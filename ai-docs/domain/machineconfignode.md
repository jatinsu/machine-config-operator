# MachineConfigNode

**API Group**: `machineconfiguration.openshift.io/v1`
**Kind**: `MachineConfigNode`
**Scope**: Cluster

**API Definition**: [types_machineconfignode.go](https://github.com/openshift/api/blob/master/machineconfiguration/v1/types_machineconfignode.go)

## Purpose

MachineConfigNode tracks the configuration state and update progress of individual nodes. Each MCN is named after its node (`metadata.name == spec.node.name`) and provides granular visibility into the MCD update lifecycle.

**Key Principle**: MCN is the per-node counterpart to MachineConfigPool. While MCP tracks pool-level progress, MCN tracks node-level phases: preparation, execution, post-actions, and completion.

## Spec Structure

```go
type MachineConfigNodeSpec struct {
    Node          MCOObjectReference                     // Reference to the node (name must match metadata.name)
    Pool          MCOObjectReference                     // Which MCP this node belongs to
    ConfigVersion MachineConfigNodeSpecMachineConfigVersion // Desired rendered config name
    ConfigImage   MachineConfigNodeSpecConfigImage       // Desired OS image (FeatureGate=ImageModeStatusReporting)
}
```

## Key Concepts

### Update State Machine

MCN conditions track the node through the update lifecycle:

```
UpdatePrepared → Cordoned → Drained → AppliedFilesAndOS → RebootedNode → Uncordoned → UpdateComplete → Updated
```

Additional conditions: `UpdateExecuted`, `UpdatePostActionComplete`, `Resumed`, `NodeDegraded`.

### Config Version Tracking

- `spec.configVersion.desired` — Set by node controller when new rendered config is available
- `status.configVersion.current` — Updated by MCD after successful application
- When `current != desired`, an update is in progress

### Image Mode (Feature-Gated)

When `FeatureGateImageModeStatusReporting` is enabled:
- `spec.configImage.desiredImage` — Target OS image digest
- `status.configImage.currentImage` / `desiredImage` — Observed image state
- Additional condition: `ImagePulledFromRegistry`

### Pinned Image Sets Status

`status.pinnedImageSets[]` tracks per-image-set progress with `currentGeneration`, `desiredGeneration`, and `lastFailedGeneration` fields.

### Irreconcilable Changes (Feature-Gated)

When `FeatureGateIrreconcilableMachineConfig` is enabled, `status.irreconcilableChanges[]` lists diffs between the node's configuration and the target that can only be applied to new nodes.

## Common Mistakes

1. **Editing MCN manually** — MCN is managed by the node controller and MCD; manual edits will be overwritten
2. **Assuming MCN name format** — MCN name must exactly match the node name; no prefixes or suffixes

## Related Concepts

- [MachineConfigPool](./machineconfigpool.md) — Pool-level config and rollout management
- [MachineConfig](./machineconfig.md) — The config being applied
