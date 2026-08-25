# Spring Data type-safe property paths

## What it is

Spring Data Core 4.1 / the Spring Data 2026.0 line adds first-class type-safe property references based on Java method references. They replace string property names such as `"firstName"` or `"address.city"` in APIs that support typed property paths.

The compiler and IDE can then validate and refactor the property reference instead of leaving a string literal behind.

## Use when

Prefer typed property paths when a property name is known at compile time, especially for:

- sorting;
- criteria/query builders that accept typed property paths;
- nested domain-property navigation;
- code that is frequently refactored.

Keep string property paths when the property is genuinely dynamic at runtime, for example a user-selected sort field or configuration value.

## How to use it

Replace string-based sorting:

```java
Sort sort = Sort.by("firstName", "lastName");
```

with method references:

```java
Sort sort = Sort.by(Person::getFirstName, Person::getLastName);
```

For nested properties, compose a typed path:

```java
import org.springframework.data.core.TypedPropertyPath;

Sort sort = Sort.by(
    TypedPropertyPath.of(Person::getAddress)
        .then(Address::getCity),
    Person::getLastName
);
```

The owner type is part of the generic signature, so mixing properties from unrelated root types is rejected by the compiler:

```java
// Does not compile: incompatible owning types.
Sort.by(Person::getFirstName, Order::getOrderDate);
```

Where a Spring Data module exposes typed criteria, the same idea applies:

```java
where(Person::getFirstName).is("Ada");
```

Check the concrete store module because typed-path coverage varies by API.

## Migration rule

When touching existing Spring Data query/sort code:

1. If the property name is static, prefer the typed overload.
2. Replace one string path at a time; this feature is additive and does not require a repository-wide migration.
3. Keep strings at dynamic boundaries and validate/allowlist them there.
4. Prefer method references over arbitrary lambdas.

This is a good small refactoring for coding agents because the compiler immediately verifies the result.

## Why it is useful

- Property renames become compiler/IDE-visible instead of failing only at runtime.
- Nested navigation keeps type information across path segments.
- No annotation processor or generated metamodel is required.
- Adoption can be incremental.

Spring Data introspects method references once and caches the resolved property representation.

## Caveats / when not to use

- Dynamic property names still require strings or an application-level mapping from allowed external names to typed paths.
- API coverage differs across Spring Data modules; do not assume every string-based query API has a typed overload.
- Prefer method references such as `Person::getFirstName`. More complex lambdas are not equivalent property references and can require extra runtime/Native Image support.
- Existing `TypedSort` APIs based on runtime proxies are a different mechanism and may have GraalVM Native Image caveats. Prefer the newer `TypedPropertyPath`/method-reference APIs where available.

## Version / compatibility

First-class typed property-reference support was introduced in the Spring Data 2026.0 development line and is present in Spring Data Core 4.1 documentation/API. Check the Spring Data version supplied by the project's Spring Boot BOM before migrating code.

## Sources

- https://spring.io/blog/2026/02/27/moving-beyond-strings-in-spring-data/
- https://docs.spring.io/spring-data/commons/reference/property-paths.html
- https://docs.spring.io/spring-data/relational/reference/data-commons/api/java/org/springframework/data/core/PropertyReference.html
- Discovery: https://www.baeldung.com/java-weekly-636

## Related

- Use JPA metamodel/Querydsl/jOOQ generated models when the project already benefits from their broader query DSL; typed property paths are mainly the low-infrastructure replacement for brittle string property references.