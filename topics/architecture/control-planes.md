# Control planes

## Current understanding

A control plane manages desired state; a data plane performs the workload. Robust control planes continuously reconcile observed state toward desired state rather than assuming one-shot imperative operations succeed permanently.

## External evidence

Werner Vogels' discussion of scalable control planes emphasizes reconciliation and **static stability**: existing workloads should continue operating when the control plane is impaired. Control-plane availability should not be unnecessarily placed on the critical path of already-running data-plane work.

Kubernetes is a familiar expression of the same model: desired state is persisted, controllers observe divergence and repeatedly attempt convergence.

## Our position

Use a reconciliation model when managing long-lived distributed resources whose actual state can drift from intent.

Prefer:

`desired state -> observe -> diff -> reconcile -> repeat`

over workflows that depend on a single imperative request permanently establishing reality.

Design the data plane for **static stability** where feasible: loss of the control plane may prevent configuration changes or provisioning, but should not automatically stop healthy existing workloads.

## Confidence and limitations

**Confidence: high.** Reconciliation is established practice across large distributed systems.

Not every application needs a separate control plane. Introducing one adds state, lifecycle, consistency and operational complexity and should correspond to a real resource-management problem.

## Practical implications

When designing a platform or orchestration service, ask:

- What is the source of desired state?
- What is the source of observed state?
- Is reconciliation idempotent?
- What happens after a partially successful operation?
- Can reconciliation safely retry forever?
- What continues working if the control plane is unavailable for an hour?
- Does the data plane synchronously depend on the control plane for ordinary requests?

## Open questions

- Which internal application domains benefit from explicit reconciliation even when Kubernetes is not involved?
- How should control-plane intent and reconciliation history be exposed to coding/operations agents?

## Sources

- https://www.allthingsdistributed.com/2026/08/on-building-scalable-control-planes.html
