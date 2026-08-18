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

## Why it is useful

- Prevents accidental ORM updates for intentionally read-only mappings.
- Avoids dirty-checking work and the entity state snapshot Hibernate normally keeps for mutable entities.
- Makes read-only intent visible in the mapping itself.

## Caveats

- **Hibernate-specific**, not a Jakarta Persistence/JPA annotation.
- For an immutable entity, in-memory changes are **silently ignored** rather than rejected. Do not expect an exception to catch misuse.
- This is not database-level protection. Use database permissions/constraints as well when writes must be impossible regardless of application code.
- In a mapped inheritance hierarchy, entity-level `@Immutable` belongs on the root entity and is inherited by subclasses. If only part of the hierarchy should be immutable, annotate attributes instead.
- An immutable collection behaves differently from an immutable entity: collection mutation throws `HibernateException`.

## Version / compatibility

Verified against Hibernate ORM 7.1 documentation. `@Immutable` is a long-standing Hibernate annotation; check the documentation for the Hibernate version used by the project.

## Sources

- https://docs.hibernate.org/orm/7.1/javadocs/org/hibernate/annotations/Immutable.html

## Related

- Hibernate session/query read-only modes (`Session#setReadOnly`, `Query#setReadOnly`) when read-only behavior should apply only to a particular operation rather than the mapping globally.
