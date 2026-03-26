---
tags:
  - kubernetes
  - kubernetes/workloads
topic: Workloads
---

# Pods

## What is a Pod?

A Pod is the **smallest deployable unit** in Kubernetes. It represents a single instance of a running process in your cluster. A Pod encapsulates one or more containers that:

- Share the same **network namespace** (same IP address, same `localhost`)
- Can share **storage volumes**
- Are always co-located and co-scheduled on the same node

Think of a Pod as a logical host for tightly coupled containers that need to work together.

```
Pod (10.244.1.5)
┌──────────────────────────────────────────────┐
│                                              │
│  ┌──────────────┐    ┌──────────────┐        │
│  │ Container A  │    │ Container B  │        │
│  │ (app)        │    │ (sidecar)    │        │
│  │ port 8080    │    │ port 9090    │        │
│  └──────┬───────┘    └──────┬───────┘        │
│         │                   │                │
│         └───── localhost ───┘                │
│                                              │
│  ┌──────────────────────────────────┐        │
│  │  Shared Volume: /var/log/app     │        │
│  └──────────────────────────────────┘        │
└──────────────────────────────────────────────┘
```

## Pod YAML Manifest

A comprehensive manifest covering the most common fields:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app                    # unique name within the namespace
  namespace: default              # namespace to create the pod in
  labels:                         # key-value pairs for selection and organization
    app: my-app
    version: v1
  annotations:                    # non-identifying metadata
    description: "My application pod"
spec:
  restartPolicy: Always           # Always | OnFailure | Never

  # --- Init containers run before app containers, in order ---
  initContainers:
    - name: init-db-check
      image: busybox:1.36
      command: ['sh', '-c', 'until nslookup db-service; do sleep 2; done']

  # --- Main application containers ---
  containers:
    - name: app
      image: my-app:1.0.0
      ports:
        - containerPort: 8080     # informational — does not publish the port
          protocol: TCP
      env:                        # environment variables
        - name: DB_HOST
          value: "db-service"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      resources:                  # resource requests and limits
        requests:
          cpu: "250m"             # 0.25 CPU cores
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
      livenessProbe:              # restarts container if it fails
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 10
      readinessProbe:             # removes pod from service endpoints if it fails
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
      startupProbe:               # disables liveness/readiness until it succeeds
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 10

  # --- Volumes available to all containers ---
  volumes:
    - name: app-logs
      emptyDir: {}                # ephemeral volume, lives with the pod

  # --- Scheduling constraints ---
  nodeSelector:
    disktype: ssd
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "app"
      effect: "NoSchedule"
  serviceAccountName: my-app-sa
  terminationGracePeriodSeconds: 30   # time to wait after SIGTERM before SIGKILL
```

## Pod Lifecycle Phases

A Pod's `status.phase` field tells you where it is in its lifecycle:

| Phase | Meaning |
|---|---|
| **Pending** | Pod accepted by the cluster but one or more containers are not yet running. Includes time spent downloading images and waiting for scheduling. |
| **Running** | Pod bound to a node, all containers created, at least one is running or starting/restarting. |
| **Succeeded** | All containers terminated successfully (exit code 0) and will not restart. |
| **Failed** | All containers terminated and at least one exited with a non-zero exit code. |
| **Unknown** | Pod state cannot be determined, usually due to a communication error with the node. |

```
                ┌──────────┐
                │ Pending  │
                └────┬─────┘
                     │
              ┌──────▼──────┐
              │   Running   │
              └──┬───────┬──┘
                 │       │
         ┌───────▼──┐ ┌──▼───────┐
         │Succeeded │ │  Failed  │
         └──────────┘ └──────────┘
```

## Container States

Each container within a Pod has its own state, visible in `status.containerStatuses`:

| State | Description |
|---|---|
| **Waiting** | Container is not yet running. Reasons include pulling the image, applying secrets, etc. The `reason` field provides details (e.g., `ContainerCreating`, `CrashLoopBackOff`, `ImagePullBackOff`). |
| **Running** | Container is executing. The `startedAt` field shows when it started. |
| **Terminated** | Container finished execution. The `exitCode`, `reason`, and `startedAt`/`finishedAt` fields provide details. |

Check container states:

```bash
kubectl describe pod my-app
kubectl get pod my-app -o jsonpath='{.status.containerStatuses}'
```

## Init Containers

Init containers run **before** any app containers start. They are defined in `spec.initContainers` and are useful for setup tasks that must complete before the main application runs.

**Key properties:**

- Run **sequentially** in the order defined (init-1 finishes, then init-2 starts, etc.)
- Each must complete successfully (exit 0) before the next one starts
- If an init container fails, Kubernetes restarts the Pod (subject to `restartPolicy`)
- Do not support `livenessProbe`, `readinessProbe`, or `startupProbe`
- Re-run on Pod restart

**Common use cases:**

- Wait for a dependency to be available (database, external service)
- Run database migrations
- Populate a shared volume with configuration files
- Register the Pod with an external system

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
    - name: run-migrations
      image: my-app:1.0.0
      command: ['./migrate', '--up']
  containers:
    - name: app
      image: my-app:1.0.0
```

## Sidecar Containers

Starting in Kubernetes 1.28 (stable in 1.29), you can define **native sidecar containers** using `restartPolicy: Always` inside `initContainers`. These start before the main containers and run alongside them for the lifetime of the Pod.

```yaml
spec:
  initContainers:
    - name: log-shipper
      image: fluent-bit:latest
      restartPolicy: Always       # makes this a sidecar — runs for the pod's lifetime
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
  containers:
    - name: app
      image: my-app:1.0.0
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
```

Native sidecars are better than regular multi-container patterns for Jobs because the sidecar terminates automatically when the main container finishes.

## Multi-Container Patterns

When a Pod runs multiple containers, they typically follow one of three patterns:

### Sidecar

An auxiliary container that extends or enhances the main container without the main container knowing about it.

```
┌─────────────────────────────────┐
│ Pod                             │
│  ┌─────────┐   ┌────────────┐  │
│  │  App    │──►│ Log Shipper│──► External logging
│  │         │   │ (sidecar)  │  │
│  └─────────┘   └────────────┘  │
│       shared volume: /logs      │
└─────────────────────────────────┘
```

**Examples:** log collectors, file synchronizers, configuration watchers.

### Ambassador

A proxy container that simplifies access to external services on behalf of the main container.

```
┌──────────────────────────────────┐
│ Pod                              │
│  ┌─────────┐   ┌─────────────┐  │
│  │  App    │──►│ Ambassador  │──► Database cluster
│  │localhost │   │ (proxy)     │  │  (routes to correct shard)
│  │ :5432   │   │ :5432       │  │
│  └─────────┘   └─────────────┘  │
└──────────────────────────────────┘
```

**Examples:** database proxies (PgBouncer, Cloud SQL Auth Proxy), service mesh proxies.

### Adapter

A container that transforms the output of the main container into a standardized format.

```
┌──────────────────────────────────┐
│ Pod                              │
│  ┌─────────┐   ┌─────────────┐  │
│  │  App    │──►│  Adapter    │──► Prometheus
│  │ (custom │   │ (transforms │  │  (/metrics)
│  │  logs)  │   │  to metrics)│  │
│  └─────────┘   └─────────────┘  │
└──────────────────────────────────┘
```

**Examples:** Prometheus exporters, log format converters.

## Pod Restart Policies

The `spec.restartPolicy` field controls what happens when containers in a Pod exit:

| Policy | Behavior | Used by |
|---|---|---|
| **Always** (default) | Always restart containers regardless of exit code | Deployments, ReplicaSets, DaemonSets |
| **OnFailure** | Restart only if the container exits with a non-zero code | Jobs |
| **Never** | Never restart containers | Jobs (for debugging) |

The restart policy applies to **all containers in the Pod**. The kubelet restarts containers with an exponential back-off delay (10s, 20s, 40s, ...) capped at 5 minutes, which resets after 10 minutes of successful execution.

## Resource Requests and Limits

Every container can declare CPU and memory **requests** (guaranteed minimum) and **limits** (enforced maximum):

```yaml
resources:
  requests:
    cpu: "250m"       # 250 millicores = 0.25 CPU
    memory: "128Mi"   # 128 mebibytes
  limits:
    cpu: "500m"
    memory: "256Mi"
```

| Concept | CPU | Memory |
|---|---|---|
| **Unit** | Millicores (`m`) or cores | Bytes (`Ki`, `Mi`, `Gi`) |
| **Request** | Used by the scheduler to find a node with enough capacity | Same |
| **Limit** | Container is throttled (not killed) if it exceeds the limit | Container is OOM-killed if it exceeds the limit |
| **No limit set** | Container can use all available CPU on the node | Container can use all available memory (risky) |
| **No request set** | Defaults to the limit value if a limit is set | Same |

**Best practice:** Always set both requests and limits. Set requests to what the container actually needs under normal load, and limits to what it might need under peak load.

## Pod DNS and Hostname

Kubernetes creates DNS records for Pods automatically:

- **Pod hostname:** defaults to `metadata.name`; override with `spec.hostname`
- **Subdomain:** set `spec.subdomain` to group Pods under a DNS name
- **DNS policy:** `spec.dnsPolicy` controls how DNS resolution works

| DNS Policy | Behavior |
|---|---|
| **ClusterFirst** (default) | Queries go to the cluster DNS service first, then upstream |
| **Default** | Inherits DNS config from the node |
| **None** | Ignores cluster DNS; you must provide `spec.dnsConfig` |
| **ClusterFirstWithHostNet** | Like ClusterFirst, but for Pods using `hostNetwork: true` |

Pod DNS record format:

```
<pod-ip-with-dashes>.<namespace>.pod.cluster.local

# Example: Pod with IP 10.244.1.5 in namespace "default"
10-244-1-5.default.pod.cluster.local
```

## Static Pods

Static Pods are managed directly by the **kubelet** on a specific node, without the API server observing them. The kubelet watches a directory (usually `/etc/kubernetes/manifests`) and automatically creates/destroys Pods based on the YAML files it finds there.

**Key characteristics:**

- Not managed by the control plane (no Deployment, ReplicaSet, etc.)
- The kubelet creates a **mirror Pod** on the API server so `kubectl` can see it, but you cannot control it from the API server
- If the static Pod crashes, the kubelet restarts it
- Cannot reference ConfigMaps, Secrets, or ServiceAccounts from the API server

**Common use cases:** Running control plane components (`kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`) on master nodes in kubeadm-based clusters.

```bash
# Default static pod manifest path
ls /etc/kubernetes/manifests/

# Check kubelet config for the path
grep staticPodPath /var/lib/kubelet/config.yaml
```

## When to Use Bare Pods vs Controllers

| Approach | When to use |
|---|---|
| **Bare Pod** | One-off debugging, quick tests, learning. Never in production. |
| **Deployment** | Stateless applications (web servers, APIs, microservices). The default choice. |
| **StatefulSet** | Stateful applications needing stable identity and persistent storage (databases, message queues). |
| **DaemonSet** | One pod per node (log collectors, monitoring agents, network plugins). |
| **Job** | Run-to-completion batch tasks. |
| **CronJob** | Scheduled batch tasks. |

Bare Pods are **not rescheduled** if a node fails. Controllers (Deployments, StatefulSets, etc.) detect the failure and recreate Pods on healthy nodes. Always use a controller for production workloads.
