# Keep the Java latency benchmark driver outside the SUT JVM

## What it is

For Java service benchmarks where garbage-collection or safepoint pauses matter, run the request-scheduling component in a different JVM from the system under test.

If scheduling and the application share one JVM, a GC pause can stop both at once. The benchmark then fails to schedule some arrivals during the pause and can report tail latency that is too optimistic. Timestamp correction cannot recreate arrivals that were never scheduled.

## Use when

Apply this to p99 and p99.9 service-latency measurements, collector comparisons, heap-tuning experiments, and investigations of GC or safepoint pauses. In-process microbenchmarks such as JMH have a different purpose.

## Recommended topology

    benchmark driver JVM
            |
            v
          SUT JVM
       application + GC
       JFR / GC logs

Record intended send time, actual send time, response time, intended arrival rate, SUT GC/safepoint events, and the CPU/memory topology.

For latency-focused SPECjbb2015 work, prefer MultiJVM or Distributed modes. A 2026 experiment comparing Composite-Net and Distributed observed roughly 2-3x p99 differences for collectors with non-trivial pauses. The authors recommend the separated modes for latency analysis. The results are experimental and hardware-specific, not official SPECjbb scores or a general collector ranking.

## Verification recipe

1. Hold the workload and request schedule constant.
2. Put the request scheduler in a separate JVM.
3. Repeat enough runs to observe variance.
4. Capture GC/safepoint logs and JFR when useful.
5. Correlate pauses with intended and actual submissions.
6. Compare percentile distributions, not averages only.
7. Repeat with production-like CPU quotas and container limits when applicable.

## Caveats

- Separate JVMs on one host can still contend for CPU; reserve resources for sensitive comparisons.
- Localhost and remote-driver setups differ in network variability.
- Closed-loop clients change arrival behavior; choose a request model matching production.
- Warmup, JIT and profiling overhead must be controlled consistently.

## Prototype experiment

Run the same Spring Boot latency benchmark with request scheduling first colocated and then isolated in a second JVM. Compare scheduling gaps and p99/p99.9 around measurable GC pauses. Keep the isolated topology if the colocated setup hides tail latency.

## Sources

- Jonas Norlinder, Anil Rajput, Tobias Wrigstad, The Limitations of Running a Workload Generator In the Same JVM as the System-Under-Test (2026-09-24): https://norlinder.nu/posts/The-Limitations-of-Running-a-Workload-Generator-In-the-Same-JVM-as-the-System-Under-Test/
- Baeldung Java Weekly #666 (2026-10-02): https://www.baeldung.com/java-weekly-666
