---
title: "Java | Sealed Classes and Pattern Matching — Modern Java Features"
date: 2026-07-11
tags: [Java, Java 17, Java 21, Pattern Matching, Sealed Classes]
style: fill
color: secondary
description: >-
  Sealed classes and pattern matching in switch expressions (Java 17–21) — replacing instanceof chains with exhaustive pattern matching, guarded patterns, and record deconstruction.
---

Java's type system got significantly more expressive between Java 16 and Java 21. Sealed classes (Java 17), pattern matching for `switch` (Java 21), and record patterns (Java 21) work together to eliminate entire categories of boilerplate. I've replaced sprawling `if-instanceof` chains in domain event handling and command routing with switch expressions that the compiler verifies for exhaustiveness. The result is shorter, safer, more readable code.

## Sealed Classes — Defining a Closed Hierarchy

A sealed class or interface declares exactly which classes are permitted to implement it. The compiler knows the complete set of subtypes and can enforce exhaustive handling:

```java
public sealed interface DomainEvent
    permits ClaimSubmitted, ClaimApproved, ClaimRejected, ClaimWithdrawn {

    String claimId();
    Instant occurredAt();
    String eventId();
}

public record ClaimSubmitted(
    String claimId, String claimantId,
    ClaimType type, Instant occurredAt, String eventId
) implements DomainEvent {}

public record ClaimApproved(
    String claimId, String assessorId,
    String reason, Instant occurredAt, String eventId
) implements DomainEvent {}

public record ClaimRejected(
    String claimId, String assessorId,
    String rejectionCode, Instant occurredAt, String eventId
) implements DomainEvent {}

public record ClaimWithdrawn(
    String claimId, String withdrawnBy,
    Instant occurredAt, String eventId
) implements DomainEvent {}
```

The `permits` clause is the contract: only these four types can implement `DomainEvent`. If you add a fifth and forget to handle it in a switch, the compiler tells you.

## Pattern Matching in Switch

Before Java 21, handling a sealed hierarchy looked like this:

```java
// Old — verbose, error-prone
String describe(DomainEvent event) {
    if (event instanceof ClaimSubmitted e) {
        return "Claim " + e.claimId() + " submitted by " + e.claimantId();
    } else if (event instanceof ClaimApproved e) {
        return "Claim " + e.claimId() + " approved by " + e.assessorId();
    } else if (event instanceof ClaimRejected e) {
        return "Claim " + e.claimId() + " rejected: " + e.rejectionCode();
    } else if (event instanceof ClaimWithdrawn e) {
        return "Claim " + e.claimId() + " withdrawn";
    }
    throw new IllegalStateException("Unknown event: " + event.getClass());
}
```

With Java 21 pattern matching switch, the compiler verifies exhaustiveness and the code is cleaner:

```java
// Modern — exhaustive, compiler-verified
String describe(DomainEvent event) {
    return switch (event) {
        case ClaimSubmitted e  -> "Claim " + e.claimId() + " submitted by " + e.claimantId();
        case ClaimApproved e   -> "Claim " + e.claimId() + " approved by " + e.assessorId();
        case ClaimRejected e   -> "Claim " + e.claimId() + " rejected: " + e.rejectionCode();
        case ClaimWithdrawn e  -> "Claim " + e.claimId() + " withdrawn";
    };
}
```

No `default` case required — the compiler verifies that all four permitted types are handled. Add `ClaimEscalated` to the sealed interface and every switch over `DomainEvent` becomes a compile error until you add the new case.

## Guarded Patterns

Guarded patterns add a `when` condition to a case:

```java
String classify(DomainEvent event) {
    return switch (event) {
        case ClaimApproved e when e.assessorId().startsWith("SENIOR-") ->
            "Senior approval for " + e.claimId();

        case ClaimApproved e ->
            "Standard approval for " + e.claimId();

        case ClaimRejected e when "FRAUD".equals(e.rejectionCode()) ->
            "Fraud rejection — escalate " + e.claimId();

        case ClaimRejected e ->
            "Standard rejection for " + e.claimId();

        case ClaimSubmitted e  -> "Submitted: " + e.claimId();
        case ClaimWithdrawn e  -> "Withdrawn: " + e.claimId();
    };
}
```

Cases are evaluated top-to-bottom, so more specific guards must come before broader ones. The compiler does not verify guard ordering — that's your responsibility.

## Record Patterns — Deconstructing Inline

Record patterns let you destructure a record directly in the case label:

```java
// Without record patterns — two steps
case ClaimApproved e -> processApproval(e.claimId(), e.assessorId(), e.reason());

// With record patterns — deconstruct inline
case ClaimApproved(var claimId, var assessorId, var reason, var at, var id) ->
    processApproval(claimId, assessorId, reason);
```

Nested deconstruction works too:

```java
sealed interface Command permits ApproveCommand, RejectCommand {}
record ApproveCommand(ClaimId claimId, AssessorId assessorId) implements Command {}
record ClaimId(String value) {}

// Nested deconstruction
case ApproveCommand(ClaimId(var idValue), var assessorId) ->
    log.info("Approving claim {}", idValue);
```

In practice, I use nested deconstruction sparingly — it can obscure intent if the hierarchy is deep. For one or two levels it's clean.

## Practical Use Cases

**Command routing in a CQRS handler:**

```java
public void dispatch(Command command) {
    switch (command) {
        case SubmitClaim c   -> submitHandler.handle(c);
        case ApproveClaim c  -> approvalHandler.handle(c);
        case RejectClaim c   -> rejectionHandler.handle(c);
        case WithdrawClaim c -> withdrawalHandler.handle(c);
    }
}
```

**Result types replacing checked exceptions:**

```java
public sealed interface Result<T> permits Result.Ok, Result.Err {
    record Ok<T>(T value) implements Result<T> {}
    record Err<T>(String message, Exception cause) implements Result<T> {}
}

Result<MarketData> result = fetchMarket(marketId);
String outcome = switch (result) {
    case Result.Ok<MarketData>(var data) -> "Fetched " + data.marketId();
    case Result.Err<MarketData>(var msg, var ex) -> "Failed: " + msg;
};
```

**Betfair trading signal routing:**

```java
return switch (compositeSignal) {
    case STRONG_STEAM -> placeBackBet(market, selectionId);
    case STRONG_DRIFT -> placeLayBet(market, selectionId);
    case NEUTRAL      -> Optional.empty();
};
```

## ProTips

- **Keep permitted types in the same compilation unit.** If they're scattered across packages, the sealed contract is harder to maintain. Colocate the sealed interface and its permitted subtypes.
- **Prefer records for sealed subtypes.** Records give you value semantics, compact constructors for validation, and free `equals`/`hashCode` — all useful for domain events and commands.
- **Use `default` carefully in switch over sealed types.** Adding `default` defeats exhaustiveness checking. Only add it if you genuinely want to ignore unrecognised subtypes (e.g. forward-compatibility with an evolving API).
- **Pattern matching in `instanceof` is available since Java 16.** For simpler cases where you don't need exhaustiveness, `if (event instanceof ClaimApproved e) { ... }` is already cleaner than the old cast-after-check pattern.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
