# ADR-0001: Get+Create/Update Over Server-Side Apply

**Status**: Accepted
**Date**: 2024-01-01 (approximate — predates formal ADR process)
**Deciders**: MCO team
**Component**: MCO

## Context

The MCO needs to create and update Kubernetes resources (MachineConfigs, DaemonSets, Deployments, etc.) during reconciliation. Two primary patterns exist in the OpenShift ecosystem:
1. Server-Side Apply (SSA) — used by some operators via `client.Apply()`
2. Get+Create/Update — traditional pattern using `Get()` then `Create()` or `Update()`

## Decision

The MCO uses Get+Create/Update via its own `lib/resourceapply/` package. Each resource type has a dedicated `Apply<Type>()` function that performs Get, then conditionally Create or Update.

## Rationale

- MCO predates widespread SSA adoption in the OpenShift ecosystem
- `lib/resourceapply/` provides MCO-specific resource types not available in library-go's resourceapply (e.g., `ApplyMachineConfig`, `ApplyMachineConfigPool`, `ApplyMachineConfigNode`, `ApplyControllerConfig`)
- The pattern is well-tested and consistent across the codebase

## Consequences

### Positive
- Explicit control over what fields are updated
- Simple debugging — Get+Create/Update is straightforward to trace
- MCO-specific types handled natively

### Negative
- No automatic field ownership tracking (SSA advantage)
- Potential for lost updates if multiple controllers modify the same resource (mitigated by MCO owning its CRDs exclusively)

## References

- `lib/resourceapply/machineconfig.go` — MCO-specific Apply functions
- `lib/resourceapply/apps.go` — Kubernetes resource Apply functions
