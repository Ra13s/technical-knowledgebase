# Rootless Kubernetes nodes for agent and test sandboxes

## What it is

Kubernetes 1.37 promotes `KubeletInUserNamespace` to beta, allowing node components such as kubelet, CRI/OCI runtimes, CNI plugins and kube-proxy to run as a non-root host user inside a Linux user namespace.

This is different from pod user namespaces (`hostUsers: false`): pod user namespaces isolate workloads while node components still run as host root. The two features can be combined.

## Use when

Use this for local or nested clusters where granting host-root privileges is unnecessary or undesirable, especially:

- coding-agent sandboxes that need a real Kubernetes API;
- integration tests that create or mutate cluster resources;
- Kubernetes-in-Kubernetes CI environments;
- temporary bootstrap/dev clusters on shared hosts.

Do not treat beta rootless mode as a drop-in production-node migration without validating networking/storage drivers.

## Quick local recipe

With rootless Docker and kind:

```bash
dockerd-rootless-setuptool.sh install
kind create cluster
```

With minikube:

```bash
dockerd-rootless-setuptool.sh install
minikube start --driver=docker
```

The user namespace is created outside Kubernetes; the Kubernetes feature gate does not create it automatically.

## Verify the boundary

Kubernetes 1.37 exposes node status through `runningInUserNamespace`:

```bash
kubectl get nodes -o yaml
```

Use that state to label/taint rootless nodes and keep workloads that require real host-root privileges away from them.

Example operational rule:

```text
if node.runningInUserNamespace == true:
    schedule normal app/test workloads
    avoid root-dependent CNI/CSI installers unless validated
```

## Agent sandbox shape

```text
coding agent
   |
   v
non-root host account
   |
Linux user namespace
   |
rootless Docker/Podman/nerdctl
   |
kind/minikube Kubernetes node
   |
agent-created test workloads
```

This reduces the blast radius if the agent runs a destructive cluster command or is manipulated by untrusted repository/web content: namespace-root is not host-root.

For stronger nesting, Kubernetes documents combining rootless node components with user-namespaced pods (`hostUsers: false`).

## Why it is useful

A Kubernetes test environment often forces an uncomfortable choice between weak mocks and a highly privileged local cluster. Rootless node components add a middle option: run realistic cluster behavior while keeping the node stack outside host-root.

This complements agent-level filesystem/network sandboxes rather than replacing them.

## Caveats

- `KubeletInUserNamespace` is beta in Kubernetes 1.37, not GA.
- Enabling the feature gate alone does not make an existing node rootless.
- Some CNI and CSI drivers can be incompatible with user namespaces; test the exact stack.
- Host configuration may still need systemd, kernel-module or sysctl changes.
- A rootless cluster does not prevent destructive actions *inside* that cluster. Use disposable clusters, scoped kubeconfigs and resource quotas where appropriate.
- Cloud-managed Kubernetes services may not expose this node configuration even when upstream Kubernetes supports it.

## Prototype experiment

For a coding-agent integration-test workflow:

1. create a disposable rootless kind cluster;
2. verify `runningInUserNamespace`;
3. run the existing Kubernetes E2E test suite;
4. attempt representative Helm/operator/network/storage workflows;
5. record incompatible CNI/CSI or privileged workloads;
6. compare host blast radius and setup time with the current privileged test cluster.

Adopt it for agent execution only if the required workload set passes without silently adding host privileges back.

## Sources

- https://kubernetes.io/blog/2026/09/04/kubernetes-v1-37-rootless-beta/
- https://kubernetes.io/docs/tasks/administer-cluster/kubelet-in-userns/

## Related

- [Contain coding agents at the environment boundary first](../ai/environment-first-agent-containment.md)
