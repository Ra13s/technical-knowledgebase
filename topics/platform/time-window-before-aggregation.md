# Bound time-series aggregation work before grouping or paging

## What it is

For a time-series API backed by an event-like relational table, **limit the indexed input time window before aggregation**, instead of grouping an entire entity history and only then paginating the groups. Also avoid exposing individual identifiers when clients actually need aggregate counts.

GitHub's October 2026 star-history redesign provides measured evidence: its former paginated stargazer API required ~2,511 requests to reconstruct the ~251,000-star `react/react` curve; naively grouping the entire history on every aggregate page would still scan ~6 million rows across 24 requests. Pagination by **indexed calendar-week range**, *then* daily grouping reduces the complete-history work to about one pass over the rows (~251,000), returning ~30 weeks per request.

## Use when

- adding dashboards for daily/weekly counts over events, audit records, changes or traffic;
- a `GROUP BY ... ORDER BY ... LIMIT/OFFSET` query still scans an entire large history;
- a public API returns person-level data only so clients can calculate counts;
- heavy aggregation or `COUNT(*)` repeated for pagination produces database timeouts.

## Query shape

Create an index matching equality filters and the bounded date predicate. Example PostgreSQL schema and query (illustrative; not GitHub's proprietary SQL):

~~~sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS event_entity_created_idx
  ON event_log (entity_id, created_at);

-- Define :start_utc and :end_utc from a calendar period
-- and prefer a half-open range. Keep both binds typed timestamptz.
SELECT date_trunc('day', created_at AT TIME ZONE 'UTC')::date AS day,
       count(*) AS event_count
FROM event_log
WHERE entity_id = :entity_id
  AND created_at >= :start_utc
  AND created_at <  :end_utc
GROUP BY day
ORDER BY day;
~~~

The key is that `created_at` is **unwrapped in the WHERE predicate**, allowing a range scan on `(entity_id, created_at)`. The bucketing function belongs only in the aggregate stage. In the API layer, clamp `weeks_per_page` and derive `[start,end)` from a stable anchor and week boundaries. A caller requesting page 2 must not force `OFFSET` through an all-history aggregate.

~~~text
API: {entity, page, weeks_per_page<=30}
  -> checked calendar window [start,end)
  -> indexed time-range scan
  -> group only matching rows into day/week buckets
  -> fill empty dates explicitly
  -> return aggregate counts, not individual identities
~~~

When the API needs a moving "most recent" page, use a pinned `as_of`/watermark to avoid duplicates and gaps as new rows arrive between requests. Document the timezone and meaning of "week" rather than assuming server-local time matches client expectations.

### Observable verification

Run both the naive all-history-grouping query and bounded version with:

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT date_trunc('day', created_at AT TIME ZONE 'UTC')::date, count(*)
FROM event_log
WHERE entity_id = :entity_id
  AND created_at >= :start_utc AND created_at < :end_utc
GROUP BY 1;
~~~

Compare *rows visited, buffers read, total execution time and p95 latency*, not response size. Test a high-volume week, sparse weeks, empty history, first/last page, DST/week boundaries and a concurrent write.

## When it does NOT preserve semantics

- GitHub's source table contains **currently existing stars**, not a full add/remove event history: a removed star disappears retroactively. Its resulting curve is **not** the number of historical star-add events.
- Fixed-duration windows still have unbounded density: a viral week can be expensive. Add request-cost budgets, maximum tenant rate, rollups/materialized aggregates or ingestion-side counters if the maximum window still scans too much.
- If an API needs arbitrary historical totals rather than interval counts, precomputed cumulative aggregates/checkpoints or an event log may be the right contract.
- A range index helps only if predicates are selective and the optimizer uses it. Inspect the actual plan; large append-only data may also benefit from partitioning or BRIN.
- Data minimization is not an excuse to hide required user-level data from authorized consumers; expose separate contracts for aggregate and identity use cases.

## Public GitHub API reference

For a current example, GitHub exposes `GET /repos/{owner}/{repo}/stargazers/history?per_page=30&page=1`, returning weekly totals and seven daily counts per week. The October 2026 docs specify a **maximum of 30 weeks per page**, **maximum page 100**, and week/day boundaries **not guaranteed to align with UTC**. Use this endpoint rather than enumerating `/stargazers` when you need the growth curve.

## Prototype experiment

Generate a PostgreSQL table with several million rows and a skewed high-volume entity. Benchmark full-history `GROUP BY + OFFSET`, window-first aggregation and a materialized rollup; measure scanned rows, memory, buffer hits, p95 and p99. Add a high-density spike and verify service-level request budgets. Repeat at a time boundary and after rows are deleted if the domain tracks current-state records rather than immutable events.

## Sources

- GitHub Engineering (2026-10-14), design and before/after: https://github.blog/engineering/architecture-optimization/from-2500-requests-to-24-rebuilding-star-history-without-the-stargazers/
- GitHub REST API, exact response and pagination: https://docs.github.com/en/rest/activity/starring#get-repository-star-history

## Related

- [DuckDB for set-based Spring Batch transforms](../spring/spring-batch-duckdb-transforms.md)
- [Agent fleet unit economics](../ai/agent-fleet-unit-economics.md)
