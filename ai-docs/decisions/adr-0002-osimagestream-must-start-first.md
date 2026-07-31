# ADR-0002: OSImageStream Controller Must Start Before Other Controllers

**Status**: Accepted
**Date**: 2024-06-01 (approximate)
**Deciders**: MCO team
**Component**: MCO (MCC startup)

## Context

The OSImageStream controller discovers available OS streams from the release payload and populates the `OSImageStream` cluster singleton. Multiple other controllers (render, node, template) depend on OS image URLs that come from this resource.

If these controllers start before OSImageStream is populated, they may operate with stale or missing OS image data, leading to incorrect rendered configs or failed updates.

**Scope**: This ADR is component-specific.

## Decision

The OSImageStream controller starts first in `cmd/machine-config-controller/start.go` with a **blocking** `EnsureOSImageStream()` call. No other controller starts until this call returns successfully.

## Rationale

- Render, node, and template controllers all read OS image URLs from the OSImageStream CR
- A missing or stale OSImageStream causes silent failures — configs render with wrong images
- The blocking call is a simple, reliable guarantee vs. retry-based eventual consistency

## Consequences

### Positive
- All downstream controllers have correct OS image data from their first reconciliation
- Eliminates a class of race conditions during MCC startup

### Negative
- MCC startup is blocked if OSImageStream creation fails — the entire controller binary terminates
- Adds a hard dependency on release payload image inspection at startup

## References

- `cmd/machine-config-controller/start.go:143` — `EnsureOSImageStream()` blocking call
- `pkg/controller/osimagestream/` — OSImageStream controller implementation
