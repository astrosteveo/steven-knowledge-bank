---
tags:
  - kubernetes
  - kubernetes/storage
topic: Storage
---

# Persistent Volumes

## The PV and PVC Model

Kubernetes separates **storage provisioning** from **storage consumption** using two resources:

- **PersistentVolume (PV)** -- a piece of storage in the cluster, provisioned by an administrator or dynamically by a [[Storage Classes|StorageClass]]. It is a cluster-level resource (no namespace).
- **PersistentVolumeClaim (PVC)** -- a request for storage made by a developer. It lives in a namespace and binds to a matching PV.

```
Administrator / StorageClass              Developer
         │                                     │
         ▼                                     ▼
   ┌───────────┐    binding           ┌──────────────┐
   │     PV    │◄────────────────────►│     PVC      │
   │  10Gi RWO │   (auto-matched     │  request 5Gi │
   │  gp3-csi  │    by size, access   │  RWO         │
   └───────────┘    mode, class)      └──────┬───────┘
                                             │
                                             ▼
                                     ┌──────────────┐
                                     │     Pod      │
                                     │  volumes:    │
                                     │  - pvc:      │
                                     │    claimName │
                                     └──────────────┘
```

The separation means developers never need to know the details of the underlying storage -- they just ask for "10Gi of ReadWriteOnce storage" and Kubernetes handles the rest.

## PersistentVolume YAML

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
  labels:
    type: ssd
spec:
  capacity:
    storage: 50Gi                      # total size of the volume
  volumeMode: Filesystem               # Filesystem (default) or Block
  accessModes:
    - ReadWriteOnce                    # see access modes table below
  persistentVolumeReclaimPolicy: Retain # what happens when PVC is deleted
  storageClassName: gp3-csi            # must match the PVC's class (or empty for manual binding)
  mountOptions:
    - hard
    - nfsvers=4.1
  nodeAffinity:                        # constrain which nodes can access this PV
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node-1
  # The source -- exactly one of these depending on the storage backend:
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0abc123def456
    fsType: ext4
  # Other source examples (mutually exclusive with csi above):
  # nfs:
  #   server: nfs.example.com
  #   path: /exports/data
  # hostPath:
  #   path: /mnt/data
```

| Field | Description |
|---|---|
| `capacity.storage` | The size of the volume |
| `volumeMode` | `Filesystem` (mounted as a directory) or `Block` (raw block device) |
| `accessModes` | How the volume can be mounted -- see table below |
| `persistentVolumeReclaimPolicy` | What to do with the PV after the PVC is deleted |
| `storageClassName` | Links the PV to a StorageClass. Set to `""` to opt out of dynamic provisioning |
| `mountOptions` | Filesystem-specific mount flags |
| `nodeAffinity` | Restricts which nodes can access this PV (important for local volumes) |

## PersistentVolumeClaim YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
  namespace: app
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 20Gi                    # minimum size requested
    limits:
      storage: 50Gi                    # maximum (alpha, not widely supported)
  storageClassName: gp3-csi            # match a StorageClass or specific PV
  selector:                            # optional: further filter which PVs can bind
    matchLabels:
      type: ssd
    matchExpressions:
      - key: environment
        operator: In
        values:
          - production
  dataSource:                          # clone from an existing PVC or snapshot
    kind: VolumeSnapshot               # or PersistentVolumeClaim
    name: my-snapshot
    apiGroup: snapshot.storage.k8s.io
```

| Field | Description |
|---|---|
| `accessModes` | Must be a subset of the PV's access modes |
| `resources.requests.storage` | Minimum storage the PVC needs. Kubernetes binds to a PV of equal or greater size |
| `storageClassName` | If set, only PVs with the same class are considered. If set to `""`, only PVs with no class are considered. If omitted, the default StorageClass is used |
| `selector` | Label selector to restrict which PVs this PVC can bind to |
| `dataSource` | Pre-populate the volume from a snapshot or existing PVC (cloning) |

## Access Modes

| Mode | Abbreviation | Description |
|---|---|---|
| `ReadWriteOnce` | RWO | Mounted as read-write by a **single node**. Multiple Pods on the same node can access it. |
| `ReadOnlyMany` | ROX | Mounted as read-only by **many nodes** simultaneously. |
| `ReadWriteMany` | RWX | Mounted as read-write by **many nodes** simultaneously. Requires a backend that supports it (NFS, CephFS, EFS). |
| `ReadWriteOncePod` | RWOP | Mounted as read-write by a **single Pod** in the entire cluster. Guarantees only one Pod writes. Requires Kubernetes 1.27+ and a CSI driver that supports it. |

Not every storage backend supports every mode. Always check your provisioner's documentation.

## Reclaim Policies

The reclaim policy determines what happens to the PV (and its underlying storage) when the PVC bound to it is deleted.

| Policy | Behavior | Use case |
|---|---|---|
| `Retain` | PV is kept and moves to `Released` state. Data is preserved. An admin must manually clean up and re-provision. | Production data you cannot afford to lose |
| `Delete` | PV and the underlying storage asset (e.g., EBS volume, GCE disk) are deleted automatically. | Ephemeral or easily reproducible data |
| `Recycle` | (**Deprecated**) Runs `rm -rf /thevolume/*` and makes the PV available again. Use dynamic provisioning instead. | Legacy clusters only |

## Volume Binding Modes

| Mode | Behavior |
|---|---|
| `Immediate` | The PV is bound to the PVC as soon as the PVC is created, regardless of whether a Pod has been scheduled. This can lead to problems in multi-zone clusters where the PV is provisioned in a zone different from where the Pod lands. |
| `WaitForFirstConsumer` | Binding and provisioning are delayed until a Pod that uses the PVC is scheduled. The scheduler picks the zone/node first, then the PV is provisioned in the right topology. **Recommended for most workloads.** |

## PV Lifecycle

```
                    PVC created
                    and matches
  ┌───────────┐    ──────────►    ┌────────┐
  │ Available  │                  │  Bound  │
  └───────────┘                   └───┬────┘
       ▲                              │
       │ admin reclaims               │ PVC deleted
       │ and cleans up                ▼
       │                        ┌───────────┐
       └──────────────────────  │ Released   │
                                └─────┬─────┘
                                      │
                              (reclaim policy)
                                      │
                    ┌─────────────────┼──────────────────┐
                    ▼                 ▼                   ▼
              Retain (manual)   Delete (auto)    Recycle (deprecated)
              data preserved    PV + storage     rm -rf, PV re-available
                                destroyed
```

- **Available** -- the PV exists and is not yet claimed.
- **Bound** -- the PV is bound to a PVC.
- **Released** -- the PVC has been deleted but the PV has not yet been reclaimed.
- **Failed** -- automatic reclamation failed.

## Static vs Dynamic Provisioning

**Static provisioning**: an administrator manually creates PV resources ahead of time. Developers create PVCs that match an existing PV by size, access mode, and StorageClass.

**Dynamic provisioning**: no pre-created PVs are needed. When a PVC references a StorageClass, Kubernetes automatically provisions the underlying storage and creates a PV. This is the preferred approach.

```
Static:
  Admin creates PV ──► PVC binds to existing PV

Dynamic:
  PVC references StorageClass ──► StorageClass triggers provisioner
                                  ──► Provisioner creates storage
                                  ──► Kubernetes creates PV
                                  ──► PVC binds to new PV
```

## Using PVCs in Pod Specs

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: data
          mountPath: /var/lib/app/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-example        # references the PVC by name
        readOnly: false               # optional, default is false
```

For StatefulSets, use `volumeClaimTemplates` to automatically create one PVC per replica:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: postgres:16
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-csi
        resources:
          requests:
            storage: 100Gi
```

This creates PVCs named `pgdata-db-0`, `pgdata-db-1`, `pgdata-db-2` -- one per Pod.

## Expanding PVCs

You can grow a PVC's storage without downtime if the StorageClass has `allowVolumeExpansion: true`.

```bash
kubectl patch pvc pvc-example -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

How expansion works:

1. The controller resizes the underlying volume (e.g., expands the EBS volume).
2. If the volume uses a filesystem, the filesystem is expanded the next time the volume is mounted to a Pod (some CSI drivers do online expansion).
3. The PVC's status updates to reflect the new size under `status.capacity.storage`.

**You cannot shrink a PVC.** If you need less space, create a new PVC and migrate data.

## Volume Snapshots

Volume snapshots let you create a point-in-time copy of a PersistentVolume's data. They require a CSI driver that supports snapshotting and the snapshot CRDs to be installed.

Three resources are involved:

| Resource | Description |
|---|---|
| `VolumeSnapshotClass` | Defines the CSI driver and deletion policy for snapshots (analogous to StorageClass for PVs) |
| `VolumeSnapshot` | A request to snapshot a specific PVC (analogous to PVC) |
| `VolumeSnapshotContent` | The actual snapshot resource (analogous to PV) |

```yaml
# Take a snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: csi-aws-snapclass
  source:
    persistentVolumeClaimName: pvc-example
---
# Restore from a snapshot by setting dataSource on a new PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-restored
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-csi
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```
