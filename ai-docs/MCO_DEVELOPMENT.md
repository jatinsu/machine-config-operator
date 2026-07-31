# MCO - Development Guide

This guide covers **MCO-specific** development practices.

## Quick Start

### Prerequisites

- Go 1.26+ (from go.mod)
- Access to OpenShift cluster (for testing)
- `KUBECONFIG` environment variable set
- Podman (for container builds)

### Build Binaries

```bash
make binaries                        # All components
make _build-component-machine-config-operator   # Operator only
make _build-component-machine-config-controller # MCC only
make _build-component-machine-config-daemon     # MCD only
```

**Binaries output**: `_output/linux/$(GOARCH)/`

### Run Tests

```bash
make test-unit                       # All unit tests
go test -v ./pkg/controller/render/... # Specific package
make lint                            # golangci-lint
make verify                          # Full verification suite
```

## Development Workflow

### 1. Local Development

```bash
# Build and run unit tests
make binaries test-unit

# Lint check
make lint
```

### 2. Testing on Cluster

```bash
# Build image
make image

# Push to registry
podman push localhost/machine-config-operator:latest quay.io/<user>/machine-config-operator:dev

# Patch the operator deployment to use dev image
oc -n openshift-machine-config-operator set image deployment/machine-config-operator machine-config-operator=quay.io/<user>/machine-config-operator:dev
```

### 3. Debugging

```bash
# View operator logs
oc logs -n openshift-machine-config-operator deployment/machine-config-operator

# View MCC logs
oc logs -n openshift-machine-config-operator deployment/machine-config-controller

# View MCD logs on a specific node
oc logs -n openshift-machine-config-operator daemonset/machine-config-daemon -c machine-config-daemon

# Check ClusterOperator status
oc get clusteroperator machine-config -o yaml

# Check pool status
oc get mcp

# Check node config status
oc get machineconfignodes
```

## Common Tasks

### Add a New Controller

1. Create package: `pkg/controller/<name>/`
2. Implement `ctrlcommon.Controller` interface with `Run(ctx context.Context, workers int)`
3. Wire in `cmd/machine-config-controller/start.go`:
   - Add to `createControllers()` return slice, OR
   - Start independently with feature gate check
4. Add required informers to `ctrlcommon.ControllerContext` if needed (`pkg/controller/common/controller_context.go`)
5. Use `lib/resourceapply/` for creating/updating MCO resources

### Add a New MachineConfig Fragment

1. Add template to `templates/` directory
2. Update template controller to render it (`pkg/controller/template/`)
3. Template receives `ControllerConfig` data — use `.Images` map for container image references

### Modify the MCD Update Flow

1. Primary logic in `pkg/daemon/update.go`
2. Update MachineConfigNode conditions as phases progress
3. Test with both reboot and rebootless paths
4. **High-churn file** — `pkg/daemon/update.go` has the most changes in the past year

### Update Dependencies

```bash
go get <module>@<version>
go mod tidy
go mod vendor
make verify  # Ensure everything still passes
```

## Build & Release

### CI Build

Component images are built by OpenShift CI on PR merge. See CI configuration in the repo.

### Release Process

MCO is released as part of the OpenShift release image, managed by CVO.

## Common Mistakes

1. **DO NOT** add controllers before OSImageStream in startup sequence — it must `EnsureOSImageStream()` first
2. **DO NOT** use raw ignition version strings — use `ctrlcommon.InternalMCOIgnitionVersion` (currently `"3.5.0"`)
3. **DO NOT** create MachineConfigs over 1.5MB — exceeds etcd request size limit (`MaxMachineConfigSize`)
4. **DO NOT** assume uniform apply patterns — check `lib/resourceapply/` for the correct method per resource type
5. **DO NOT** hand-edit vendored code — use `go mod vendor` after dependency changes
6. **DO NOT** skip `make verify` before submitting — it runs lint, template verification, and helper verification

## Known Operational Issues

- `pkg/daemon/update.go` disables SELinux enforcement during some operations (`HACK` comment at line ~2830)
- Certificate rotation for MCS CA bundles requires coordination with `machine-config-server-ca` ConfigMap/Secret

## SME Review Recommended

- Detailed recipes for adding new MCD update phases (cordon/drain/apply/reboot sequence)
- Cross-component interaction patterns between MCD and drain controller during upgrades
- Best practices for testing MCD changes (requires real cluster or specific mocking approach)

## See Also

- [Testing Guide](./MCO_TESTING.md)
- [Architecture](./architecture/components.md)
- [HACKING guide](../docs/HACKING.md) — Original developer guide
