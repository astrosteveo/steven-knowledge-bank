---
tags:
  - kubernetes
  - kubernetes/operations
topic: Operations
---

# Rolling Updates and Rollbacks

## How Rolling Updates Work

When you update a Deployment (for example, change the container image), Kubernetes performs a **rolling update** by default. It incrementally replaces old Pods with new ones so the application stays available throughout the process.

```
Rolling Update Flow (maxSurge=1, maxUnavailable=0, replicas=3)

Step 1: Create 1 new Pod (surge)
  Old: [v1] [v1] [v1]    New: [v2 starting...]
  Available: 3/3

Step 2: New Pod ready, terminate 1 old Pod
  Old: [v1] [v1]          New: [v2]
  Available: 3/3

Step 3: Create another new Pod
  Old: [v1] [v1]          New: [v2] [v2 starting...]
  Available: 3/3

Step 4: New Pod ready, terminate 1 old Pod
  Old: [v1]               New: [v2] [v2]
  Available: 3/3

Step 5-6: Repeat until all Pods are v2
  Old: (none)             New: [v2] [v2] [v2]
  Available: 3/3
```

Under the hood, the Deployment controller creates a **new ReplicaSet** for the updated Pod template and scales it up while scaling the old ReplicaSet down.

## Strategy Fields

The `spec.strategy` section controls how the rollout proceeds:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate          # default (other option: Recreate)
    rollingUpdate:
      maxSurge: 1               # max Pods above desired count during update
      maxUnavailable: 0          # max Pods that can be unavailable during update
  template:
    # ...
```

| Field | Accepts | Default | Meaning |
|---|---|---|---|
| `maxSurge` | Integer or percentage | `25%` | How many extra Pods can exist above the desired replica count during the update |
| `maxUnavailable` | Integer or percentage | `25%` | How many Pods can be unavailable (not Ready) during the update |

Common combinations:

| `maxSurge` | `maxUnavailable` | Behavior |
|---|---|---|
| `1` | `0` | Zero-downtime, slow rollout (one pod at a time, always at full capacity) |
| `0` | `1` | No extra resources used, but capacity temporarily reduced |
| `25%` | `25%` | Default balance of speed and availability |
| `100%` | `0` | Blue-green style: spin up all new Pods first, then remove old ones |

Setting `strategy.type: Recreate` kills all existing Pods before creating new ones. This causes downtime but avoids running two versions simultaneously.

## What Triggers a Rollout

A new rollout is triggered only when the **Pod template** (`spec.template`) changes. This includes:

- Container image change (`spec.template.spec.containers[*].image`)
- Environment variable changes
- Resource requests/limits changes
- Label or annotation changes on the Pod template
- Volume mount changes

Changes to fields outside the Pod template (like `replicas`, `strategy`, or Deployment labels) do **not** trigger a rollout.

```bash
# Trigger a rollout by updating the image
kubectl set image deployment/my-app my-app=my-app:v2

# Or edit the manifest directly
kubectl edit deployment my-app

# Or apply an updated YAML file
kubectl apply -f deployment.yaml
```

## Monitoring a Rollout

```bash
# Watch the rollout progress in real time
kubectl rollout status deployment/my-app

# Example output:
# Waiting for deployment "my-app" rollout to finish: 2 out of 4 new replicas have been updated...
# Waiting for deployment "my-app" rollout to finish: 3 out of 4 new replicas have been updated...
# deployment "my-app" successfully rolled out

# Check ReplicaSets to see old and new
kubectl get replicasets -l app=my-app
# NAME               DESIRED   CURRENT   READY   AGE
# my-app-6b7d8c9f4   4         4         4       10s    (new)
# my-app-5a6c7b8e3   0         0         0       1h     (old, scaled to 0)
```

The command exits with code `0` on success or non-zero on failure/timeout.

## Rollout History

Kubernetes keeps a history of Deployment revisions (one per ReplicaSet):

```bash
# View rollout history
kubectl rollout history deployment/my-app
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
# 3         <none>

# View details of a specific revision
kubectl rollout history deployment/my-app --revision=2
```

The `CHANGE-CAUSE` column is populated from the `kubernetes.io/change-cause` annotation on the Deployment. Set it when deploying:

```bash
kubectl annotate deployment/my-app kubernetes.io/change-cause="upgrade to v2.1.0"
```

Or include it in the YAML:

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "upgrade to v2.1.0"
```

## Rolling Back

```bash
# Roll back to the previous revision
kubectl rollout undo deployment/my-app

# Roll back to a specific revision
kubectl rollout undo deployment/my-app --to-revision=2
```

A rollback is itself a rollout -- it creates a new revision by re-using the Pod template from the target revision. The old ReplicaSet is scaled back up and the current one is scaled down.

```
Rollback Flow

Revision 1: image=v1  (ReplicaSet A)
Revision 2: image=v2  (ReplicaSet B) ← current
Revision 3: image=v3  (ReplicaSet C)

After: kubectl rollout undo --to-revision=1

Revision 1: image=v2  (ReplicaSet B)
Revision 2: image=v3  (ReplicaSet C)
Revision 3: image=v1  (ReplicaSet A) ← current (re-uses rev 1 template)
```

## Pausing and Resuming Rollouts

Pausing a Deployment lets you make multiple changes to the Pod template without triggering intermediate rollouts:

```bash
# Pause the rollout
kubectl rollout pause deployment/my-app

# Make multiple changes (none of these trigger a rollout yet)
kubectl set image deployment/my-app my-app=my-app:v3
kubectl set resources deployment/my-app -c my-app --limits=cpu=200m,memory=256Mi

# Resume -- a single rollout with all changes fires now
kubectl rollout resume deployment/my-app
```

While paused, the Deployment controller will not reconcile new changes. You cannot roll back a paused Deployment -- you must resume it first.

## Revision History Limit

```yaml
spec:
  revisionHistoryLimit: 10   # default
```

This controls how many old ReplicaSets (revisions) the Deployment keeps around. Each old ReplicaSet is scaled to zero but still exists so you can roll back to it. Setting this to `0` means you cannot roll back at all. The default of `10` is usually sufficient.

## minReadySeconds

```yaml
spec:
  minReadySeconds: 30   # default: 0
```

A Pod is considered **available** only after it has been Ready for at least `minReadySeconds` without any container crashing. This adds a buffer after readiness probes pass, giving the application additional stabilization time before the rollout proceeds to replace the next batch of Pods.

This is particularly useful for applications that pass health checks quickly but need time to warm caches or finish initialization.

## Deployment Conditions

The Deployment status exposes conditions you can inspect programmatically:

```bash
kubectl get deployment my-app -o jsonpath='{.status.conditions[*].type}'
```

| Condition | Meaning |
|---|---|
| `Progressing` | The Deployment is creating/scaling ReplicaSets or waiting for Pods. Set to `True` while a rollout is in progress |
| `Available` | The Deployment has the minimum required Pods available (`minReadySeconds` satisfied). Set to `True` when enough Pods are running |
| `ReplicaFailure` | The Deployment cannot create a Pod (e.g., quota exceeded, image pull failure). Set to `True` on failure |

A Deployment is considered **stuck** when the `Progressing` condition transitions to `status: "False"` with `reason: ProgressDeadlineExceeded`. Control the deadline with:

```yaml
spec:
  progressDeadlineSeconds: 600   # default: 600 (10 minutes)
```

## Blue-Green Deployments

Blue-green deploys two full environments and switches traffic atomically via a Service selector. Kubernetes does not provide this natively, but you can implement it with separate Deployments:

```yaml
# Blue deployment (currently serving traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
        - name: my-app
          image: my-app:v1
---
# Green deployment (new version, not yet receiving traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
        - name: my-app
          image: my-app:v2
---
# Service points to blue; switch to green by changing the selector
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: blue        # change to "green" to switch traffic
  ports:
    - port: 80
      targetPort: 8080
```

Switch traffic by patching the Service:

```bash
kubectl patch service my-app -p '{"spec":{"selector":{"version":"green"}}}'
```

## Canary Deployments

Canary sends a small percentage of traffic to the new version to validate it before full rollout. The simplest approach uses two Deployments behind the same Service:

```yaml
# Stable deployment (9 replicas)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app        # Service selects on this label
        track: stable
    spec:
      containers:
        - name: my-app
          image: my-app:v1
---
# Canary deployment (1 replica = ~10% of traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app        # same label, so the Service routes here too
        track: canary
    spec:
      containers:
        - name: my-app
          image: my-app:v2
---
# Service selects all Pods with app=my-app (both stable and canary)
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app            # matches both deployments
  ports:
    - port: 80
      targetPort: 8080
```

Traffic distribution is proportional to the number of Pod endpoints. For more precise traffic splitting, use a service mesh (Istio, Linkerd) or an Ingress controller that supports weighted routing.

```
Canary Traffic Flow

                  ┌──────────────────────────┐
                  │      Service: my-app      │
                  │   selector: app=my-app    │
                  └─────┬────────────┬────────┘
                        │            │
               90% of   │            │  10% of
              endpoints  │            │  endpoints
                        ▼            ▼
              ┌──────────────┐  ┌──────────────┐
              │ Stable (v1)  │  │ Canary (v2)  │
              │ 9 replicas   │  │ 1 replica    │
              └──────────────┘  └──────────────┘
```
