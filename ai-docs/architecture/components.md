# MCO Architecture

## Repository Layout

```text
cmd/
├── machine-config-operator/       # Top-level operator: renders sub-component manifests, manages ClusterOperator status
├── machine-config-controller/     # MCC: runs render, template, node, kubelet-config, container-runtime-config, drain, certrotation, pinnedimageset, osimagestream, bootimage, internalreleaseimage controllers
├── machine-config-daemon/         # MCD: on-node agent, applies MachineConfig to the host OS (files, systemd, OS updates, reboots)
├── machine-config-server/         # MCS: serves Ignition configs to bootstrapping nodes via HTTPS
├── machine-os-builder/            # MOB: on-cluster image builds (OCL) using buildah Jobs
├── machine-config-osimagestream/  # OS image stream binary
├── apiserver-watcher/             # Watches for kube-apiserver readiness
├── machine-config-tests-ext/      # Extended test binary
├── common/                        # Shared CLI helpers (leader election, signal handling)
pkg/
├── controller/
│   ├── common/                    # Shared: constants, helpers, featuregates, metrics, controller context, reconcile utilities
│   ├── render/                    # Merges MachineConfigs per pool → rendered-<pool>-<hash>
│   ├── template/                  # Generates base MachineConfigs from templates/ + ControllerConfig
│   ├── node/                      # Coordinates node updates: sets desiredConfig on MCN, manages pool status, cordon/drain orchestration
│   ├── kubelet-config/            # Translates KubeletConfig CRs → MachineConfig fragments
│   ├── container-runtime-config/  # Translates ContainerRuntimeConfig CRs → MachineConfig fragments
│   ├── build/                     # OCL build controller: MachineOSConfig → MachineOSBuild → Job
│   ├── drain/                     # Node drain controller with PDB-aware eviction
│   ├── certrotation/              # MCS TLS cert rotation using library-go cert rotation
│   ├── bootimage/                 # Updates MachineSet/CPMS boot images to match current OS
│   ├── pinnedimageset/            # Validates PinnedImageSet images, manages pool synchronizer status
│   ├── osimagestream/             # Discovers OS streams from release payload, populates OSImageStream CR
│   ├── internalreleaseimage/      # Manages InternalReleaseImage for no-registry installs
│   ├── bootstrap/                 # Bootstrap-time rendering (no API server)
lib/
│   └── resourceapply/             # MCO-specific Apply helpers (ApplyMachineConfig, ApplyDaemonSet, etc.)
internal/
│   └── clients/                   # Client builder for kube, MCO, config, operator API clients
pkg/daemon/                        # MCD implementation: update.go (main update logic), daemon.go (lifecycle)
pkg/operator/                      # Top-level operator: sync.go (reconciles sub-component Deployments/DaemonSets)
pkg/server/                        # MCS implementation: serves Ignition via HTTPS
pkg/helpers/                       # Miscellaneous helpers
pkg/apihelpers/                    # API object helpers
pkg/constants/                     # Additional constants
templates/                         # Go templates → base MachineConfig (systemd units, configs)
manifests/                         # CRD YAMLs installed by CVO
install/                           # Installer manifests
```

## Component Hierarchy

The MCO has a three-tier architecture:

| Tier | Binary | Runs As | Purpose |
|------|--------|---------|---------|
| **Operator** | `machine-config-operator` | Deployment (1 replica, leader-elected) | Renders and syncs all sub-component manifests (MCC Deployment, MCD DaemonSet, MCS DaemonSet). Reports ClusterOperator status. |
| **Controller** | `machine-config-controller` | Deployment (1 replica, leader-elected) | Runs all controllers that reconcile MCO CRs. |
| **Daemon** | `machine-config-daemon` | DaemonSet (every node) | On-node agent that applies MachineConfig changes: writes files, sets up systemd units, updates OS image, performs reboots. |

Supporting components: `machine-config-server` (DaemonSet on control-plane, serves Ignition to joining nodes), `machine-os-builder` (Deployment, OCL image builds).

## Controller Registration & Startup

### Operator (`cmd/machine-config-operator/start.go`)
1. Creates `clients.Builder` → `ControllerContext` with all informer factories
2. Connects `FeatureGatesHandler` (reads `featuregates.config.openshift.io/cluster`)
3. Creates single `operator.New(...)` with all informers
4. Starts all informer factories, runs operator with 2 workers
5. Uses leader election via `leaderelection.RunOrDie`

### MCC (`cmd/machine-config-controller/start.go`)
1. Creates `ControllerContext`, connects FeatureGates
2. **OSImageStream controller starts FIRST** with blocking `EnsureOSImageStream()` — other controllers depend on it for OS image URLs
3. `createControllers()` returns: render, template, kubelet-config, container-runtime-config, node
4. Additional controllers started conditionally or separately: drain (5 workers), certrotation (1 worker), pinnedimageset (2 workers), bootimage (feature-gated), internalreleaseimage (feature-gated)
5. All controllers implement `ctrlcommon.Controller` interface: `Run(ctx, workers)`

### Adding a New Controller
1. Create package under `pkg/controller/<name>/`
2. Implement `ctrlcommon.Controller` interface (`Run(ctx context.Context, workers int)`)
3. Wire in `cmd/machine-config-controller/start.go` — either add to `createControllers()` slice or start independently
4. Use `ctrlcommon.CreateControllerContext()` for informers and clients

## Resource Application Pattern

**DO NOT assume uniform apply methods.** Each controller area uses different patterns:

| Controller | Apply Method | Code Reference |
|-----------|-------------|----------------|
| render | `lib/resourceapply.ApplyMachineConfig()` (Get+Create or Get+Update) | `pkg/controller/render/render_controller.go` |
| template | `lib/resourceapply.ApplyMachineConfig()` | `pkg/controller/template/template_controller.go` |
| node | Direct status updates via `client.MachineconfigurationV1().MachineConfigNodes()` | `pkg/controller/node/node_controller.go` |
| kubelet-config | `lib/resourceapply.ApplyMachineConfig()` for generated MCs | `pkg/controller/kubelet-config/` |
| container-runtime-config | `lib/resourceapply.ApplyMachineConfig()` for generated MCs | `pkg/controller/container-runtime-config/` |
| build (OCL) | Direct client Create/Update for MachineOSBuild, Jobs | `pkg/controller/build/` |
| certrotation | library-go cert rotation framework | `pkg/controller/certrotation/` |
| operator (top-level) | `lib/resourceapply.ApplyDaemonSet()`, `ApplyDeployment()` | `pkg/operator/sync.go` |

`lib/resourceapply/` implements Get-then-Create-or-Update pattern — NOT Server-Side Apply (SSA).

## Feature Gates

Feature gates flow: `featuregates.config.openshift.io/cluster` → `FeatureGatesHandler.Connect()` → `handler.Enabled(featureName)`.

Key feature-gated controllers:
- `FeatureGateNoRegistryClusterInstall` → InternalReleaseImage controller, IRI informers in template controller
- `FeatureGateOSStreams` → OSImageStream controller (always starts but feature must be enabled for full behavior)
- `FeatureGateImageModeStatusReporting` → Image mode fields on MachineConfigNode
- `FeatureGateIrreconcilableMachineConfig` → Irreconcilable changes tracking on MCN
- Boot image controller: `IsBootImageControllerRequired()` checks infrastructure platform type

**Runtime check pattern**: `ctrlctx.FeatureGatesHandler.Enabled(features.FeatureGateName)` at controller creation time.

## MCD Update Flow (`pkg/daemon/update.go`)

The MCD is the most critical and complex component. Update sequence:
1. Detect desired config differs from current (via MachineConfigNode spec vs status)
2. Update MachineConfigNode status conditions (UpdatePrepared)
3. Cordon node → Drain node (PDB-aware) → Apply files and OS changes → Reboot (if needed) → Uncordon
4. Report completion via MachineConfigNode status

## Key Namespaces and Labels

| Constant | Value | Usage |
|----------|-------|-------|
| `MCONamespace` | `openshift-machine-config-operator` | All MCO workloads |
| `MachineConfigRoleLabel` | `machineconfiguration.openshift.io/role` | MachineConfig → pool selection |
| `RenderedMachineConfigPrefix` | `rendered-` | Rendered MC naming convention |
| `GeneratedByControllerVersionAnnotationKey` | `machineconfiguration.openshift.io/generated-by-controller-version` | Version tracking |
| `InternalMCOIgnitionVersion` | `3.5.0` | Internal ignition spec version |
| `MaxMachineConfigSize` | `1572864` (1.5MB) | Matches etcd request limit |

## OpenShift Integrations

- **Proxy**: Reads `proxies.config.openshift.io/cluster`, propagates to ControllerConfig.Spec.Proxy
- **Infrastructure**: Reads `infrastructures.config.openshift.io/cluster` for platform type, embedded in ControllerConfig
- **DNS**: Reads `dns.config.openshift.io/cluster`, embedded in ControllerConfig
- **ClusterVersion**: Used by OSImageStream and boot image controllers for version tracking
- **ClusterOperator**: Operator reports status to `clusteroperators.config.openshift.io/machine-config`
- **Image mirrors**: ICSP, IDMS, ITMS informers used by render, container-runtime-config, osimagestream controllers
- **TLS profiles**: KubeletConfig controller reads `apiservers.config.openshift.io/cluster` for TLS settings
- **Cert rotation**: Uses library-go cert rotation for MCS TLS certificates

## Anti-Patterns

1. **DO NOT** hand-edit files under `vendor/` — use `go mod vendor`
2. **DO NOT** assume all controllers use the same apply method — check per-controller (see table above)
3. **DO NOT** start new controllers before OSImageStream — it must complete `EnsureOSImageStream()` first
4. **DO NOT** use ignition version strings directly — use `ctrlcommon.InternalMCOIgnitionVersion` constant
5. **DO NOT** create MachineConfigs exceeding 1.5MB (`MaxMachineConfigSize`) — matches etcd request limit

## SME Review Recommended

- Implementation recipes for adding new MCD update phases
- Rationale behind Get+Create/Update vs SSA choice in `lib/resourceapply`
- Cross-component interaction patterns between MCD and node controller during upgrades
