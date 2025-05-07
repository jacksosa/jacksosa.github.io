---
title: Java | Why I Avoid Checked Exceptions
tags: [ Java, Exceptions, Clean Code, Spring Boot ]
style: fill
color: primary
description: As a Java developer, I’ve ditched checked exceptions - here’s why they’re a hassle and how to handle exceptions better.
---

As a Java developer who’s built systems like Mosaic Smart Data’s real-time API pipeline, Co-op’s competitor pricing
reports, ESG Global’s BOL Engine, and Ribby Hall Village’s data warehouse, I’ve learned to steer clear of checked
exceptions. Early on, I wrestled with them in Co-op’s pricing system, writing endless try-catch blocks that bloated my
code and annoyed my team. Checked exceptions, meant to force error handling, often create more problems than they solve.
Here’s why I avoid them, with an example from my projects and tips for cleaner exception handling.

## The Problem with Checked Exceptions

Checked exceptions (like `IOException`) force you to catch or declare them, which sounds great but backfires. In Co-op’s
pricing system, I dealt with `FileNotFoundException` for reading price files, and it was a mess:

- **Boilerplate Overload**: Every method touching a file needed try-catch or `throws`, cluttering my code.
- **Client Burden**: Callers had to handle exceptions they didn’t care about, frustrating my team in ESG’s engine.
- **Maintenance Pain**: Adding a new checked exception broke method signatures, forcing cascading changes in Mosaic’s
  pipeline.

**ProTip**: If your code feels like a try-catch jungle, checked exceptions might be the culprit.

## A Real-World Example

In Co-op’s pricing system, I read price data from a file using checked exceptions:

```java
public String readPriceData(String filePath) throws FileNotFoundException {
    Scanner scanner = new Scanner(new File(filePath));
    return scanner.nextLine();
}
```

Every caller needed:

```java
try {
    String data = readPriceData("prices.txt");
} catch (FileNotFoundException e) {
    // Handle or rethrow
}
```

This bloated my Spring Boot controllers and annoyed teammates who just wanted the data. I switched to unchecked
exceptions for cleaner code.

## Why Unchecked Exceptions Are Better

Unchecked exceptions (like `RuntimeException`) don’t force handling, giving you flexibility. In Ribby Hall’s sync, I
wrapped checked exceptions in a custom unchecked one:

```java
public class DataAccessException extends RuntimeException {
    public DataAccessException(Throwable cause) {
        super(cause);
    }
}

public String readPriceData(String filePath) {
    try {
        Scanner scanner = new Scanner(new File(filePath));
        return scanner.nextLine();
    } catch (FileNotFoundException e) {
        throw new DataAccessException(e);
    }
}
```

Now, callers can handle it only if needed:

```java
String data = readPriceData("prices.txt"); // No try-catch required
```

In Mosaic’s pipeline, I used Spring’s `ResponseStatusException` for API errors, keeping controllers clean:

```java
throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Price file missing");
```

## When to Use Checked Exceptions

Checked exceptions make sense for critical, recoverable errors where the caller *must* act, like database rollbacks in
ESG’s engine. But they’re overused. I stick to unchecked exceptions 90% of the time for simplicity and maintainability.

## Best Practices for Exception Handling

Here’s what I’ve learned:

- **Wrap Checked Exceptions**: Convert checked exceptions to unchecked ones (e.g., `DataAccessException`) to reduce
  boilerplate.
- **Use Descriptive Exceptions**: In Co-op’s system, custom exceptions clarified error causes.
- **Centralize Handling**: In Spring Boot apps, I use `@ControllerAdvice` to handle exceptions globally, keeping
  Mosaic’s APIs clean.
- **Avoid Swallowing Exceptions**: Logging and ignoring errors in Ribby Hall’s sync hid bugs—always propagate or handle
  meaningfully.

**ProTip**: Centralize exception handling in Spring Boot with `@ControllerAdvice` to keep your controllers lean.

## Conclusion

Checked exceptions have burned me too many times. In Co-op’s pricing system, they bloated my code; in Mosaic’s pipeline,
they complicated maintenance. Unchecked exceptions, like custom `RuntimeException` subclasses, keep my code clean and
flexible. Next time you’re tempted to `throws IOException`, consider wrapping it in an unchecked exception. Check
Oracle’s Java docs or my clean code tips [here]({{ site.baseurl }}/blog/clean-coding/) for more.

Got an exception handling trick? Share it with me [here]({{ site.baseurl }}/contact/), I’d love to swap stories!

<p class="text-center">
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Exception.html" text="Exceptions" %}
{% include elements/button.html link="https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882" text="Clean Code" %}
</p>