---
tags:
  - kubernetes
  - kubernetes/security
topic: Security
---

# Security Contexts

## Overview

A **security context** defines privilege and access control settings for a Pod or container. These settings control the Linux kernel-level security mechanisms that constrain what a process can do on the node.

Settings can be applied at two levels:

| Level | Field | Applies to |
|---|---|---|
| **Pod-level** | `spec.securityContext` | All containers in the Pod (init, sidecar, and regular) |
| **Container-level** | `spec.containers[*].securityContext` | A single container |

When both are set, the **container-level setting wins** for fields that exist at both levels.

```
  Pod securityContext          Container securityContext
  (applies to all containers)  (overrides per container)
  ┌─────────────────────────┐  ┌─────────────────────────┐
  │ runAsUser: 1000         │  │ runAsUser: 2000         │  ◄── container wins (2000)
  │ runAsGroup: 3000        │  │                         │  ◄── pod applies (3000)
  │ fsGroup: 2000           │  │ readOnlyRootFilesystem: │
  │ runAsNonRoot: true      │  │   true                  │
  └─────────────────────────┘  └─────────────────────────┘
```

## Pod-Level Settings

### runAsUser and runAsGroup

Set the UID and primary GID for all containers in the Pod. Processes inside the container run as this user/group instead of whatever is defined in the container image.

```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
```

### fsGroup

Sets the group ownership of all volume mounts. When a Pod mounts a volume, Kubernetes chowns the files to `fsGroup` and sets the group permission bits so the Pod's processes can read/write them.

```yaml
spec:
  securityContext:
    fsGroup: 2000
```

This is critical for StatefulSets and any workload using PersistentVolumeClaims -- without it, the container process may not have permission to write to the mounted volume.

### supplementalGroups

Adds additional group IDs to the container's process beyond `runAsGroup`.

```yaml
spec:
  securityContext:
    runAsGroup: 1000
    supplementalGroups: [4000, 5000]
```

### runAsNonRoot

When set to `true`, the kubelet validates that the container is not running as UID 0 at startup. If the image runs as root and you haven't set `runAsUser`, the Pod will fail to start.

```yaml
spec:
  securityContext:
    runAsNonRoot: true
```

## Container-Level Settings

### readOnlyRootFilesystem

Mounts the container's root filesystem as read-only. The process cannot write to the filesystem -- any writes must go to explicitly mounted volumes (emptyDir, PVC, etc.).

```yaml
spec:
  containers:
    - name: app
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

This prevents attackers from modifying binaries or dropping files into the container's filesystem. Applications that need to write temp files, logs, or caches should use mounted volumes for those paths.

### allowPrivilegeEscalation

Controls whether a process can gain more privileges than its parent. Setting this to `false` prevents the use of `setuid` binaries and the `PR_SET_NO_NEW_PRIVS` Linux prctl flag.

```yaml
spec:
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
```

### capabilities

Linux capabilities split root privileges into fine-grained units. Instead of running as root (which grants all capabilities), you can drop all capabilities and add back only the ones your application needs.

```yaml
spec:
  containers:
    - name: app
      securityContext:
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE    # bind to ports < 1024
```

Common capabilities:

| Capability | What it allows |
|---|---|
| `NET_BIND_SERVICE` | Bind to privileged ports (below 1024) |
| `NET_RAW` | Use raw sockets (needed by `ping`, some network tools) |
| `SYS_PTRACE` | Trace processes (debugging tools) |
| `SYS_ADMIN` | Broad admin capability -- avoid granting this |
| `CHOWN` | Change file ownership |
| `DAC_OVERRIDE` | Bypass file permission checks |
| `SETUID` / `SETGID` | Change process UID/GID |

Best practice: **drop ALL capabilities** and add back only what you need.

### Privileged Containers

A privileged container has almost unrestricted access to the host. It gets all Linux capabilities, access to all devices, and can modify host kernel parameters.

```yaml
spec:
  containers:
    - name: debug
      securityContext:
        privileged: true     # avoid this in production
```

**Why to avoid privileged containers:** A compromised privileged container has effectively root access to the node. It can read all secrets, escape to the host filesystem, access the container runtime socket, and compromise other Pods on the same node. Only use privileged mode for infrastructure-level DaemonSets that genuinely need host access (CNI plugins, storage drivers, node monitoring agents).

### seccompProfile

Seccomp (secure computing mode) restricts which Linux syscalls a container can make. This limits the kernel attack surface.

```yaml
spec:
  containers:
    - name: app
      securityContext:
        seccompProfile:
          type: RuntimeDefault    # use the container runtime's default profile
```

| Type | Description |
|---|---|
| `RuntimeDefault` | Uses the container runtime's default seccomp profile (blocks ~60 dangerous syscalls). Recommended baseline. |
| `Localhost` | Uses a custom profile from the node's filesystem (`/var/lib/kubelet/seccomp/`). |
| `Unconfined` | No seccomp filtering. Not recommended. |

```yaml
# Using a custom seccomp profile
spec:
  containers:
    - name: app
      securityContext:
        seccompProfile:
          type: Localhost
          localhostProfile: profiles/my-app.json
```

### SELinux Options

SELinux provides mandatory access control on systems where it is enabled (RHEL, CentOS, Fedora). You can set SELinux labels on containers.

```yaml
spec:
  containers:
    - name: app
      securityContext:
        seLinuxOptions:
          level: "s0:c123,c456"
          type: "container_t"
          user: "system_u"
          role: "system_r"
```

Most clusters running in cloud environments don't use SELinux, but it's common in enterprise and government environments. If you're not sure whether your cluster uses SELinux, check with `getenforce` on the nodes.

## Complete Example: Hardened Pod

This example applies all recommended security settings:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
  namespace: prod
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false    # don't mount SA token if not needed
  securityContext:
    runAsUser: 10000
    runAsGroup: 10000
    fsGroup: 10000
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: my-app:1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      resources:
        limits:
          memory: "256Mi"
          cpu: "500m"
        requests:
          memory: "128Mi"
          cpu: "100m"
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
  volumes:
    - name: tmp
      emptyDir:
        sizeLimit: 64Mi
    - name: cache
      emptyDir:
        sizeLimit: 128Mi
```

## Best Practices Summary

| Practice | Setting | Why |
|---|---|---|
| Run as non-root | `runAsNonRoot: true`, `runAsUser: <non-zero>` | Prevents container processes from having root privileges |
| Drop all capabilities | `capabilities.drop: [ALL]` | Removes all privileged kernel operations by default |
| Read-only root filesystem | `readOnlyRootFilesystem: true` | Prevents filesystem tampering; use emptyDir for writable paths |
| No privilege escalation | `allowPrivilegeEscalation: false` | Prevents setuid binaries from granting elevated privileges |
| Use seccomp | `seccompProfile.type: RuntimeDefault` | Blocks dangerous syscalls at the kernel level |
| Never use privileged | `privileged: false` (default) | Privileged containers can escape to the host |
| Set resource limits | `resources.limits` | Prevents resource exhaustion (not strictly security context, but complements it) |
