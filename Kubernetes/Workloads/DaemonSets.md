---
tags:
  - kubernetes
  - kubernetes/workloads
topic: Workloads
---

# DaemonSets

## What DaemonSets Do

A DaemonSet ensures that **exactly one copy of a Pod runs on every node** (or a specific subset of nodes) in the cluster. When a new node joins the cluster, the DaemonSet controller automatically schedules a Pod on it. When a node is removed, the Pod is garbage collected.

```
Cluster
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Node A    │  │   Node B    │  │   Node C    │
│             │  │             │  │             │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │fluentd  │ │  │ │fluentd  │ │  │ │fluentd  │ │
│ │(daemon) │ │  │ │(daemon) │ │  │ │(daemon) │ │
│ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │
│  + app pods │  │  + app pods │  │  + app pods │
└─────────────┘  └─────────────┘  └─────────────┘

New node D added → fluentd Pod automatically scheduled on D
```

## Use Cases

| Use Case | Example |
|---|---|
| **Log collection** | Fluentd, Fluent Bit, Filebeat on every node |
| **Monitoring agents** | Prometheus Node Exporter, Datadog Agent |
| **Network plugins** | Calico, Cilium, kube-proxy, Weave |
| **Storage daemons** | GlusterFS, Ceph, local volume provisioners |
| **Security agents** | Falco, Sysdig, antivirus scanners |
| **Node configuration** | GPU drivers, kernel module loaders |

## Complete YAML Example

A Fluentd log collector DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1             # max pods that can be down during update
      maxSurge: 0                   # DaemonSets support maxSurge since k8s 1.22
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule        # allow scheduling on control plane nodes
      containers:
        - name: fluentd
          image: fluent/fluentd:v1.17
          resources:
            requests:
              cpu: "100m"
              memory: "200Mi"
            limits:
              cpu: "200m"
              memory: "400Mi"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
      terminationGracePeriodSeconds: 30
```

## Node Selectors and Tolerations

By default, a DaemonSet runs on **every schedulable node**. Use `nodeSelector`, `nodeAffinity`, and `tolerations` to control placement.

### Run only on specific nodes

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd                   # only nodes with label disk=ssd
```

Or with node affinity for more expressive rules:

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                      - linux
```

### Tolerations for tainted nodes

Control plane nodes are typically tainted so regular workloads do not run there. DaemonSets for system-level components (networking, monitoring) need tolerations to run on those nodes:

```yaml
spec:
  template:
    spec:
      tolerations:
        # Run on control plane nodes
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
        # Run on nodes that are not ready
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoExecute
        # Run on nodes with unreachable network
        - key: node.kubernetes.io/unreachable
          operator: Exists
          effect: NoExecute
```

To run on **all** nodes regardless of taints, add a blanket toleration (use carefully):

```yaml
tolerations:
  - operator: Exists
```

## Update Strategies

### RollingUpdate (default)

Pods are updated one node at a time. The old Pod is terminated and a new one is created on the same node before moving to the next.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1          # how many nodes can lack a running Pod during update
      maxSurge: 0                # how many extra Pods can exist during update
```

| Parameter | Default | Effect |
|---|---|---|
| `maxUnavailable` | 1 | Number of nodes that can be without the DaemonSet Pod simultaneously. Higher = faster updates. |
| `maxSurge` | 0 | Number of extra Pods created before old ones are deleted (surge-based update). Set to 1 for zero-downtime DaemonSet updates. |

With `maxSurge: 1`, the new Pod starts on a node **before** the old Pod is terminated — useful when you cannot tolerate a gap (e.g., a networking plugin).

### OnDelete

Pods are **not automatically updated**. The old Pod continues running until you manually delete it, at which point the DaemonSet creates a replacement with the new spec. This gives you full control over the rollout pace.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

## How Scheduling Works

DaemonSets have special scheduling behavior:

1. **The DaemonSet controller** (not the default scheduler) is responsible for creating Pods on each eligible node. It sets `spec.nodeName` directly on the Pod.
2. **Taints and tolerations** are respected. The DaemonSet controller evaluates node taints against the Pod template's tolerations before scheduling.
3. **Unschedulable nodes** (`spec.unschedulable: true` or cordoned) do not prevent DaemonSet Pods from running. DaemonSet Pods tolerate `node.kubernetes.io/unschedulable` by default.
4. **DaemonSet Pods have priority.** They get the `system-node-critical` or `system-cluster-critical` priority class when running in `kube-system`, preventing eviction during resource pressure.
5. **Node addition/removal** is handled automatically. The controller watches for node events and creates or removes Pods accordingly.

Check which nodes have DaemonSet Pods:

```bash
# See all pods managed by the DaemonSet
kubectl get pods -l app=fluentd -o wide -n kube-system

# Check the DaemonSet status
kubectl get daemonset fluentd -n kube-system
# NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
# fluentd   3         3         3       3             3           <none>          5d

# DESIRED  = number of nodes that should run the Pod
# CURRENT  = number of Pods created
# READY    = number of Pods passing readiness checks
# UP-TO-DATE = number of Pods with the latest template
# AVAILABLE  = number of Pods that have been ready for minReadySeconds
```
