# Enhancement Proposals & Design Docs

Catalog of design documentation relevant to the MCO. Enhancement proposals are the source of truth for feature design — this file is an index only.

## Core MCO Enhancements (`enhancements/machine-config/`)

| Title | Status | Link |
|-------|--------|------|
| On-Cluster Layering | graduating | [on-cluster-layering.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/on-cluster-layering.md) |
| MachineConfigNode (MCN) | graduating | [machine-config-node.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/machine-config-node.md) |
| Admin Defined Node Disruption Policy | graduating | [admin-defined-node-disruption-policy.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/admin-defined-node-disruption-policy.md) |
| OS Image Streams | provisional | [os-images-streams.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/os-images-streams.md) |
| Pin and Pre-load Images | graduating | [pin-and-pre-load-images.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/pin-and-pre-load-images.md) |
| Managing Boot Images via MCO | graduating | [manage-boot-images.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/manage-boot-images.md) |
| MachineConfig Irreconcilable Changes | provisional | [machine-config-irreconcilable-changes.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/machine-config-irreconcilable-changes.md) |
| Block runc on RHCOS 10 Upgrade | provisional | [block-runc-on-rhcos10-upgrade.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/block-runc-on-rhcos10-upgrade.md) |
| Install-Time Image Mode | provisional | [install-time-support-for-image-mode-on-openshift.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/install-time-support-for-image-mode-on-openshift.md) |
| Additional Storage Config for CRI-O | implementable | [additional-storage-config-crio.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/additional-storage-config-crio.md) |
| Store User Ignition in MachineConfig | implemented | [custom-ignition-machineconfig.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/custom-ignition-machineconfig.md) |
| Machines user-data Managed by MCO | implementable | [user-data-secret-managed.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/user-data-secret-managed.md) |
| Certificate Authorities for Image Registries | implemented | [certificate-authorities-for-image-registries.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/certificate-authorities-for-image-registries.md) |
| CGROUPSv2 Enablement | implementable | [mco-cgroupsv2-support.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/mco-cgroupsv2-support.md) |
| MCO Network Configuration (Baremetal) | implemented | [mco-network-configuration.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/mco-network-configuration.md) |
| GOMAXPROCS Configuration | provisional | [gomaxprocs-injection.md](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/gomaxprocs-injection.md) |

## Cross-Component Enhancements

| Title | Status | Link |
|-------|--------|------|
| OCP CoreOS Layering (foundational) | implemented | [ocp-coreos-layering.md](https://github.com/openshift/enhancements/blob/master/enhancements/ocp-coreos-layering/ocp-coreos-layering.md) |
| Standardized CoreOS Bootimage Metadata | provisional | [coreos-bootimages.md](https://github.com/openshift/enhancements/blob/master/enhancements/coreos-bootimages.md) |
| Don't Require Registry During Reboot | provisional | [dont-require-registry-during-reboot-and-upgrade.md](https://github.com/openshift/enhancements/blob/master/enhancements/update/dont-require-registry-during-reboot-and-upgrade.md) |
| EUS Upgrades MVP | provisional | [eus-upgrades-mvp.md](https://github.com/openshift/enhancements/blob/master/enhancements/update/eus-upgrades-mvp.md) |
| MachineConfig Support in NTO | implemented | [machine-config-support-in-nto.md](https://github.com/openshift/enhancements/blob/master/enhancements/node-tuning/machine-config-support-in-nto.md) |
| RHCOS Extensions | provisional | [extensions.md](https://github.com/openshift/enhancements/blob/master/enhancements/rhcos/extensions.md) |
| Split RHCOS into Layers | provisional | [split-rhcos-into-layers.md](https://github.com/openshift/enhancements/blob/master/enhancements/rhcos/split-rhcos-into-layers.md) |
| Support for Realtime Kernel | provisional | [support-for-realtime-kernel.md](https://github.com/openshift/enhancements/blob/master/enhancements/support-for-realtime-kernel.md) |
| Workload Partitioning | implementable | [management-workload-partitioning.md](https://github.com/openshift/enhancements/blob/master/enhancements/workload-partitioning/management-workload-partitioning.md) |
| Kubelet Auto Node Sizing | implementable | [kubelet-node-sizing.md](https://github.com/openshift/enhancements/blob/master/enhancements/kubelet/kubelet-node-sizing.md) |

## Local Design Docs (`docs/`)

| Document | Purpose |
|----------|---------|
| [MachineConfig.md](../../docs/MachineConfig.md) | MachineConfig object specification |
| [MachineConfigController.md](../../docs/MachineConfigController.md) | MCC architecture and sub-controllers |
| [MachineConfigDaemon.md](../../docs/MachineConfigDaemon.md) | MCD update process and rebootless updates |
| [MachineConfigServer.md](../../docs/MachineConfigServer.md) | MCS Ignition serving to bootstrapping nodes |
| [MachineOSBuilderDesign.md](../../docs/MachineOSBuilderDesign.md) | On-cluster image build (OCL) design |
| [ContainerRuntimeConfigDesign.md](../../docs/ContainerRuntimeConfigDesign.md) | ContainerRuntimeConfig CRD design |
| [KubeletConfigDesign.md](../../docs/KubeletConfigDesign.md) | KubeletConfig CRD design |
| [NodeDisruptionPolicy.md](../../docs/NodeDisruptionPolicy.md) | User-defined node disruption actions |
| [custom-pools.md](../../docs/custom-pools.md) | Custom MachineConfigPool design |
| [ImageMirrorSetDesign.md](../../docs/ImageMirrorSetDesign.md) | IDMS/ITMS mirror configuration |
| [HACKING.md](../../docs/HACKING.md) | Developer guide |
| [FAQ.md](../../docs/FAQ.md) | Frequently asked questions |
