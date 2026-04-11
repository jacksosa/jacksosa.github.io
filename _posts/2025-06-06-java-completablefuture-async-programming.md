---
title: "Java | CompletableFuture — Mastering Async Programming"
date: 2026-06-06
tags: [Java, CompletableFuture, Async, Concurrency, Spring Boot]
style: fill
color: warning
description: >-
  A comprehensive guide to CompletableFuture in Java — covering thenApply, thenCompose, allOf, exception handling, custom executors, and the pitfalls to avoid in real-world async pipelines.
---

Before Java 21's virtual threads became stable, `CompletableFuture` was the standard tool for non-blocking, concurrent Java in Spring Boot services. I've used it extensively in real-time data ingestion pipelines at Mosaic Smart Data, where we needed to fan out requests to multiple data providers, aggregate the results, and respond within strict latency budgets. Used well, it's powerful. Used carelessly, it produces code that's difficult to reason about and hides failures.

## The Basics — Creating a CompletableFuture

A `CompletableFuture` represents an asynchronous computation that may or may not have completed:

```java
// Run async, return nothing
CompletableFuture<Void> future = CompletableFuture.runAsync(
    () -> publishEvent(event),
    executor
);

// Run async, return a value
CompletableFuture<MarketData> dataFuture = CompletableFuture.supplyAsync(
    () -> dataProvider.fetchMarket(marketId),
    executor
);
```

Always provide an explicit `Executor`. The default (`ForkJoinPool.commonPool()`) is shared across the JVM — a saturated pool in one part of the application stalls unrelated async work. Create purpose-specific executors:

```java
ExecutorService ioExecutor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() * 4,
    new ThreadFactory() {
        private final AtomicInteger count = new AtomicInteger();
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "io-async-" + count.incrementAndGet());
            t.setDaemon(true);
            return t;
        }
    }
);
```

## Transforming Results — thenApply and thenCompose

`thenApply` transforms the result of a completed future synchronously (on the completing thread):

```java
CompletableFuture<String> json = dataFuture
    .thenApply(market -> objectMapper.writeValueAsString(market));
```

`thenCompose` chains another async operation, flattening the nested `CompletableFuture<CompletableFuture<T>>` that `thenApply` would produce:

```java
CompletableFuture<EnrichedMarket> enriched = dataFuture
    .thenComposeAsync(
        market -> enrichmentService.enrich(market),
        ioExecutor
    );
```

Use `thenApplyAsync` and `thenComposeAsync` variants (with an explicit executor) when the transformation itself is slow — otherwise the completion thread does the work, which may block the I/O thread pool.

## Combining Multiple Futures — thenCombine and allOf

When you need results from two independent async operations:

```java
CompletableFuture<RunnerData> runners = fetchRunners(marketId);
CompletableFuture<PriceData>  prices  = fetchPrices(marketId);

CompletableFuture<MarketView> view = runners.thenCombine(
    prices,
    (r, p) -> new MarketView(r, p)
);
```

For three or more futures, use `allOf`:

```java
List<CompletableFuture<MarketData>> futures = marketIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> fetchMarket(id), ioExecutor))
    .toList();

CompletableFuture<Void> allDone = CompletableFuture.allOf(
    futures.toArray(new CompletableFuture[0])
);

// Collect results after all complete
CompletableFuture<List<MarketData>> allResults = allDone.thenApply(
    v -> futures.stream()
        .map(CompletableFuture::join) // safe here — all futures are complete
        .toList()
);
```

Note the `join()` call: this is only safe inside `thenApply` after `allOf` has completed. Calling `join()` on an incomplete future blocks the current thread.

## anyOf — First to Complete Wins

When you want the first result from any of several sources:

```java
CompletableFuture<Object> first = CompletableFuture.anyOf(
    fetchFromSource1(id),
    fetchFromSource2(id),
    fetchFromSource3(id)
);

String result = (String) first.join();
```

The type of `anyOf` is `CompletableFuture<Object>` — the cast is unavoidable. The other futures continue running in the background; they don't get cancelled. If you need cancellation, handle it manually.

## Exception Handling

Exceptions in a `CompletableFuture` propagate through the chain as `CompletionException`. Three methods handle them:

```java
CompletableFuture<MarketData> withFallback = fetchMarket(marketId)
    // Recover with a default value — equivalent to try/catch
    .exceptionally(ex -> {
        log.warn("Fetch failed for {}, using cached data", marketId, ex);
        return cache.get(marketId);
    });

CompletableFuture<MarketData> withHandle = fetchMarket(marketId)
    // Handle both success and failure in one place
    .handle((data, ex) -> {
        if (ex != null) {
            metrics.increment("market.fetch.failure");
            return MarketData.empty();
        }
        metrics.increment("market.fetch.success");
        return data;
    });

CompletableFuture<MarketData> withPeek = fetchMarket(marketId)
    // Inspect the exception but rethrow — good for logging without swallowing
    .whenComplete((data, ex) -> {
        if (ex != null) log.error("Fetch failed", ex);
    });
```

I use `exceptionally` when I have a meaningful fallback, `handle` when I need to record metrics regardless of outcome, and `whenComplete` for pure side effects like logging.

## Avoiding Common Pitfalls

**Don't block inside a chain.** Calling `future.get()` or `future.join()` inside a `thenApply`/`thenCompose` defeats the purpose and risks deadlock if the thread pool is saturated.

**Don't use `ForkJoinPool.commonPool()` for IO.** The common pool has thread count equal to available CPUs. IO-bound work blocks those threads and starves CPU-bound work in the rest of the application.

**Set timeouts.** A future that hangs forever hangs the threads waiting on it:

```java
CompletableFuture<MarketData> withTimeout = fetchMarket(marketId)
    .orTimeout(2, TimeUnit.SECONDS)
    .exceptionally(ex -> {
        if (ex instanceof TimeoutException) {
            log.warn("Market fetch timed out for {}", marketId);
            return MarketData.empty();
        }
        throw new CompletionException(ex);
    });
```

## ProTips

- **Prefer virtual threads for new code.** If you're on Java 21, `StructuredTaskScope` gives you fork/join with cleaner code and better exception propagation. `CompletableFuture` remains valuable for complex async pipelines and where you need Java 11+ compatibility.
- **Log the causing exception, not `CompletionException`.** `CompletionException.getCause()` gives you the original exception. The wrapper is rarely useful in logs.
- **Test async code with `CompletableFuture.failedFuture()`.** Inject a pre-failed future to test error-handling paths without needing real failures from downstream services.
- **Name your threads.** Custom `ThreadFactory` names (e.g. `io-async-1`, `betfair-fetch-3`) make thread dumps interpretable when debugging in production.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
