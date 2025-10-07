---
title: Spring Boot | Adding Robustness with Spring Retry Annotations
tags: [ Spring Boot, Retryable, Backoff, Resilience, Java, Clean Code ]
style: fill
color: secondary
description: How to use Spring Retry to transparently handle transient failures, including configurable max attempts and exponential backoff, with practical examples.
---

---

# Adding Robustness with Spring Retry Annotations

When dealing with external systems (e.g. APIs, databases, or third-party integrations), transient failures are
inevitable.  
These failures are often temporary — a network hiccup, a short rate-limit window, or a timeout — and can be resolved
simply by **trying again after a short delay**. Rather than littering your code with manual retry loops, Spring Boot
provides a powerful and declarative way to handle this: the [
`@Retryable`](https://docs.spring.io/spring-retry/docs/current/api/org/springframework/retry/annotation/Retryable.html)
annotation.

In this post, I’ll walk through how I added **Spring Retry** to a core service method in my application, including
exponential backoff and selective exception handling.

## Contents

- [Why Retry?](#why-retry)
- [Enabling Spring Retry](#enabling-spring-retry)
- [Using @Retryable on a Service Method](#using-retryable-on-a-service-method)
- [Configuring Backoff and Exception Handling](#configuring-backoff-and-exception-handling)
- [Practical Example in a Base Provider Service](#practical-example-in-a-base-provider-service)
- [Conclusion](#conclusion)

---

## Why Retry?

When a method calls an external API, that call can fail due to reasons beyond our control — temporary outages,
throttling, or network latency spikes. In such scenarios, retrying the operation after a delay is often enough to
recover gracefully.

Without a retry mechanism, these transient errors can lead to unnecessary failures, noisy logs, and degraded user
experience. Implementing retry logic manually can also clutter your code and lead to subtle bugs.

Spring Retry provides a **declarative**, **configurable**, and **centralized** way to handle this.

---

## Enabling Spring Retry

To use Spring Retry, you need to add the dependency and enable it:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>

<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

Then, enable retry support in your Spring Boot application:

```java

@SpringBootApplication
@EnableRetry
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## Using @Retryable on a Service Method

The `@Retryable` annotation can be placed on any Spring-managed bean method to make it automatically retry on failure.

Here’s the basic structure:

```java

@Retryable(
        maxAttempts = 5,
        backoff = @Backoff(delay = 1000, multiplier = 2.0)
)
public ApiResponse callExternalApi(String param) {
    // potentially flaky operation
}

```

* `maxAttempts` specifies how many times to try before giving up.
* `@Backoff` controls the delay between attempts.
    * `delay` = 1000 means start with a 1-second delay.
    * `multiplier` = 2.0 means each subsequent retry doubles the delay (exponential backoff).

If the method keeps failing, the last thrown exception is propagated.

## Configuring Backoff and Exception Handling

One of the strengths of Spring Retry is that its parameters can be **externalized to configuration**, allowing different
environments (e.g. dev vs prod) to tune retry behavior without code changes.

```java

@Retryable(
        maxAttemptsExpression = "${retry.max.attempts:5}",
        noRetryFor = {ApiException.class, DailyLimitExceededException.class},
        backoff = @Backoff(
                delayExpression = "${retry.backoff.delay:1000}",
                multiplierExpression = "${retry.backoff.multiplier:2.0}"
        )
)
public ApiResponse callExternalApi(String param) {
    ...
}

```

### Key points:

* `maxAttemptsExpression` lets you read values from application properties, with `5` as a fallback default.
* `noRetryFor` specifies exceptions that should not trigger retries — for example, business logic errors like
* `DailyLimitExceededException` should fail immediately.
* `delayExpression` and `multiplierExpression` allow runtime configuration of backoff behavior.

Sample `application.yml`

```yaml
retry:
  max:
    attempts: 5
  backoff:
    delay: 1000
    multiplier: 2.0

```

## Practical Example in a Base Provider Service

In my application, I have an abstract base class called `ProviderService`, which provides common functionality for
multiple data provider integrations. One of its key methods, `lookup(String lookup)`, queries an external API.

Here’s how the retry mechanism was applied:

```java

@LogExecutionTime
@Retryable(
        maxAttemptsExpression = "${retry.max.attempts:5}",
        noRetryFor = {ApiException.class, DailyLimitExceededException.class},
        backoff = @Backoff(
                delayExpression = "${retry.backoff.delay:1000}",
                multiplierExpression = "${retry.backoff.multiplier:2.0}"
        )
)
public abstract E lookup(final String lookup)
        throws ApiException, DailyLimitExceededException;

```

👉 This allows **every concrete provider** (e.g. CRM, Zone, Prem) to inherit the retry behavior automatically, without
repeating logic in each subclass.

If a lookup fails due to a transient error (e.g. timeout), Spring will automatically retry it up to 5 times with
exponential backoff.
If the failure is due to a business rule (e.g. daily limit exceeded), the method will fail fast without retries.

## Conclusion

Spring Retry is a powerful tool for adding **resilience** and **fault tolerance** to your application with minimal code.
By
using `@Retryable`:

* ✅ You keep your service methods clean and focused.
* ⚙️ You make retry behavior configurable without redeploying.
* 🧠 You avoid retrying on exceptions that should fail fast.
* ⏳ You implement exponential backoff with one line of configuration.

This small annotation can have a **big impact** on the stability and reliability of your integration-heavy services.


<p class="text-center">
{% include elements/button.html link="https://docs.spring.io/spring-retry/docs/current/api/org/springframework/retry/annotation/Retryable.html" text="Spring Retry Documentation" %}
{% include elements/button.html link="https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.spring-retry" text="Spring Boot Retry Feature" %}
</p>