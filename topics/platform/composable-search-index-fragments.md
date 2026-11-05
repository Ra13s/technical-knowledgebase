# Compose shared search indexes from independently owned fragments

## What it is

Create one denormalized search document from **multiple service-owned source fragments** without giving one service ownership of every upstream field. The **index schema** declares field search behavior; each **fragment** declares its source, mappings, parent relationships and transformations. A shared platform derives batch/bootstrap, CDC, index updates, validation and mappings from the same definitions.

Uber Freight described a production Java implementation on October 27, 2026. Its private `@Index`, `@MappingField`, `@LinkedEntity` and related annotations illustrate the design, **not a public library that can be installed**.

## Use when

- search filters/sorts need fields from multiple microservices and different databases;
- joining those services at query time cannot meet interactive latency SLOs;
- teams must add searchable fields independently with explicit ownership and lineage;
- bulk bootstrap and streaming updates must produce the *same* document schema;
- a search-engine migration should not require reimplementing domain projection logic.

Avoid it for small monoliths where a direct SQL join and one owned schema already suffice: the platform and CDC consistency cost may dwarf any search benefit.

## Design contracts

~~~text
Service A -------- fragment A ------+
Service B -------- fragment B ------+-> logical document keyed by entity ID
Service C / CDC -- fragment C ------+   (field ownership + mapping manifest)
             |                          |
  bootstrap snapshot                     +-> index mapping generator
  and live event stream                   +-> denormalizer / fan-out
             |                            +-> OpenSearch (partial field writes)
             +-> source-to-index validator +-> drift/rebuild alerts
~~~

Specify and validate one owner for every indexed field. For example:

~~~yaml
index: shipment
id_field: shipmentId
fields:
  reference: { type: keyword, case_insensitive: true, owner: shipment }
  grossWeight: { type: double, owner: weight }
  customerName: { type: text, owner: customer }
fragments:
  shipment: { source: shipments_cdc, key: id }
  weight: { source: weights_cdc, key: shipment_id }
  customer:
    source: customers_cdc
    join: { shipment.customer_id: customer.id }
    fanout_to: shipment
~~~

This YAML is **proposed prototype configuration**, not Uber's actual DSL. Validate that two fragments cannot assign the same output field; require explicit field-removal/migration plans.

### One projection function in two pipelines

Treat the core operation as a versioned, deterministic transformation:

~~~java
record FragmentEvent(
    String entityId, String owner, long sourceVersion,
    Map<String, Object> ownedFields, boolean deleted) {}

Map<String, Object> project(
    Map<String, Object> currentDocument,
    FragmentEvent event) {
    // 1. Validate owner -> allowed field paths and input schema.
    // 2. Ignore an event older than last version for that owner+entity.
    // 3. Apply only fields owned by that fragment; remove on tombstone.
    // 4. Return next document plus per-owner version metadata.
    throw new UnsupportedOperationException("prototype sketch");
}
~~~

- **Bootstrap:** parallel snapshots through JDBC/warehouse/RPC connectors; create fragments and fold by entity ID. Record source offsets/watermarks with the snapshot.
- **Live:** consume CDC from the corresponding offset/watermark and run the same fragment transformation; upsert only owned fields when safe rather than rebuilding everything.
- **Many-to-one:** when a customer changes, find all shipments depending on it and fan out updates. Keep a reverse dependency index and enforce a fan-out budget/backpressure.
- **One-to-many:** collect child rows for a parent; compute an explicit, deterministic fold (sort and aggregate). On a child update, read remaining siblings and recompute the fold without a full source scan; handle deletes and out-of-order events.
- **Validation:** sample or continuously compare source rows to derived index state through the same connector contract. Track missing documents, stale versions and mismatched owned fields.

### Compatibility and migration

The source describes Spark bootstrap, Flink CDC and a SIA-to-OpenSearch dual-output transition. These are **implementation choices**, not required parts of the pattern. Prototype with PostgreSQL logical CDC + a small worker + OpenSearch before operating a multi-engine system.

For engine migration, generate mappings for old/new search backends from one business declaration, dual-write from the same fragments, replay a sampled query corpus, compare result sets/ranking and switch reads gradually. Roll back reads on divergence; reconcile snapshot/CDC gaps before declaring parity.

## Evidence, failure modes and trade-offs

Uber Freight reports 40+ microservices, roughly 100 sources/tables, more than 70 topics, 300 fragments across 20+ indices, thousands of complex QPS, and few-second freshness with hundreds-of-milliseconds p99 latency. These are **first-party production results**, not isolated proof that fragment composition caused the performance.

- Ownership collisions and schema drift can silently overwrite another service's fields.
- Bootstrap-to-stream cutover can lose or duplicate changes unless offset/watermark semantics are explicit.
- CDC out of order, duplicate events and tombstones need per-source versioning and deterministic replay.
- Linked-entity fan-out can turn one small update into an enormous write burst; bound it.
- Denormalization trades cheap reads for update cost, storage and eventual-consistency lag.
- Search analyzers (exact, prefix, substring, geo) alter index size and performance. Treat them as declared product semantics and test concrete queries.
- A dual-write migration requires comparing index *content* and query behavior, not merely successful writes.

## Prototype experiment

Create a `shipment` index from three PostgreSQL tables (shipment, customer, stops). Bootstrap 10,000 shipments; stream updates through one CDC adapter; use an explicit reverse-link table and 1:N stop fold. Inject duplicate/out-of-order events, a customer name update affecting 2,000 shipments, a deleted stop, and a restart at the bootstrap/stream boundary. Assert no cross-owner overwrites, eventual convergence to authoritative SQL, <configured> p99 query latency, and a measured fan-out queue. Then dual-write the same fragments to a second index and compare results before swapping read aliases.

## Sources

- Uber Engineering, *Freight Search: Composing a Shared Search Index from Independently Owned Fragments* (2026-10-27): https://www.uber.com/gp/en/blog/high-performance-search-platform/

## Related

- [Durable document ingestion](durable-document-ingestion-pipeline.md)
- [Time-window pushdown before aggregation](time-window-before-aggregation.md)
- [Per-key write serialization](per-key-write-serialization-object-store-replicas.md)
