# MachineOSConfig & MachineOSBuild

**API Group**: `machineconfiguration.openshift.io/v1`
**Kinds**: `MachineOSConfig`, `MachineOSBuild`
**Scope**: Cluster

**API Definition**: [types_machineosconfig.go](https://github.com/openshift/api/blob/master/machineconfiguration/v1/types_machineosconfig.go), [types_machineosbuild.go](https://github.com/openshift/api/blob/master/machineconfiguration/v1/types_machineosbuild.go)

## Purpose

MachineOSConfig and MachineOSBuild enable On-Cluster Layering (OCL) — building custom OS images directly on the cluster. MachineOSConfig defines what to build (pool, image destination, custom Containerfile content). MachineOSBuild represents a specific build execution.

**Key Principle**: One MachineOSConfig per MachineConfigPool (enforced by validation: `metadata.name == spec.machineConfigPool.name`). Each new rendered MachineConfig triggers a new MachineOSBuild.

## MachineOSConfig Spec

```go
type MachineOSConfigSpec struct {
    MachineConfigPool      MachineConfigPoolReference   // Pool name (must match MOSConfig name)
    ImageBuilder           MachineOSImageBuilder        // Builder type (currently only "Job")
    BaseImagePullSecret    *ImageSecretObjectReference   // Pull secret for base image (optional, defaults to cluster-wide)
    RenderedImagePushSecret ImageSecretObjectReference   // Push secret for built image (required, MCO namespace)
    RenderedImagePushSpec  ImageTagFormat                // Registry destination for built image (required)
    Containerfile          []MachineOSContainerfile      // Custom Dockerfile content per arch (max 4)
}
```

## MachineOSBuild Spec

MachineOSBuild spec is **immutable once set**:

```go
type MachineOSBuildSpec struct {
    MachineConfig        MachineConfigReference   // Which rendered MC this build targets
    MachineOSConfig      MachineOSConfigReference // Parent MachineOSConfig
    RenderedImagePushSpec ImageTagFormat           // Where to push the built image
}
```

## Build Lifecycle

1. **Prepared** — Build inputs gathered and validated
2. **Building** — Build Job running (uses buildah in a pod)
3. **Succeeded** — Image pushed to registry, digest available in `status.digestedImagePushSpec`
4. **Failed** / **Interrupted** — Terminal states; conditions become immutable

Once a build reaches `Failed`, `Interrupted`, or `Succeeded`, its status conditions are frozen.

## Key Details

- Builder type is currently only `Job` (pod-based buildah)
- Push and pull secrets must be separate (principle of least privilege)
- Push secret needs write access, only required on MCC node
- Pull secret needs read access, required on all nodes
- Containerfile content max 4096 chars per arch entry
- Supported architectures: `AMD64`, `ARM64`, `PPC64LE`, `S390X`, `NoArch` (default)

## Related Concepts

- [MachineConfigPool](./machineconfigpool.md) — Pool that receives the built image
- [MachineConfig](./machineconfig.md) — Rendered config baked into the image
- [MachineConfigNode](./machineconfignode.md) — Tracks image rollout per node
