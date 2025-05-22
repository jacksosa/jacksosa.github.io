---
title: Java | Pattern match Optional in Java 21
tags: [ Java, Pattern Matching, Optional, Java 21 ]
style: fill
color: dark
description: A trick to get pattern matching on Optional in Java 21, but one you'll probably never actually use.
---

---

## Using Optional

As of Java 21, Pattern matching in Java allows us to check a value against a type like an instanceof with a new variable
being declared of the correct type. Pattern matching can handle simple types and the deconstruction of records. But
pattern matching of arbitrary classes like Optional is not yet supported. (Work to support pattern match methods is
ongoing).

In normal code, the best way to use Optional is with one of the functional methods:

```java
var addressOpt = findAddress(personId);
var addressStr = addressOpt
.map(address -> address.format())
.orElse("No address available");
```

This works well in most cases. But sometimes you want to use the Optional with a return statement. This results in code
using get() like this:

```java
var addressOpt = findAddress(personId);
if (addressOpt.isPresent()) {
// early return if address found
return addressOpt.get().format();
}
// lots of other code to handle case when address not found
```
One way to improve this is to write a simple method:

```java
/**
* Converts an optional to an iterable for use in the for-each statement.
*
* @param <lT> the type of optional element
* @param optional the optional
* @return an iterable representation of the optional
*/
public static <lT> Iterable<lT> inOptional(Optional<lT> optional) {
return optional.isPresent() ? List.of(optional.get()): List.of();
}
```
Which allows the following neat form:

```java
for (var address : inOptional(findAddress(personId))) {
// early return if address found
return address.format();
}
// lots of other code to handle case when address not found
```

This is a great approach providing that you don't need an else branch.

## Using Optional with Pattern matching

With Java 21 and pattern matching we have a new way to do this!

```java
if (findAddress(personId).orElse(null) instanceof Address address) {
// early return if address found
return address.format();
} else {
// lots of other code to handle case when address not found
}
```

This makes use of the fact that `instanceof` rejects `null`. Any expression can be used that creates an `Optional` -
just pop `.orElse(null)` on the end.

Note that this does not work with Java 17 in most cases, because unconditional patterns were not permitted. (In most
cases, the expression will return a value of the type being checked for, and the compiler rejects it as being `always
true`. Of course this was a simplification in earlier Java versions, as the `null` really needed to be checked for.)

Is this a useful trick?

In reality, this is of marginal use. Where I think it could be of is a long `if-else` chain, for example:

```java
if (obj instanceof Integer value) {
// use value
} else if (obj instanceof Long value) {
// use value
} else if (obj instanceof Double value) {
// use value
} else if (lookupConverter(obj).orElse(null) instanceof Converter conv) {
// use converter
} else {
// even more options
}
```

(It is valuable here as it avoids calling `lookupConverter(obj)` until necessary.)

Anyway, it is a fun trick even if you never use it!

One day soon I imagine we will use something like this:

```java
if (lookupConverter(obj) instanceof Optional.of(Converter conv)) {
// use converter
} else {
// even more options
}
```

which I think we'd all agree is the better approach.

<p class="text-center">
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://openjdk.org/jeps/441" text="JEP 441" %}
{% include elements/button.html link="https://openjdk.org/jeps/441#pattern-matching-for-instanceof" text="Pattern matching for instanceof" %}
{% include elements/button.html link="https://openjdk.org/jeps/441#pattern-matching-for-optional" text="Pattern matching for Optional" %}
{% include elements/button.html link="https://www.oracle.com/java/technologies/javase/jdk21-compatibility-notes.html" text="Java 21 Compatibility Notes" %}
{% include elements/button.html link="https://www.oracle.com/java/technologies/javase/21-relnotes.html" text="Java 21 Release Notes" %}
</p>