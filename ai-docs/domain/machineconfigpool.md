# MachineConfigPool

**API Group**: `machineconfiguration.openshift.io/v1`
**Kind**: `MachineConfigPool`
**Scope**: Cluster
**Short Name**: `mcp`

**API Definition**: [types.go](https://github.com/openshift/api/blob/master/machineconfiguration/v1/types.go)

## Purpose

MachineConfigPool groups nodes by label selector and defines which MachineConfigs apply to them. It tracks rollout progress and health of configuration updates across the pool's member nodes.

**Key Principle**: Every node belongs to exactly one pool. The pool's `machineConfigSelector` determines which MachineConfigs are merged into the pool's rendered config. The pool's `nodeSelector` determines which nodes are members.

## Spec Structure

```go
type MachineConfigPoolSpec struct {
    MachineConfigSelector *metav1.LabelSelector           // Selects MachineConfigs for this pool
    NodeSelector          *metav1.LabelSelector           // Selects nodes for this pool
    Paused                bool                            // Stop generating new rendered configs and updating nodes
    MaxUnavailable        *intstr.IntOrString             // Max nodes unavailable during update (default: 1)
    Configuration         MachineConfigPoolStatusConfiguration // Current target config
    PinnedImageSets       []PinnedImageSetRef             // Pre-loaded images (max 100)
    OSImageStream         OSImageStreamReference          // OS stream override (FeatureGate=OSStreams)
}
```

## Key Concepts

### Built-in Pools

- `master` — Control plane nodes (`node-role.kubernetes.io/master` label)
- `worker` — Worker nodes (`node-role.kubernetes.io/worker` label)

### Custom Pools

Custom pools inherit from the worker pool. A node can only belong to one pool — if a node matches both `worker` and a custom pool selector, the custom pool takes precedence.

### Pausing

Setting `paused: true` stops both rendering new configs and rolling out updates. Use this to batch changes before applying them together.

### Status Conditions

| Condition | Meaning |
|-----------|---------|
| `Updated` | All nodes are at the desired rendered config |
| `Updating` | At least one node is updating |
| `Degraded` | Overall pool degradation (aggregates NodeDegraded, RenderDegraded, etc.) |
| `NodeDegraded` | A specific node failed to update |
| `RenderDegraded` | Rendered config generation failed |
| `PinnedImageSetsDegraded` | Pinned image set processing failed |

### Machine Counts

Status tracks: `machineCount`, `updatedMachineCount`, `readyMachineCount`, `unavailableMachineCount`, `degradedMachineCount`.

## Lifecycle

1. **Creation**: Pool is created with node and MC selectors → render controller generates rendered config
2. **Update**: New/changed MachineConfigs trigger re-rendering → node controller coordinates rolling update respecting `maxUnavailable`
3. **Completion**: All nodes reach desired config → `Updated=True`, `Updating=False`

## Common Mistakes

1. **Overlapping nodeSelectors** — Nodes should only match one pool
2. **Setting maxUnavailable to 0** — Defaults back to 1; use `paused: true` to stop updates
3. **Forgetting that drain respects PDBs** — Even with `maxUnavailable > 1`, etcd quorum guards and other PDBs are honored

## Related Concepts

- [MachineConfig](./machineconfig.md) — Individual config fragments merged per pool
- [MachineConfigNode](./machineconfignode.md) — Per-node update tracking
