# Hibernate `@EmbeddedTable`

## What it is

`org.hibernate.annotations.EmbeddedTable` maps a whole `@Embedded` value to a named `@SecondaryTable` without repeating `@AttributeOverride` / `@AssociationOverride` table declarations for every embedded member.

It is a Hibernate-specific, `@Incubating` annotation introduced in Hibernate ORM 7.2.

## Use when

Use it when:

- an entity uses `@SecondaryTable`;
- one complete embeddable belongs in that secondary table; and
- Hibernate-specific mapping is acceptable.

Prefer it over a pile of per-property overrides when the mapping rule is simply "all fields of this embedded value live in this table".

## How to use

```java
@Entity
@Table(name = "person")
@SecondaryTable(name = "person_address")
class Person {

    @Id
    Long id;

    @Embedded
    @EmbeddedTable("person_address")
    Address address;
}

@Embeddable
class Address {
    String street;
    String city;
}
```

Without `@EmbeddedTable`, Jakarta Persistence portable mappings generally need table information repeated through `@AttributeOverride` and, where associations are involved, `@AssociationOverride`.

## Why it is useful

It makes the mapping express the actual rule once and avoids override boilerplate that becomes fragile when fields are added to the embeddable.

## Caveats

- Hibernate extension, not Jakarta Persistence portable.
- `@Incubating`: API details may evolve.
- Supported only for an embedded value declared directly on an entity or mapped superclass. Invalid placement can result in `AnnotationPlacementException`.
- Use the standard override annotations when provider portability is required.

## Version constraints

- Hibernate ORM **7.2+**.
- Hibernate's compatibility matrix maps ORM 7.2 to Spring Boot 4.0 and ORM 7.4 to Spring Boot 4.1. Spring Boot 3.4–3.5 uses Hibernate 6.6, so this annotation is not available there by default.

## Sources

- Hibernate ORM 7.2 Javadoc: https://docs.hibernate.org/orm/7.2/javadocs/org/hibernate/annotations/EmbeddedTable.html
- Hibernate ORM release/compatibility matrix: https://hibernate.org/orm/releases/
