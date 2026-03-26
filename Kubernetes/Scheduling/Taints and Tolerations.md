---
tags:
  - kubernetes
  - kubernetes/scheduling
topic: Scheduling
---

# Taints and Tolerations

## What Are Taints and Tolerations?

Taints and tolerations work together to **repel Pods from nodes** unless the Pod explicitly tolerates the taint. Think of it as nodes saying "keep out unless you have the right pass."

```
  Taint on Node                Toleration on Pod
  ┌──────────────┐             ┌──────────────┐
  │ Node A       │             │ Pod X        │
  │              │  REJECTED   │ (no matching │
  │ gpu=true:    │◄────────────│  toleration) │
  │  NoSchedule  │             └──────────────┘
  │              │
  │              │  ACCEPTED   ┌──────────────┐
  │              │◄────────────│ Pod Y        │
  │              │             │ (tolerates   │
  └──────────────┘             │  gpu=true)   │
                               └──────────────┘
```

Taints are set on **nodes**. Tolerations are set on **Pods**. A taint without a matching toleration prevents scheduling; a toleration without a matching taint does nothing (a Pod that tolerates everything can still land on untainted nodes).

## Taint Effects

Each taint has an **effect** that determines what happens to Pods that don't tolerate it.

| Effect | Behavior |
|---|---|
| `NoSchedule` | Pods that don't tolerate the taint will **not be scheduled** on the node. Already-running Pods are unaffected. |
| `PreferNoSchedule` | Kubernetes will **try to avoid** scheduling Pods that don't tolerate the taint, but it's not guaranteed. A soft version of `NoSchedule`. |
| `NoExecute` | Pods that don't tolerate the taint will **not be scheduled** on the node AND **existing Pods** that don't tolerate it will be **evicted**. |

## Adding and Removing Taints

```bash
# Add a taint to a node
kubectl taint nodes worker-1 gpu=true:NoSchedule

# Add a taint with no value (key-only)
kubectl taint nodes worker-1 dedicated:NoSchedule

# Add multiple taints
kubectl taint nodes worker-1 gpu=true:NoSchedule env=production:NoExecute

# Remove a taint (trailing minus sign, must match key and effect)
kubectl taint nodes worker-1 gpu=true:NoSchedule-

# Remove all taints with a given key (regardless of effect)
kubectl taint nodes worker-1 gpu-

# View taints on a node
kubectl describe node worker-1 | grep -A5 Taints
```

## Toleration YAML in Pod Spec

Tolerations go in `spec.tolerations` on the Pod (or Pod template inside a Deployment, etc.).

### Matching a Specific Key-Value Pair

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: cuda-app
      image: nvidia/cuda:12.0-base
```

### Matching a Key (Any Value)

```yaml
tolerations:
  - key: "gpu"
    operator: "Exists"
    effect: "NoSchedule"
```

### Tolerating All Taints with a Key

```yaml
# Matches any effect for the "gpu" key
tolerations:
  - key: "gpu"
    operator: "Exists"
```

### Tolerating Everything (Wildcard)

```yaml
# An empty key with Exists matches all taints -- use with caution
tolerations:
  - operator: "Exists"
```

## Operator: Equal vs Exists

| Operator | Behavior | Requires `value`? |
|---|---|---|
| `Equal` | Matches when key, value, AND effect all match the taint | Yes |
| `Exists` | Matches when the key exists on the node (value is ignored) | No (omit `value`) |

If `effect` is omitted from the toleration, it matches all effects for that key.

## Use Cases

### Dedicated Nodes for a Team or Workload

```bash
# Taint the node so only specific workloads land here
kubectl taint nodes gpu-node-1 dedicated=ml-team:NoSchedule
```

```yaml
# Only ML team pods tolerate this
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "ml-team"
    effect: "NoSchedule"
```

### Nodes with Special Hardware

Reserve GPU nodes for workloads that actually need GPUs.

```bash
kubectl taint nodes gpu-node-1 nvidia.com/gpu=present:NoSchedule
```

### Eviction During Node Problems

When a node becomes unhealthy, Kubernetes automatically applies `NoExecute` taints. Pods without a matching toleration are evicted.

```bash
# Simulate marking a node for maintenance
kubectl taint nodes worker-3 maintenance=true:NoExecute
# All pods without a toleration for this taint are immediately evicted
```

## Built-in Taints

Kubernetes automatically adds these taints to nodes when conditions arise. The node lifecycle controller manages them.

| Taint | When Applied |
|---|---|
| `node.kubernetes.io/not-ready` | Node condition `Ready` is `False` |
| `node.kubernetes.io/unreachable` | Node condition `Ready` is `Unknown` (node controller lost contact) |
| `node.kubernetes.io/memory-pressure` | Node is running low on memory |
| `node.kubernetes.io/disk-pressure` | Node is running low on disk space |
| `node.kubernetes.io/pid-pressure` | Node has too many processes |
| `node.kubernetes.io/network-unavailable` | Node network is not configured correctly |
| `node.kubernetes.io/unschedulable` | Node is cordoned (`kubectl cordon`) |

When you `kubectl cordon` a node, it applies `node.kubernetes.io/unschedulable:NoSchedule`. This prevents new Pods from being scheduled but does not evict existing ones. To also evict, use `kubectl drain`.

## tolerationSeconds for NoExecute

When a `NoExecute` taint is added to a node, Pods that don't tolerate it are evicted immediately. But you can delay eviction using `tolerationSeconds` -- this gives the Pod a grace period before it's removed.

```yaml
tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300    # stay for 5 minutes after node becomes not-ready
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300
```

Kubernetes adds default tolerations for `not-ready` and `unreachable` with `tolerationSeconds: 300` to every Pod that doesn't explicitly set them. This gives the node 5 minutes to recover before Pods are evicted.

If `tolerationSeconds` is omitted on a `NoExecute` toleration, the Pod tolerates the taint indefinitely and is never evicted for that reason.

## Taints and Tolerations vs Node Affinity

These two mechanisms serve complementary but different purposes.

| Aspect | Taints & Tolerations | Node Affinity |
|---|---|---|
| **Perspective** | Node-centric: "Keep pods away from me" | Pod-centric: "Schedule me on a specific node" |
| **Default behavior** | Repels all pods that don't tolerate | Does nothing unless configured on the pod |
| **Guarantees exclusivity** | Yes -- only pods with tolerations can be scheduled | No -- other pods can still land on the same node |
| **Hard vs soft** | `NoSchedule` (hard), `PreferNoSchedule` (soft) | `required` (hard), `preferred` (soft) |
| **Eviction** | `NoExecute` can evict running pods | No eviction capability |
| **Use together** | Taint a node AND use node affinity on the pod to both repel others and attract the right workloads |

A common pattern for dedicated nodes combines both:

1. **Taint** the node to keep unwanted Pods away.
2. **Node affinity** on the desired Pods to attract them to the tainted node.

This ensures only the right workloads run on dedicated hardware.
