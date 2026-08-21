# Spring / Java null-safety with JSpecify and NullAway

## What it is

Spring Framework 7 and the Spring Boot 4 generation expose nullability with the standard JSpecify annotations. In application code, use `@NullMarked` to make a package non-null by default, use JSpecify `@Nullable` on the exceptional nullable type uses, and optionally run NullAway in the build so violations fail compilation instead of remaining IDE warnings.

## Use when

Use this for Spring Boot 4 / Spring Framework 7 Java code when you want null contracts to be explicit and machine-checked, especially for:

- service and library APIs where `null` is part of the contract;
- Java/Kotlin mixed systems;
- packages with recurring production `NullPointerException` risk;
- libraries whose public API should carry nullability to downstream callers.

Adopt it package-by-package rather than trying to annotate an entire legacy codebase in one change.

## 1. Mark a package non-null by default

Create `package-info.java`:

```java
@NullMarked
package com.example.orders;

import org.jspecify.annotations.NullMarked;
```

Inside a null-marked package, unannotated reference type uses are treated as non-null.

## 2. Mark only the nullable type uses

```java
import org.jspecify.annotations.Nullable;

public interface CustomerLookup {

    @Nullable Customer findByExternalId(String externalId);
}
```

JSpecify annotations are `TYPE_USE` annotations. Placement matters for arrays and generics:

```java
List<@Nullable String> values;   // list is non-null; elements may be null
String @Nullable [] values;     // array may be null; elements are non-null
@Nullable String[] values;      // array is non-null; elements may be null
```

When overriding methods, copy the intended JSpecify contract explicitly; nullability annotations are not inherited onto the overriding declaration.

## 3. Make the build enforce it with NullAway

NullAway can limit analysis to code explicitly marked with `@NullMarked`:

```text
-Xep:NullAway:ERROR
-XepOpt:NullAway:OnlyNullMarked=true
-XepOpt:NullAway:CustomContractAnnotations=org.springframework.lang.Contract
```

The Spring `@Contract` option lets NullAway understand contracts such as `Assert.notNull(...)` instead of producing avoidable warnings after a successful assertion.

For fuller JSpecify semantics, including more checks around arrays and generics, add as a second migration step:

```text
-XepOpt:NullAway:JSpecifyMode=true
```

Do not start with full JSpecify mode while the basic package annotations still produce a large backlog of warnings. First get `OnlyNullMarked` packages clean, then strengthen the checker.

## Compiler compatibility

NullAway's JSpecify mode needs a javac capable of reading type-use annotations from bytecode:

- JDK 22+ works directly;
- supported OpenJDK builds of JDK 21.0.8+ and 17.0.19+ can use `-XDaddTypeAnnotationsToSymbol=true`;
- Oracle JDK 21/17 does not support that flag according to NullAway's current documentation;
- a newer compiler can still target an older runtime with `--release`, e.g. Java 25 compiler + `--release 17`.

Spring recommends using a recent compiler and treating full JSpecify mode as a second step because NullAway's complete JSpecify support is still evolving.

## Migration from old Spring null annotations

Spring Framework 7 deprecates `org.springframework.lang.Nullable`, `NonNull`, `NonNullApi`, and `NonNullFields` in favor of JSpecify.

Do not mechanically move an old `@Nullable` annotation without checking type-use placement. The array example is the common trap: old declaration-level semantics and JSpecify type-use semantics can describe different things.

## Why it is useful

- Turns many nullability mistakes into build failures.
- Makes Java APIs more explicit without changing their runtime signatures.
- Spring's JSpecify annotations are automatically understood as Kotlin nullability rather than unsafe platform types.
- `@NullMarked` keeps the common non-null case terse instead of decorating every declaration.

## Caveats

- Third-party libraries without useful nullability metadata remain a weak boundary; introduce local adapters or NullAway library models rather than assuming their contracts are correct.
- NullAway JSpecify mode is still under development and may require targeted suppressions for tooling limitations.
- `@Nullable` is a contract, not validation. External input still needs runtime validation where appropriate.
- Avoid blanket suppressions. When a suppression is necessary, document why the checker cannot prove the code safe.

## Sources

- Spring Framework null-safety reference: https://docs.spring.io/spring-framework/reference/core/null-safety.html
- Spring Boot 4 null-safety guidance: https://spring.io/blog/2025/11/12/null-safe-applications-with-spring-boot-4/
- NullAway configuration: https://github.com/uber/NullAway/wiki/Configuration
- NullAway JSpecify support: https://github.com/uber/NullAway/wiki/JSpecify-Support
