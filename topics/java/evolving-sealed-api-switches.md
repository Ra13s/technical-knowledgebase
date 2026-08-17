# Switching over evolving sealed APIs

## Decision rule

Treat exhaustive switches differently depending on who owns the sealed hierarchy.

**Hierarchy owned and released with the consuming code:** omit a catch-all branch when every subtype should be handled explicitly. Adding a subtype then turns missing handling into a compile-time failure.

**Hierarchy comes from an independently versioned dependency:** if forward compatibility with newer dependency versions matters, add an explicit catch-all branch or guarantee that consumers are always recompiled and tested against every dependency upgrade.

## Why

Adding a permitted subtype to a sealed class or interface is binary compatible in Java, so an already compiled consumer still links. But an old exhaustive switch can encounter the new subtype at runtime and throw `MatchException` because that subtype did not exist when the switch was compiled.

That creates a subtle distinction:

- exhaustive switch without `default` is excellent for **closed, co-evolved domain models**;
- it can be a runtime compatibility hazard for **externally evolving sealed APIs**.

## Example: internal closed hierarchy

```java
sealed interface PaymentResult permits Paid, Declined {}

String message(PaymentResult result) {
    return switch (result) {
        case Paid paid -> "Paid " + paid.id();
        case Declined declined -> "Declined: " + declined.reason();
    };
}
```

If we add another subtype in the same codebase and recompile, the compiler points at every switch that needs a decision. Do not add `default` merely to silence that useful check.

## Example: external/evolving hierarchy

```java
String describe(ExternalResult result) {
    return switch (result) {
        case ExternalResult.Success success -> success.value();
        case ExternalResult.Failure failure -> failure.message();
        default -> throw new UnsupportedOperationException(
            "Unsupported result subtype: " + result.getClass().getName()
        );
    };
}
```

The explicit catch-all prevents the JVM's implicit exhaustive-switch failure mode and gives us a controlled place to log, meter, degrade gracefully, or fail with domain-specific diagnostics.

If silently accepting unknown variants would be dangerous, throwing is preferable to inventing fallback behavior. The point is to control the failure path, not to hide new variants.

## Upgrade check

For dependencies exposing sealed result/state hierarchies:

1. Recompile consumers when upgrading the dependency.
2. Run tests that exercise variant dispatch.
3. If runtime dependency substitution is possible, use explicit catch-all handling at that boundary.

## Version constraints

The `MatchException` behavior described here applies to modern pattern-matching switches; Java SE 26 documents the runtime failure explicitly. The broader design rule is relevant whenever independently compiled consumers switch exhaustively over an evolving sealed hierarchy.

## Sources

- Java SE 26 JLS §13.4.2.1 and §13.5.2: https://docs.oracle.com/en/java/javase/26/docs/specs/jls/jls-13.html
- `java.lang.MatchException`: https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/lang/MatchException.html
- OpenJDK sealed-types design notes: https://openjdk.org/projects/amber/design-notes/records-and-sealed-classes
