# Durable document ingestion with parser fallbacks and small workflow histories

## What it is

Turn file ingestion for search/RAG into observable per-file durable workflows, not an opaque synchronous job that restarts every stage after a failure. StackAI's October 2026 Temporal case study gives a concrete production design: staged download, extraction, chunking and indexing; child-workflow parser fallbacks; async third-party callbacks; content-addressed caching; workload-separated queues; and per-file visibility.

## Use when

A connected document corpus contains mixed file types and large files, paid extraction providers or rate-limited APIs; an error in a later indexing stage should not re-download or re-parse a successful earlier result.

## Workflow outline

~~~text
discover source item
  -> record source version/etag + tenant + file ID
  -> download (I/O task queue)
  -> parse by ordered parser chain (memory-heavy task queue)
     -> success: cache content hash + parser configuration
     -> temporary failure: bounded retry with backoff
     -> permanent failure: try next compatible parser / fail visibly
  -> chunk + normalize
  -> index upsert keyed by document version / chunk ID
  -> verify searchable state and publish success
~~~

Each file has its own durable state and a stable idempotency key such as `tenant/source/file/version`. Track the index generation so a superseded file cannot publish stale chunks over newer content. Chunk IDs should be deterministic for a known document version; purge obsolete chunks when a new generation becomes authoritative.

## Concrete controls

1. **Stage-granular retry.** Classify HTTP 429/timeout/network errors as transient and back off; corrupt/unsupported documents and non-retryable API validation failures should not waste retry budgets.
2. **Parser chain.** Choose candidate extractors based on content type, file size and features; put each parser behind an isolated child workflow. Try the next option on timeout or a known permanent failure.
3. **Cache expensive results.** Use a key covering document content hash plus parser name, model/version and extraction settings. A hash of bytes alone is unsafe when extraction settings change.
4. **Asynchronous external parsers.** Submit a job with a correlation ID, then wait durably with a deadline (not a sleeping worker). Validate callbacks, signal the matching workflow with a *small status*, and fetch the document payload from object storage in a subsequent activity. Timeout should trigger the configured fallback or visible error.
5. **Keep history small.** Large PDFs and extracted text live in object/blob storage; workflow records carry references/checksums, not entire document payloads.
6. **Isolate resource classes.** Download workers mainly wait on I/O; PDF/OCR workers can be memory-heavy. Give them separate task queues and scaling plans.
7. **Observability and fairness.** Tag workflows by tenant, knowledge base and file ID. Bound intake bursts so a large customer's import cannot starve existing in-flight work.

## Failure handling

~~~text
file status: DISCOVERED -> DOWNLOADED -> PARSED -> CHUNKED
             -> INDEXED -> VERIFIED
on transient failure: retry current stage within bound
on permanent parser failure: next compatible extractor
on callback loss: durable timeout and fallback
on document superseded: cancel/mark obsolete generation
on repeated fatal failure: dead-letter + explicit error reason
~~~

Do not treat Workflow success as proof of an accurate search index: verify document counts, chunk coverage and representative retrieval. Make index writes idempotent. Callback deduplication and authentication belong to the inbound endpoint, not the model.

## Evidence and caveats

In a first-party interview, StackAI reports ingestion failure rates moving from about 20% to under 0.1%, and parsing costs falling to about one tenth. The migration was incremental from Celery and simultaneously changed parsing, caching and retry behavior; those numbers **cannot** be attributed to Temporal alone or assumed transferable.

This design is portable to other durable workflow engines or a carefully implemented task/outbox system. Temporal adds useful execution history, timers, child workflows and recoverability but also operational complexity.

## Prototype experiment

Build three tiny fake sources (plain text, PDF-like blob, oversized file) and two parsers (one occasionally timing out). Run a 200-document batch, kill workers at every stage, send duplicate/late callbacks, rate-limit the source and publish two versions of one file. Assert no unnecessary repeat of paid parsing, no duplicate chunks, bounded retries and the newest version indexed. Track P95 ingestion lag and cost/document.

## Sources

- https://temporal.io/blog/how-stackai-used-temporal-cut-indexing-failures-from-20-percent-to-below-1-percent

## Related

- [Durable agent tool effects](../ai/durable-agent-tool-effects-and-fencing.md)
- [Model-call record/replay testing](../testing/ai-model-vcr-record-replay-tests.md)
