---
tags:
  - kubernetes
  - kubernetes/storage
topic: Storage
---

# Storage Classes

## What StorageClasses Enable

Without StorageClasses, an administrator must manually create PersistentVolumes before any Pod can use persistent storage. This does not scale.

A **StorageClass** lets you define a "blueprint" for storage. When a PVC references a StorageClass, Kubernetes automatically provisions the underlying storage and creates a PV -- this is **dynamic provisioning**. Different classes can represent different tiers of storage (fast SSD, cheap HDD, replicated, encrypted).

```
Developer creates PVC               Kubernetes
  storageClassName: fast-ssd            │
          │                             ▼
          │                   StorageClass "fast-ssd"
          │                     provisioner: ebs.csi.aws.com
          │                     parameters:
          │                       type: gp3
          │                       iops: "5000"
          │                             │
          │                             ▼
          │                   Provisioner creates
          │                   cloud disk + PV
          │                             │
          ▼                             ▼
        PVC  ◄──── bound ────►  auto-created PV
```

## StorageClass YAML

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # mark as default
provisioner: ebs.csi.aws.com          # which plugin creates the volumes
parameters:                            # provisioner-specific key-value pairs
  type: gp3
  iops: "5000"
  throughput: "250"
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Delete                  # Delete or Retain (default: Delete)
volumeBindingMode: WaitForFirstConsumer # Immediate or WaitForFirstConsumer
allowVolumeExpansion: true             # allow PVCs to be resized
mountOptions:                          # passed to the mount command
  - discard
  - noatime
allowedTopologies:                     # restrict provisioning to specific zones
  - matchLabelExpressions:
      - key: topology.kubernetes.io/zone
        values:
          - us-east-1a
          - us-east-1b
```

| Field | Description |
|---|---|
| `provisioner` | The volume plugin that creates storage. Can be an in-tree plugin or a CSI driver. |
| `parameters` | Key-value pairs passed to the provisioner. Completely provisioner-specific. |
| `reclaimPolicy` | What happens to dynamically provisioned PVs when the PVC is deleted. `Delete` (default) or `Retain`. |
| `volumeBindingMode` | `Immediate` (provision as soon as PVC is created) or `WaitForFirstConsumer` (delay until a Pod is scheduled). |
| `allowVolumeExpansion` | If `true`, PVCs using this class can be expanded by editing `resources.requests.storage`. |
| `mountOptions` | Filesystem mount flags applied when the volume is mounted. |
| `allowedTopologies` | Restricts provisioning to specific topology segments (zones, regions). |

## Provisioners: In-Tree vs CSI

Historically, Kubernetes shipped volume plugins compiled directly into the kubelet binary -- these are **in-tree provisioners** (e.g., `kubernetes.io/aws-ebs`). They are now deprecated and being migrated to **CSI (Container Storage Interface)** drivers.

| Aspect | In-Tree | CSI |
|---|---|---|
| **Deployed as** | Part of the Kubernetes binary | Separate Pods (DaemonSet + Deployment) |
| **Release cycle** | Tied to Kubernetes releases | Independent -- vendors ship updates on their own schedule |
| **Adding new storage** | Requires changes to Kubernetes core | Anyone can write a CSI driver |
| **Status** | Deprecated, being migrated | The standard going forward |

## Common Provisioners

| Provider | In-Tree (deprecated) | CSI Driver | StorageClass parameter examples |
|---|---|---|---|
| **AWS EBS** | `kubernetes.io/aws-ebs` | `ebs.csi.aws.com` | `type`: gp3, io2, st1; `iops`; `throughput`; `encrypted` |
| **GCE Persistent Disk** | `kubernetes.io/gce-pd` | `pd.csi.storage.gke.io` | `type`: pd-standard, pd-ssd, pd-balanced; `replication-type` |
| **Azure Disk** | `kubernetes.io/azure-disk` | `disk.csi.azure.com` | `skuName`: Premium_LRS, StandardSSD_LRS; `cachingMode` |
| **Azure File** | `kubernetes.io/azure-file` | `file.csi.azure.com` | `skuName`; `secretNamespace` |
| **Local** | -- | `local.csi.k8s.io` (or built-in `no-provisioner`) | Uses `nodeAffinity` on the PV; no dynamic provisioning with the built-in local type |
| **NFS** | `kubernetes.io/nfs` (limited) | `nfs.csi.k8s.io` | `server`; `share` |
| **Ceph RBD** | `kubernetes.io/rbd` | `rbd.csi.ceph.com` | `clusterID`; `pool`; `imageFeatures` |
| **Amazon EFS** | -- | `efs.csi.aws.com` | `provisioningMode`; `fileSystemId`; `directoryPerms` |

## Parameters Field

The `parameters` field is an opaque map of strings passed directly to the provisioner. What you put here depends entirely on the CSI driver.

```yaml
# AWS EBS CSI example
parameters:
  type: gp3
  iops: "5000"
  throughput: "250"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/abc-123"

# GCE PD CSI example
parameters:
  type: pd-ssd
  replication-type: regional-pd

# Azure Disk CSI example
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
```

You can also reference Secrets for sensitive parameters using `csi.storage.k8s.io/provisioner-secret-name` and related keys.

## Reclaim Policy

StorageClasses set the reclaim policy for all PVs they dynamically create.

| Policy | Behavior |
|---|---|
| `Delete` | **(Default)** The PV and its backing storage are deleted when the PVC is deleted. Use for ephemeral or easily-recreated data. |
| `Retain` | The PV is kept after the PVC is deleted. An admin must manually clean up. Use for data you cannot afford to lose. |

You can change a PV's reclaim policy after creation with `kubectl patch`, but the StorageClass sets the initial value.

## Volume Binding Mode

| Mode | When PV is provisioned | Best for |
|---|---|---|
| `Immediate` | As soon as the PVC is created | Single-zone clusters or when you know the topology upfront |
| `WaitForFirstConsumer` | After a Pod using the PVC is scheduled to a node | **Multi-zone clusters** -- ensures the volume is created in the same zone as the Pod |

`WaitForFirstConsumer` is the recommended default. Without it, a PV might be provisioned in `us-east-1a` while the scheduler places the Pod in `us-east-1b`, causing the Pod to be unschedulable.

## Allow Volume Expansion

Set `allowVolumeExpansion: true` on the StorageClass to let users grow PVCs after creation.

```yaml
allowVolumeExpansion: true
```

When a user edits a PVC to request more storage:

1. The CSI driver expands the underlying volume (e.g., modifies the EBS volume size).
2. On the next mount (or online, if the driver supports it), the filesystem is grown.
3. The PVC's `status.capacity` updates to reflect the new size.

Shrinking is not supported.

## Default StorageClass

Mark a StorageClass as the **default** by setting this annotation:

```yaml
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
```

When a PVC does **not** specify a `storageClassName`, Kubernetes uses the default class. If no default exists, the PVC will remain `Pending` until a matching PV is manually created.

Only one StorageClass should be marked as default. If multiple are marked, the behavior is undefined and varies by cluster version.

```bash
# Check which StorageClass is the default
kubectl get storageclass
# The default class is marked with "(default)" in the output
```

## CSI (Container Storage Interface)

CSI is an industry standard that defines how container orchestrators (Kubernetes, Mesos, etc.) communicate with storage providers. It replaced the old in-tree volume plugin model.

### Why CSI Exists

- **Decoupling** -- storage vendors ship drivers independently of Kubernetes releases.
- **Extensibility** -- anyone can write a CSI driver; no changes to Kubernetes core required.
- **Standardization** -- the same driver interface works across orchestrators.

### How CSI Drivers Work

A CSI driver is deployed as a set of Pods in the cluster:

```
┌─────────────────────────────────────────────────┐
│  CSI Driver Deployment                          │
│                                                 │
│  ┌───────────────────┐  ┌────────────────────┐  │
│  │  Controller Plugin │  │   Node Plugin      │  │
│  │  (Deployment)      │  │   (DaemonSet)      │  │
│  │                    │  │                    │  │
│  │  - CreateVolume    │  │  - NodeStageVolume │  │
│  │  - DeleteVolume    │  │  - NodePublish     │  │
│  │  - ControllerPubl │  │    Volume           │  │
│  │    ishVolume       │  │  - NodeUnpublish   │  │
│  │  - CreateSnapshot  │  │    Volume           │  │
│  └───────────────────┘  └────────────────────┘  │
│                                                 │
│  Sidecar containers (from Kubernetes SIG):      │
│  - external-provisioner  (watches PVCs)         │
│  - external-attacher     (attaches volumes)     │
│  - external-snapshotter  (handles snapshots)    │
│  - external-resizer      (handles expansion)    │
│  - node-driver-registrar (registers with        │
│                           kubelet)              │
└─────────────────────────────────────────────────┘
```

| Component | Runs as | Responsibility |
|---|---|---|
| **Controller Plugin** | Deployment (1-3 replicas) | Manages volume lifecycle: create, delete, attach/detach, snapshot |
| **Node Plugin** | DaemonSet (every node) | Mounts/unmounts volumes on the node where Pods run |
| **Sidecar containers** | Alongside the controller/node Pods | Bridge between Kubernetes API objects (PVC, VolumeAttachment) and the CSI driver's gRPC interface |

### CSI Driver Installation

Most CSI drivers are installed via Helm or operator:

```bash
# Example: AWS EBS CSI driver via Helm
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system
```

After installation, the driver registers a `CSIDriver` object:

```bash
kubectl get csidriver
# NAME                  ATTACHREQUIRED   PODINFOONMOUNT   MODES
# ebs.csi.aws.com       true             false            Persistent
```

You can then reference the driver name in your StorageClass's `provisioner` field.
