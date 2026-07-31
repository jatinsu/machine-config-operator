# Machine Config Operator - Agentic Documentation

**Component**: Machine Config Operator (MCO)
**Repository**: openshift/machine-config-operator

> **Generic Platform Patterns**: See Platform documentation (openshift/enhancements/ai-docs/) for operator patterns, testing practices, security guidelines, and cross-repo ADRs.

## What is MCO?

The MCO manages the OS-level configuration of OpenShift nodes — everything between the kernel and kubelet. It renders Ignition configs, coordinates rolling updates across node pools, manages OS image updates, and handles certificate rotation.

**Key Principle**: Configuration is declarative via MachineConfig CRDs, merged per pool into immutable rendered configs (`rendered-<pool>-<hash>`), and applied by the on-node MachineConfigDaemon.

## Core Components

| Component | Binary | Role |
|-----------|--------|------|
| **Operator** | `machine-config-operator` | Renders sub-component manifests, reports ClusterOperator status |
| **Controller (MCC)** | `machine-config-controller` | Runs render, template, node, kubelet-config, container-runtime-config, drain, certrotation, pinnedimageset, osimagestream, bootimage, internalreleaseimage controllers |
| **Daemon (MCD)** | `machine-config-daemon` | On-node agent: applies configs, manages OS updates, performs reboots |
| **Server (MCS)** | `machine-config-server` | Serves Ignition to bootstrapping nodes over TLS |
| **Builder (MOB)** | `machine-os-builder` | On-cluster image builds (OCL) using buildah Jobs |

**Quick Start**: `oc get clusteroperator machine-config` | `oc get mcp` | `oc get machineconfignodes`

## Critical Patterns

1. **Never assume uniform apply methods** — `lib/resourceapply/` uses Get+Create/Update (NOT SSA). Each controller type has its own Apply function. Check `lib/resourceapply/` for the correct method.

2. **OSImageStream must start first** — In MCC startup, `EnsureOSImageStream()` blocks before other controllers start. Never add controller startup before this call. See `cmd/machine-config-controller/start.go:143`.

3. **Never exceed 1.5MB MachineConfig size** — `MaxMachineConfigSize = 1572864` matches etcd's request limit. Use `ctrlcommon.InternalMCOIgnitionVersion` (`"3.5.0"`) for ignition version, not hardcoded strings.

## Documentation Structure

```text
ai-docs/
├── domain/                    # MachineConfig, MachineConfigPool, MachineConfigNode, MachineOSConfig/Build
├── architecture/
│   └── components.md          # Repo layout, controller wiring, apply patterns, feature gates
├── decisions/                 # Component ADRs (apply pattern, OSImageStream ordering, rendered config naming)
├── exec-plans/                # Feature planning
├── references/
│   ├── ecosystem.md           # Links to Platform docs
│   └── enhancements.md        # Enhancement proposals catalog (50+ design docs)
├── MCO_DEVELOPMENT.md         # Build, debug, common tasks
└── MCO_TESTING.md             # Test suites, sharding, framework
```

**AI Agent Path**: domain/ → architecture/ → decisions/ → MCO_DEVELOPMENT.md

## CRD Quick Reference

| Kind | Short | Scope | Key Purpose |
|------|-------|-------|-------------|
| MachineConfig | mc | Cluster | OS config fragment (Ignition, kernel args, extensions) |
| MachineConfigPool | mcp | Cluster | Groups nodes, selects MCs, tracks rollout |
| MachineConfigNode | — | Cluster | Per-node update progress tracking |
| ControllerConfig | — | Cluster | Cluster-wide config for template rendering |
| KubeletConfig | — | Cluster | Kubelet tuneables → MC fragment |
| ContainerRuntimeConfig | ctrcfg | Cluster | CRI-O tuneables → MC fragment |
| MachineOSConfig | — | Cluster | OCL build config (one per pool) |
| MachineOSBuild | — | Cluster | Individual build execution |
| PinnedImageSet | — | Cluster | Pre-loaded container images |
| OSImageStream | — | Cluster | OS stream discovery (singleton "cluster") |

## Key Shared Utilities

| Package | Purpose |
|---------|---------|
| `pkg/controller/common/` | Constants, helpers, feature gates, metrics, controller context |
| `lib/resourceapply/` | `ApplyMachineConfig()`, `ApplyDaemonSet()`, `ApplyDeployment()`, etc. |
| `internal/clients/` | Client builder for kube, MCO, config, operator API clients |
| `cmd/common/` | Leader election config, signal handling |

## External References

- [Product Docs](https://docs.openshift.com/) | [Enhancement Proposals](https://github.com/openshift/enhancements/tree/master/enhancements/machine-config/) | [Local Design Docs](docs/)

---

**Platform Documentation**: openshift/enhancements/ai-docs/
