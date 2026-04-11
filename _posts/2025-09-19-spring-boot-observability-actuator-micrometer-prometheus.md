---
title: "Spring Boot | Production Observability with Actuator, Micrometer, and Prometheus"
tags: [Java, Spring Boot, Observability, Prometheus, Micrometer, Actuator]
style: fill
color: success
description: Building production-grade observability into Spring Boot — Actuator health endpoints, custom health indicators, Micrometer metrics, Prometheus export, and instrumenting business-level metrics.
---

Observability is the difference between knowing your system is behaving correctly and hoping it is. I've been on teams that discovered a critical processing failure from a customer complaint rather than from a dashboard — and I've been on teams where we caught and resolved a Kafka consumer lag spike before a single business transaction was affected. The difference was instrumentation. Spring Boot Actuator, Micrometer, and Prometheus give you the building blocks; the craft is knowing what to measure and how to make the dashboards actionable.

## Spring Boot Actuator — Health Endpoints

Actuator exposes operational endpoints over HTTP. The most important for production is `/actuator/health`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, loggers
  endpoint:
    health:
      show-details: when-authorized  # don't expose internals publicly
      show-components: when-authorized
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

`/actuator/health/liveness` tells Kubernetes whether the process is alive (should it restart?). `/actuator/health/readiness` tells it whether the service is ready to receive traffic.

Custom health indicators let you express the health of dependencies:

```java
@Component("betfairStream")
public class BetfairStreamHealthIndicator implements HealthIndicator {

    private final BetfairStreamClient streamClient;

    @Override
    public Health health() {
        if (!streamClient.isConnected()) {
            return Health.down()
                .withDetail("reason", "Streaming connection down")
                .withDetail("lastConnected", streamClient.getLastConnectedAt())
                .build();
        }

        Duration lastHeartbeat = streamClient.timeSinceLastHeartbeat();
        if (lastHeartbeat.toSeconds() > 30) {
            return Health.down()
                .withDetail("reason", "No heartbeat for " + lastHeartbeat.toSeconds() + "s")
                .build();
        }

        return Health.up()
            .withDetail("subscriptions", streamClient.getSubscriptionCount())
            .withDetail("lastHeartbeat", lastHeartbeat.toMillis() + "ms ago")
            .build();
    }
}
```

## Micrometer — The Metrics API

Micrometer is Spring Boot's metrics abstraction. You write metrics once against the Micrometer API; the backend (Prometheus, Datadog, CloudWatch, etc.) is a dependency you swap.

The four core types:

```java
@Service
public class MarketMetrics {

    private final Counter betsPlaced;
    private final Counter betsFailed;
    private final Timer betPlacementLatency;
    private final Gauge openPositions;
    private final DistributionSummary stakeDistribution;

    public MarketMetrics(MeterRegistry registry, PositionTracker tracker) {
        this.betsPlaced = Counter.builder("trading.bets.placed")
            .description("Total bets successfully placed")
            .tag("exchange", "betfair")
            .register(registry);

        this.betsFailed = Counter.builder("trading.bets.failed")
            .description("Total bet placement failures")
            .tag("exchange", "betfair")
            .register(registry);

        this.betPlacementLatency = Timer.builder("trading.bet.placement.latency")
            .description("Time from order decision to API response")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        this.openPositions = Gauge.builder("trading.positions.open", tracker,
                PositionTracker::getOpenPositionCount)
            .description("Number of currently open positions")
            .register(registry);

        this.stakeDistribution = DistributionSummary.builder("trading.stake.distribution")
            .description("Distribution of bet stakes in pence")
            .baseUnit("pence")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
}
```

Use tags to slice metrics by meaningful dimensions:

```java
// Record bet placement with outcome tags
Timer.Sample sample = Timer.start();
try {
    PlaceExecutionReport report = betfairClient.placeOrders(ssoid, request);
    betsPlaced.increment();
    stakeDistribution.record(request.getTotalStakePence());
    sample.stop(betPlacementLatency.tag("result", "success").register(registry));
} catch (Exception e) {
    betsFailed.increment();
    sample.stop(betPlacementLatency.tag("result", "failure").register(registry));
}
```

## Exporting to Prometheus

Add the Prometheus dependency:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Expose the scrape endpoint:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

Configure Prometheus to scrape:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'trading-service'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets: ['trading-service:8080']
```

## Business-Level Metrics in Grafana

Technical metrics (JVM heap, HTTP request rate, database pool size) are valuable but not sufficient. The dashboards I find most useful in production combine technical metrics with business metrics:

```java
// Kafka consumer lag — technical + business signal
Gauge.builder("kafka.consumer.lag", consumerLag, ConsumerLagProvider::getTotalLag)
    .tags("topic", "claim-events", "consumer-group", "claims-processor")
    .register(registry);

// Daily P&L — business metric
Gauge.builder("trading.daily.pnl.pence", pnlTracker, PnlTracker::getDailyPnlPence)
    .register(registry);

// Kill switch state — operational metric
Gauge.builder("trading.kill.switch.active",
        riskController, rc -> rc.isKillSwitchActive() ? 1.0 : 0.0)
    .register(registry);
```

In Grafana, set alerts on:
- `trading.kill.switch.active == 1` → page immediately
- `kafka.consumer.lag > 1000` for a sustained 5 minutes → investigate
- `trading.bets.failed` rate > 5% → investigate
- `jvm.memory.used / jvm.memory.max > 0.85` → memory pressure

## Distributed Tracing

For request flows that span multiple services, add distributed tracing:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.1  # sample 10% of requests
```

Traces propagate automatically across HTTP calls and Kafka messages in Spring Boot 3.x. The `traceId` appears in structured log output, linking logs to the trace in Zipkin or Grafana Tempo.

## ProTips

- **Instrument the things that change your decision about the system.** Metrics exist to inform action. Before adding a metric, ask: "what would I do differently if this number changed?" If the answer is nothing, skip the metric.
- **Use `@Timed` for quick wins.** Spring's `@Timed` annotation on `@RequestMapping` methods and `@KafkaListener` handlers instruments latency automatically without boilerplate.
- **Keep Actuator endpoints off the public port.** Use `management.server.port=8090` to expose Actuator on a separate port, accessible only within your VPC. Don't expose internal health details to the internet.
- **Add runbook links to your Grafana alerts.** When an alert fires, the responder should immediately know what to check and what actions are available. A link to a runbook in the alert annotation makes this automatic.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
