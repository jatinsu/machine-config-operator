# MCO - Testing Guide

This guide covers **MCO-specific** test suites and testing practices.

## Test Organization

### Directory Structure

```text
test/
├── e2e-1of2/               # E2E tests, first shard
├── e2e-2of2/               # E2E tests, second shard
├── e2e-bootstrap/          # Bootstrap E2E tests
├── e2e-iri/                # InternalReleaseImage E2E tests
├── e2e-ocl-1of2/           # On-Cluster Layering E2E, first shard
├── e2e-ocl-2of2/           # On-Cluster Layering E2E, second shard
├── e2e-ocl-shared/         # Shared OCL test utilities
├── e2e-shared-tests/       # Shared E2E test functions
├── e2e-single-node/        # Single Node OpenShift E2E tests
├── e2e-techpreview/        # TechPreview feature E2E tests
├── e2e-techpreview-shared/ # Shared TechPreview test utilities
├── extended/               # Extended tests (openshift-tests integration)
├── extended-priv/          # Privileged extended tests
├── framework/              # Test framework utilities
└── helpers/                # Test helper functions
```

### Unit Tests

Unit tests live alongside the code in `pkg/`, `lib/`, `internal/`:
- `pkg/controller/*/` — Controller unit tests
- `pkg/daemon/` — MCD unit tests
- `pkg/operator/` — Operator unit tests
- `lib/resourceapply/` — Apply function unit tests
- `pkg/controller/common/` — Shared utility tests

### Running Unit Tests

```bash
# All unit tests
make test-unit

# Specific package
go test -v ./pkg/controller/render/...

# With JUnit output (CI pattern)
make install-go-junit-report
go test ./pkg/... 2>&1 | go-junit-report > junit.xml

# With race detector
go test -race ./pkg/...
```

## E2E Tests

### Running E2E Tests

```bash
# Requires KUBECONFIG pointing to a real OpenShift cluster
export KUBECONFIG=/path/to/kubeconfig

# Run first shard
go test -v ./test/e2e-1of2/ -timeout 120m

# Run second shard
go test -v ./test/e2e-2of2/ -timeout 120m

# Run single-node tests
go test -v ./test/e2e-single-node/ -timeout 120m

# Run TechPreview tests (requires TechPreview cluster)
go test -v ./test/e2e-techpreview/ -timeout 120m
```

### E2E Test Sharding

Tests are split into shards (`e2e-1of2`, `e2e-2of2`) for CI parallelism. Similarly, OCL tests are split into `e2e-ocl-1of2` and `e2e-ocl-2of2`.

### Extended Tests

`test/extended/` and `test/extended-priv/` contain tests that integrate with the `openshift-tests` framework:

```bash
# These are typically run via CI, not locally
go test -v ./test/extended/...
go test -v ./test/extended-priv/...
```

Key files in `test/extended-priv/`:
- `node.go` — Node-level test helpers
- `machineconfigpool.go` — MCP test helpers
- `mco_ocb.go` — On-cluster build test helpers
- `util.go` — General test utilities

## Test Framework

`test/framework/` provides test utilities:
- Client creation and cluster interaction
- Wait/polling helpers for async operations
- Resource creation and cleanup

`test/helpers/` provides:
- MachineConfig builders for test fixtures
- Pool management helpers
- Node state assertion helpers

## Component-Specific Test Patterns

### Controller Tests

Controller tests use fake clientsets:
```go
// Pattern from pkg/controller/render/
client := fake.NewSimpleClientset(existingMCs...)
controller := New(informers..., client)
err := controller.syncHandler("pool-key")
```

### MCD Tests

MCD tests are complex due to host OS interactions. Most MCD logic is tested via:
- Unit tests for pure functions in `pkg/daemon/`
- E2E tests that create MachineConfigs and verify node state

### Build Controller Tests

`pkg/controller/build/` has its own test infrastructure:
- `buildrequest/` — Build request construction tests
- `reconciler.go` / `reconciler_test.go` — Reconciler logic tests

## CI/CD Testing

### PR Testing

CI runs on every PR via OpenShift CI:
- Unit tests (`make test-unit`)
- Lint checks (`make lint`)
- Template verification (`make verify-templates`)
- E2E tests on ephemeral clusters

### Verification Suite

```bash
make verify  # Runs: lint, verify-templates, verify-helpers, verify-e2e
```

## Debugging Failing Tests

### Unit Test Failures

```bash
go test -v -run TestSpecificFunction ./pkg/controller/render/...
go test -race ./pkg/...  # Check for race conditions
```

### E2E Test Failures

```bash
# Collect must-gather
oc adm must-gather

# Check MCO component logs
oc logs -n openshift-machine-config-operator deployment/machine-config-operator
oc logs -n openshift-machine-config-operator deployment/machine-config-controller
oc logs -n openshift-machine-config-operator daemonset/machine-config-daemon -c machine-config-daemon --all-pods

# Check resource states
oc get mcp -o wide
oc get machineconfignodes
oc get machineconfig -l machineconfiguration.openshift.io/role=worker
```

## SME Review Recommended

- Patterns for mocking MCD host interactions in unit tests
- Best practices for E2E test cleanup to avoid flaky tests
- How to add tests to the correct shard (e2e-1of2 vs e2e-2of2)

## See Also

- [Development Guide](./MCO_DEVELOPMENT.md)
- [Architecture](./architecture/components.md)
