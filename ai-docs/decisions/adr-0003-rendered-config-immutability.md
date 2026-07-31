# ADR-0003: Rendered MachineConfig Immutability and Hash-Based Naming

**Status**: Accepted
**Date**: 2024-01-01 (approximate — foundational design)
**Deciders**: MCO team
**Component**: MCO (render controller)

## Context

The render controller merges multiple MachineConfig fragments per pool into a single rendered config. Nodes need a deterministic way to detect whether their config has changed and whether an update is needed.

**Scope**: This ADR is component-specific.

## Decision

Rendered MachineConfigs are named `rendered-<pool>-<hash>` where the hash is derived from the merged content. They are treated as immutable — when inputs change, a new rendered config is created rather than updating the existing one.

## Rationale

- Hash-based naming provides content-addressable identity: same inputs → same name → no spurious updates
- Immutability simplifies rollback reasoning — the previous rendered config still exists
- Node controller compares `current` vs `desired` config names on MachineConfigNode to determine if an update is needed
- The `rendered-` prefix (`ctrlcommon.RenderedMachineConfigPrefix`) makes rendered configs easy to identify and filter

## Consequences

### Positive
- Deterministic change detection without deep comparison
- Clean audit trail of rendered configs over time
- Simple upgrade/rollback logic in the node controller

### Negative
- Old rendered configs accumulate and must be garbage-collected
- Config name changes even for semantically equivalent changes (e.g., annotation-only changes)

## References

- `pkg/controller/render/render_controller.go` — Render controller
- `pkg/controller/common/constants.go:43` — `RenderedMachineConfigPrefix = "rendered-"`
- `pkg/controller/common/helpers.go:71` — `MergeMachineConfigs()`
