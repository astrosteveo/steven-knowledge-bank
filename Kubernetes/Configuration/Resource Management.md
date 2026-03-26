---
tags:
  - kubernetes
  - kubernetes/configuration
topic: Configuration
---

# Resource Management

## Why Resource Management Matters

Every container in Kubernetes runs on a node with finite CPU and memory. Without resource management, a single runaway container can starve everything else on the node — including system daemons and other applications. Resource requests and limits let you tell Kubernetes how much CPU and memory each container needs and how much it's allowed to use.

```
Node Capacity (e.g., 4 CPU, 16 Gi memory)
┌────────────────────────────────────────────────────┐
│  System reserved   │  kube-reserved  │  Allocatable │
│  (OS daemons)      │  (kubelet, etc) │  (for Pods)  │
└────────────────────────────────────────────────────┘
                                        ▲
                    Pods compete for this portion
```

## Requests and Limits

| | Request | Limit |
|---|---|---|
| **What it means** | The **minimum** resources the container needs to run | The **maximum** resources the container is allowed to use |
| **Used by** | The **scheduler** — to decide which node has enough room | The **kubelet** — to enforce a ceiling at runtime |
| **What happens if unset** | Container gets whatever's available (no guarantees) | Container can consume unlimited resources on the node |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:1.0
      resources:
        requests:
          cpu: "250m"       # 250 millicores = 0.25 CPU
          memory: "128Mi"   # 128 mebibytes
        limits:
          cpu: "500m"       # 500 millicores = 0.5 CPU
          memory: "256Mi"   # 256 mebibytes
```

## Resource Units

### CPU

CPU is measured in **cores**. Kubernetes uses millicores for precision:

| Value | Meaning |
|---|---|
| `1` | 1 full CPU core |
| `"500m"` | 500 millicores = 0.5 CPU core |
| `"250m"` | 250 millicores = 0.25 CPU core |
| `"100m"` | 100 millicores = 0.1 CPU core |
| `0.1` | Same as `"100m"` |

One CPU core in Kubernetes is equivalent to 1 vCPU (AWS), 1 Core (GCP), 1 vCore (Azure), or 1 hyperthread on bare metal.

### Memory

Memory is measured in **bytes** with standard SI or binary suffixes:

| Suffix | Type | Meaning |
|---|---|---|
| `Ki` | Binary | Kibibyte (1024 bytes) |
| `Mi` | Binary | Mebibyte (1024^2 = 1,048,576 bytes) |
| `Gi` | Binary | Gibibyte (1024^3 bytes) |
| `K` | Decimal | Kilobyte (1000 bytes) |
| `M` | Decimal | Megabyte (1000^2 bytes) |
| `G` | Decimal | Gigabyte (1000^3 bytes) |

**Use binary suffixes** (`Mi`, `Gi`) — they're the convention in Kubernetes and match how operating systems actually report memory.

## QoS Classes

Kubernetes assigns a **Quality of Service** class to every Pod based on its resource configuration. QoS determines which Pods get killed first when a node runs out of memory:

| QoS Class | Criteria | Eviction Priority |
|---|---|---|
| **Guaranteed** | Every container has `requests` == `limits` for both CPU and memory | Last to be killed (highest priority) |
| **Burstable** | At least one container has a `request` or `limit` set, but they're not all equal | Killed after BestEffort |
| **BestEffort** | No container has any `requests` or `limits` set | First to be killed (lowest priority) |

### Guaranteed

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"       # must equal request
    memory: "256Mi"   # must equal request
```

### Burstable

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"       # different from request
    memory: "256Mi"   # different from request
```

### BestEffort

```yaml
# No resources block at all
containers:
  - name: app
    image: myapp:1.0
```

## How Scheduling Uses Requests

The scheduler places Pods on nodes based on **requests**, not limits. It sums the requests of all Pods on a node and compares that to the node's allocatable capacity:

```
Node: 4 CPU, 8 Gi allocatable
├── Pod A: requests 1 CPU, 2 Gi     ── scheduled
├── Pod B: requests 1.5 CPU, 3 Gi   ── scheduled
├── Pod C: requests 1 CPU, 2 Gi     ── scheduled
│                                    ── Total: 3.5 CPU, 7 Gi
└── Pod D: requests 1 CPU, 2 Gi     ── PENDING (insufficient memory)
```

Even if actual usage is low, the scheduler reserves capacity based on requests. This means over-requesting wastes cluster resources, and under-requesting leads to overcommitment and potential OOM kills.

## What Happens When Limits Are Exceeded

### CPU: Throttling

When a container tries to use more CPU than its limit, the kernel **throttles** it — the container is forced to wait. It doesn't get killed, but it runs slower. This can cause latency spikes that are hard to diagnose.

### Memory: OOM Kill

When a container exceeds its memory limit, the kernel's OOM (Out of Memory) killer **terminates** the container. Kubernetes then restarts it according to the Pod's `restartPolicy`.

```
CPU over limit  ──►  Throttled (slowed down, still running)
Memory over limit ──►  OOMKilled (container terminated, restarted)
```

You'll see `OOMKilled` as the container's last termination reason:

```bash
kubectl describe pod app
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

## LimitRange

A **LimitRange** sets default, minimum, and maximum resource constraints for individual Pods or containers within a namespace. It prevents any single container from requesting too much (or too little).

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
    # Defaults applied to containers that don't specify their own
    - type: Container
      default:           # default limits
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:    # default requests
        cpu: "100m"
        memory: "128Mi"
      min:               # minimum allowed
        cpu: "50m"
        memory: "64Mi"
      max:               # maximum allowed
        cpu: "2"
        memory: "2Gi"

    # Limits on the total resources of a single Pod
    - type: Pod
      max:
        cpu: "4"
        memory: "4Gi"
```

| Field | Effect |
|---|---|
| `default` | Sets default **limits** for containers that don't specify any |
| `defaultRequest` | Sets default **requests** for containers that don't specify any |
| `min` | Rejects Pods where any container requests less than this |
| `max` | Rejects Pods where any container requests/limits more than this |
| `type: Pod` | Applies to the sum of all containers in a Pod |

LimitRange is enforced at admission time — it only affects Pods created **after** the LimitRange exists.

## ResourceQuota

A **ResourceQuota** limits the **total** resources that can be consumed across all Pods in a namespace. It prevents one team or environment from consuming the entire cluster.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    # Compute resources
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"

    # Object counts
    pods: "50"
    services: "20"
    configmaps: "100"
    secrets: "100"
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"
    services.nodeports: "5"
```

Check current usage against the quota:

```bash
kubectl describe resourcequota team-alpha-quota -n team-alpha
# Name:                  team-alpha-quota
# Namespace:             team-alpha
# Resource               Used    Hard
# --------               ----    ----
# requests.cpu           3500m   10
# requests.memory        8Gi     20Gi
# limits.cpu             7       20
# limits.memory          16Gi    40Gi
# pods                   12      50
```

**Important:** when a ResourceQuota for compute resources exists in a namespace, every Pod in that namespace **must** specify resource requests and limits (or have them injected by a LimitRange). Pods without them are rejected.

## LimitRange vs ResourceQuota

| | LimitRange | ResourceQuota |
|---|---|---|
| **Scope** | Individual container or Pod | Entire namespace (aggregate) |
| **Purpose** | Set defaults, enforce min/max per workload | Cap total resource consumption |
| **Example use** | "No single container can use more than 2 Gi" | "This namespace can use at most 40 Gi total" |
| **Enforcement** | Admission time (Pod creation) | Admission time (Pod creation) |

They work well together — use LimitRange to set sane defaults and guard rails per container, and ResourceQuota to cap the namespace total.

## Best Practices

1. **Always set requests** — without them, the scheduler can't make informed decisions and Pods get BestEffort QoS (first to be evicted).

2. **Set requests based on actual usage** — monitor your workloads with `kubectl top` or a metrics stack (Prometheus/Grafana) and set requests to the P95 or P99 of observed usage.

3. **Be cautious with CPU limits** — CPU throttling causes hard-to-diagnose latency. Many teams set CPU requests but omit CPU limits, letting containers burst when cores are available. Set CPU limits for multi-tenant clusters where fairness matters.

4. **Always set memory limits** — unlike CPU (throttled), exceeding memory limits kills the container. Without a limit, a memory leak takes down the entire node.

5. **Use Guaranteed QoS for critical workloads** — set requests equal to limits for your most important services so they're last to be evicted.

6. **Deploy LimitRange in every namespace** — ensures no Pod is created without resource constraints, even if a developer forgets.

7. **Use ResourceQuota for multi-tenant clusters** — prevents any single team from consuming disproportionate resources.

8. **Leave headroom between requests and node capacity** — don't pack nodes to 100% of allocatable. Leave 10-20% headroom for burst traffic and system overhead.

9. **Monitor for OOMKilled and throttling** — both are signals that your resource configuration needs adjustment.
