---
tags:
  - kubernetes
  - kubernetes/storage
topic: Storage
---

# Volumes

## Why Volumes Exist

Container filesystems are **ephemeral** -- when a container restarts, its filesystem is wiped clean and rebuilt from the image. This creates two problems:

1. **Data loss** -- any files written at runtime (logs, uploads, database files) disappear on crash or restart.
2. **No sharing** -- containers in the same Pod cannot see each other's filesystems.

A **Volume** solves both problems by providing a directory that lives outside the container's own filesystem layer. It has a defined lifecycle tied to the Pod and can be backed by many different storage types.

```
Pod
+-----------------------------------------+
|  Container A         Container B        |
|  /var/log ──┐    ┌── /data/shared       |
|             │    │                       |
|             v    v                       |
|         [ Volume: shared-data ]         |
|         (survives container restarts,   |
|          destroyed when Pod is deleted) |
+-----------------------------------------+
```

## Volume vs PersistentVolume

| | Volume | PersistentVolume (PV) |
|---|---|---|
| **Lifecycle** | Tied to the Pod -- deleted when the Pod is deleted | Independent of any Pod -- survives Pod deletion |
| **Defined in** | `pod.spec.volumes` | A separate cluster-level resource |
| **Provisioned by** | The developer, inline in the Pod spec | An administrator (static) or a StorageClass (dynamic) |
| **Use case** | Scratch space, config injection, sidecar sharing | Databases, stateful apps, anything that must outlive a Pod |
| **Claim mechanism** | Direct reference in the Pod spec | Developer creates a PersistentVolumeClaim (PVC) |

## Volume Types

| Type | Description | Data persists after Pod deletion? |
|---|---|---|
| `emptyDir` | Temporary directory created when a Pod is assigned to a node. Starts empty. All containers in the Pod can read/write. Useful for scratch space, caches, and inter-container data exchange. | No |
| `hostPath` | Mounts a file or directory from the **node's** filesystem into the Pod. Useful for accessing node-level logs (`/var/log`), Docker internals (`/var/lib/docker`), or device files. **Risk**: ties your Pod to a specific node and can expose the host to security vulnerabilities. Avoid in production unless absolutely necessary. | Yes (on that node) |
| `configMap` | Mounts the key-value pairs of a ConfigMap as files in a directory. Each key becomes a filename, each value becomes the file's content. | N/A (read-only config) |
| `secret` | Same as `configMap` but for sensitive data. Files are backed by tmpfs (RAM) so they are never written to disk on the node. | N/A (read-only config) |
| `projected` | Combines multiple volume sources (configMap, secret, downwardAPI, serviceAccountToken) into a single directory. Useful when a container needs config from several sources in one mount point. | N/A |
| `downwardAPI` | Exposes Pod and container metadata (labels, annotations, resource limits, Pod name, namespace) as files. | N/A |
| `nfs` | Mounts an existing NFS (Network File System) share. Data persists independently. Multiple Pods can mount the same NFS share simultaneously for `ReadWriteMany` access. | Yes |
| `persistentVolumeClaim` | Mounts a PersistentVolume into the Pod. This is the standard way to use durable storage -- see [[Persistent Volumes]]. | Yes |

## Volume Mount YAML Example

This Pod runs two containers that share an `emptyDir` volume. The `writer` container writes data, and the `reader` container reads it.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "echo 'hello from writer' > /output/message.txt && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /output

    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "cat /input/message.txt && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /input
          readOnly: true

  volumes:
    - name: shared-data
      emptyDir: {}
```

Key points:
- `spec.volumes` declares the volume at the Pod level.
- `spec.containers[*].volumeMounts` mounts the volume inside each container.
- The `name` field links a mount to its volume declaration.

## mountPath and subPath

`mountPath` is the directory inside the container where the volume appears. If the directory already exists in the image, the volume **shadows** it completely.

`subPath` mounts a **single file or subdirectory** from the volume instead of the entire volume. This prevents the volume from obscuring other files already at the mount path.

```yaml
# Mount only the "app.conf" key from a ConfigMap
# without hiding everything else in /etc/config/
volumeMounts:
  - name: config-vol
    mountPath: /etc/config/app.conf
    subPath: app.conf
```

| Field | What it does |
|---|---|
| `mountPath` | Where the volume appears inside the container |
| `subPath` | Selects a specific file or subdirectory from the volume to mount at `mountPath` |
| `subPathExpr` | Same as `subPath` but supports environment variable expansion (e.g., `$(POD_NAME)`) |

**Caveat**: volumes mounted with `subPath` do **not** receive updates when the underlying ConfigMap or Secret changes. If you need automatic updates, mount the entire volume and let your app read the specific file.

## readOnly Mounts

Add `readOnly: true` to a volumeMount to prevent the container from writing to the volume. This is a good practice for config files, secrets, and any volume a container should only consume.

```yaml
volumeMounts:
  - name: app-secrets
    mountPath: /etc/secrets
    readOnly: true
```

You can mount the same volume read-write in one container and read-only in another within the same Pod.

## volumeDevices (Raw Block Storage)

By default, volumes are mounted as **filesystems**. Some workloads (databases with custom I/O engines, high-performance storage) need direct access to the raw block device without a filesystem layer.

Use `volumeDevices` instead of `volumeMounts` to expose a PersistentVolume as a raw block device:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: raw-block-demo
spec:
  containers:
    - name: db
      image: custom-db:latest
      volumeDevices:
        - name: raw-pv
          devicePath: /dev/xvda
  volumes:
    - name: raw-pv
      persistentVolumeClaim:
        claimName: raw-block-pvc
```

| Field | Description |
|---|---|
| `volumeDevices[*].devicePath` | The path inside the container where the block device appears (e.g., `/dev/xvda`) |
| `volumeDevices[*].name` | Must match a volume name that references a PVC with `volumeMode: Block` |

The PV and PVC must both set `volumeMode: Block` for this to work. See [[Persistent Volumes]] for the full PV/PVC spec.
