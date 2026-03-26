---
tags:
  - kubernetes
  - kubernetes/workloads
topic: Workloads
---

# ReplicaSets

## Purpose

A ReplicaSet ensures that a **specified number of identical Pod replicas** are running at any given time. If a Pod crashes or is deleted, the ReplicaSet controller detects the discrepancy and creates a new Pod to replace it. If there are too many Pods, it terminates the excess.

```
ReplicaSet (desired: 3)
┌────────────────────────────────────────────┐
│                                            │
│  Watches pods matching label selector      │
│                                            │
│  ┌───────┐    ┌───────┐    ┌───────┐      │
│  │ Pod 1 │    │ Pod 2 │    │ Pod 3 │      │
│  │app=web│    │app=web│    │app=web│      │
│  └───────┘    └───────┘    └───────┘      │
│                                            │
│  If Pod 2 dies → creates Pod 4 (app=web)  │
└────────────────────────────────────────────┘
```

## How ReplicaSets Maintain Desired State

The ReplicaSet controller runs a continuous reconciliation loop:

1. **Observe** — Query the API server for Pods matching the label selector in the ReplicaSet's namespace
2. **Compare** — Count running Pods vs the `spec.replicas` value
3. **Act** — Create or delete Pods to match the desired count

This is a level-triggered system (it acts on the current state, not on events), so it self-heals even if it misses an event.

## Label Selectors and Ownership

ReplicaSets use **label selectors** to determine which Pods they own. A Pod is considered part of a ReplicaSet if:

1. The Pod's labels match the ReplicaSet's `spec.selector`
2. The Pod has no existing `metadata.ownerReferences` from another controller (or it references this ReplicaSet)

When a ReplicaSet creates a Pod, it sets itself as the **owner** in `metadata.ownerReferences`. This means:

- If the ReplicaSet is deleted with `--cascade=foreground`, its Pods are also deleted
- If deleted with `--cascade=orphan`, the Pods continue running without an owner
- An existing Pod whose labels match can be **adopted** by a ReplicaSet if it has no owner

**Warning:** If you create a bare Pod with labels matching a ReplicaSet's selector, the ReplicaSet may adopt it or terminate it (if the replica count is already satisfied). Be careful with label collisions.

## Full YAML Manifest

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
  namespace: default
  labels:
    app: web
    tier: frontend
spec:
  replicas: 3                       # desired number of pod replicas
  selector:                         # must match .spec.template.metadata.labels
    matchLabels:
      app: web
    # matchExpressions:             # alternative: set-based selectors
    #   - key: app
    #     operator: In
    #     values: [web, web-v2]
  template:                         # pod template — same structure as a Pod spec
    metadata:
      labels:
        app: web                    # must satisfy .spec.selector
        tier: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
```

The `spec.selector` field is **immutable** after creation. If you need to change it, delete the ReplicaSet and create a new one.

## Scaling ReplicaSets

```bash
# Imperative scaling
kubectl scale replicaset web-rs --replicas=5

# Check current state
kubectl get replicaset web-rs

# Output:
# NAME     DESIRED   CURRENT   READY   AGE
# web-rs   5         5         5       10m
```

You can also patch the `spec.replicas` field directly:

```bash
kubectl patch replicaset web-rs -p '{"spec": {"replicas": 5}}'
```

Or edit the manifest and apply:

```bash
kubectl apply -f replicaset.yaml
```

## Why You Almost Never Create ReplicaSets Directly

In practice, you manage ReplicaSets through **Deployments**. A Deployment creates and manages ReplicaSets for you, and adds critical features on top:

| Feature | ReplicaSet | Deployment |
|---|---|---|
| Maintain replica count | Yes | Yes (via ReplicaSet) |
| Rolling updates | No | Yes |
| Rollback | No | Yes (keeps old ReplicaSets as revision history) |
| Pause/resume rollouts | No | Yes |
| Declarative updates | Must delete and recreate | Automatic via rolling update |

```
Deployment
└── ReplicaSet (revision 2, current) ──► Pod, Pod, Pod
└── ReplicaSet (revision 1, scaled to 0) ──► (kept for rollback)
```

The only time you might use a ReplicaSet directly is if you need custom update orchestration that Deployments do not support, which is rare.

## ReplicaSet vs ReplicationController

ReplicationController is the **legacy predecessor** to ReplicaSet. The key difference is selector support:

| Feature | ReplicationController | ReplicaSet |
|---|---|---|
| API group | `v1` (core) | `apps/v1` |
| Equality selectors (`app=web`) | Yes | Yes |
| Set-based selectors (`app in (web, api)`) | No | Yes |
| Managed by Deployments | No | Yes |
| Status | Deprecated | Current |

ReplicationControllers only support simple `key=value` selectors, while ReplicaSets support both equality-based and set-based selectors via `matchLabels` and `matchExpressions`.

**Bottom line:** Never use ReplicationControllers. Use Deployments (which create ReplicaSets) for all production workloads.
