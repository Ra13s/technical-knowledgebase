# Hibernate ORM 7.4 core `@Audited`

## What it is

Hibernate ORM 7.4 adds an **incubating** core audit-log model through `org.hibernate.annotations.Audited`. An audited entity keeps its normal current-state table plus an audit table containing historical state, a changeset id, and the modification type.

This is different from the long-standing Envers annotation with the same simple name (`org.hibernate.envers.Audited`). Hibernate 7.4 core auditing is integrated with Hibernate's normal `Session`/HQL model and can also read an existing Envers-compatible audit schema.

## Use when

Consider core `@Audited` when all of these are true:

- the application runs Hibernate ORM 7.4+;
- you need durable entity change history rather than only application logs;
- querying historical state through normal Hibernate sessions/HQL is useful; and
- you accept an incubating Hibernate-specific API.

For an existing, stable Envers installation, do **not** migrate merely because core auditing exists. Migrate only if the simpler core APIs or temporal-session queries solve a concrete problem and the incubating status is acceptable.

## How to use it

Use the core annotation explicitly so it is not confused with Envers:

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import org.hibernate.annotations.Audited;

@Entity
@Audited
public class Account {
    @Id
    private Long id;

    private String status;

    @Audited.Excluded
    private String transientNote;
}
```

Hibernate stores current state in the entity table and historical changes in an audit table. Use `@Audited.Table` when the audit-table mapping needs custom names/schema/catalog.

### Give changesets real metadata

Do not rely on the fallback JVM-instant changeset identifier for a serious audit trail. Prefer either:

- a domain `@Changelog` entity, which can carry timestamp/user/comment metadata and whose generated id becomes the changeset id; or
- a custom `ChangesetIdentifierSupplier` configured with `hibernate.temporal.changeset_id_supplier`.

### Read historical state with normal Hibernate APIs

A historical entity can be loaded through a temporal session:

```java
try (Session historical = sessionFactory.withOptions()
        .atChangeset(changesetId)
        .openSession()) {
    Account account = historical.find(Account.class, accountId);
}
```

For revision-oriented operations, Hibernate also exposes `AuditLog`, including operations equivalent to Envers revision lookup/history. Custom historical queries can use HQL in an `atChangeset()` session, or the `changesetId()` / `modificationType()` HQL functions when querying all changesets.

## Migrating from Envers

Hibernate 7.4 provides an optional migration path with compatible default `REV` / `REVTYPE` audit columns and no required DDL change for a standard Envers schema.

Typical mapping changes are:

```text
org.hibernate.envers.Audited       -> org.hibernate.annotations.Audited
org.hibernate.envers.NotAudited    -> Audited.Excluded
RevisionEntity                     -> Changelog
RevisionNumber                     -> Changelog.ChangesetId
RevisionTimestamp                  -> Changelog.Timestamp
```

A pre-existing `REVINFO` table can be mapped by a `@Changelog` entity so revision ids continue from the same sequence.

## Why it is useful

- Audit history becomes part of Hibernate ORM core instead of requiring a separate query model.
- Historical reads can use ordinary `Session.find()` and HQL within a changeset-scoped session.
- Existing Envers schemas can be migrated without rewriting the audit tables in the default-compatible case.
- `@Changelog` makes transaction-level audit metadata explicit and queryable.

## Caveats / when not to use

- **Incubating in Hibernate 7.4.** API/behavior can still change; avoid it when long-term API stability is more important than the new integration.
- Hibernate-specific, not Jakarta Persistence.
- Envers remains supported. There is no blanket reason to migrate a working Envers system.
- Audit history is not automatically a security-grade immutable ledger. Database permissions, retention, tamper controls, and sensitive-data policy still matter.
- Choose deliberately which attributes are audited; copying secrets or unnecessary large payloads into history can create security/storage problems.

## Version / compatibility

Introduced in Hibernate ORM 7.4 and verified against 7.4.7.Final documentation. The feature is marked `@Incubating`.

## Sources

- https://docs.hibernate.org/orm/7.4/whats-new/
- https://docs.hibernate.org/orm/7.4/javadocs/org/hibernate/annotations/Audited.html
- https://docs.hibernate.org/orm/7.4/migration-guide/
