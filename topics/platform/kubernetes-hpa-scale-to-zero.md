# Scale queue-driven Kubernetes workloads to zero with HPA 1.37

## What it is

Kubernetes 1.37 promotes HorizontalPodAutoscaler scale-to-zero to **Beta** and enables it by default. An HPA can use an Object or External metric to reduce a workload to zero replicas and later wake it when that external signal changes.

The wake-up signal must exist independently of running Pods.

## Use when

Use this for queue consumers, asynchronous workers, batch processors, or expensive CPU/GPU workers whose pending work survives while no Pod is running.

Do not treat this as a generic HTTP scale-to-zero feature. A Kubernetes Service does not buffer requests while no Pods are ready.

## Preconditions

Before setting `minReplicas: 0`:

1. use Kubernetes 1.37+;
2. configure at least one Object or External metric;
3. prove that metric remains observable while the target has zero Pods;
4. make sure pending work survives cold start;
5. start the target with at least one replica so the HPA can own the scale-down.

CPU and memory Resource metrics alone cannot wake a zero-replica workload because those signals come from running Pods.

## Example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: queue-worker
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-worker
  minReplicas: 0
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: queue_consumer_lag
          selector:
            matchLabels:
              name: worker_tasks
        target:
          type: Value
          value: "30"
```

When the external metric reaches zero, the HPA can reduce the Deployment to zero. When the metric rises again, the HPA can calculate a new replica count.

## Understand ownership of zero

A zero replica count can mean either an HPA-controlled idle state or an operator-controlled pause.

Kubernetes 1.37 uses the HPA `ScaledToZero` condition to distinguish those states.

```bash
kubectl describe hpa queue-worker
```

When the HPA owns the zero state, expect `ScaledToZero=True`.

Do not manually set the Deployment to zero expecting the HPA to wake it; manual zero remains a pause.

## Failure handling

If the configured external metric cannot be read, the HPA reports `ScalingActive=False` with a reason such as `FailedGetExternalMetric`.

Treat the external-metrics adapter as part of the workload's availability path. At zero replicas, it is the component that provides the wake signal.

Measure end-to-end wake latency:

```text
external metric changes
  -> HPA observes it
  -> Pod is scheduled
  -> application becomes ready
  -> queued work begins processing
```

If this latency is unacceptable, keep `minReplicas: 1` or change the buffering/activation design.

## Upgrade / rollback rule

Both the API server and controller manager participate in this feature.

During an upgrade, wait until both support scale-to-zero before creating HPAs with `minReplicas: 0`.

Before disabling the feature or downgrading:

1. change affected HPAs to `minReplicas: 1` or higher;
2. return workloads currently at zero to at least one replica;
3. then proceed with the rollback.

## Why it is useful

For queue-backed workloads, one permanently idle Pod can be pure reservation cost. Core HPA scale-to-zero removes that baseline without requiring a separate autoscaling product.

The decision rule is:

> Use HPA scale-to-zero only when both the wake signal and pending work survive the absence of Pods.

## Caveats

- Kubernetes 1.37 status is Beta, not GA.
- A broken external-metrics path can leave the workload at zero.
- HTTP requests are not buffered by a Kubernetes Service.
- CPU/memory-only HPAs cannot use `minReplicas: 0`.
- Cold-start cost may outweigh savings for latency-sensitive workloads.

## Prototype experiment

For one non-critical queue consumer:

1. expose queue depth or lag as an External metric;
2. configure the HPA with `minReplicas: 0`;
3. let the HPA scale to zero;
4. enqueue a canary job;
5. measure time from metric change to first processing;
6. compare idle resource-hours and wake latency against the previous `minReplicas: 1` baseline.

Promote only if the savings justify the cold-start and metrics-pipeline dependency.

## Sources

- Kubernetes, *Kubernetes v1.37: Scale Workloads to Zero with HorizontalPodAutoscaler* (2026-09-02): https://kubernetes.io/blog/2026/09/02/kubernetes-v1-37-hpa-scale-to-zero-beta/
- Kubernetes v1.37 release: https://kubernetes.io/blog/2026/08/26/kubernetes-v1-37-release/

## Related

- [Protect shared agent platforms with workload classes and bounded recovery](../ai/agent-platform-capacity-isolation.md)
