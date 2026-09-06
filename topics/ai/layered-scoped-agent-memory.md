# Layered scoped agent memory with stateless reasoning sessions

## What it is

An agent-memory pattern that keeps durable knowledge outside the model context, separates recent observations from curated long-term summaries, retrieves only relevant memory on demand, and starts each reasoning turn from fresh live task state.

Coinbase's CEEcil support teammate uses this shape: opt-in scoped observations, nightly consolidation into durable summaries, Git-versioned knowledge, on-demand memory tools, and stateless reasoning sessions that reread the live Slack thread before acting.

## Use when

Use this when an agent needs continuity across days, people, incidents, repositories or operational shifts, for example:

- support/on-call assistants;
- coding agents that must remember repository conventions or prior decisions;
- incident assistants that need recent operational history;
- team agents that answer recurring questions from evolving institutional knowledge;
- long-running workflows where model context cannot be the durable state store.

## Architecture

```text
scoped event sources
  -> recent observations
  -> scheduled consolidation
  -> durable summaries / knowledge
             |
live task ---+----> memory/search tools
             |            |
             +------------v
                    fresh reasoning session
                             |
                    deterministic service
                  publish / audit / workers /
                  dedupe / limits / kill switch
```

The model receives the current task plus memory selected for that task, not a dump of everything the system has ever seen.

## Implementation rules

### 1. Scope observation by construction

Define which channels, repositories, services or data sources are observable before collection starts. Prefer explicit opt-in scopes and deny private/sensitive sources rather than relying on the model to remember what it should not read.

Example policy:

```yaml
memory_sources:
  slack:
    allowed_channels:
      - incident-payments
      - support-internal
    direct_messages: deny
    private_channels: deny
  github:
    repositories:
      - payments-service
```

Do not retroactively ingest unapproved history merely because the agent later gains access.

### 2. Separate recent observations from durable knowledge

Raw or near-raw observations should have short retention and narrow scope. Consolidate recurring facts, decisions and useful context into a smaller durable layer.

```text
raw observations -> short retention
        |
        v
scheduled consolidation
        |
        v
curated summaries / reusable knowledge
```

This prevents every transient conversation detail from becoming permanent memory.

### 3. Retrieve memory on demand

Expose narrow retrieval tools such as:

```text
search_team_knowledge(query)
get_recent_entity_summary(entity)
get_decision_history(topic)
```

Let the agent request relevant memory when needed. Avoid prepending the full memory corpus to every prompt: that increases cost, hides important evidence in noise and makes stale context harder to detect.

### 4. Keep reasoning sessions disposable

At invocation time, reread the authoritative live thread/ticket/repository state and reconstruct the working context. Persist durable knowledge and workflow state externally rather than depending on a long conversational transcript.

```text
new invocation
  -> read current source-of-truth state
  -> retrieve relevant durable memory
  -> reason / act
  -> write durable state explicitly if needed
```

This makes retries, model swaps and failover easier because continuity is not trapped inside one model session.

### 5. Keep operational behavior outside memory/reasoning

Background schedules, publishing, deduplication, retries, audit, spend limits and kill switches belong in a deterministic service layer. The model can decide what a response should say; it should not be the only system responsible for remembering whether a scheduled job already ran or whether a message was published.

### 6. Prefer human-readable, versionable durable knowledge

For a bounded corpus, Git-versioned Markdown or a structured text bundle is often enough. Keep provenance and freshness visible.

Example front matter:

```yaml
---
topic: payments-rollout
updated: 2026-09-06
sources:
  - incident-1234
  - adr-0091
owners:
  - payments-platform
---
```

Google's Open Knowledge Format is one example of a Git-friendly packaging format for human/agent knowledge. The principle matters more than the specific format.

Do not introduce a vector database merely because the system has memory. Start with the simplest retrieval that meets corpus size and recall needs, then add semantic retrieval when measured misses justify it.

### 7. Surface freshness and provenance

Memory is evidence with an expiration problem. Retrieval results should expose where a fact came from and when it was last refreshed so the agent can prefer current live state over stale summaries.

```json
{
  "fact": "service X uses rollout policy Y",
  "source": "adr-0091",
  "updated_at": "2026-08-30",
  "confidence": "curated"
}
```

### 8. Tier cognition when the workflow permits it

Not every request needs a full agent loop. A cheap classifier/router can send deterministic lookups to direct API or knowledge paths and reserve expensive agent reasoning for open-ended multi-step work.

```text
request
  -> route
      -> known lookup: deterministic API/knowledge path
      -> open-ended task: full agent + tools + memory
```

Measure routing errors; do not trade correctness for token savings blindly.

## Why it is useful

The pattern distinguishes **memory** from **context**. Large context windows do not provide durable, scoped, auditable organizational memory, and long chat transcripts are poor databases.

Externalized memory also makes model upgrades safer: the model can be replaced while knowledge, provenance, workflow state and retention policy remain stable.

## Failure modes / caveats

- **Consolidation loss:** summaries can erase qualifiers or disagreement. Keep links/provenance back to source material.
- **Stale memory:** a durable summary can conflict with live system state. Authoritative live state wins.
- **Privacy creep:** broad collection creates a surveillance system very quickly; scope inputs before ingestion.
- **Memory poisoning:** untrusted content should not silently become curated durable knowledge.
- **Over-consolidation:** not every observation deserves promotion into long-term memory.
- **Retrieval ceiling:** simple substring/search approaches eventually fail as corpus size and vocabulary diversity grow; measure misses before changing architecture.

## Prototype experiment

Build a small opt-in memory pilot for one engineering workflow:

1. observe one explicitly allowed Slack/issue stream;
2. keep raw observations for a short retention period;
3. generate a daily entity/topic summary with source links;
4. store durable team knowledge in Git Markdown or OKF-like bundles;
5. expose `search_memory()` and `get_recent_summary()` tools;
6. start every agent invocation from the current live thread/task;
7. keep publishing, dedupe and scheduled consolidation in deterministic workers.

Measure repeated re-briefing time, stale-memory errors, retrieval misses, human corrections, latency and token cost.

## Sources

- https://www.coinbase.com/blog/ceecil-engineering-a-support-teammate-with-human-memory
- https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing/
- https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md

## Related

- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
- [Bind agent approvals to the exact side effect](enforcement-bound-agent-approvals.md)
