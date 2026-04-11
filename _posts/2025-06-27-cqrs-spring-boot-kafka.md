---
title: "Spring Boot | CQRS Pattern with Kafka and Spring Boot"
tags: [Java, Spring Boot, CQRS, Apache Kafka, Architecture]
style: fill
color: info
description: Implementing CQRS in Spring Boot with Kafka — separating command and query models, publishing events, building read projections, and knowing when CQRS is worth the complexity.
---

CQRS — Command Query Responsibility Segregation — separates the write path (commands that change state) from the read path (queries that return data). At DWP Digital, we applied it to the benefit claims platform because the read requirements (claimant-facing status pages, caseworker dashboards, reporting) were fundamentally different from the write requirements (claim state transitions, eligibility decisions, payment instructions). One data model couldn't serve all of them well without compromising either throughput or query flexibility.

## The Core Idea

In a traditional CRUD application, you have one model used for both writing and reading. CQRS says: optimise separately. The write model enforces business invariants and records facts (events). The read model is denormalised and shaped for specific query patterns — you can have as many read models as you have distinct query needs.

With Kafka as the event bus:

1. A command arrives at the write side
2. The command handler validates, applies business logic, and saves to the command store
3. Domain events are published to Kafka
4. One or more read model projectors consume the events and update read stores
5. Queries go directly to the read stores

## The Command Side

Commands express intent to change state. They're validated and handled by command handlers:

```java
public record ApproveClaim(String claimId, String assessorId, String reason) {}

@Service
public class ClaimCommandHandler {

    private final ClaimRepository writeRepository;
    private final KafkaTemplate<String, DomainEvent> kafka;

    @Transactional
    public void handle(ApproveClaim command) {
        BenefitClaim claim = writeRepository.findById(new ClaimId(command.claimId()))
            .orElseThrow(() -> new ClaimNotFoundException(command.claimId()));

        claim.approve(new AssessorId(command.assessorId()), command.reason());

        writeRepository.save(claim);

        claim.pullEvents().forEach(event ->
            kafka.send("claim-events", claim.getId().value(), event)
        );
    }
}
```

The write-side repository uses the full domain model with business invariants. The database here might be PostgreSQL with normalised tables — optimised for consistency and transactional integrity, not query performance.

## Publishing Events to Kafka

Domain events carry enough information for all downstream projectors to build their read models without querying back:

```java
public record ClaimApprovedEvent(
    String claimId,
    String claimantId,
    String assessorId,
    String reason,
    Instant approvedAt,
    String eventId          // unique per event — idempotency key for projectors
) implements DomainEvent {}
```

Include a unique `eventId` on every event. Projectors must be idempotent — Kafka at-least-once delivery means the same event may arrive more than once.

## Building Read Model Projections

Each read model is a Kafka consumer that listens to the event stream and updates a denormalised store:

```java
@Component
public class ClaimStatusProjector {

    private final ClaimStatusViewRepository viewRepo;
    private final ProcessedEventRepository processedEvents;

    @KafkaListener(topics = "claim-events", groupId = "claim-status-projector")
    @Transactional
    public void project(DomainEvent event, Acknowledgment ack) {
        if (processedEvents.existsById(event.eventId())) {
            ack.acknowledge();
            return; // idempotency — skip duplicates
        }

        switch (event) {
            case ClaimSubmittedEvent e -> viewRepo.save(
                new ClaimStatusView(e.claimId(), "PENDING", e.submittedAt()));

            case ClaimApprovedEvent e -> viewRepo.updateStatus(
                e.claimId(), "APPROVED", e.approvedAt(), e.assessorId());

            case ClaimRejectedEvent e -> viewRepo.updateStatus(
                e.claimId(), "REJECTED", e.rejectedAt(), e.reason());

            default -> log.debug("Ignoring event type: {}", event.getClass().getSimpleName());
        }

        processedEvents.save(new ProcessedEvent(event.eventId(), Instant.now()));
        ack.acknowledge();
    }
}
```

The `ClaimStatusView` table is denormalised — everything a status page needs is in one row. No joins required. The caseworker dashboard projector might maintain a completely different table with aggregated counts per assessor. Each read model is shaped for its consumer.

## The Query Side

Queries go directly to the read store — no command-side business logic involved:

```java
@RestController
@RequestMapping("/claims")
public class ClaimQueryController {

    private final ClaimStatusViewRepository viewRepo;
    private final ClaimSearchRepository searchRepo;

    @GetMapping("/{claimId}/status")
    public ClaimStatusResponse getStatus(@PathVariable String claimId) {
        return viewRepo.findById(claimId)
            .map(view -> new ClaimStatusResponse(view.status(), view.lastUpdated()))
            .orElseThrow(() -> new ClaimNotFoundException(claimId));
    }

    @GetMapping("/search")
    public Page<ClaimSummary> search(
            @RequestParam String assessorId,
            @RequestParam ClaimStatus status,
            Pageable pageable) {
        return searchRepo.findByAssessorAndStatus(assessorId, status, pageable);
    }
}
```

The read store for status queries might be PostgreSQL. The search capability might sit in Elasticsearch if text search is required. Each is optimised for its query pattern.

## Eventual Consistency

The write side and read side are eventually consistent — after a claim is approved, there's a short window (typically milliseconds on a healthy system) where the write model has the new state but the read model doesn't. This is usually acceptable for the use cases CQRS is applied to.

Design your UI accordingly. After submitting a command, don't immediately query the read model and expect the latest state. Either poll with a short delay, use optimistic updates in the UI, or return the new state directly from the command response for display purposes.

## When CQRS Is and Isn't Worth It

CQRS adds real complexity: two models to maintain, event publishing infrastructure, projector management, eventual consistency to handle. The ROI is only positive when:

- Read and write load characteristics are significantly different
- You have multiple distinct read patterns that can't be served by one model
- The domain model enforces complex invariants that would be corrupted by read-optimisation

For a simple CRUD admin panel, CQRS is pure overhead. For a high-throughput platform where event sourcing and multiple downstream consumers are requirements, CQRS is the right architecture.

## ProTips

- **Version your events.** `ClaimApprovedEvent_v1` → `ClaimApprovedEvent_v2`. When you need to add fields, create a new version and handle both in projectors during migration.
- **Monitor projector lag.** If projectors fall behind, your read models become stale. Alert on Kafka consumer lag > threshold and have a replay strategy ready.
- **Make projectors rebuildable.** Projectors should be able to rebuild from scratch by replaying the event log from the beginning. Design read stores to be droppable and repopulated.
- **Don't put business logic in projectors.** A projector's job is to translate events into read model state — nothing more. Business logic belongs on the command side.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
