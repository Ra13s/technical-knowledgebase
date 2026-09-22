# Migrate stored Kubernetes API objects with StorageVersionMigration

## What it is

Kubernetes 1.37 makes the built-in `storagemigration.k8s.io/v1` `StorageVersionMigration` API and controller stable and enabled by default. It declaratively rewrites existing API objects through the API server so they are stored using the resource's current preferred storage version.

This replaces ad-hoc `kubectl get | replace` rewrite scripts or an extra out-of-tree storage-version-migrator for a common upgrade task.

## Use when

Use `StorageVersionMigration` after changing the preferred storage version of an API when old objects may still be serialized in an older version, especially:

- promoting a CRD from an alpha/beta storage version to a stable one;
- preparing to remove an old CRD version from `.status.storedVersions` / serving support;
- rewriting objects after enabling encryption at rest;
- rewriting objects after rotating Kubernetes encryption keys/configuration so existing data uses the current encryption settings.

It is not a schema-conversion strategy by itself: conversion/defaulting/webhook behavior for the API must already be correct before rewriting the stored objects.

## Basic recipe

First update the CRD/API so the desired version is the current storage version. Then create a migration for the resource:

```yaml
apiVersion: storagemigration.k8s.io/v1
kind: StorageVersionMigration
metadata:
  name: crontabs-migration
spec:
  resource:
    group: example.com
    resource: crontabs
```

Apply it:

```bash
kubectl apply -f crontabs-migration.yaml
```

The built-in controller rewrites existing objects using the API server's current default storage version for that resource.

## Verify completion

Treat migration status as the gate, not successful creation of the object:

```bash
kubectl get \
  storageversionmigration.storagemigration.k8s.io/crontabs-migration \
  -o yaml
```

Wait for a successful condition:

```yaml
status:
  conditions:
    - type: Succeeded
      status: "True"
```

For CRDs, also verify that the CRD's stored-version state now contains only versions you intend to keep:

```bash
kubectl get crd crontabs.example.com \
  -o jsonpath='{.status.storedVersions}'
```

Do **not** remove an old served/storage version merely because new writes use the new version. Existing objects can remain stored under the old representation until they are rewritten.

## Upgrade workflow

A safe CRD-version retirement looks like:

```text
1. add / validate new API version + conversion behavior
2. make new version the storage version
3. deploy the updated CRD
4. create StorageVersionMigration
5. wait for Succeeded
6. verify CRD .status.storedVersions
7. if CRD changed during migration, rerun migration
8. only then remove obsolete stored/served version support
```

Kubernetes documents that if `.status.storedVersions` is not updated after a successful migration because the CRD itself changed during the run, the migration should be retried before deprecating the older version.

## Encryption-key rotation workflow

Changing encryption-at-rest configuration does not retroactively rewrite existing etcd values. Use a storage migration to force existing objects back through the API server after the new encryption configuration is active.

```text
install/rotate encryption config
       |
       v
validate API server configuration
       |
       v
StorageVersionMigration for affected resource(s)
       |
       v
wait for Succeeded + verify
```

This is especially useful for resource classes containing secrets/sensitive configuration, but plan the scope and load rather than blindly rewriting every API at once.

## Bundle migration with CRD delivery when appropriate

Because migration is a declarative Kubernetes resource, CRD authors/operators can ship the updated CRD and a corresponding `StorageVersionMigration` in the same release/manifests while still treating migration success as a separate rollout gate.

Example structure:

```yaml
# updated CRD with v1 storage: true
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
...
---
apiVersion: storagemigration.k8s.io/v1
kind: StorageVersionMigration
metadata:
  name: myresource-v1-storage
spec:
  resource:
    group: example.com
    resource: myresources
```

## Why it is useful

Storage representation is hidden state. Merely changing an API/CRD definition does not mean old objects have been rewritten. `StorageVersionMigration` turns that invisible maintenance step into a standard Kubernetes object with observable completion state.

This is safer and easier to automate than scripting read/replace loops, and it gives release tooling an explicit gate before removing old API versions or assuming an encryption rotation is complete.

## Caveats

- Requires Kubernetes 1.37+ for the stable `storagemigration.k8s.io/v1` API enabled by default.
- Migration rewrites objects; test conversion/defaulting/webhook behavior before running at scale.
- Large migrations generate API-server/storage load. Roll out and observe rather than launching many high-volume migrations simultaneously.
- A successful migration can need to be rerun if the CRD changed during the operation and `.status.storedVersions` does not reflect the desired final state.
- Do not equate migration success with application compatibility; clients still need to support whatever served API versions remain.
- For encryption rotation, verify the current API-server encryption configuration separately; the migration does not prove the encryption policy itself is correct.

## Prototype experiment

In a disposable Kubernetes 1.37 cluster:

1. create a CRD with an older storage version and several objects;
2. introduce the new storage version and conversion path;
3. create a `StorageVersionMigration`;
4. watch conditions until `Succeeded=True`;
5. verify `.status.storedVersions` and application reads/writes;
6. repeat while modifying the CRD during migration to validate the retry rule;
7. run the same workflow after an encryption-at-rest key rotation and observe API-server/storage load.

## Sources

- Kubernetes, *Kubernetes v1.37: Storage Version Migration Enabled by Default* (2026-08-31): https://kubernetes.io/blog/2026/08/31/kubernetes-v1-37-storage-version-migration-ga/

## Related

- [Migrate Kubernetes extended resources to DRA without changing workloads](kubernetes-dra-extended-resource-migration.md)
