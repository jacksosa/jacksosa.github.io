---
title: "Java 21 | Virtual Threads and Project Loom — A Practical Guide"
tags: [Java, Java 21, Virtual Threads, Project Loom, Concurrency]
style: fill
color: success
description: Deep dive into Java 21 virtual threads — how they work, how to enable them in Spring Boot 3, where they shine for IO-bound workloads, and where they won't help you.
---

Java's threading model has been a source of friction for two decades. Platform threads are expensive — each one maps 1:1 to an OS thread, consuming 1MB of stack space by default. Scaling a service to handle 10,000 concurrent requests meant thread pools, reactive programming, and complexity that the business logic didn't deserve. Project Loom, delivered in Java 21 as a stable feature, changes the calculus entirely. I've migrated production services from reactive pipelines to virtual threads, and the code clarity improvement is significant.

## Platform Threads vs Virtual Threads

A platform thread is a thin wrapper over an OS thread. When it blocks on IO, the OS thread is occupied — the JVM cannot use it for anything else. Your thread pool of 200 threads means a maximum of 200 concurrent blocking operations, regardless of how much CPU headroom you have.

A virtual thread is scheduled by the JVM, not the OS. When a virtual thread blocks on IO, the JVM unmounts it from its carrier (an OS thread) and parks it. The carrier thread is immediately free to run another virtual thread. Millions of virtual threads can exist simultaneously — each one is cheap to create and park.

```java
// Platform thread — heavyweight, ~1MB stack
Thread platformThread = new Thread(() -> {
    // blocks OS thread on IO
    String data = callExternalService();
});

// Virtual thread — lightweight, starts at ~1KB
Thread virtualThread = Thread.ofVirtual().start(() -> {
    // carrier thread is freed while this blocks
    String data = callExternalService();
});

// Or via executor
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> processMarketData(marketId));
}
```

The key insight: virtual threads are cheap enough that you can create one per task rather than pooling them.

## Enabling Virtual Threads in Spring Boot 3

In Spring Boot 3.2+, enabling virtual threads for the embedded Tomcat/Undertow container takes a single property:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

That's it. Spring will configure Tomcat to use a virtual-thread-per-request executor. Every HTTP request gets its own virtual thread, meaning thousands of simultaneous requests are no longer a concern from a threading perspective.

For `@Async` tasks, replace the thread pool executor:

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}
```

For scheduled tasks, Spring Boot 3.2 also picks up virtual threads for `@Scheduled` methods when the property is set.

## Where Virtual Threads Shine — IO-Bound Work

Virtual threads are transformative for IO-bound workloads. Consider a service that enriches each incoming event with data from three downstream APIs. With platform threads, you'd reach for `CompletableFuture.allOf()` or reactive chains. With virtual threads, you can write blocking code that the JVM parallelises automatically:

```java
@Service
public class MarketEnrichmentService {

    public EnrichedMarket enrich(String marketId) {
        // These run concurrently via structured concurrency
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            var runnerTask   = scope.fork(() -> runnerService.getRunners(marketId));
            var priceTask    = scope.fork(() -> priceService.getPrices(marketId));
            var formTask     = scope.fork(() -> formService.getForm(marketId));

            scope.join().throwIfFailed();

            return new EnrichedMarket(
                runnerTask.get(),
                priceTask.get(),
                formTask.get()
            );
        }
    }
}
```

`StructuredTaskScope` (also a Java 21 preview, finalised in 21) gives you fork/join semantics with automatic cancellation on failure. The three IO calls run concurrently, but the code reads top-to-bottom. No callbacks, no reactive operators, no mental overhead.

## Where Virtual Threads Won't Help — CPU-Bound Work

Virtual threads are not magic for CPU-bound work. If your thread is crunching numbers, it holds the carrier thread for the duration — unmounting only happens at blocking points. A calculation that takes 200ms of CPU time will occupy a carrier thread for 200ms regardless of whether it runs on a virtual or platform thread.

In my Betfair trading framework, signal calculation (WoM, LTP velocity, OFI) is CPU-bound. Virtual threads gave no improvement there. Platform threads and a sized thread pool remain the right choice for CPU-intensive processing.

Also be aware of **pinning**: if a virtual thread calls a `synchronized` block, it pins to its carrier thread. This was a significant limitation in early Project Loom previews. In Java 21, most JDK IO operations have been updated to avoid pinning — but if you use third-party libraries with heavy `synchronized` usage, check for pinning events via JFR:

```bash
-Djdk.tracePinnedThreads=full
```

## Observability

Virtual threads appear in thread dumps, but you'll see millions of them. Use JFR to analyse virtual thread behaviour:

```java
// Programmatic JFR recording
Recording recording = new Recording();
recording.enable("jdk.VirtualThreadStart").withStackTrace();
recording.enable("jdk.VirtualThreadPinned");
recording.start();
```

Spring Boot Actuator and Micrometer work as normal with virtual threads — the executor metrics report task throughput rather than thread pool sizes, which is actually more useful.

## ProTips

- **Don't pool virtual threads.** The entire point is to create one per task. `Executors.newFixedThreadPool(200)` of virtual threads defeats the purpose — and may actually perform worse.
- **Remove reactive libraries incrementally.** If you're on WebFlux, there's no obligation to migrate immediately. Virtual threads are a compelling reason to consider Spring MVC + virtual threads, but only migrate when the simplification is worth the cost.
- **Check for `synchronized` pinning in your dependencies.** JDBC drivers, older JPA providers, and legacy Netty versions are common offenders. Monitor `jdk.VirtualThreadPinned` events before going to production.
- **Structured concurrency is the right pairing.** Use `StructuredTaskScope` rather than `CompletableFuture.allOf()` for concurrent IO — it's cleaner, safer, and integrates naturally with virtual threads.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
