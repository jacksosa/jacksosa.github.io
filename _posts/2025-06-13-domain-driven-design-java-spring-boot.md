---
title: "Architecture | Domain-Driven Design in Java Spring Boot"
tags: [Java, Spring Boot, DDD, Architecture, Clean Code]
style: fill
color: success
description: A practical DDD guide for Java Spring Boot developers — covering aggregates, value objects, domain events, bounded contexts, and how to keep JPA out of your domain model.
---

Most Spring Boot projects I've walked into have the same structural problem: JPA annotations scattered across domain objects, business logic living in services that know too much about persistence, and a codebase that makes it hard to answer the question "what does this system actually do?". Domain-Driven Design isn't a silver bullet — it adds real complexity — but on systems with genuine domain complexity (like the DWP Digital benefit platform or a Betfair trading strategy engine), it produces code that's far easier to evolve and reason about.

## Aggregates and Aggregate Roots

An aggregate is a cluster of domain objects treated as a single unit for the purposes of data changes. Every aggregate has an aggregate root — the only entry point for external interaction. The classic rule: reference other aggregates by ID, not by object reference.

```java
public class BenefitClaim {  // aggregate root

    private final ClaimId id;
    private ClaimStatus status;
    private final ClaimantId claimantId;  // reference to Claimant aggregate by ID
    private final List<Assessment> assessments = new ArrayList<>();
    private final List<DomainEvent> events = new ArrayList<>();

    public static BenefitClaim submit(ClaimantId claimantId, ClaimType type) {
        BenefitClaim claim = new BenefitClaim(ClaimId.generate(), claimantId, type);
        claim.events.add(new ClaimSubmitted(claim.id, claimantId, type, Instant.now()));
        return claim;
    }

    public void approve(AssessorId assessorId, String reason) {
        if (this.status != ClaimStatus.PENDING) {
            throw new IllegalStateException("Only PENDING claims can be approved");
        }
        this.status = ClaimStatus.APPROVED;
        this.events.add(new ClaimApproved(this.id, assessorId, reason, Instant.now()));
    }

    public List<DomainEvent> pullEvents() {
        List<DomainEvent> pending = List.copyOf(events);
        events.clear();
        return pending;
    }
}
```

Business rules live on the aggregate. `approve()` enforces the state machine. The service layer calls `claim.approve()` and handles persistence and event publication — it doesn't contain the rule itself.

## Value Objects

A value object has no identity — it's defined entirely by its attributes. Coordinates, money amounts, email addresses: these should be value objects rather than primitives.

```java
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Money amount cannot be negative");
        }
        amount = amount.setScale(2, RoundingMode.HALF_UP);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

Using Java records for value objects is clean and idiomatic in Java 16+. The `compact constructor` provides validation. Equality is structural by default.

## Domain Events

Domain events record what happened, and are the mechanism for integration between bounded contexts:

```java
public sealed interface DomainEvent permits ClaimSubmitted, ClaimApproved, ClaimRejected {
    ClaimId claimId();
    Instant occurredAt();
}

public record ClaimApproved(
    ClaimId claimId,
    AssessorId assessorId,
    String reason,
    Instant occurredAt
) implements DomainEvent {}
```

The application service publishes events after saving the aggregate:

```java
@Service
@Transactional
public class ClaimApplicationService {

    private final ClaimRepository repository;
    private final ApplicationEventPublisher eventPublisher;

    public void approveClaim(ClaimId claimId, AssessorId assessorId, String reason) {
        BenefitClaim claim = repository.findById(claimId)
            .orElseThrow(() -> new ClaimNotFoundException(claimId));

        claim.approve(assessorId, reason);

        repository.save(claim);

        // Publish domain events after commit
        claim.pullEvents().forEach(eventPublisher::publishEvent);
    }
}
```

Other services can listen to `ClaimApproved` events via `@EventListener` (same JVM) or via Kafka for cross-service integration.

## Keeping JPA Out of the Domain Model

The most important structural decision in DDD with Spring Boot: don't put `@Entity`, `@Column`, or `@OneToMany` on your domain objects. JPA concerns (lazy loading, proxy objects, persistence state) are infrastructure concerns. Mix them into your domain model and you couple your business logic to your ORM.

Instead, create separate persistence models:

```java
// Domain object — pure Java, no JPA annotations
public class BenefitClaim {
    private final ClaimId id;
    private ClaimStatus status;
    // ...
}

// JPA entity — infrastructure layer
@Entity
@Table(name = "benefit_claims")
class ClaimJpaEntity {
    @Id
    private String id;
    @Enumerated(EnumType.STRING)
    private ClaimStatus status;
    // ...
}

// Repository implementation — maps between the two
@Repository
class JpaClaimRepository implements ClaimRepository {

    private final ClaimJpaRepository jpa;

    @Override
    public Optional<BenefitClaim> findById(ClaimId id) {
        return jpa.findById(id.value())
            .map(this::toDomain);
    }

    @Override
    public void save(BenefitClaim claim) {
        jpa.save(toEntity(claim));
    }

    private BenefitClaim toDomain(ClaimJpaEntity entity) {
        return BenefitClaim.reconstitute(
            new ClaimId(entity.getId()),
            entity.getStatus()
            // ...
        );
    }
}
```

Yes, this is more code. The payoff is a domain model that's testable without a Spring context, portable to a different persistence technology, and readable without knowing JPA.

## Bounded Contexts

A bounded context is a boundary within which a particular domain model applies. In a large system, different teams maintain different models of the same concept. In the DWP system, "Claimant" means something different to the Identity service, the Eligibility service, and the Payment service. Don't try to create a single unified model — create three context-appropriate models and translate between them at context boundaries.

```java
// Eligibility bounded context
public class EligibleClaimant {
    private final NiNumber niNumber;
    private final int age;
    private final EmploymentStatus employmentStatus;
    // eligibility-specific domain logic
}

// Payment bounded context
public class PaymentRecipient {
    private final NiNumber niNumber;
    private final BankAccount bankAccount;
    // payment-specific domain logic
}
```

Translation between contexts happens in an anti-corruption layer — a class whose sole job is to translate between models without polluting either.

## ProTips

- **Start with the domain, not the database.** Write the domain model first, then figure out how to persist it. The reverse leads to anemic domain models where all behaviour ends up in services.
- **Keep aggregates small.** A large aggregate with 15 fields and 8 associations is difficult to load, save, and reason about. If you feel the urge to make it bigger, consider whether you have two aggregates that should reference each other by ID.
- **Use domain events for cross-aggregate coordination.** Never directly modify one aggregate from inside another aggregate's method.
- **Not every project needs full DDD.** CRUD applications with simple business logic don't benefit from aggregates and bounded contexts — they just add overhead. Apply DDD where the domain complexity justifies it.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
