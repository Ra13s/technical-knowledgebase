# Opt into Kubernetes 1.37 Memory QoS deliberately

## What it is

Kubernetes 1.37 promotes Memory QoS to Beta on Linux nodes using cgroup v2. The feature gate is enabled by default, but the default kubelet configuration does **not** apply memory throttling or tiered reservation.

This makes upgrade behavior intentionally conservative: no memory.high, memory.min or memory.low values are written unless operators configure the corresponding fields.

## Use when

Evaluate Memory QoS when a cluster has memory-sensitive workloads and you want cgroup v2 to provide softer pressure controls before an OOM event, or to protect higher-QoS workloads during node pressure.

Do not assume that upgrading to 1.37 activates new memory behavior by itself.

## Configuration

Enable throttling for Burstable and BestEffort containers:

    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    memoryThrottlingFactor: 0.9

Enable tiered reservation as well:

    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    memoryThrottlingFactor: 0.9
    memoryReservationPolicy: TieredReservation

Or enable tiered reservation without throttling:

    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    memoryReservationPolicy: TieredReservation

In 1.37, memoryThrottlingFactor defaults to null and memoryReservationPolicy defaults to None.

## Rollout rule

Treat these as node-level scheduling/runtime policy changes, not harmless feature flags.

1. Canary a small node pool.
2. Record memory pressure, throttling, OOMs, latency and reclaim behavior before the change.
3. Enable one behavior at a time.
4. Compare Guaranteed, Burstable and BestEffort workloads separately.
5. Expand only after observing the expected cgroup values and workload effects.

## Caveats

- Requires Linux with cgroup v2.
- Memory QoS is Beta in Kubernetes 1.37.
- Tiered reservation is node-wide rather than a per-Pod opt-in.
- Stronger reservation can keep memory unavailable to neighboring workloads, including page cache.
- Existing kubelet configurations with an explicit throttling factor keep that value during upgrade.
- Disabling the feature again requires removing or adjusting incompatible Memory QoS fields.

## Prototype experiment

On one non-critical 1.37 node pool, run a Guaranteed service beside Burstable and BestEffort memory consumers. Compare default behavior, throttling only, and tiered reservation. Verify cgroup values and measure application latency, OOM behavior and node reclaim before deciding whether to expand.

## Sources

- Kubernetes, Kubernetes v1.37: Memory QoS Graduates to Beta (2026-09-14): https://kubernetes.io/blog/2026/09/14/kubernetes-v1-37-memory-qos-graduates-to-beta/

## Related

- [Scale queue-driven workloads to zero with Kubernetes HPA 1.37](kubernetes-hpa-scale-to-zero.md)
