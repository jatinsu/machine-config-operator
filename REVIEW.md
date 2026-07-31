# Review instructions

## Must fix before merge

- Incorrect reconciliation logic (wrong Get/Create/Update sequence, missing conflict retry, skipped `resourcemerge.Ensure*` check)
- Resource leaks: informers started without corresponding stop, objects created without owner references or cleanup
- Upgrade/downgrade safety: changes that break rolling update (MCD applies new config but old MCC cannot render it)
- Breaking changes to GA `machineconfiguration.openshift.io/v1` API fields or semantics
- `Available=False` or `Degraded=True` set during normal upgrade path — `Available=False` is reserved for "midnight admin page" severity only (`pkg/operator/status.go`)
- Premature `operator` version bump in ClusterOperator status before operands are ready
- Security: credentials logged, pull secrets exposed, TLS downgrade, or missing RBAC scoping
- MachineConfig exceeding 1.5 MB (`MaxMachineConfigSize = 1572864`), which hits the etcd request limit
- Hardcoded ignition version instead of `ctrlcommon.InternalMCOIgnitionVersion`
- Controller startup added before `EnsureOSImageStream()` in `cmd/machine-config-controller/start.go`

## Minor issue volume

Report at most five minor issues. Overflow: "plus N similar items" in the summary. If all minor, lead with "No blocking issues."

## Do not report

- Lint, gofmt, govet, type-check — CI-enforced via `make verify`
- Template verification — CI-enforced via `make verify-templates`
- `vendor/**` — vendored upstream code
- `go.sum` — lockfile noise (review `go.mod` dep bumps separately)
- `zz_generated*`, `**/clientset/**`, `**/informers/**`, `**/listers/**` — generated code
- `manifests/**` — embedded manifest YAMLs (compiled into binary)
- `templates/**/*.yaml` — platform-specific Ignition templates (validated by `verify-templates`)

## Always check

- Alpha features behind `TechPreviewNoUpgrade` feature gate; TP CRDs gated by `release.openshift.io/feature-gate` annotation (`guidelines/supportability.md`)
- No bool fields in new CRD APIs; config API defaults belong in controller, not in the API (`dev-guide/api-conventions.md`)
- No CRD pointer fields unless distinguishing zero from unset is required (`dev-guide/api-conventions.md`)
- Resource requests declared; resource limits NOT set — components must not be OOM-killed by limits (`CONVENTIONS.md`)
- CPU request never below 5m (`CONVENTIONS.md`)
- Operators must not tolerate `node.kubernetes.io/unschedulable` (`CONVENTIONS.md`)
- "OpenShift" capitalization, never "Openshift" (`CONVENTIONS.md`)
- `Available`/`Degraded`/`Progressing` conditions: reason+message for both happy and sad states; `Progressing` message ≤10 words (`dev-guide/clusteroperator.md`)
- Apply via `lib/resourceapply/` (Get+Create/Update pattern), not SSA — verify new controllers match existing pattern
- MachineConfigNode conditions follow the state machine: UpdatePrepared → Cordoned → Drained → AppliedFilesAndOS → RebootedNode → Uncordoned → UpdateComplete → Updated
- OCL builds: terminal conditions (Succeeded/Failed/Interrupted) must freeze `.status` — no further mutations

## Verification bar

Every comment must cite file:line evidence from the diff or linked source. If you cannot point to a specific line, do not post the comment. Read surrounding context (at minimum the enclosing function) before flagging — the answer may be ten lines below the diff hunk.

## Re-review

On re-review of an updated PR, only comment on lines that changed since the last review. Do not re-raise resolved issues or introduce new nits on unchanged code. Converge toward approval.

## Path-specific rules

### `pkg/controller/**`

Verify apply method uses `lib/resourceapply/` — never SSA. New controllers must implement `ctrlcommon.Controller` interface with `Run(ctx, workers)`. Check that informer event handlers enqueue the correct object key. Render controller uses `renderDelay = 5s`; node controller uses `defaultUpdateDelay = 5s` — respect existing timing.

### `pkg/daemon/**`

High-churn area (`update.go` has 41 changes/year). Check MachineConfigNode condition updates match the state machine. Verify rebootless vs reboot paths are both tested. Flag any new SELinux workarounds for SME review.

### `pkg/operator/**`

`sync.go` is second-highest churn. Verify `syncAvailableStatus` never sets `Available=False` for non-critical issues. Check `createDiscoveredControllerConfigSpec` handles all Infrastructure platforms. Verify image version matches operator version in `images.json` parsing.

### `lib/resourceapply/**`

Changes here affect ALL controllers. Verify `IsApplyErrorRetriable` covers the new error class. Ensure `resourcemerge.Ensure*` functions correctly set the `modified` bool.

### `cmd/**`

Controller startup ordering is critical. `EnsureOSImageStream()` must remain the first blocking call in MCC. Feature-gated controllers must check the gate before starting.

### `test/**`

E2E tests must target the correct shard (`e2e-1of2` or `e2e-2of2`). OCL tests go in `e2e-ocl-*`. TechPreview tests go in `e2e-techpreview`. Verify test cleanup avoids flaky state leaks.
