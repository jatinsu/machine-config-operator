# Platform Ecosystem References

This document links to generic OpenShift/Kubernetes patterns in the Platform ecosystem hub. The MCO inherits these platform-wide patterns and practices.

## Operator Patterns

**Location**: [ai-docs/platform/operator-patterns/](https://github.com/openshift/enhancements/tree/master/ai-docs/platform/operator-patterns/)

- **Controller Runtime**: Reconciliation loops, event handling, client patterns
- **Status Conditions**: Available, Progressing, Degraded condition semantics
- **Finalizers**: Resource cleanup patterns
- **RBAC**: Service account and permissions

**MCO Usage**:
- MCO uses custom controller framework (not controller-runtime) with `ctrlcommon.Controller` interface
- Certrotation controller uses library-go's `controller/factory` framework
- ClusterOperator status reported via `pkg/operator/status.go`

## Testing Practices

**Location**: openshift/enhancements/ai-docs/practices/testing/

- **Test Pyramid**: Unit > Integration > E2E ratio
- **E2E Framework**: OpenShift E2E test patterns

**MCO Usage**:
- See `MCO_TESTING.md` for component-specific test suites
- E2E tests split across `test/e2e-1of2/`, `test/e2e-2of2/`, and specialized directories

## Security Practices

**Location**: openshift/enhancements/ai-docs/practices/security/

- **RBAC Guidelines**: Role and ClusterRole design

**MCO Usage**:
- MCS serves Ignition over TLS with cert rotation via library-go
- FIPS mode support via MachineConfig.Spec.FIPS
- Pull secret management for node bootstrapping

## Reliability Practices

**Location**: openshift/enhancements/ai-docs/practices/reliability/

- **Observability**: Metrics, logging patterns

**MCO Usage**:
- Prometheus metrics exposed via `pkg/controller/common/metrics.go`
- Metrics served over HTTPS with TLS client certs

## OpenShift Fundamentals

**Location**: [ai-docs/domain/openshift/](https://github.com/openshift/enhancements/tree/master/ai-docs/domain/openshift/)

- **ClusterOperator**: Cluster operator status reporting
- **ClusterVersion**: Platform upgrade orchestration

**MCO Usage**:
- Reports to `clusteroperators.config.openshift.io/machine-config`
- Reads ClusterVersion for version tracking and boot image management

## Cross-Repository ADRs

**Location**: openshift/enhancements/ai-docs/decisions/

Platform-wide architectural decisions:
- **Immutable Nodes**: Why RHCOS + rpm-ostree (directly relevant to MCO)
- **CVO Orchestration**: Why CVO orchestrates upgrades (MCO is a CVO-managed operator)

**Component-Specific ADRs**: See `ai-docs/decisions/` for MCO-specific decisions

---

**Note**: These links point to Platform (ecosystem hub) documentation. Component-specific patterns and decisions are documented in the `ai-docs/` directory of this repository.

**Last Updated**: 2026-07-31
