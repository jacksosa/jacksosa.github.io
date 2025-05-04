---
title: Java | Records for Simpler, Cleaner Code
tags: [ Java, Records, Clean Code, Spring Boot, Immutability ]
style: fill
color: danger
description: As a Java developer, I’ve used records to cut boilerplate in projects like Mosaic Smart Data’s pipeline. Here’s my beginner-friendly guide to mastering Java records.
---

As a Java developer who’s tackled systems like Mosaic Smart Data’s real-time API pipeline, Co-op’s competitor pricing
reports, ESG Global’s BOL Engine, and Ribby Hall Village’s data warehouse, I’ve learned that Java records are a
game-changer for writing clean, concise code. Back when I started, I’d spend hours writing boilerplate for data
classes—getters, setters, `toString`, you name it. When Java 14 introduced records, I could finally focus on solving
problems instead of wrestling with syntax. From modeling trade events in Mosaic to handling pricing data at Co-op,
records have saved me time and headaches. Here’s my beginner-friendly guide to Java records, packed with examples from
my projects and lessons I’ve learned the hard way.

## What Are Java Records?

Java records, introduced as a preview in Java 14 and finalized in Java 16, are a special kind of class designed for
immutable data carriers. They’re perfect for modeling data that doesn’t change, like a trade event or a product price.
In Mosaic’s pipeline, I used records to represent trade events, cutting boilerplate and making my code scream clarity.
Records automatically provide getters, `equals()`, `hashCode()`, and `toString()`, so you don’t have to write them
yourself.

**ProTip**: Use records for simple data holders to slash boilerplate and keep your code clean.

## Why Were Records Introduced?

Before records, I’d write verbose classes for data objects. For example, in Co-op’s pricing system, a `Price` class
looked like this:

````java
public class Price {
    private final double amount;
    private final String currency;

    public Price(double amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public double getAmount() {
        return amount;
    }

    public String getCurrency() {
        return currency;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Price price = (Price) o;
        return Double.compare(price.amount, amount) == 0 &&
                Objects.equals(currency, currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }

    @Override
    public String toString() {
        return "Price{amount=" + amount + ", currency='" + currency + "'}";
    }
}
````

That’s a lot of code for a simple data holder! Records were introduced to eliminate this boilerplate, making immutable
data classes concise and readable. In ESG’s BOL Engine, records let me focus on workflow logic instead of getter-setter
noise.

## Creating a Java Record

Creating a record is dead simple. In Mosaic’s pipeline, I modeled a `TradeEvent` like this:

````java
public record TradeEvent(String symbol, double price, int size) {
}
````

This one-liner gives you:

- A final class with private, final fields (`symbol`, `price`, `size`).
- Public accessor methods (e.g., `symbol()`, not `getSymbol()`).
- A canonical constructor.
- Implementations of `equals()`, `hashCode()`, and `toString()`.

You can use it like this:

````java
TradeEvent event = new TradeEvent("AAPL", 150.0, 100);
System.out.println(event.symbol()); // AAPL
System.out.println(event); // TradeEvent[symbol=AAPL, price=150.0, size=100]
````

**ProTip**: Stick to records for immutable data to leverage their built-in features and avoid manual boilerplate.

## Customizing Records

Records are flexible. You can add custom constructors, methods, or static fields. In Co-op’s pricing system, I added
validation to a `Price` record:

````java
public record Price(double amount, String currency) {
    public Price {
        if (amount < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        if (currency == null || currency.isBlank()) {
            throw new IllegalArgumentException("Currency cannot be null or blank");
        }
    }

    public String formattedPrice() {
        return String.format("%s %.2f", currency, amount);
    }
}
````

Used like this:

````java
Price price = new Price(19.99, "GBP");
System.out.println(price.formattedPrice()); // GBP 19.99
````

In Ribby Hall’s data sync, I added a static method to a `Config` record:

````java
public record Config(String endpoint, int timeout) {
    public static Config defaultConfig() {
        return new Config("localhost:8080", 30);
    }
}
````

## Records and Immutability

Records are inherently immutable—their fields are final, and there are no setters. This makes them perfect for
thread-safe data models. In Mosaic’s pipeline, I used `TradeEvent` records to ensure trade data wasn’t modified
accidentally in multi-threaded streams. Immutability also simplifies reasoning about code, as I found in ESG’s workflow
engine where records prevented state-related bugs.

## Use Cases for Records

Records shine in several scenarios:

- **Data Transfer Objects (DTOs)**: In Co-op’s Spring Boot API, I used a `PriceDTO` record to transfer pricing data,
  reducing boilerplate.
- **Domain Models**: In Mosaic’s pipeline, `TradeEvent` records modeled trade data cleanly.
- **Configuration Objects**: In Ribby Hall’s sync, `Config` records held immutable settings.
- **Event Objects**: In ESG’s BOL Engine, records represented workflow events, ensuring immutability.

Here’s a `PriceDTO` example from Co-op:

````java
public record PriceDTO(double amount, String currency) {
}
````

Used in a controller:

````java

@RestController
public class PriceController {
    @GetMapping("/prices")
    public PriceDTO getPrice() {
        return new PriceDTO(19.99, "GBP");
    }
}
````

## Benefits of Using Records

Records have saved me time and effort:

- **Less Boilerplate**: No need for getters, setters, or `equals()`. In Co-op’s system, records cut my `Price` class
  from 30 lines to 3.
- **Immutability**: Built-in final fields prevent accidental changes, critical for Mosaic’s multi-threaded pipeline.
- **Readability**: Concise syntax makes intent clear, as I saw in ESG’s workflow models.
- **Standardized Behavior**: Consistent `toString()`, `equals()`, and `hashCode()` reduce bugs.

## Limitations of Records

Records aren’t perfect. Here’s what I’ve run into:

- **Immutability Only**: No setters mean you can’t use records for mutable data. In ESG’s system, I used a regular class
  for mutable workflow state.
- **No Inheritance**: Records can’t extend other classes or be extended (they implicitly extend `Record`). I worked
  around this in Mosaic by using composition.
- **Final Fields**: You can’t have non-final instance fields, which limited me in Ribby Hall’s sync for dynamic configs.
- **Restricted Constructors**: The canonical constructor is fixed, though compact constructors help. I added validation
  in Co-op’s `Price` record to handle this.

**ProTip**: Use regular classes when you need mutability or inheritance, but lean on records for immutable data holders.

## Records vs. Classes vs. Other Alternatives

### Records vs. Regular Classes

Regular classes require manual boilerplate for immutable data. In Co-op’s early pricing system, I wrote a verbose
`Price` class (see above). A `Price` record is just:

````java
public record Price(double amount, String currency) {
}
````

Records are concise but lack mutability and inheritance, so I use classes for complex logic or mutable state in ESG’s
engine.

### Records vs. Lombok

Lombok’s `@Data` or `@Value` annotations reduce boilerplate, but they’re not part of the Java language and require
external dependencies. In Ribby Hall’s sync, I switched from Lombok to records for a `Config` class to avoid build
complexity:

````java

@Data // Lombok
public class Config {
    private final String endpoint;
    private final int timeout;
}
````

Vs. a record:

````java
public record Config(String endpoint, int timeout) {
}
````

Records are cleaner and dependency-free, which I prefer for long-term maintenance.

### Records vs. Immutables Library

The Immutables library generates immutable classes with custom features, but it’s overkill for simple cases. In Mosaic’s
pipeline, I replaced an Immutables-based `TradeEvent` with a record, simplifying my codebase:

````java
public record TradeEvent(String symbol, double price, int size) {
}
````

Use Immutables for advanced immutability needs, but records for straightforward data holders.

## Practical Examples

Here are real-world examples from my projects:

### Example 1: Modeling a Domain Object

In Mosaic’s pipeline, I used a `TradeEvent` record:

````java
public record TradeEvent(String symbol, double price, int size) {
}
````

Processed in a stream:

````java
List<String> symbols = tradeEvents.stream()
        .map(TradeEvent::symbol)
        .collect(Collectors.toList());
````

### Example 2: DTO in a Spring Boot Application

In Co-op’s API, I used a `PriceDTO` record:

````java
public record PriceDTO(double amount, String currency) {
}
````

Returned from a controller:

````java

@GetMapping("/prices")
public PriceDTO getPrice() {
    return new PriceDTO(19.99, "GBP");
}
````

### Example 3: Custom Constructor with Validation

In ESG’s BOL Engine, I validated a `WorkflowEvent` record:

````java
public record WorkflowEvent(String id, String name) {
    public WorkflowEvent {
        if (id == null || id.isBlank()) {
            throw new IllegalArgumentException("ID cannot be null or blank");
        }
    }
}
````

### Example 4: Static Factory Method

In Ribby Hall’s sync, I added a factory method to a `Config` record:

````java
public record Config(String endpoint, int timeout) {
    public static Config defaultConfig() {
        return new Config("localhost:8080", 30);
    }
}
````

Used like:

````java
Config config = Config.defaultConfig();
````

## Common Pitfalls and Best Practices

Records are awesome, but I’ve hit bumps:

- **Overusing Records**: In ESG’s system, I tried using a record for mutable state and had to refactor to a class. Stick
  to immutable data.
- **Ignoring Validation**: In Co-op’s pricing, I skipped constructor validation early on, causing bugs. Always validate
  inputs in the compact constructor.
- **Complex Logic**: Records aren’t for heavy logic. In Mosaic’s pipeline, I moved complex trade processing to a service
  class.
- **Forgetting Accessor Names**: Record accessors use `fieldName()`, not `getFieldName()`. I tripped on this in Ribby
  Hall’s sync—double-check your calls.

**ProTip**: Use records for simple, immutable data, and profile their performance in streams with tools like VisualVM to
catch overhead.

## Conclusion

Java records have been a lifesaver in my projects. In Mosaic’s pipeline, they simplified trade event modeling, keeping
my code clean and thread-safe. At Co-op, they streamlined pricing DTOs, cutting boilerplate. In ESG’s BOL Engine and
Ribby Hall’s sync, they made immutable data a breeze. Records are perfect for beginners and pros alike—just define your
fields and go. Start small: replace one verbose class with a record and see the difference. Check Oracle’s Java docs or
my clean code tips [here]({{ site.baseurl }}/blog/clean-coding/) for more.

Have you used records to simplify your code? Share your wins with me [here]({{ site.baseurl }}/contact/)—I’d love to
hear your story!

<p class="text-center">
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Record.html" text="Java Records" %}
</p>