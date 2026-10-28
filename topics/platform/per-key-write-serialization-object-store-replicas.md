# Serialize shared-state writes by key and separate durable storage from read replicas

## What it is

When agents/workers write frequently to many **independent logical objects** inside one shared namespace, a namespace-wide write lock can become a queueing bottleneck even though the individual writes do not conflict. GitHub's October 21, 2026 account of its Git storage migration is a high-scale case study: it is moving from a repository-wide serialized quorum to **per-reference ordering**, while adding a durable object-storage quorum participant and independently scalable ephemeral read replicas.

The reusable system-design pattern is **coordinate only the smallest authoritative value that must be ordered; store immutable payloads separately; acknowledge writes only after durability; make stale reads explicit**. This is a design pattern, not a drop-in GitHub API.

## Use when

- a high-concurrency coding-agent fleet creates/checkpoints many unrelated branches in one repository;
- a multi-tenant workflow engine updates unrelated task keys but serializes all writes through one lock or global coordinator;
- immutable blobs/artifacts/checkpoints are written frequently and read many times;
- geographic replication makes the slowest healthy replica's latency dominate a broad write lock;
- you are designing a ref/index/metadata store that needs per-key monotonic read-after-write semantics.

## Workload and invariants

GitHub reports that its earlier Spokes quorum had **five replicas** for most repositories and serialized writes under a repository-wide lock, waiting for all healthy replicas up to a timeout even though quorum acknowledgement requires only a majority. A roughly **25 ms** cross-region round trip became part of every serialized write for the hottest repositories. Git's critical ordering property is per reference: once `refs/heads/agent-42` is updated and acknowledged, a client must not later observe an older value of **that same ref**.

A transfer to other systems should make invariants explicit:

- each key/reference has a totally ordered sequence of accepted updates;
- an acknowledged update is durable before success is returned;
- independent keys can be updated in parallel;
- a compare-and-swap or expected-version precondition prevents lost updates to the **same** key;
- reads requiring causality must observe at least the version acknowledged to their caller;
- multi-key transactions are a **separate requirement**, not automatically solved by independent per-key updates.

## Architecture sketch

```text
writers (many agents, workers)
  -> immutable payload writes (content hash, idempotent put)
  -> conditional ref/key update {key, expectedVersion, newObjectHash}
      -> serialize for key only (lock/CAS/quorum)
      -> durable quorum incl. object-store write participant
      -> acknowledge {newVersion, hash}
  -> cache/read replicas (scalable, ephemeral)
      -> background reconciliation for eventually fresh workloads
      -> version-aware reconciliation or authoritative read for strict reads
  -> read result {key, version, object}
```

The GitHub bridge design first moves quorums from repository-wide to **individual refs**. Concurrent pushes to different refs no longer occupy one write slot. It next introduces a virtual write replica that durably streams data to Azure Blob Storage and can coalesce disjoint updates into fewer object-store round trips. Read replicas may be short-lived and warm from the durable object store; **initial background reconciliation cannot serve every strict-consistency workload**.

## Concrete implementation sketch

A simple key-value service can start with one DB row per mutable key and an immutable content-addressed object store:

```sql
CREATE TABLE mutable_refs (
  namespace_id TEXT NOT NULL,
  ref_name TEXT NOT NULL,
  version BIGINT NOT NULL,
  object_hash TEXT NOT NULL,
  PRIMARY KEY(namespace_id, ref_name)
);
```

Conceptual update workflow:

```text
1. putImmutable(objectHash, payload)       # verify checksum; idempotent
2. begin transaction
3. UPDATE mutable_refs
     SET version = oldVersion + 1, object_hash = objectHash
     WHERE namespace_id = ns AND ref_name = key AND version = oldVersion
4. if 0 rows affected -> conflict, re-read that ref; do NOT overwrite
5. commit durable update
6. return newVersion and objectHash
```

The relational database provides authoritative per-row serialization; the immutable object must already be durable before the ref points to it. This **does not** exactly reproduce GitHub's multi-replica quorum; it implements a smaller experiment that tests the contention/consistency principle. A real replicated design must specify quorum protocol, leader failure, repair, object lifetime and atomic visibility.

Read path:

```text
read(key, minimumVersion?)
  -> if cache.version >= minimumVersion: return cached
  -> otherwise: refresh key/ref from authoritative store (or wait with timeout)
  -> read content-addressed object by hash; verify existence/integrity
  -> return object and version
```

For versionless analytical or feed-like reads, accept bounded staleness explicitly; for an agent continuing after a push, require a minimum version/expected head SHA. Avoid using stale cache reads to decide mergeability, permissions or whether an external tool effect happened.

## Evidence

GitHub reports **99.99999%** of its repository population already had copies in Azure during the migration, with the hottest exceptions blocked by the previous global write serialization. Its internal tests of per-reference serialization found approximately **3–4× higher write throughput** with lower latency under realistic concurrent branch writes. GitHub reports 100–1,000 reads per write in many repositories, motivating cheap ephemeral read replicas.

These are **GitHub-reported results**, not independent performance measurements or a guarantee that a row-level CAS/lock will yield the same throughput. Benefit depends on the percentage of *independent keys*; if all workers update one `main` ref, it remains serialized. GitHub also explicitly notes that its virtual-read-replica reconciliation was not yet suitable for every read consistency requirement.

## Failure modes and trade-offs

- **Hot key remains hot:** a merge queue that updates `main` cannot escape same-key ordering; use batching, admission control or hierarchical queues.
- **Cross-key invariants:** per-key serialization can break constraints spanning two refs. Keep multi-key transactions under an explicit higher-level gate.
- **Stale reads:** object-store and read-cache replicas may trail the acknowledged ref. Enforce minimum versions on causally dependent reads.
- **Dangling objects:** object write before ref update can leave unreachable blobs after a conflict. Garbage collection must tolerate delayed readers and concurrent mutations.
- **CAS retries:** retries may observe a new head; recompute work against that head, do not blindly reuse a patch/checkpoint.
- **Failure fencing:** a stale worker must not install an old ref update; bind writes to expected version/epoch.
- **Latency versus durability:** a remote durable write participant can add per-write latency; throughput gains come from removing unnecessary *serialization*, not from eliminating network costs.
- **Operational scope:** GitHub's infrastructure is bespoke; this pattern is best viewed as an experiment/hotspot diagnosis for our own control plane.

## Prototype experiment

Implement a small Postgres + object-store-backed `mutable_refs` toy service, then load-test 100 concurrent agents each updating a separate ref versus 100 agents contending on one ref. Compare (A) a namespace-wide lock, (B) row-level CAS. Inject process death after blob put and before/after commit, stale-cache reads, and retries using old expected versions. Measure p50/p99 commit latency, throughput, CAS conflict rate, stale reads, garbage object accumulation and read-your-writes correctness. Repeat with 10% hot-key skew to see when benefits disappear.

## Sources

- GitHub Engineering (2026-10-21), *The bridge to agent-scale Git infrastructure*: https://github.blog/engineering/architecture-optimization/the-bridge-to-agent-scale-git-infrastructure/
- GitHub Engineering (2026-10-06), *Building Git infrastructure for agent-scale development*: https://github.blog/engineering/building-git-infrastructure-for-agent-scale-development/

## Related

- [Durable agent tool effects and fencing](../ai/durable-agent-tool-effects-and-fencing.md)
- [Idempotent side effects](../ai/idempotent-agent-tool-side-effects.md)
- [Evidence-gated coding-agent edits](../ai/evidence-gated-coding-agent-edits.md)
