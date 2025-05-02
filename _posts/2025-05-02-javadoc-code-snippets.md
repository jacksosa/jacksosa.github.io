---
title: Java | Formatting Code Snippets in Javadoc
tags: [ Java, Javadoc, Documentation ]
style: fill
color: danger
description: As a Java developer, I’ve used Javadoc to document APIs for projects like Mosaic Smart Data. Here’s how to format code snippets correctly with Javadoc.
---

---

As a Java developer who’s built systems like Mosaic Smart Data’s real-time API pipeline, Co-op’s competitor pricing
reports, ESG Global’s BOL Engine, and Ribby Hall Village’s data warehouse, I know that clear documentation is as
critical as clean code. When I was documenting a Spring Boot REST API for Mosaic’s financial analytics, I hit a snag:
formatting code snippets in Javadoc. Special characters like `<`, `>`, and `@` kept breaking the rendered HTML, making
my docs unreadable. After some trial and error, I mastered the use of `<pre>`, `<code>`, and `{@code}` to create
polished, maintainable Javadoc. Here’s my guide to formatting code snippets correctly, with practical tips to make your
API docs shine.

## The Javadoc Code Markup Puzzle

When documenting APIs, you often need to embed code snippets in Javadoc comments to show usage examples. But Javadoc’s
HTML rendering can mangle special characters like `<` (for generics), `>` (for XML), or `@` (for annotations) if you
don’t mark up the code properly. I ran into this while documenting a Kafka consumer for Mosaic’s pipeline, where a
snippet with `@KafkaListener` and `List<String>` looked like gibberish in the generated HTML.

Java offers three markup options for code snippets: `<pre>`, `<code>`, and `{@code}`. Each behaves differently with
indentation, line breaks, and special characters. Choosing the wrong one can break your docs or force tedious escaping
with HTML codes like `&#60;` for `<`. Let’s break down each option and when to use it.

**ProTip**: Always preview your Javadoc with `mvn javadoc:javadoc` or your IDE’s Javadoc view to catch rendering issues
early.

## Using <pre> for Multi-Line Code

The `<pre>` HTML tag is designed for preformatted text, preserving indentation and line breaks—perfect for multi-line
code snippets. I used it to document a utility class for Co-op’s pricing system, where I needed to show a formatted
method with annotations and generics.

Here’s an example:

````java
/**
 * <pre>
 * public class PriceParser {
 *   // Indentation and line breaks are preserved
 *
 *   &#64;Override
 *   public List&#60;String&#62; parsePrices() {
 *     // Method body
 *   }
 * }
 * </pre>
 */
public class PriceParser {
}
````

What it does:

* Keeps indentation and line breaks intact.
* Requires escaping special characters: @ as &#64;, < as &#60;, and > as &#62;.
* Renders correctly as formatted code in Javadoc HTML.

Downside: Escaping special characters is a pain, especially for Java code heavy with annotations or generics. I spent
extra time double-checking HTML codes for Mosaic’s API docs to avoid broken output.

**ProTip**: Use `<pre>` for multi-line Java snippets with annotations, but budget time for escaping `@,` `<`, and `>`
manually.

## Using `<code>` for Inline Code

The `<code>` HTML tag is meant for inline code, like a single variable or method name. I tried it for a multi-line
snippet in ESG’s BOL Engine docs, but it was a mess. Here’s what happened:

````java
/**
 * Using &#60;code&#62; for a snippet:
 * &#60;code&#62;@KafkaListener List&#60;String&#62; processData()&#60;/code&#62;
 */
public class DataProcessor {
}
````

What it does:

* Wraps text in a monospace font, ideal for inline terms like processData.
* Loses indentation and line breaks for multi-line snippets.
* Requires escaping `@,` `<`, and `>` with HTML codes.

Downside: It’s useless for multi-line snippets, as formatting collapses. I stopped using `<code>` for anything but
single-line references, like documenting a method name in Ribby Hall’s GraphQL API.

**ProTip**: Reserve `<code>` for inline code, like `List<String>` or `@Override`, in narrative Javadoc text.

## Using `{@code}` for Hassle-Free Special Characters

Introduced in Java 5, `{@code}` is a Javadoc tag that simplifies code formatting by automatically escaping special
characters. I used it to document a Spring Boot controller for Mosaic’s pipeline, where I needed to show a snippet with
generics and annotations without manual escaping.

Example:

````java
/**
 * Using {@code @KafkaListener} with a generic {@code List<String>}:
 * {@code
 * @KafkaListener
 * List<String> consumeEvents()
 * }
 */
public class EventConsumer {
}
````

What it does:

* Automatically escapes `<`, `>`, and `@`, so no HTML codes needed.
* Loses indentation and line breaks, making it tricky for multi-line snippets.
* Renders as monospace text in Javadoc HTML.

Downside: The loss of formatting makes `{@code}` alone impractical for complex snippets. I hit this issue when
documenting a multi-line method for Co-op’s pricing parser.

**ProTip**: Use `{@code}` for inline snippets or single-line examples where special characters like `@` or `<` are
common.

## Combining <pre> and `{@code}` for the Best of Both Worlds

For multi-line snippets with special characters, combining `<pre>` and `{@code}` seemed like the holy grail. I tested it
for ESG’s Activiti workflow docs, hoping to preserve formatting and skip escaping. Here’s how it looks:

````java
/**
 * <pre>{@code
 * public class WorkflowEngine {
 *   // Indentation and line breaks preserved
 *
 *   @BpmnProcess
 *   public List<String> startProcess() {
 *     // Method body
 *   }
 * }
 * }</pre>
 */
public class WorkflowEngine {
}
````

What it does:

* `<pre>` keeps indentation and line breaks.
* `{@code}` escapes `<` and `>` automatically.
* Renders beautifully for multi-line snippets.

Gotcha: The `@` character is still treated as a Javadoc tag inside `{@code}`, and you can’t escape it with &#64; (it
renders literally). You can use `{@literal @}` before `@`, but it adds an unwanted space, which I found ugly in Mosaic’s
docs.

ProTip: Use `<pre>{@code ...}</pre>` for multi-line HTML or XML snippets where `<` and `>` dominate, but fall back to
`<pre>` for Java code with annotations.

## Choosing the Right Markup for Your Snippet

There’s no one-size-fits-all solution, but here’s my rule of thumb based on years of documenting APIs:

Inline snippets: Use `{@code ...}` for single-line examples (e.g., `{@code List<String>}`) since it handles special
characters effortlessly.

Multi-line Java code: Use `<pre>...</pre>` to preserve formatting and escape `@`, `<`, and `>` with HTML codes. This
works best for annotation-heavy code, like `@KafkaListener`.

Multi-line HTML/XML code: Use `<pre>{@code ...}</pre>` for snippets where `<` and `>` are common, accepting the `@`
limitation.

I applied these rules to Mosaic’s pipeline docs, ensuring snippets were readable and maintainable. For Co-op’s pricing
API, `<pre>` saved time for multi-line parser examples. Clear docs reduced onboarding time for new devs, letting us
focus on coding, not deciphering Javadoc.

**ProTip**: Document your team’s Javadoc markup conventions in a README or wiki to ensure consistency across your
codebase.

## Why Javadoc Formatting Matters

Clean Javadoc isn’t just about aesthetics, it’s about making your code accessible to teammates and clients. In projects
like Mosaic’s real-time analytics or ESG’s smart meter workflows, well-formatted docs helped developers understand APIs
faster, cutting integration time. By mastering `<pre>`, `<code>`, and `{@code}`, you’ll create documentation that’s as
robust as your Spring Boot microservices.

Start small: pick one markup for your next Javadoc comment, test the output, and refine.

Have you wrestled with Javadoc formatting? Share your tricks with me [here]({{ site.baseurl }}/contact/), or ask me for
help, I’d love to swap war stories!

<p class="text-center">
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://www.oracle.com/uk/technical-resources/articles/java/javadoc-tool.html" text="Javadoc" %}
</p>