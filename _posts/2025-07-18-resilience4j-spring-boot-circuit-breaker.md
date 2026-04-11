---
title: "Resilience4j | Circuit Breakers, Rate Limiters, and Bulkheads in Spring Boot"
date: 2026-07-18
tags: [Java, Spring Boot, Resilience4j, Circuit Breaker, Resilience]
style: fill
color: warning
description: >-
  Practical guide to Resilience4j in Spring Boot — configuring @CircuitBreaker, @RateLimiter, @Bulkhead, and @Retry annotations, and testing resilience patterns in integration tests.
---

Distributed systems fail. The question isn't whether a downstream service will become unavailable — it's whether your system degrades gracefully or takes everything down with it. At Mosaic Smart Data, where our real-time financial data API depended on feeds from multiple third-party providers, a poorly isolated failure in one provider would cascade and degrade unrelated client queries. Resilience4j, combined with Spring Boot's annotation-driven configuration, gave us the isolation and recovery behaviour we needed with a fraction of the code Hystrix required.

## Adding Resilience4j to Spring Boot

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

The AOP dependency is required — Resilience4j's annotations are implemented as aspects.

## Circuit Breaker

The circuit breaker has three states: **Closed** (normal operation), **Open** (failing — calls short-circuit immediately), and **Half-Open** (testing recovery). When the failure rate in the sliding window exceeds the configured threshold, the circuit opens:

```java
@Service
public class MarketDataClient {

    @CircuitBreaker(name = "market-data", fallbackMethod = "fallbackMarketData")
    public MarketData fetchMarket(String marketId) {
        return externalProvider.fetch(marketId); // may throw
    }

    private MarketData fallbackMarketData(String marketId, Exception ex) {
        log.warn("Circuit breaker open for market-data, returning cached data for {}", marketId);
        return cache.get(marketId).orElse(MarketData.empty());
    }
}
```

Configuration in `application.yml`:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      market-data:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20
        failure-rate-threshold: 50          # open when 50% of 20 calls fail
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
        slow-call-rate-threshold: 80        # also open on slow calls
        slow-call-duration-threshold: 2s
        register-health-indicator: true
```

The fallback method must have the same return type and the same parameters as the original method, plus an additional `Exception` parameter as the last argument. Spring's AOP wiring handles the routing.

## Rate Limiter

Rate limiters prevent your service from overwhelming a downstream dependency:

```java
@RateLimiter(name = "betfair-api", fallbackMethod = "rateLimitedFallback")
public PlaceExecutionReport placeOrder(PlaceOrdersRequest request) {
    return betfairClient.placeOrders(request);
}

private PlaceExecutionReport rateLimitedFallback(
        PlaceOrdersRequest request, RequestNotPermitted ex) {
    throw new TradingException("API rate limit reached — order rejected", ex);
}
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      betfair-api:
        limit-for-period: 10          # 10 calls
        limit-refresh-period: 1s      # per second
        timeout-duration: 100ms       # wait up to 100ms for a permit
```

Unlike circuit breakers (which respond to failures), rate limiters enforce throughput limits regardless of success or failure.

## Bulkhead — Isolating Thread Pools

A bulkhead isolates concurrent calls to a dependency, preventing resource starvation. The `ThreadPoolBulkhead` runs calls in a dedicated thread pool:

```java
@Bulkhead(name = "price-data", type = Bulkhead.Type.THREADPOOL,
          fallbackMethod = "priceDataFallback")
@Async
public CompletableFuture<PriceData> fetchPrices(String marketId) {
    return CompletableFuture.completedFuture(priceProvider.fetch(marketId));
}

private CompletableFuture<PriceData> priceDataFallback(
        String marketId, BulkheadFullException ex) {
    return CompletableFuture.completedFuture(PriceData.empty());
}
```

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      price-data:
        max-thread-pool-size: 10
        core-thread-pool-size: 5
        queue-capacity: 20
```

If the thread pool is full and the queue is at capacity, calls go to the fallback immediately rather than blocking the calling thread. This is the key isolation property — a slow or failing price data feed doesn't exhaust your shared thread pool.

## Retry

`@Retry` retries on failure with configurable backoff:

```java
@Retry(name = "external-validation", fallbackMethod = "validationFallback")
@CircuitBreaker(name = "external-validation")
public ValidationResult validate(ClaimData claim) {
    return validationService.validate(claim);
}
```

```yaml
resilience4j:
  retry:
    instances:
      external-validation:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
        ignore-exceptions:
          - com.example.ValidationException  # don't retry business validation failures
```

Combine `@Retry` with `@CircuitBreaker` — the retry sits inside the circuit breaker. Three failed retries count as one failure against the circuit breaker's sliding window.

## Monitoring via Actuator and Micrometer

Resilience4j integrates with Micrometer automatically when `register-health-indicator: true` is set:

```
GET /actuator/health
{
  "status": "UP",
  "components": {
    "circuitBreakers": {
      "status": "UP",
      "details": {
        "market-data": {
          "status": "UP",
          "state": "CLOSED",
          "failureRate": "5.0%",
          "slowCallRate": "0.0%"
        }
      }
    }
  }
}
```

Micrometer exposes circuit breaker state, failure rate, call duration, and buffered call counts as metrics. Wire them into Grafana dashboards to alert when circuits open.

## Testing Circuit Breakers

Testing resilience patterns requires controlling failure injection:

```java
@SpringBootTest
class MarketDataCircuitBreakerTest {

    @Autowired
    private CircuitBreakerRegistry registry;

    @MockBean
    private ExternalProvider externalProvider;

    @Test
    void shouldOpenCircuitAfterThresholdFailures() {
        when(externalProvider.fetch(any())).thenThrow(new RuntimeException("Provider down"));

        // Trigger 10 failures (50% of 20 = threshold)
        IntStream.range(0, 10).forEach(i -> {
            try { marketDataClient.fetchMarket("1.12345"); }
            catch (Exception ignored) {}
        });

        CircuitBreaker cb = registry.circuitBreaker("market-data");
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);
    }
}
```

## ProTips

- **Name instances after the downstream dependency, not the calling class.** `betfair-api`, `external-validation`, `price-data` are more useful than `MarketDataService` when diagnosing open circuits in production.
- **Test your fallbacks explicitly.** A fallback that throws `NullPointerException` is worse than no fallback. Write tests that verify the fallback path returns sensible data.
- **Combine patterns thoughtfully.** Rate limiter wrapping a circuit breaker wrapping a retry is a reasonable pattern for a critical external API call. But stacking all four patterns on every method is complexity for its own sake.
- **Alert on circuit opens, not individual failures.** Individual failures are noise. A circuit opening means a meaningful failure rate — that deserves a page.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
