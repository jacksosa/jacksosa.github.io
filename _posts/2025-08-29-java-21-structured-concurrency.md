---
title: "Java | Structured Concurrency in Java 21"
tags: [Java, Java 21, Structured Concurrency, Concurrency, Threading]
style: fill
color: secondary
description: Structured concurrency in Java 21 — StructuredTaskScope, ShutdownOnFailure, ShutdownOnSuccess, how it prevents thread leaks, and where it produces cleaner code than CompletableFuture.
---

Before structured concurrency, managing a group of concurrent tasks in Java was error-prone. If you forked three async operations with `CompletableFuture` and one failed, the other two kept running — consuming resources, potentially making external calls, and potentially producing results that would never be used. Structured concurrency, finalised in Java 21, gives you a disciplined model: tasks are scoped, they have clear lifetimes, and the scope doesn't exit until all tasks have finished or been cancelled.

## The Core Idea

Structured concurrency enforces a simple invariant: **a task's lifetime is bounded by the scope that created it**. The scope doesn't complete until all forked tasks have completed. If you exit the scope with tasks still running, they're cancelled. This eliminates the most common class of thread leak in concurrent Java.

`StructuredTaskScope` is the central API:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var runnerTask = scope.fork(() -> fetchRunners(marketId));
    var priceTask  = scope.fork(() -> fetchPrices(marketId));
    var formTask   = scope.fork(() -> fetchForm(marketId));

    scope.join();           // wait for all tasks to complete
    scope.throwIfFailed();  // propagate any exceptions

    return new MarketView(
        runnerTask.get(),
        priceTask.get(),
        formTask.get()
    );
} // scope closes here — all tasks guaranteed done or cancelled
```

Three fetches run concurrently. If any one throws, `scope.join()` returns, `throwIfFailed()` throws the exception, and the `try-with-resources` close cancels the remaining tasks. No dangling threads.

## ShutdownOnFailure — All Must Succeed

`ShutdownOnFailure` shuts down the scope as soon as any task fails, then cancels remaining tasks:

```java
public EnrichedMarket enrichMarket(String marketId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<RunnerData>  runners = scope.fork(() -> runnerService.fetch(marketId));
        Subtask<FormData>    form    = scope.fork(() -> formService.fetch(marketId));
        Subtask<WeatherData> weather = scope.fork(() -> weatherService.fetch(marketId));

        scope.join().throwIfFailed(); // join and propagate first failure

        return EnrichedMarket.of(runners.get(), form.get(), weather.get());
    }
}
```

`Subtask.get()` is safe to call after `throwIfFailed()` — all tasks either succeeded (state = `SUCCESS`) or were cancelled. Calling `get()` on a cancelled subtask throws `IllegalStateException` — which is why you check the scope result before calling `get()`.

If you need a timeout:

```java
scope.joinUntil(Instant.now().plusSeconds(5)); // deadline, not duration
scope.throwIfFailed();
```

If the deadline passes before all tasks finish, the scope cancels remaining tasks and `throwIfFailed()` throws a `TimeoutException`.

## ShutdownOnSuccess — First to Win

`ShutdownOnSuccess` completes as soon as the first task succeeds, cancels the rest:

```java
public MarketData fetchFromFastestSource(String marketId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<MarketData>()) {
        scope.fork(() -> primarySource.fetch(marketId));
        scope.fork(() -> secondarySource.fetch(marketId));
        scope.fork(() -> tertiarySource.fetch(marketId));

        scope.join();
        return scope.result(); // returns the first successful result
    }
}
```

This is a redundant-fetch pattern: three providers are queried simultaneously, and the fastest successful response wins. The others are cancelled. It's useful when you need low latency and have multiple data sources of varying reliability.

## Preventing Thread Leaks

The classic `CompletableFuture` leak pattern:

```java
// Old — if processA fails, processB keeps running and nobody notices
CompletableFuture<ResultA> a = CompletableFuture.supplyAsync(() -> processA());
CompletableFuture<ResultB> b = CompletableFuture.supplyAsync(() -> processB());

ResultA ra = a.join();  // throws if a failed
ResultB rb = b.join();  // b is still running, but we've already thrown
```

With structured concurrency:

```java
// New — if processA fails, processB is cancelled before the scope exits
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var a = scope.fork(() -> processA());
    var b = scope.fork(() -> processB());

    scope.join().throwIfFailed();
    return process(a.get(), b.get());
}
// b is guaranteed done or cancelled here
```

The scope is the mechanism that provides the guarantee. As a `AutoCloseable`, it's compatible with `try-with-resources`, and the close operation waits for tasks to terminate.

## Custom Scope Policies

For more complex fan-out/fan-in scenarios, extend `StructuredTaskScope` with a custom policy:

```java
public class MajorityResultScope<T> extends StructuredTaskScope<T> {

    private final List<T> results = Collections.synchronizedList(new ArrayList<>());
    private final int required;

    public MajorityResultScope(int required) {
        this.required = required;
    }

    @Override
    protected void handleComplete(Subtask<? extends T> subtask) {
        if (subtask.state() == Subtask.State.SUCCESS) {
            results.add(subtask.get());
            if (results.size() >= required) {
                shutdown(); // enough results — cancel remaining
            }
        }
    }

    public List<T> results() { return List.copyOf(results); }
}
```

## Structured Concurrency vs CompletableFuture

Both have their place in Java 21:

| Concern | Structured Concurrency | CompletableFuture |
|---|---|---|
| Code readability | Sequential, clear | Callback chains, dense |
| Thread leak risk | Impossible by design | Requires careful handling |
| Exception propagation | Clean, first failure wins | Wrapped in CompletionException |
| Complex async pipelines | Cumbersome | Natural (thenCompose, etc.) |
| Debugging | Standard thread dumps | Async stack traces are poor |
| Java version | Java 21+ (stable) | Java 8+ |

I use structured concurrency for fan-out patterns where I need all results (or the first result) and clean cancellation behaviour. I still use `CompletableFuture` for complex reactive pipelines where thenCompose chains read more naturally than nested scopes.

## ProTips

- **Virtual threads are the natural partner.** `StructuredTaskScope.fork()` creates virtual threads by default in Java 21. The combination gives you millions of lightweight concurrent tasks with disciplined scoping.
- **Avoid mixing structured concurrency with `CompletableFuture.allOf`.** Wrapping `CompletableFuture` inside a structured scope defeats the purpose — the futures run independently of the scope's lifecycle.
- **Propagate interruption correctly.** Tasks forked in a scope receive an interrupt when the scope shuts down. Ensure your tasks check `Thread.currentThread().isInterrupted()` or use interruptible IO — otherwise they won't cancel promptly.
- **Structured concurrency is not for long-running tasks.** A scope that forks a Kafka consumer that runs indefinitely doesn't fit the structured model. Use it for bounded, request-scoped operations.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
