# Migrate Kubernetes extended resources to DRA without changing workloads

## What it is

Kubernetes 1.37 makes **DRA-backed extended resources** stable. A `DeviceClass` can declare an `extendedResourceName` such as `example.com/gpu`, and existing Pods can keep requesting that familiar extended resource through `resources.requests` while Kubernetes allocates the device through Dynamic Resource Allocation (DRA).

This provides a migration bridge from the legacy device-plugin consumption model to DRA without forcing every workload to adopt `ResourceClaim` objects immediately.

## Use when

Use this when a cluster already has workloads that request accelerators or specialized devices as extended resources and you want to move device allocation to DRA incrementally.

Examples:

- GPU fleets currently exposed as `vendor.example/gpu`;
- network/accelerator devices moving to a DRA driver;
- mixed clusters where only some nodes have migrated;
- platform teams that want DRA policy and device lifecycle while preserving existing application manifests.

## DeviceClass bridge

Create a `DeviceClass` whose `extendedResourceName` matches the resource name existing workloads already request.

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu.example.com
spec:
  selectors:
    - cel:
        expression: >-
          device.driver == 'gpu.example.com' &&
          device.attributes['gpu.example.com'].type == 'gpu'
  extendedResourceName: example.com/gpu
```

The application manifest can remain unchanged:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  containers:
    - name: app
      image: example/app:1
      resources:
        limits:
          example.com/gpu: 1
        requests:
          example.com/gpu: 1
```

The workload does not need an explicit `ResourceClaim` just to consume the DRA-backed resource.

## Migration shape

```text
existing Pod
resources.requests[example.com/gpu]
              |
              v
        Kubernetes scheduler
              |
       +------+------+
       |             |
legacy device     DRA DeviceClass
plugin nodes      / DRA driver nodes
       |             |
       +------ device allocation
```

Kubernetes allows the same extended-resource name to be backed by a device plugin on some nodes and by DRA on other nodes. That enables node-by-node migration rather than a flag day.

## Migration recipe

### 1. Inventory the current contract

Record:

- extended-resource names used by workloads;
- node pools that advertise them;
- driver/runtime dependencies;
- scheduling constraints, topology and taints;
- workloads that depend on device-plugin-specific behavior.

The resource name is part of the application contract. Preserve it during the first migration step.

### 2. Install and validate the DRA driver on a bounded node pool

Do not remove the old provider first. Bring up a small DRA-backed pool and verify that the driver exposes the expected device attributes and lifecycle.

### 3. Map the existing resource name through `DeviceClass`

Set `spec.extendedResourceName` to the same name applications already request.

Keep selectors narrow enough that the class only matches devices equivalent to the legacy resource contract.

### 4. Run unchanged workloads against migrated nodes

Verify:

```bash
kubectl get deviceclasses
kubectl describe pod <pod>
kubectl get resourceclaims
```

Check that Pods requesting the existing extended resource schedule successfully and receive the intended device.

### 5. Migrate nodes incrementally

Move node pools from the legacy device-plugin backing to the DRA driver while keeping the workload resource name stable.

Avoid unintentionally advertising the same logical capacity twice on one node. Treat each node's source of truth for that resource as either the legacy provider or the DRA path unless the specific driver documents a safe coexistence model.

### 6. Adopt native DRA APIs only where they add value

Once allocation is stable, selected workloads can move to explicit DRA `ResourceClaim` / claim-template APIs when they need richer device selection, sharing or lifecycle semantics.

Do not force that application migration merely to get the operational benefits of DRA.

## Shortcut for a DeviceClass without a custom extended-resource name

Kubernetes also supports the special resource-name prefix:

```text
deviceclass.resource.kubernetes.io/<device-class-name>
```

A Pod can request that extended resource and Kubernetes creates the corresponding device allocation without the workload explicitly defining a `ResourceClaim`.

Use a stable organization-specific extended resource name when preserving an existing application contract; use the `deviceclass.resource.kubernetes.io/...` form when direct DeviceClass naming is acceptable.

## Why it is useful

DRA gives platform teams a richer device-allocation model, but application teams may have years of manifests, charts and operators built around classic extended-resource requests.

The 1.37 bridge lets the **backend allocation mechanism change before the workload API contract changes**. That reduces migration blast radius and supports mixed old/new node pools during rollout.

## Caveats

- DRA-backed extended-resource support is stable starting with Kubernetes **1.37**.
- The exact driver capabilities and installation model remain vendor-specific.
- Preserve workload semantics, not only the resource name: topology, sharing and device initialization can still differ.
- Test scheduler behavior on mixed node pools before broad rollout.
- Resource claims created implicitly still consume DRA machinery; observe allocation failures and cleanup.
- Advanced DRA features can require explicit claims or other APIs; this bridge is deliberately the compatibility path.

## Prototype experiment

For one non-production accelerator pool:

1. choose an existing extended resource such as `example.com/gpu`;
2. deploy the equivalent DRA driver to a small node pool;
3. create a `DeviceClass` with `extendedResourceName: example.com/gpu`;
4. schedule the existing unmodified workload manifest onto both legacy and DRA-backed pools;
5. compare allocation, teardown, topology, observability and failure behavior;
6. migrate one node pool only after parity is demonstrated;
7. document which workloads actually need native `ResourceClaim` features later.

## Sources

- Kubernetes, *Kubernetes v1.37: DRA Updates* (2026-09-03): https://kubernetes.io/blog/2026/09/03/kubernetes-v1-37-dra-updates/
- Kubernetes, *DRA API Objects — Extended resource allocation by DRA*: https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/dra-api/
- Kubernetes, *Assign Extended Resources to a Container*: https://kubernetes.io/docs/tasks/configure-pod-container/extended-resource/

## Related

- [Rootless Kubernetes nodes for agent and test sandboxes](rootless-kubernetes-agent-sandbox.md)
