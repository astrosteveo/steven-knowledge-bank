---
tags:
  - kubernetes
  - kubernetes/scheduling
topic: Scheduling
---

# Pod Affinity and Anti-Affinity

Pod affinity and anti-affinity let you constrain scheduling based on **which other Pods are already running on a node (or topology domain)**. Where node affinity says "put me on a node with these labels," pod affinity says "put me near (or away from) Pods with these labels."

## Pod Affinity (Co-locate Pods)

Pod affinity attracts a Pod toward nodes where Pods matching a label selector are already running. Common use case: co-locate a frontend with its cache for low-latency communication.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - redis-cache
          topologyKey: kubernetes.io/hostname
  containers:
    - name: frontend
      image: frontend:1.0
```

This says: "Schedule this Pod only on a node where a Pod with `app=redis-cache` is already running."

## Pod Anti-Affinity (Spread Pods Apart)

Pod anti-affinity repels a Pod away from nodes where matching Pods exist. Common use case: spread replicas of the same application across nodes or zones for high availability.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - web
              topologyKey: kubernetes.io/hostname
      containers:
        - name: web
          image: web:1.0
```

This says: "Never place two Pods with `app=web` on the same node." With 3 replicas and this rule, each replica lands on a different node. If there aren't enough nodes, excess Pods stay `Pending`.

## topologyKey Concept

`topologyKey` defines the **scope** of the affinity or anti-affinity rule. It's a node label key. Pods are considered "co-located" if they run on nodes that share the same value for that label.

| topologyKey | Scope | Meaning |
|---|---|---|
| `kubernetes.io/hostname` | Single node | Pods on the same node |
| `topology.kubernetes.io/zone` | Availability zone | Pods in the same AZ |
| `topology.kubernetes.io/region` | Region | Pods in the same region |

```
  Zone us-east-1a              Zone us-east-1b
  ┌────────────────────┐       ┌────────────────────┐
  │  Node A    Node B  │       │  Node C    Node D  │
  │  [Pod 1]  [Pod 2]  │       │  [Pod 3]  [     ]  │
  └────────────────────┘       └────────────────────┘

  topologyKey: kubernetes.io/hostname
    Pod 1 and Pod 2 are in DIFFERENT topologies (different nodes)

  topologyKey: topology.kubernetes.io/zone
    Pod 1 and Pod 2 are in the SAME topology (same zone: us-east-1a)
    Pod 1 and Pod 3 are in DIFFERENT topologies (different zones)
```

## Hard vs Soft (required vs preferred)

Just like node affinity, pod affinity and anti-affinity support both hard and soft rules.

| Type | Behavior |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard constraint. Pod stays `Pending` if no valid node exists. |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft preference. Scheduler tries its best but schedules elsewhere if needed. |

### Soft Anti-Affinity Example

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - web
          topologyKey: topology.kubernetes.io/zone
```

This says: "Try to spread `app=web` Pods across zones, but if you can't, schedule them wherever there's room."

### Combined Affinity and Anti-Affinity

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: redis-cache
        topologyKey: topology.kubernetes.io/zone
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname
```

This says: "Schedule me in the same zone as `redis-cache` AND never on the same node as another `web` Pod."

## Use Cases

| Scenario | Rule Type | Configuration |
|---|---|---|
| Co-locate frontend with its cache | Pod affinity | `podAffinity` with `topologyKey: kubernetes.io/hostname` |
| Co-locate in same zone for low latency | Pod affinity | `podAffinity` with `topologyKey: topology.kubernetes.io/zone` |
| Spread replicas across nodes | Pod anti-affinity | `podAntiAffinity` with `topologyKey: kubernetes.io/hostname` |
| Spread replicas across zones | Pod anti-affinity | `podAntiAffinity` with `topologyKey: topology.kubernetes.io/zone` |
| Keep competing workloads apart | Pod anti-affinity | Anti-affinity matching the other workload's labels |

## Topology Spread Constraints

`topologySpreadConstraints` provide finer-grained control over how Pods are distributed across topology domains. While pod anti-affinity is binary ("don't co-locate"), topology spread constraints let you define **how evenly** Pods should be distributed.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web
      containers:
        - name: web
          image: web:1.0
```

### Key Fields

| Field | Description |
|---|---|
| `maxSkew` | The maximum allowed difference in Pod count between any two topology domains. A value of 1 means domains can differ by at most 1 Pod. |
| `topologyKey` | The node label that defines topology domains (same concept as in pod affinity). |
| `whenUnsatisfiable` | What to do when the constraint can't be satisfied. |
| `labelSelector` | Selects which Pods to count when computing skew. |

### whenUnsatisfiable

| Value | Behavior |
|---|---|
| `DoNotSchedule` | Don't schedule the Pod if it would violate `maxSkew`. Pod stays `Pending`. |
| `ScheduleAnyway` | Schedule the Pod anyway, but favor domains that minimize the skew. |

### How maxSkew Works

With `maxSkew: 1` and 3 zones, 6 replicas distribute as evenly as possible:

```
  Zone A          Zone B          Zone C
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ [Pod]    │   │ [Pod]    │   │ [Pod]    │
  │ [Pod]    │   │ [Pod]    │   │ [Pod]    │
  └──────────┘   └──────────┘   └──────────┘
  Count: 2        Count: 2        Count: 2
  Skew: 0 (max - min = 2 - 2 = 0)  ✓ satisfies maxSkew: 1
```

If you tried to add a 7th Pod, only one zone could get it (making the count 3-2-2, skew = 1), which still satisfies `maxSkew: 1`. An 8th Pod could go to any of the remaining zones with count 2.

### Multiple Constraints

You can combine multiple topology spread constraints. All must be satisfied.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web
```

This says: "Strictly spread across zones (hard). Also try to spread across nodes (soft)."

## Pod Topology Spread vs Pod Anti-Affinity

| Aspect | Pod Anti-Affinity | Topology Spread Constraints |
|---|---|---|
| **Granularity** | Binary: "don't co-locate" or "try not to" | Numeric: "differ by at most N" (`maxSkew`) |
| **Distribution** | Can result in uneven spread (one-per-node leaves remaining Pods pending) | Actively balances across domains |
| **Scaling** | Hard anti-affinity limits replicas to number of domains | maxSkew allows controlled imbalance as you scale |
| **Multiple domains** | One topology key per rule | Can combine zone-level and node-level constraints |
| **When to use** | Strict separation (at most one per node) | Even distribution across zones or nodes |
| **Performance** | Higher scheduling cost in large clusters | More efficient for the scheduler |

**General guidance**: Use pod anti-affinity when you need strict "at most one per node/zone" semantics. Use topology spread constraints when you need balanced distribution that scales gracefully.
