# MachineConfig

**API Group**: `machineconfiguration.openshift.io/v1`
**Kind**: `MachineConfig`
**Scope**: Cluster
**Short Name**: `mc`

**API Definition**: [types.go](https://github.com/openshift/api/blob/master/machineconfiguration/v1/types.go)

## Purpose

MachineConfig defines the desired OS-level configuration for a group of nodes. It is the fundamental unit of host configuration in the MCO, encoding Ignition configs, kernel arguments, extensions, OS image references, FIPS mode, and kernel type into a single declarative object.

**Key Principle**: MachineConfigs are additive and merged by the render controller into a single "rendered" config per pool. The role label (`machineconfiguration.openshift.io/role`) determines which pool(s) a MachineConfig targets.

## Spec Structure

```go
type MachineConfigSpec struct {
    OSImageURL                     string               // OS update payload container image (optional)
    BaseOSExtensionsContainerImage string               // Extensions container matching OS image (optional)
    Config                         runtime.RawExtension // Ignition Config object (optional)
    KernelArguments                []string             // Kernel arguments to append (optional, nullable)
    Extensions                     []string             // Host extensions to enable (optional)
    FIPS                           bool                 // Enable FIPS mode (optional)
    KernelType                     string               // "default", "realtime", or "64k-pages" (optional)
}
```

## Key Concepts

### Role-Based Selection

MachineConfigs are selected by MachineConfigPools via the `machineconfiguration.openshift.io/role` label:
- `role: master` → master pool
- `role: worker` → worker pool
- Custom roles target custom pools

### Rendered MachineConfigs

The render controller merges all matching MachineConfigs for a pool into a single rendered config named `rendered-<pool>-<hash>`. This rendered config is what the MCD actually applies to nodes.

### Ignition Config

The `Config` field holds an Ignition specification (currently spec 3.x internally converted to 3.5.0 via `ctrlcommon.InternalMCOIgnitionVersion`). This includes files, systemd units, users, and other OS primitives.

### Size Limit

MachineConfigs must not exceed 1.5MB (`MaxMachineConfigSize = 1572864` bytes) to stay within etcd's request size limit.

## Lifecycle

1. **Creation**: User or controller creates MC with role label → render controller detects, re-renders pool config
2. **Update**: Changes trigger re-rendering → new `rendered-<pool>-<hash>` → node controller coordinates rollout
3. **Deletion**: Removing an MC triggers re-rendering of the pool's rendered config

## Example: Adding a File

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-custom-config
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
        - path: /etc/my-custom-config
          mode: 0644
          contents:
            source: data:,my-config-content
```

## Common Mistakes

1. **Missing role label** — MC will not be selected by any pool and has no effect
2. **Exceeding 1.5MB** — Will be rejected; split large configs across multiple MCs
3. **Using wrong Ignition version** — MCO converts internally to 3.5.0; use 3.2.0+ for user-created MCs
4. **Conflicting files** — Multiple MCs writing the same file path causes render errors

## Related Concepts

- [MachineConfigPool](./machineconfigpool.md) — Groups nodes and selects MachineConfigs
- [MachineConfigNode](./machineconfignode.md) — Tracks per-node update progress
