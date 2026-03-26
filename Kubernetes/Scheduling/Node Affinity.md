---
tags:
  - kubernetes
  - kubernetes/scheduling
topic: Scheduling
---

# Node Affinity

Node affinity controls **which nodes a Pod can be scheduled on** based on node labels. It ranges from a simple one-line constraint (`nodeSelector`) to expressive rules with hard and soft preferences.

## nodeSelector (Simple Constraint)

The simplest form of node selection. The Pod will only be scheduled on nodes that have **all** the specified labels.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    disktype: ssd
    gpu: "true"
  containers:
    - name: app
      image: my-app:1.0
```

```bash
# Label a node first so nodeSelector can match it
kubectl label nodes worker-1 disktype=ssd
kubectl label nodes worker-1 gpu=true
```

`nodeSelector` is purely hard -- if no node matches, the Pod stays in `Pending`. For soft preferences or more complex logic, use node affinity.

## Node Affinity Types

Node affinity rules go under `spec.affinity.nodeAffinity` and come in two flavors:

| Type | Short Name | Behavior |
|---|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | **Hard** | Pod will NOT be scheduled unless a matching node exists. Like `nodeSelector` but more expressive. |
| `preferredDuringSchedulingIgnoredDuringExecution` | **Soft** | Scheduler TRIES to find a matching node but will schedule elsewhere if none is available. |

The `IgnoredDuringExecution` part means that if a node's labels change after a Pod is already running, the Pod is **not** evicted. The affinity rules only apply at scheduling time.

## Hard Affinity (required)

The Pod must land on a node matching the rules. If no node qualifies, the Pod stays `Pending`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: zone-restricted
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
  containers:
    - name: app
      image: my-app:1.0
```

Rules within a single `nodeSelectorTerms` entry are ANDed. Multiple `nodeSelectorTerms` entries are ORed.

```yaml
# OR logic: schedule on nodes in us-east-1a OR nodes with gpu=true
nodeSelectorTerms:
  - matchExpressions:                    # term 1
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-east-1a"]
  - matchExpressions:                    # term 2 (OR)
      - key: gpu
        operator: In
        values: ["true"]
```

```yaml
# AND logic: node must be in us-east-1a AND have gpu=true
nodeSelectorTerms:
  - matchExpressions:
      - key: topology.kubernetes.io/zone  # condition 1
        operator: In
        values: ["us-east-1a"]
      - key: gpu                          # AND condition 2
        operator: In
        values: ["true"]
```

## Soft Affinity (preferred)

The scheduler scores nodes against the preference and favors matching ones. Each preference has a **weight** (1-100) that influences the scheduling score.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prefer-ssd
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
        - weight: 20
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
  containers:
    - name: app
      image: my-app:1.0
```

The scheduler adds the weights of all matching preferences to each node's score. Higher total weight means the node is more attractive.

```
  Node A (disktype=ssd, zone=us-east-1a)   --> score += 80 + 20 = 100
  Node B (disktype=ssd, zone=us-east-1b)   --> score += 80      = 80
  Node C (disktype=hdd, zone=us-east-1a)   --> score += 20      = 20
  Node D (disktype=hdd, zone=us-east-1c)   --> score += 0       = 0
```

## Combining Hard and Soft Affinity

You can mix both types. Hard rules filter the candidate nodes, and soft rules rank the remaining candidates.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/os
              operator: In
              values: ["linux"]
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        preference:
          matchExpressions:
            - key: disktype
              operator: In
              values: ["ssd"]
```

This says: "The node MUST run Linux. Among Linux nodes, prefer ones with SSD storage."

## Operators

The `operator` field in `matchExpressions` supports these values:

| Operator | Meaning | Requires `values`? |
|---|---|---|
| `In` | Label value is in the given set | Yes |
| `NotIn` | Label value is NOT in the given set | Yes |
| `Exists` | Label key exists (any value) | No |
| `DoesNotExist` | Label key does not exist | No |
| `Gt` | Label value is greater than (parsed as integer) | Yes (single value) |
| `Lt` | Label value is less than (parsed as integer) | Yes (single value) |

## Common Node Labels

Kubernetes automatically applies several well-known labels to nodes.

| Label | Description | Example Value |
|---|---|---|
| `kubernetes.io/hostname` | The hostname of the node | `worker-1` |
| `kubernetes.io/os` | Operating system | `linux`, `windows` |
| `kubernetes.io/arch` | CPU architecture | `amd64`, `arm64` |
| `topology.kubernetes.io/zone` | Cloud availability zone | `us-east-1a` |
| `topology.kubernetes.io/region` | Cloud region | `us-east-1` |
| `node.kubernetes.io/instance-type` | Cloud instance type | `m5.xlarge` |

```bash
# View all labels on a node
kubectl get node worker-1 --show-labels

# View specific labels
kubectl get nodes -L kubernetes.io/os,topology.kubernetes.io/zone
```

## nodeName (Direct Assignment)

`nodeName` bypasses the scheduler entirely. The Pod is placed directly on the specified node with no affinity checks, taint checks, or resource evaluation.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pinned-pod
spec:
  nodeName: worker-2
  containers:
    - name: app
      image: my-app:1.0
```

**Limitations of nodeName:**

- If the named node doesn't exist, the Pod won't run and may be automatically deleted.
- If the node lacks resources, the Pod fails on that node with no fallback.
- Taints, affinity rules, and resource requests are all ignored.
- Not suitable for production workloads. Primarily useful for debugging or DaemonSet-like one-off tasks.

## Quick Comparison

| Mechanism | Complexity | Hard/Soft | Scheduler Involved? |
|---|---|---|---|
| `nodeName` | Trivial | Hard (no fallback) | No (bypassed) |
| `nodeSelector` | Simple | Hard | Yes |
| Node affinity `required` | Expressive | Hard | Yes |
| Node affinity `preferred` | Expressive | Soft (best-effort) | Yes |
