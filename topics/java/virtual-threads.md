# Java Virtual Threads

## Current understanding

Virtual Threads make thread-per-request blocking Java substantially cheaper, but they do not make downstream resources unlimited. JDK 24 removed the major `synchronized`-related carrier-thread pinning limitation, making Virtual Threads more practical for conventional server applications.

## External evidence

InfoQ's review of Virtual Threads after JDK 24 highlights JEP 491's removal of most monitor-related pinning. Remaining pinning cases include some native/JNI, class-loading and platform-specific I/O scenarios.

The article also demonstrates a less obvious cost: `ThreadLocal` initialization that was amortized across a small reusable platform-thread pool can occur once per short-lived Virtual Thread. In its benchmark, initialization increased by orders of magnitude.

## Our position

For conventional blocking Java HTTP services, **evaluate Spring MVC + Virtual Threads before adopting reactive programming solely for scalability**.

Virtual Threads change where concurrency must be controlled. Explicitly bound scarce resources such as:

- database connections
- downstream request concurrency
- rate-limited APIs
- file descriptors
- memory-heavy operations

Cheap threads are not a substitute for backpressure or capacity management.

Use reactive approaches where their semantics are themselves useful — for example streaming, SSE/WebSockets or explicit reactive backpressure — rather than because blocking threads are assumed to be intrinsically too expensive.

Audit `ThreadLocal` usage before enabling Virtual Threads broadly. Request context should not depend on expensive per-thread initialization; Scoped Values are worth evaluating on modern JDKs where appropriate.

## Confidence and limitations

**Confidence: high** for the JDK behavior and resource-bounding principle; **medium** for framework choice because workload characteristics matter.

Virtual Threads do not remove all pinning and do not automatically improve CPU-bound workloads.

## Practical implications

A migration checklist should include:

1. identify blocking I/O paths;
2. inspect `ThreadLocal` usage;
3. identify every downstream concurrency limit;
4. load-test those downstream limits rather than only application thread count;
5. monitor carrier-thread pinning and resource-pool saturation.

## Open questions

- What production observability best exposes accidental unbounded concurrency after a Virtual Thread migration?
- Which common Java libraries still contain patterns that behave poorly with per-task threads?

## Sources

- https://www.infoq.com/articles/virtual-threads-after-jdk24/
