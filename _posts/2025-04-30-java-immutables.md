---
title: Java | Immutables - No Setters Allowed
tags: [ Java, Immutables, Lombok, Clean Code, Spring Boot, Apache Kafka ]
style: fill
color: light
description: As a Java developer, I’ve used immutables to make projects like Mosaic Smart Data’s pipeline bulletproof. Here’s why immutables shine and how to avoid setters.
---

As a Java developer who’s tackled systems like Mosaic Smart Data’s real-time API pipeline, Co-op’s competitor pricing
reports, ESG Global’s BOL Engine, and Ribby Hall Village’s data warehouse, I’ve learned that mutable objects can be a
recipe for chaos. Early in my career, I battled bugs from unexpected state changes in a multi-threaded Spring Boot
service, costing hours of debugging. That’s when I embraced immutable objects—classes whose state can’t change after
construction. They’ve been a game-changer for reliability and maintainability, from Kafka consumers to Activiti
workflows. Here’s why immutables are essential, how to implement them right, and why setters (and their sneaky cousins)
have no place in them.

## 1. Why Immutables Are Your Secret Weapon

An immutable object is simple: its state is set at construction and stays fixed. This predictability makes immutables a
powerhouse for robust code. In Mosaic’s pipeline, I used immutables for trade event DTOs, ensuring high-velocity data
stayed consistent across threads. Here’s why they’re worth adopting:

- **Predictable State**: No one can alter an immutable’s state, preventing runtime errors. At Co-op, immutable price
  objects eliminated inconsistencies in reports parsing 50,000 products.
- **Valid State Guaranteed**: Immutables validate inputs at construction. For ESG’s BOL Engine, I ensured meter data
  objects rejected invalid values upfront, slashing null checks elsewhere.
- **Compiler Support**: Final fields and required constructors let compilers catch mistakes. When I added a field to a
  Ribby Hall DTO, the compiler flagged every missing initialization, saving me from production bugs.

**ProTip**: Use immutables in data-intensive projects to lock in state and reduce debugging time.

## 2. Building Immutables the Right Way

Creating immutables is straightforward but requires discipline. Here’s how I implement them, drawing from my Mosaic and
ESG projects.

### Use Final Fields and Constructors

A basic immutable needs `final` fields and a constructor to set them. Here’s a `TradeEvent` class I used in Mosaic’s
pipeline:

```java
class TradeEvent {
    private final Long id;
    private final String symbol;

    TradeEvent(Long id, String symbol) {
        this.id = id;
        this.symbol = symbol;
    }
}
```

The `final` keyword ensures fields can’t change, and the constructor sets all values upfront.

**ProTip**: Make fields `private final` by default to enforce immutability and encapsulation.

### Leverage Lombok for Clean Code

Writing constructors manually is tedious, so I use Lombok’s `@RequiredArgsConstructor` to generate them. Here’s a
cleaner version:

```java

@RequiredArgsConstructor
class TradeEvent {
    private final Long id;
    private final String symbol;
}
```

This generates a constructor for all `final` fields, keeping my code concise. I used this in Co-op’s pricing system to
streamline DTOs.

**ProTip**: Watch out for Lombok’s field order—reordering fields changes the constructor signature, so document it
clearly.

### Add Factory Methods for Flexibility

Optional fields, like an ID for unsaved objects, need careful handling. Instead of passing `null` to constructors (a
code smell), use factory methods. For ESG’s workflow engine, I did this:

```java

@RequiredArgsConstructor(access = AccessLevel.PRIVATE)
class ProcessInstance {
    private final Long id;
    private final String name;

    static ProcessInstance newProcess(String name) {
        return new ProcessInstance(null, name);
    }

    static ProcessInstance existingProcess(Long id, String name) {
        return new ProcessInstance(id, name);
    }
}
```

Factory methods like `newProcess` clarify intent and prevent invalid states. I made the constructor private to force
clients to use factories.

**ProTip**: Name factory methods descriptively (e.g., `newProcess`) to signal valid field combinations.

### Validate State at Construction

Immutables should reject invalid inputs. For Mosaic’s trade events, I added validation:

```java
class TradeEvent {
    private final Long id;
    private final String symbol;

    TradeEvent(Long id, String symbol) {
        if (id != null && id < 0) {
            throw new IllegalArgumentException("ID must be >= 0");
        }
        if (symbol == null || symbol.isEmpty()) {
            throw new IllegalArgumentException("Symbol must not be empty");
        }
        this.id = id;
        this.symbol = symbol;
    }
}
```

This ensures only valid objects are created. Alternatively, I’ve used Bean Validation for declarative checks in ESG’s
DTOs:

```java
class TradeEvent extends SelfValidating<TradeEvent> {
    @Min(0)
    private final Long id;
    @NotEmpty
    private final String symbol;

    TradeEvent(Long id, String symbol) {
        this.id = id;
        this.symbol = symbol;
        this.validateSelf();
    }
}
```

**ProTip**: Centralize validation in constructors or use Bean Validation to keep rules close to fields.

### Handle Optional Fields with `Optional`

To avoid `NullPointerExceptions`, return `Optional` for nullable fields. In Ribby Hall’s data sync, I did this:

```java

@RequiredArgsConstructor(access = AccessLevel.PRIVATE)
class LedgerEntry {
    private final Long id;
    private final String description;

    static LedgerEntry newEntry(String description) {
        return new LedgerEntry(null, description);
    }

    Optional<Long> getId() {
        return Optional.ofNullable(id);
    }
}
```

Clients know `id` might be absent and handle it safely.

**ProTip**: Never use `Optional` as a field type—it’s meant for return values, not storage, to avoid null checks inside
the class.

## 3. Immutable Pitfalls to Avoid

Immutables are powerful, but certain patterns undermine them. Here’s what I steer clear of, based on painful lessons.

### No Setters or Withers

Setters and “wither” methods (e.g., `withId`) mimic mutability by returning new objects:

```java
class TradeEvent {
    private final Long id;
    private final String symbol;

    TradeEvent setId(Long id) {
        return new TradeEvent(id, this.symbol);
    }
}
```

This confuses clients, who expect immutables to be immutable. In Mosaic’s pipeline, I banned setters to keep DTOs
predictable. If you need state changes, use a mutable class instead.

**ProTip**: If you’re tempted to add setters or withers, reconsider whether the class should be mutable.

### Skip Builders for Immutables

Builders, like those from Lombok’s `@Builder`, let you set fields step-by-step:

```java
TradeEvent event = TradeEvent.builder()
        .id(42L)
        .build(); // Oops, forgot symbol
```

This risks incomplete objects, as I learned when a builder-created DTO caused a null error in Co-op’s pricing system.
Factory methods are safer, as they enforce valid combinations.

**ProTip**: Use factory methods over builders to let the compiler catch missing fields at compile time.

### Don’t Auto-Generate Getters

Lombok’s `@Getter` or IDE-generated getters can expose mutable state. In an early ESG project, I made this mistake:

```java

@Getter
class User {
    private final Long id;
    private final List<String> roles;
}
```

Clients could modify `roles` via `getRoles().add("admin")`, breaking immutability. Instead, return immutable types or
copies:

```java
class User {
    private final Long id;
    private final List<String> roles;

    List<String> getRoles() {
        return List.copyOf(roles);
    }
}
```

**ProTip**: Only provide getters for immutable types (e.g., `String`, `Long`) or return defensive copies for
collections.

## 4. Where Immutables Shine

Immutables excel in specific scenarios. Here’s where I’ve seen them transform projects:

- **Concurrency**: In Mosaic’s multi-threaded Kafka pipeline, immutable trade events eliminated race conditions,
  ensuring thread safety without locks.
- **Value race conditions**: ensuring thread safety without locks.
- **Value Objects**: For Co-op’s pricing system, immutable `Price` objects (e.g., amount and currency) guaranteed
  consistent data across reports.
- **DTOs**: In ESG’s smart meter system, immutable DTOs simplified data transfers between microservices, making
  debugging easier.
- **Domain Objects**: For Ribby Hall’s accounting sync, I made domain objects partially immutable (final fields with
  targeted methods), balancing flexibility and safety.

**ProTip**: Default to immutables for DTOs and value objects in Spring Boot apps to streamline data flows.

## Why Immutables Matter for Your Projects

Immutables aren’t just a nice-to-have—they’re a cornerstone of clean, reliable code. In Mosaic’s pipeline, they ensured
sub-second latency by preventing state-related bugs. In Co-op’s reports, they cut validation overhead. By making fields
`final` and avoiding setters, you’ll catch errors at compile time and sleep better knowing your state is locked down.

Start small: convert one DTO to an immutable, add a factory method, and validate its inputs. For inspiration, check
Oracle’s Java docs or my clean code principles [here]({{ site.baseurl }}/blog/clean-coding/). If you’re

Have you used immutables to tame complex systems? Share your wins with me [here]({{ site.baseurl }}/contact/), or ask me
for tips, I’d love to hear your story!

<p class="text-center">
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://projectlombok.org/" text="Lombok" %}
{% include elements/button.html link="https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882" text="Clean Code" %}
{% include elements/button.html link="https://www.spring.io/projects/spring-boot" text="Spring Boot" %}
</p>