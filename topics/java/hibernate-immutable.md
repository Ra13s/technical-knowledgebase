# Hibernate `@Immutable`

## What it is

`org.hibernate.annotations.Immutable` tells Hibernate that an entity, attribute, collection, or converter-backed Java type is immutable.

For an immutable **entity**, Hibernate skips dirty checking/state snapshots and does not synchronize in-memory changes back to the database.

## Use when

Use it for Hibernate-managed data that the application should read but never update through the ORM, for example:

- reference/lookup tables
- historical snapshots
- imported read-only data
- entities mapped to read-only database views

Prefer it over relying only on developer discipline when a mapped entity is intentionally read-only everywhere.

Also consider field-level immutability for expensive custom/basic values such as JSON when the application follows a **replace, never mutate in place** rule.

## How to use it

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import org.hibernate.annotations.Immutable;

@Entity
@Immutable
public class Country {
    @Id
    private String code;

    private String name;
}
```

If a managed `Country` is mutated later, Hibernate ignores the entity state change and emits no update for it.

You may also apply `@Immutable` to an individual basic attribute:

```java
@Immutable
private String externalId;
```

or to a collection:

```java
@Immutable
@OneToMany(mappedBy = "catalog")
private List<CatalogEntry> entries;
```

The collection form is stricter: adding or removing an element causes `HibernateException`.

### Expensive JSON/custom basic values

Entity-level `@Immutable` does **not** necessarily make the Java type's own Hibernate `MutabilityPlan` immutable. For JSON/custom values where Hibernate would otherwise deep-copy or serialize the value for snapshots/cache assembly, use field-level `@Mutability(Immutability.class)` if the value is never mutated in place:

```java
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.annotations.Mutability;
import org.hibernate.type.SqlTypes;
import org.hibernate.type.descriptor.java.Immutability;

@JdbcTypeCode(SqlTypes.JSON)
@Mutability(Immutability.class)
private JsonNode data;
```

This is only correct when code replaces the reference:

```java
entity.setData(newJsonNode);
```

and does not mutate the existing object:

```java
entity.getData().put("status", "done"); // unsafe with immutable mutability plan
```

For immutable entities placed in Hibernate second-level cache, prefer a read-only cache strategy when appropriate:

```java
@Entity
@Immutable
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
class ReferenceData { ... }
```

A 2026 Hibernate case study found that combining entity `@Immutable`, `@Cache(READ_ONLY)`, and field-level `@Mutability(Immutability.class)` removed substantial dirty-check/deep-copy/cache-serialization overhead in a JSON-heavy workload. Treat the published speedups as workload-specific; the reusable rule is to profile for expensive `deepCopy`/serialization and mark values immutable only when their mutation semantics really allow it.

## Why it is useful

- Prevents accidental ORM updates for intentionally read-only mappings.
- Avoids dirty-checking work and the entity state snapshot Hibernate normally keeps for mutable entities.
- Can remove expensive deep-copy/cache serialization for immutable-by-convention JSON/custom values when `@Mutability(Immutability.class)` is safe.
- Makes read-only intent visible in the mapping itself.

## Diagnostic recipe

If Hibernate-heavy code is unexpectedly CPU/allocation-heavy, profile and look for:

- `deepCopy`
- `performDirtyCheck` / `findDirty` / `isDirty`
- `FormatMapperBasedJavaType.deepCopy`
- Jackson JSON serialize/deserialize calls around Hibernate snapshots or L2 cache

If a custom/basic value is never mutated in place, field-level `@Mutability(Immutability.class)` is a concrete optimization candidate.

## Caveats

- **Hibernate-specific**, not a Jakarta Persistence/JPA annotation.
- For an immutable entity, in-memory changes are **silently ignored** rather than rejected. Do not expect an exception to catch misuse.
- `@Mutability(Immutability.class)` is a correctness promise. Do not use it when callers mutate the existing object/collection in place.
- `@Cache(READ_ONLY)` is only appropriate for data that truly never changes through the application.
- This is not database-level protection. Use database permissions/constraints as well when writes must be impossible regardless of application code.
- In a mapped inheritance hierarchy, entity-level `@Immutable` belongs on the root entity and is inherited by subclasses. If only part of the hierarchy should be immutable, annotate attributes instead.
- An immutable collection behaves differently from an immutable entity: collection mutation throws `HibernateException`.

## Version / compatibility

`@Immutable` is long-standing Hibernate functionality. `@Mutability`/`Immutability` behavior and second-level cache details should be checked against the Hibernate version used by the project. Examples were reviewed against Hibernate ORM 7.x material.

## Sources

- https://docs.hibernate.org/orm/7.1/javadocs/org/hibernate/annotations/Immutable.html
- https://in.relation.to/2026/05/29/hibernate-json-dirty-checking-performance/
- Baeldung Java Weekly #660 discovery note: https://www.baeldung.com/java-weekly-660

## Related

- Hibernate session/query read-only modes (`Session#setReadOnly`, `Query#setReadOnly`) when read-only behavior should apply only to a particular operation rather than the mapping globally.
