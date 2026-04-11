---
title: "Java | Event Sourcing with Spring Boot and MongoDB"
tags: [Java, Spring Boot, Event Sourcing, MongoDB, Architecture]
style: fill
color: info
description: Implementing event sourcing in Java Spring Boot with MongoDB as the event store — event appending, aggregate reconstruction, snapshots, and publishing events to Kafka for downstream projections.
---

Event sourcing stores the history of state changes, not the current state. Instead of updating a row to record that a claim was approved, you append an immutable `ClaimApproved` event to the event log. The current state is derived by replaying the events. This sounds like extra work — and it is — but the benefits for audit-heavy systems, complex temporal queries, and event-driven integration are substantial. At DWP Digital, where every state change to a benefit claim had legal and audit significance, event sourcing was the right architecture.

## The Event Store

The event store is append-only. Events are never updated or deleted. In MongoDB, each document in the event store represents a single domain event:

```java
@Document(collection = "event_store")
public class StoredEvent {

    @Id
    private String id;

    @Indexed
    private String aggregateId;     // which aggregate this event belongs to

    private String aggregateType;   // "BenefitClaim", "Claimant", etc.

    @Indexed
    private long sequenceNumber;    // version within this aggregate

    private String eventType;       // "ClaimApproved", "ClaimSubmitted", etc.
    private String eventData;       // JSON-serialised event payload

    private Instant occurredAt;
    private String causedBy;        // command that triggered this event
}
```

The `(aggregateId, sequenceNumber)` compound index is critical — you'll query by aggregate ID ordered by sequence to reconstruct state:

```java
@Component
public class MongoEventStore {

    private final MongoTemplate mongoTemplate;
    private final ObjectMapper objectMapper;

    public void append(String aggregateId, String aggregateType,
                       long expectedVersion, List<DomainEvent> events) {
        long nextSeq = expectedVersion + 1;
        List<StoredEvent> stored = new ArrayList<>();

        for (DomainEvent event : events) {
            StoredEvent se = new StoredEvent();
            se.setId(UUID.randomUUID().toString());
            se.setAggregateId(aggregateId);
            se.setAggregateType(aggregateType);
            se.setSequenceNumber(nextSeq++);
            se.setEventType(event.getClass().getSimpleName());
            se.setEventData(serialise(event));
            se.setOccurredAt(event.occurredAt());
            stored.add(se);
        }

        // Optimistic concurrency: fail if another process has already written at this version
        long count = mongoTemplate.count(
            new Query(Criteria.where("aggregateId").is(aggregateId)
                .and("sequenceNumber").gte(expectedVersion + 1)),
            StoredEvent.class
        );

        if (count > 0) {
            throw new OptimisticConcurrencyException(
                "Conflict: aggregate " + aggregateId + " has been modified");
        }

        mongoTemplate.insertAll(stored);
    }

    public List<StoredEvent> load(String aggregateId) {
        Query query = new Query(Criteria.where("aggregateId").is(aggregateId))
            .with(Sort.by(Sort.Order.asc("sequenceNumber")));
        return mongoTemplate.find(query, StoredEvent.class);
    }
}
```

The optimistic concurrency check prevents two processes from appending events at the same sequence number. For true atomic guarantees under concurrent load, consider using MongoDB transactions on the `append` + version check.

## Reconstructing Aggregate State

Replaying events to rebuild current state:

```java
public class BenefitClaim {

    private String id;
    private ClaimStatus status;
    private String claimantId;
    private List<Assessment> assessments = new ArrayList<>();
    private long version = 0;

    // Private constructor — only reconstituted from events
    private BenefitClaim() {}

    public static BenefitClaim reconstitute(List<StoredEvent> events, EventDeserialiser deserialiser) {
        BenefitClaim claim = new BenefitClaim();
        for (StoredEvent se : events) {
            DomainEvent event = deserialiser.deserialise(se);
            claim.apply(event);
            claim.version = se.getSequenceNumber();
        }
        return claim;
    }

    private void apply(DomainEvent event) {
        switch (event) {
            case ClaimSubmittedEvent e -> {
                this.id = e.claimId();
                this.status = ClaimStatus.PENDING;
                this.claimantId = e.claimantId();
            }
            case ClaimApprovedEvent e -> this.status = ClaimStatus.APPROVED;
            case ClaimRejectedEvent e -> this.status = ClaimStatus.REJECTED;
            case AssessmentAddedEvent e -> this.assessments.add(new Assessment(e));
            default -> log.debug("Unhandled event type in reconstitute: {}", event.getClass());
        }
    }
}
```

The `apply` method is pure — no side effects, no external calls. This makes it fast and testable. Loading a claim with 200 events takes milliseconds.

## Snapshots for Performance

Replaying hundreds or thousands of events on every load becomes slow for long-lived aggregates. Snapshots solve this: periodically persist a snapshot of the aggregate state and, on next load, replay only events after the snapshot:

```java
@Document(collection = "aggregate_snapshots")
public class AggregateSnapshot {
    @Id private String aggregateId;
    private long snapshotVersion;
    private String aggregateType;
    private String stateJson;
    private Instant createdAt;
}

@Component
public class SnapshotStrategy {

    private static final int SNAPSHOT_EVERY = 50; // snapshot every 50 events

    private final MongoTemplate mongoTemplate;

    public void maybeSnapshot(BenefitClaim claim) {
        if (claim.getVersion() % SNAPSHOT_EVERY == 0) {
            AggregateSnapshot snapshot = new AggregateSnapshot();
            snapshot.setAggregateId(claim.getId());
            snapshot.setSnapshotVersion(claim.getVersion());
            snapshot.setStateJson(serialise(claim));
            mongoTemplate.save(snapshot);
        }
    }

    public BenefitClaim loadFromSnapshot(String claimId, List<StoredEvent> allEvents) {
        Optional<AggregateSnapshot> snapshot = findLatestSnapshot(claimId);

        if (snapshot.isEmpty()) {
            return BenefitClaim.reconstitute(allEvents, deserialiser);
        }

        BenefitClaim claim = deserialise(snapshot.get().getStateJson());
        List<StoredEvent> remaining = allEvents.stream()
            .filter(e -> e.getSequenceNumber() > snapshot.get().getSnapshotVersion())
            .toList();

        remaining.forEach(e -> claim.apply(deserialiser.deserialise(e)));
        return claim;
    }
}
```

## Publishing Events to Kafka

After persisting events, publish them for downstream consumers:

```java
@Service
@Transactional
public class ClaimCommandService {

    private final MongoEventStore eventStore;
    private final KafkaTemplate<String, DomainEvent> kafka;

    public void approveClaim(ApproveClaim command) {
        List<StoredEvent> history = eventStore.load(command.claimId());
        BenefitClaim claim = BenefitClaim.reconstitute(history, deserialiser);

        List<DomainEvent> newEvents = claim.approve(command.assessorId(), command.reason());
        eventStore.append(command.claimId(), "BenefitClaim", claim.getVersion(), newEvents);

        // Publish after successful persist
        newEvents.forEach(event ->
            kafka.send("claim-events", command.claimId(), event));
    }
}
```

For stronger guarantees between event store and Kafka, apply the outbox pattern: write events to an `outbox` collection in the same MongoDB write, then publish from there via a separate process (like Debezium CDC). This prevents publishing events that didn't actually persist.

## ProTips

- **Event schema evolution is real.** Version your events (`ClaimApproved_v1`, `ClaimApproved_v2`) and maintain upcasters that translate old schema to new. Old events are never rewritten — the translator runs at read time.
- **Keep the `apply` method free of side effects.** Reconstitution should be pure and repeatable. If `apply` sends emails, makes API calls, or logs audit trails, those run on every replay — which will be wrong.
- **Think about event granularity.** One `ClaimFullyUpdated` event with every changed field is less useful than separate `ClaimStatusChanged`, `AssessmentAdded` events. Finer-grained events make projections more maintainable.
- **Event sourcing + CQRS is a natural pairing.** The event store is the write side; Kafka projectors populate read-side stores. Together they give you complete audit history, temporal queries, and fast read access.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
