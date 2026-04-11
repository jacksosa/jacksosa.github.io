---
title: "Spring Boot | Deep Dive into Apache Kafka Consumers"
date: 2026-05-09
tags: [Java, Spring Boot, Apache Kafka, Event-Driven Architecture]
style: fill
color: info
description: >-
  A thorough guide to robust Kafka consumers in Spring Boot — covering @KafkaListener, consumer groups, manual commit, error handling, dead letter topics, and idempotent processing patterns.
---

Kafka consumer configuration has more knobs than most developers realise, and the defaults are not production-ready. I've worked on event-driven systems at DWP Digital and Mosaic Smart Data where getting the consumer configuration wrong meant duplicate payments, lost events, or cascading failures. The Spring Kafka library does a good job of hiding complexity — but only up to the point where you actually need to understand what's happening underneath.

## The Basic @KafkaListener

Spring's `@KafkaListener` annotation gets you a working consumer in a few lines:

```java
@Component
public class BenefitClaimConsumer {

    @KafkaListener(
        topics = "benefit.claims",
        groupId = "claims-processor",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consume(
            @Payload BenefitClaimEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {

        log.debug("Processing claim {} from {}-{} at offset {}",
            event.getClaimId(), topic, partition, offset);

        claimService.process(event);
    }
}
```

The container factory wires together the deserialiser, concurrency, error handler, and commit strategy. Most problems originate in the container factory configuration — so that's where I spend most time.

## Consumer Groups and Partition Assignment

Every consumer belongs to a group. Kafka distributes partitions across group members, so if your topic has 12 partitions and you have 3 consumer instances, each instance handles 4 partitions. Add a fourth instance and Kafka triggers a rebalance.

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, BenefitClaimEvent>
        kafkaListenerContainerFactory() {

    ConcurrentKafkaListenerContainerFactory<String, BenefitClaimEvent> factory =
        new ConcurrentKafkaListenerContainerFactory<>();

    factory.setConsumerFactory(consumerFactory());
    factory.setConcurrency(3); // 3 consumer threads per instance
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

    return factory;
}
```

Set `concurrency` to match or be less than your partition count — a consumer thread with no assigned partition sits idle.

## Manual vs Auto Commit

Auto-commit (`enable.auto.commit=true`) is dangerous for most business use cases. Kafka auto-commits the offset at a fixed interval, regardless of whether your application actually processed the message successfully. If the application crashes between commit and processing, the message is lost.

Switch to manual commit:

```java
@KafkaListener(topics = "benefit.claims", groupId = "claims-processor")
public void consume(BenefitClaimEvent event, Acknowledgment ack) {
    try {
        claimService.process(event);
        ack.acknowledge(); // commit only after successful processing
    } catch (RecoverableException e) {
        // Do NOT acknowledge — message will be redelivered after rebalance
        log.warn("Recoverable error processing claim {}, will retry", event.getClaimId(), e);
        throw e;
    }
}
```

With `AckMode.MANUAL_IMMEDIATE`, the offset is committed immediately when you call `ack.acknowledge()`. This gives you exactly-once processing semantics when combined with idempotent business logic.

## Error Handling with @RetryableTopic

Spring Kafka 2.7+ introduced `@RetryableTopic`, which implements non-blocking retry using retry topics. Instead of blocking the consumer thread with `Thread.sleep()` retries (which stops all messages on that partition), failed messages are routed to a retry topic with a delay:

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 10_000),
    topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_DELAY_VALUE,
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
@KafkaListener(topics = "benefit.claims")
public void consume(BenefitClaimEvent event) {
    claimService.process(event); // throws on failure, triggers retry routing
}

@DltHandler
public void handleDlt(BenefitClaimEvent event,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    log.error("Message exhausted retries, routing to DLT: {}", event.getClaimId());
    deadLetterService.record(event, topic);
}
```

Spring creates `benefit.claims-retry-1000`, `benefit.claims-retry-2000`, `benefit.claims-retry-4000` topics automatically and routes the message through them with the specified delays. The DLT (`benefit.claims-dlt`) receives messages that exhaust all retries.

## Idempotent Processing

Retries introduce the possibility of processing a message more than once. Your consumer must be idempotent — processing the same message twice should produce the same result as processing it once.

The standard approach is to track processed message IDs:

```java
@Service
public class IdempotentClaimProcessor {

    private final ClaimRepository claimRepository;
    private final ProcessedEventRepository processedEvents;

    @Transactional
    public void process(BenefitClaimEvent event) {
        String eventId = event.getEventId(); // unique ID per message

        if (processedEvents.existsById(eventId)) {
            log.info("Duplicate event {}, skipping", eventId);
            return;
        }

        // Process the claim
        claimRepository.save(Claim.from(event));

        // Record as processed within the same transaction
        processedEvents.save(new ProcessedEvent(eventId, Instant.now()));
    }
}
```

The deduplication check and the business operation must be in the same transaction. If they're not, a crash between the two leaves you in an inconsistent state.

## Ordering Guarantees

Kafka guarantees ordering within a partition, not across partitions. If order matters (e.g. you must process `CLAIM_CREATED` before `CLAIM_UPDATED` for the same claim), ensure all messages for a given key land on the same partition by setting the message key:

```java
kafkaTemplate.send("benefit.claims", event.getClaimId(), event);
```

Kafka uses consistent hashing on the key to assign partitions, so the same claim ID always lands on the same partition — and the consumer for that partition sees messages in order.

## ProTips

- **Set `max.poll.records` deliberately.** The default is 500. If each record takes 100ms to process, 500 records means 50 seconds between polls — you'll breach the `max.poll.interval.ms` timeout and trigger a rebalance. Size it to match your processing time.
- **Watch out for poison pills.** A malformed message that always throws will block the partition forever if you acknowledge on failure. Route to a DLT rather than retrying indefinitely.
- **Monitor consumer lag.** `records-lag-max` in Actuator/Micrometer tells you how far behind your consumers are. Lag growing means throughput is insufficient — time to increase concurrency or scale instances.
- **Test with Testcontainers Kafka.** An embedded Kafka broker for unit tests is fine, but integration tests should use Testcontainers with a real Kafka image to catch configuration issues before production.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
