---
title: Java | Java NullPointerException Avoidance and Enhancement Tactics
tags: [ Java, NullPointerException, TimeUnit, Objects, String, Enum, Clean Code ]
style: fill
color: secondary
description: A guide to avoiding NullPointerExceptions in Java, with practical tips and techniques to enhance code quality.
---

---

# Java NullPointerException Avoidance and Enhancement Tactics

An encountered `NullPointerException` can be a useful mechanism for highlighting when a certain code flow or certain
data has led to unexpected results (and the messages provided by `NullPointerException` are much improved with JDK 15).
However, there are other times when the presence of `null` is not an exceptional condition, and for those cases, there
are several tactics that can be used to easily and cleanly avoid an unwanted `NullPointerException`. Even when the
occurrence of a `NullPointerException` helps identify problems, there are other tactics we can use to make the most of
these opportunities.

## Contents

- [Elegantly Avoiding Unnecessary NullPointerExceptions](#elegantly-avoiding-unnecessary-nullpointerexceptions)
    - [Implicit Java String Conversion](#implicit-java-string-conversion)
    - [Null-safe String Representation with String.valueOf(Object)](#null-safe-string-representation-with-stringvalueofobject)
    - [Null-safe String Representation with Objects.toString(Object)](#null-safe-string-representation-with-objectstostringobject)
    - [Null-safe String Representation with Objects.toString(Object, String)](#null-safe-string-representation-with-objectstostringobject-string)
    - [Default Value Replacement of null for Any Object](#default-value-replacement-of-null-for-any-object)
    - [Comparing enums Safely](#comparing-enums-safely)
    - [Comparing Objects Safely with Known Non-null Object on LHS of .equals(Object)](#comparing-objects-safely-with-known-non-null-object-on-lhs-of-equalsobject)
    - [Case Insensitive Comparison of Strings Safely with Known Non-null String on LHS of .equals(Object)](#case-insensitive-comparison-of-strings-safely-with-known-non-null-string-on-lhs-of-equalsobject)
    - [Safely Comparing Objects When Neither is Known to be Non-null](#safely-comparing-objects-when-neither-is-known-to-be-non-null)
    - [Null-safe Hashing](#null-safe-hashing)
- [Elegantly Handling Useful NullPointerExceptions](#elegantly-handling-useful-nullpointerexceptions)
    - [Controlling When and What Related to Unexpected null](#controlling-when-and-what-related-to-unexpected-null)
- [Other Null-Handling Tactics](#other-null-handling-tactics)
- [Conclusion](#conclusion)

## Elegantly Avoiding Unnecessary NullPointerExceptions

### Implicit Java String Conversion

There are often times when we want the string representation of something that is potentially `null`, and we do not want
the access of that string representation to result in a `NullPointerException`. An example of this is when we log
certain conditions, and the context we include in the logged message includes a variable or field that is `null`. It is
highly unlikely in such a case that we want a `NullPointerException` to be possibly thrown during the attempted logging
of some potentially different condition. Fortunately, Java's string conversion is often available in these situations.

Even when the field variable `NULL_OBJECT` of type `Object` is `null`, the following code will **NOT** result in a
`NullPointerException` thanks to Java's string conversion handling a `null` implicitly and converting it to the `"null"`
string instead.

```java
/**
 * Demonstrates that Java string conversion avoids {@link NullPointerException}.
 */
public void demonstrateNullSafeStringConversion() {
    executeOperation(
            "Implicit Java String Conversion",
            () -> "The value of the 'null' object is '" + NULL_OBJECT + "'.");
}
```

**Output:**

```
Feb 25, 2021 9:26:19 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Implicit Java String Conversion' completed without exception!
```

The implicit Java string conversion avoided a `NullPointerException`. When `toString()` is explicitly called on that
same `null`, a `NullPointerException` is encountered. This is shown in the next code listing and the output it leads to.

```java
/**
 * Demonstrates that explicit {@link Object#toString()} on {@code null} leads to
 * {@link NullPointerException}.
 */
public void demonstrateNullUnsafeExplicitToString() {
    executeOperation(
            "Unsafe Explicit toString() Invocation on null",
            () -> "The value of the 'null' object is '" + NULL_OBJECT.toString() + "'.");
}
```

**Output:**

```
Feb 25, 2021 9:32:06 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
SEVERE: Exception encountered while trying to run operation for demonstration 'Unsafe Explicit toString() Invocation on null': java.lang.NullPointerException: Cannot invoke "Object.toString()" because "dustin.examples.nullsafe.tactics.NullSafeTactics.NULL_OBJECT" is null
```

Note that these examples in this post have been executed with a JDK 17 early access release, so the
`NullPointerException`s shown benefit from the better NPE messages introduced with JDK 14 (and enabled by default since
JDK 15).

### Null-safe String Representation with String.valueOf(Object)

Allowing Java's implicit string conversion to represent `null` as the `"null"` string is the cleanest and easiest way to
handle `null` when constructing strings. However, there are many times when we need a string representation of a Java
object when implicit string conversion is not available. In such cases, `String.valueOf(Object)` can be used to achieve
functionality similar to the implicit string conversion. When an object is passed to `String.valueOf(Object)`, that
method will return the results of the object's `toString()` if that object is not `null` or will return the `"null"`
string if the object is `null`.

The following code listing demonstrates `String.valueOf(Object)` in action and the output from running that code is
shown after the code listing.

```java
/**
 * Demonstrates that {@link String#valueOf(Object)} will render {@code null} safely
 * as "null" string.
 *
 * In many cases, use of {@link String#valueOf(Object)} is unnecessary because Java's
 * string conversion will perform the same effect. {@link String#valueOf(Object)} is
 * necessary when Java is not able to implicitly convert to a {@link String}.
 *
 */
public void demonstrateNullSafeStringValueOf() {
    executeOperation(
            "Null-safe String Representation with String.valueOf(Object)",
            () -> "The value of the 'null' object is '" + String.valueOf(NULL_OBJECT) + "'.");
}
```

**Output:**

```
Feb 25, 2021 10:05:52 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Null-safe String Representation with String.valueOf(Object)' completed without exception!
```

There are several overloaded versions of `String#valueOf` accepting parameter types other than `Object`, but they all
behave similarly.

### Null-safe String Representation with Objects.toString(Object)

The `Objects` class provides several methods to allow for elegant handling of potential `null`s. One of these,
`Objects.toString(Object)`, works exactly like the just-discussed `String.valueOf(Object)`. In fact, as described in the
post ["String.valueOf(Object) versus Objects.toString(Object)"](https://marxsoftware.blogspot.com/2018/08/string-valueof-vs-objects-tostring.html),
the `Objects.toString(Object)` method delegates to the `String.valueOf(Object)` method.

The following code listing demonstrates use of `Objects.toString(Object)` and the output from running it follows the
code listing.

```java
/**
 * Demonstrates that {@link Objects#toString(Object)} will render {@code null} safely
 * as "null" string.
 *
 * In many cases, use of {@link Objects#toString(Object)} is unnecessary because Java's
 * string conversion will perform the same effect. {@link Objects#toString(Object)} is
 * necessary when Java is not able to implicitly convert to a {@link String}.
 */
public void demonstrateObjectsToString() {
    executeOperation(
            "Null-safe String Representation with Objects.toString(Object)",
            () -> "The value of the 'null' object is '" + Objects.toString(NULL_OBJECT) + "'.");
}
```

**Output:**

```
Feb 25, 2021 10:19:52 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Null-safe String Representation with Objects.toString(Object)' completed without exception!
```

I tend to use `String.valueOf(Object)` instead of `Objects.toString(Object)` because the latter calls the former anyway
and because there are overloaded versions of `String#valueOf`.

### Null-safe String Representation with Objects.toString(Object, String)

The approaches covered so far in this post (implicit string conversion, `String#valueOf` methods, and
`Objects.toString(Object)`) all result in the `"null"` string when a `null` is presented to them. There are times when
we may prefer to have something other than the `"null"` string be presented as the string representation of `null`. An
example of this is when we want to return an empty string from a method rather than returning `null` from a method. The
following code listing demonstrates using `Objects.toString(Object, String)` to have an empty string be provided when
the first passed-in argument turns out to be `null`.

```java
/**
 * Demonstrates that {@link Objects#toString(Object, String)} will render {@code null}
 * potentially safely as the "default" string specified as the second argument.
 *
 * In many cases, use of {@link Objects#toString(Object, String)} is unnecessary because
 * Java's string conversion will perform the same effect. {@link Objects#toString(Object)}
 * is necessary when Java is not able to implicitly convert to a {@link String} or when
 * it is desired that the string representation of the {@code null} be something other
 * than the "null" string.
 */
public void demonstrateObjectsToStringWithDefault() {
    executeOperation(
            "Null-safe String Representation with Objects.toString(Object,String) Using Empty String Default",
            () -> "The value of the 'null' object is '" + Objects.toString(NULL_OBJECT, "") + "'.");
}
```

**Output:**

```
Feb 25, 2021 10:33:16 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Null-safe String Representation with Objects.toString(Object,String) Using Empty String Default' completed without exception!
```

### Default Value Replacement of null for Any Object

The JDK-provided methods covered so far are useful for safely acquiring string representation of objects that might be
`null`. Sometimes, we may want to handle a potential instance that might be `null` of a class other than `String`. In
that case, the `Objects.requireNonNullElse(T, T)` method allows specification of a default value that should be used if
the object in question (first parameter to the method) is `null`. This is demonstrated with the following code listing
and its accompanying output.

```java
/**
 * Demonstrates that {@link Objects#requireNonNullElse(Object, Object)} will render
 * {@code null} safely for any potential {@code null} passed to it by returning the
 * supplied default instead when the object in question is {@code null}. Two
 * examples are included in this method's demonstration:
 * <ol>
 *    <li>{@code null} {@link Object} safely rendered as custom supplied default "null" string</li>
 *    <li>{@code null} {@link TimeUnit} safely rendered as custom supplied default {@link TimeUnit#SECONDS}</li>
 * </ol>
 *
 * In many cases, use of {@link Objects#requireNonNullElse(Object, Object)} is not
 * necessary because Java's string conversion will perform the same effect.
 * {@link Objects#requireNonNullElse(Object, Object)} is necessary when Java is not
 * able to implicitly convert to a {@link String} or when the potentially {@code null}
 * object is not a {@link String} or when the object to have a default returned
 * when it is {@code null} is of class other than {@link String}.
 */
public void demonstrateNullSafeObjectsRequireNonNullElse() {
    executeOperation(
            "Null-safe String Representation with Objects.requireNonNullElse(Object, Object)",
            () -> "The value of the 'null' object is '" + Objects.requireNonNullElse(NULL_OBJECT, "null") + "'");
    executeOperation(
            "Null-safe TimeUnit access with Objects.requireNonNullElse(Object, Object)",
            () -> "The value used instead of 'null' TimeUnit is '" + Objects.requireNonNullElse(NULL_TIME_UNIT, TimeUnit.SECONDS) + "'");
}
```

**Output:**

```
Feb 28, 2021 2:54:45 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Null-safe String Representation with Objects.requireNonNullElse(Object, Object)' completed without exception!
Feb 28, 2021 2:54:45 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Null-safe TimeUnit access with Objects.requireNonNullElse(Object, Object)' completed without exception!
```

Another `Objects` method with a slightly different name (`requireNonNullElseGet(T, Supplier<? extends T>)`) allows the
default that will be used in place of `null` to be specified using a `Supplier`. The advantage of this approach is that
the operation used to compute that default value will only be executed if the object is `null`, and the cost of
executing that `Supplier` is **NOT** incurred if the specified object is not `null` (Supplier deferred execution).

### Comparing enums Safely

Although Java `enum`s can be compared for equality using `Enum.equals(Object)`, I prefer to use the operators `==` and
`!=` for comparing `enum`s because the latter is `null`-safe (and arguably makes for easier reading).

The code listing and associated output that follow demonstrate that comparing enums with `==` is `null`-safe but
comparing enums with `.equals(Object)` is NOT `null`-safe.

```java
/**
 * Demonstrates that comparing a potentially {@code null} enum is
 * {@code null}-safe when the {@code ==} operator (or {@code !=}
 * operator) is used, but that potentially comparing a {@code null}
 * enum using {@link Enum#equals(Object)} results in a
 * {@link NullPointerException}.
 *
 */
public void demonstrateEnumComparisons() {
    executeOperation(
            "Using == with enums is null Safe",
            () -> NULL_TIME_UNIT == TimeUnit.MINUTES);
    executeOperation(
            "Using .equals On null Enum is NOT null Safe",
            () -> NULL_TIME_UNIT.equals(TimeUnit.MINUTES));
}
```

**Output:**

```
INFO: Demonstration 'Using == with enums is null Safe' completed without exception!
Feb 28, 2021 4:30:17 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
SEVERE: Exception encountered while trying to run operation for demonstration 'Using .equals On null Enum is NOT null Safe': java.lang.NullPointerException: Cannot invoke "java.util.concurrent.TimeUnit.equals(Object)" because "dustin.examples.nullsafe.tactics.NullSafeTactics.NULL_TIME_UNIT" is null
```

### Comparing Objects Safely with Known Non-null Object on LHS of .equals(Object)

When we know that at least one of two objects being compared is definitely **NOT** `null`, we can safely compare the two
objects (even if the other one may be `null`), by calling `Object.equals(Object)` against the known non-`null` object.
There still is an element of risk here if the class which you're calling `.equals(Object)` against has its
`Object.equals(Object)` method implemented in such a way that passing in a `null` argument leads to a
`NullPointerException`. However, I've never encountered a JDK class or even custom class that has made that mistake (and
it is a mistake in my opinion to have an `Object.equals(Object)` overridden method not be able to handle a supplied
`null` and simply return `false` in that case instead of throwing `NullPointerException`). The tactic of calling
`.equals(Object)` against the known non-null object is demonstrated in the next code listing and associated output.

```java
/**
 * Demonstrates that comparisons against known non-{@code null} strings can be
 * {@code null}-safe as long as the known non-{@code null} string is on the left
 * side of the {@link Object#equals(Object)} method ({@link Object#equals(Object)})
 * is called on the known non-{@code null} string rather than on the unknown
 * and potential {@code null}.
 */
public void demonstrateLiteralComparisons() {
    executeOperation(
            "Using known non-null literal on left side of .equals",
            () -> "Inspired by Actual Events".equals(NULL_STRING));
    executeOperation(
            "Using potential null variable on left side of .equals can result in NullPointerExeption",
            () -> NULL_STRING.equals("Inspired by Actual Events"));
}
```

**Output:**

```
Feb 28, 2021 4:46:20 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Using known non-null literal on left side of .equals' completed without exception!
Feb 28, 2021 4:46:20 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
SEVERE: Exception encountered while trying to run operation for demonstration 'Using potential null variable on left side of .equals can result in NullPointerExeption': java.lang.NullPointerException: Cannot invoke "String.equals(Object)" because "dustin.examples.nullsafe.tactics.NullSafeTactics.NULL_STRING" is null
```

Although it was specifically `String.equals(Object)` demonstrated above, this tactic applies to instances of any class
as long as the class's `.equals(Object)` method can gracefully handle a supplied `null` (and I cannot recall ever
encountering one that didn't handle `null`).

### Case Insensitive Comparison of Strings Safely with Known Non-null String on LHS of .equals(Object)

Placing the known non-`null` object on the left side of the `.equals(Object)` call is a general `null`-safe tactic for
any object of any type. For `String` in particular, there are times when we want a `null`-safe way to compare two
strings without regard to the case of the characters in the strings (case-insensitive comparison). The
`String.equalsIgnoreCase(String)` method works well for this and will be a `null`-safe operation if we use a known non-
`null` `String` on the left side of that method (method called against the known non-`null` `String`).

The code listing and associated output that follow demonstrate `null`-safe use of `String.equalsIgnoreCase(String)`.

```java
/**
 * Demonstrates that case-insensitive comparisons against known non-{@code null}
 * strings can be {@code null}-safe as long as the known non-{@code null} string
 * is on the left side of the {@link Object#equals(Object)} method
 * ({@link Object#equals(Object)}) is called on the known non-{@code null} String
 * rather than on the unknown potential {@code null}).
 */
public void demonstrateLiteralStringEqualsIgnoreCase() {
    executeOperation(
            "String.equalsIgnoreCase(String) is null-safe with literal string on left side of method",
            () -> "Inspired by Actual Events".equalsIgnoreCase(NULL_STRING));
    executeOperation(
            "Using potential null variable of left side of .equalsIgnoreCase can result in NPE",
            () -> NULL_STRING.equalsIgnoreCase("Inspired by Actual Events"));
}
```

**Output:**

```
Feb 28, 2021 7:03:42 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'String.equalsIgnoreCase(String) is null-safe with literal string on left side of method' completed without exception!
Feb 28, 2021 7:03:42 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
SEVERE: Exception encountered while trying to run operation for demonstration 'Using potential null variable of left side of .equalsIgnoreCase can result in NPE': java.lang.NullPointerException: Cannot invoke "String.equalsIgnoreCase(String)" because "dustin.examples.nullsafe.tactics.NullSafeTactics.NULL_STRING" is null
```

These last two demonstrations used literal strings as the "known non-`null`" strings against which methods were called,
but other strings and objects could also be used. Constants and known previously initialized fields and variables are
all candidates for the objects against which the comparison methods can be called safely as long as it's known that
those fields and variables can never be changed to `null`. For fields, this condition is only guaranteed if that field
is always initialized to a non-`null` value with the instance and is immutable. For variables, this condition is only
guaranteed if that variable is initialized to a non-`null` value and is `final`. There are many "in between" cases where
it is most likely that certain objects are not `null`, but guarantees cannot be made. In those cases, it is less risky
to explicitly check each object being compared for `null` before comparing them with `.equals(Object)` or to use the
`Objects.equals(Object, Object)` method, which is covered next.

### Safely Comparing Objects When Neither is Known to be Non-null

The `Objects.equals(Object, Object)` method is a highly convenient way to compare two objects for equality when we don't
know whether either or both might be `null`. This convenience method's documentation explains its behavior: "Returns
`true` if the arguments are equal to each other and `false` otherwise. Consequently, if both arguments are `null`,
`true` is returned. Otherwise, if the first argument is not `null`, equality is determined by calling the `equals`
method of the first argument with the second argument of this method. Otherwise, `false` is returned."

This is demonstrated in the next code listing and associated output.

```java
/**
 * Demonstrates that comparisons of even potential {@code null}s is safe
 * when {@link Objects#equals(Object, Object)} is used.
 */
public void demonstrateObjectsEquals() {
    executeOperation(
            "Using Objects.equals(Object, Object) is null-safe",
            () -> Objects.equals(NULL_OBJECT, LocalDateTime.now()));
}
```

**Output:**

```
Feb 28, 2021 5:11:19 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Using Objects.equals(Object, Object) is null-safe' completed without exception!
```

I like to use the `Objects.equals(Object, Object)` to quickly build my own class's `.equals(Object)` methods in a `null`
-safe manner.

The method `Objects.deepEquals(Object, Object)` is not demonstrated here, but it's worth pointing out its existence. The
method's documentation states: "Returns `true` if the arguments are deeply equal to each other and `false` otherwise.
Two `null` values are deeply equal. If both arguments are arrays, the algorithm in `Arrays.deepEquals` is used to
determine equality. Otherwise, equality is determined by using the `equals` method of the first argument."

### Null-safe Hashing

The methods `Objects.hashCode(Object)` (single object) and `Objects.hash(Object...)` (sequences of objects) can be used
to safely generate hash codes for potentially `null` references.

```java
/**
 * Demonstrates that {@link Objects#hashCode(Object)} is {@code null}-safe.
 */
public void demonstrateObjectsHashCode() {
    executeOperation(
            "Using Objects.hashCode(Object) is null-safe",
            () -> Objects.hashCode(NULL_OBJECT));
}

/**
 * Demonstrates that {@link Objects#hash(Object...)} is {@code null}-safe.
 */
public void demonstrateObjectsHash() {
    executeOperation(
            "Using Objects.hash(Object...) is null-safe",
            () -> Objects.hash(NULL_OBJECT, NULL_STRING, NULL_TIME_UNIT));
}
```

**Output:**

```
Feb 28, 2021 5:11:19 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Using Objects.hashCode(Object) is null-safe' completed without exception!
Feb 28, 2021 5:11:19 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
INFO: Demonstration 'Using Objects.hash(Object...) is null-safe' completed without exception!
```

These methods can be convenient for generating one's own `null`-safe `hashCode()` methods on custom classes.

It's also important to note that there is a warning in the documentation that the hash code generated by
`Objects.hash(Object...)` for a single supplied `Object` is not likely to be the same value as a hash code generated for
that same `Object` when calling the `Object`'s own `hashCode()` method or when calling `Objects.hashCode(Object)` on
that `Object`.

## Elegantly Handling Useful NullPointerExceptions

The tactics described and demonstrated so far were primarily aimed at avoiding `NullPointerException` in situations
where we fully anticipated a reference being `null`, but that presence of a `null` is in no way exceptional, and so we
don't want any exception (including `NullPointerException`) to be thrown. The remainder of the descriptions and examples
in this post will focus instead on situations where we want to handle a truly unexpected (and therefore exceptional)
`null` as elegantly as possible. In many of these cases, we do **NOT** want to preclude the `NullPointerException` from
being thrown because its occurrence will communicate to us some unexpected condition (often bad data or faulty upstream
code logic) that we need to address.

The improved `NullPointerException` messages have made unexpected NullPointerExceptions far more meaningful. However, we
can often take a few additional tactics to further improve the usefulness of the `NullPointerException` that is thrown
when we run into an unanticipated `null`. These tactics include adding our own custom context details to the exception
and throwing the exception early so that a bunch of logic is not performed needlessly that may also need to be reverted.

### Controlling When and What Related to Unexpected null

I like to use `Objects.requireNonNull(T, String)` at the beginning of my public methods that accept arguments which will
lead to a `NullPointerException` if a passed-in argument is `null`. While a `NullPointerException` is thrown in either
case (either implicitly when an attempt to dereference the `null` or when `Objects.requireNonNull(T, String)` is
called), I like the ability to be able to specify a string with details and context about what's happening when the
`null` is unexpectedly encountered.

The `Objects.requireNonNull(T)` method does not allow one to specify a string with additional context, but it is still a
useful guard method. Both of these methods allow the developer to take control of when a `NullPointerException` will be
thrown for the unexpected `null`, and this control allows the developer to preclude unnecessary logic from being
performed. We'd rather not waste time/cycles on something that is going to lead to that exception anyway, and we can
often choose places in the code where it's cleaner to check for `null` and throw the exception to avoid having to "undo"
or "revert" logic that has been performed.

The following code listing and associated output demonstrate both of these methods in action.

```java
/**
 * Demonstrates using {@link Objects#requireNonNull(Object)} and
 * {@link Objects#requireNonNull(Object, String)} to take control of
 * when an {@link NullPointerException} is thrown. The method accepting
 * a {@link String} also allows control of the context that is provided
 * in the exception message.
 *
 * It is not demonstrated here, but a similar method is
 * {@link Objects#requireNonNull(Object, Supplier)} that allows a
 * {@link Supplier} to be used to provide the message for when an
 * unexpected {@code null} is encountered.
 */
public void demonstrateObjectsRequiresNonNullMethods() {
    executeOperation(
            "Using Objects.requireNonNull(T)",
            () -> Objects.requireNonNull(NULL_OBJECT));
    executeOperation(
            "Using Objects.requireNonNull(T, String)",
            () -> Objects.requireNonNull(NULL_OBJECT, "Cannot perform logic on supplied null object."));
}
```

**Output:**

```
Feb 28, 2021 5:59:42 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
SEVERE: Exception encountered while trying to run operation for demonstration 'Using Objects.requireNonNull(T)': java.lang.NullPointerException
Feb 28, 2021 5:59:42 PM dustin.examples.nullsafe.tactics.NullSafeTactics executeOperation
SEVERE: Exception encountered while trying to run operation for demonstration 'Using Objects.requireNonNull(T, String)': java.lang.NullPointerException: Cannot perform logic on supplied null object.
```

The output shows us that the method that accepted a `String` was able to provide that additional context in its message,
which can be very useful when figuring out why the unexpected `null` occurred.

I don't demonstrate it here, but it's worth noting that another overloaded version of this method (
`Objects.requireNonNull(T, Supplier<String>)`) allows a developer to use a `Supplier` to supply a custom
`NullPointerException` for complete control over the exception that is thrown. The use of the `Supplier` and its
deferred execution means that this exception generation will only be performed when the object is `null`. One might
choose to implement this `Supplier` as a relatively expensive operation checking various data sources and/or instance
values and would not need to worry about incurring that cost unless the unexpected `null` was encountered.

## Other Null-Handling Tactics

There are other tactics that can be used to either avoid unnecessary `NullPointerException`s or to make
`NullPointerException`s due to unexpected `null`s more useful. These include explicit checking for `null` in
conditionals and use of `Optional`.

## Conclusion

This post has discussed and demonstrated tactics for using standard JDK APIs to appropriately avoid unnecessary
`NullPointerException`s and to more effectively use `NullPointerException`s to indicate unexpected `null`s. There are
several simple tactics to ensure that expected `null`s do not lead to `NullPointerException`. There are also tactics
available to control when a `NullPointerException` is thrown and what details are provided in it when an unexpected
`null` is encountered.


<p class="text-center">
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Objects.html" text="Java Objects Class" %}
{% include elements/button.html link="https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/String.html" text="Java String Class" %}
{% include elements/button.html link="https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Enum.html" text="Java Enum Class" %}
{% include elements/button.html link="https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/TimeUnit.html" text="Java TimeUnit Class" %}
{% include elements/button.html link="https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/NullPointerException.html" text="Java NullPointerException Class" %}
{% include elements/button.html link="https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Object.html" text="Java Object Class" %}
</p>