---
tags:
  - kubernetes
  - kubernetes/workloads
topic: Workloads
---

# StatefulSets

## When to Use StatefulSets vs Deployments

| Requirement | Deployment | StatefulSet |
|---|---|---|
| Pods are interchangeable | Yes | No — each Pod has a unique, stable identity |
| Persistent storage per Pod | Shared or ephemeral | Each Pod gets its own PersistentVolumeClaim |
| Ordered startup/shutdown | No guarantee | Pods are created and deleted in order |
| Stable network identity | No — Pod names and IPs change on restart | Yes — Pod name and DNS are stable across restarts |
| Typical workloads | Web servers, APIs, microservices | Databases, message brokers, distributed caches |

Use a StatefulSet when your application requires one or more of: **stable identity**, **stable storage**, or **ordered operations**.

## Guarantees

StatefulSets provide three guarantees that Deployments do not:

### 1. Stable Network Identity

Each Pod gets a predictable hostname based on the StatefulSet name and an ordinal index:

```
StatefulSet name: mysql
Pods: mysql-0, mysql-1, mysql-2

DNS records (requires a headless Service):
  mysql-0.mysql-headless.default.svc.cluster.local
  mysql-1.mysql-headless.default.svc.cluster.local
  mysql-2.mysql-headless.default.svc.cluster.local
```

If `mysql-1` is deleted and recreated, the new Pod is still named `mysql-1` with the same DNS entry.

### 2. Stable Persistent Storage

Each Pod gets its own PersistentVolumeClaim via `volumeClaimTemplates`. When a Pod is rescheduled, it reattaches to the **same** PersistentVolume:

```
mysql-0 ──► PVC: data-mysql-0 ──► PV (10Gi on node-a)
mysql-1 ──► PVC: data-mysql-1 ──► PV (10Gi on node-b)
mysql-2 ──► PVC: data-mysql-2 ──► PV (10Gi on node-c)
```

PVCs are **not deleted** when you scale down or delete the StatefulSet. This is a safety feature that prevents accidental data loss. You must delete PVCs manually.

### 3. Ordered Deployment, Scaling, and Deletion

| Operation | Order |
|---|---|
| **Create** | mysql-0 must be Running and Ready before mysql-1 starts |
| **Scale up** | New Pods are created in ascending index order |
| **Scale down** | Pods are terminated in descending index order (highest first) |
| **Delete** | Same as scale down — highest ordinal first |
| **Updates** | Pods are updated in reverse ordinal order (highest first) |

## Headless Service Requirement

StatefulSets require a **headless Service** (`clusterIP: None`) to provide DNS records for individual Pods. This Service does not load-balance; it returns the IPs of all matching Pods directly.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  labels:
    app: mysql
spec:
  clusterIP: None                 # headless — no virtual IP
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

You can (and often should) also create a regular Service for client access that load-balances across all Pods.

## Pod Naming Convention

Pods are named `<statefulset-name>-<ordinal>`, starting at 0:

```
StatefulSet: redis       Pods: redis-0, redis-1, redis-2
StatefulSet: cassandra   Pods: cassandra-0, cassandra-1, cassandra-2, cassandra-3
```

The ordinal is stable: if `redis-1` is deleted, the replacement is also named `redis-1`.

DNS records for each Pod follow the pattern:

```
<pod-name>.<headless-service>.<namespace>.svc.cluster.local
```

## VolumeClaimTemplates

Instead of referencing an existing PVC, StatefulSets define templates that create a unique PVC for each Pod:

```yaml
volumeClaimTemplates:
  - metadata:
      name: data                    # volume name referenced in volumeMounts
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

The resulting PVCs are named `<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>`:

```
data-mysql-0
data-mysql-1
data-mysql-2
```

## Update Strategies

### RollingUpdate (default)

Pods are updated one at a time in **reverse ordinal order** (highest index first). Each Pod must become Ready before the next one is updated.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0                # only update pods with ordinal >= partition
```

The `partition` field enables staged rollouts: set `partition: 2` to update only Pods with ordinal >= 2 (i.e., `mysql-2` and above). Pods with lower ordinals keep the old spec. Lower the partition progressively to roll the update out further.

```
partition: 2 → updates mysql-2 only
partition: 1 → also updates mysql-1
partition: 0 → updates all pods (complete rollout)
```

### OnDelete

Pods are **not automatically updated**. You must manually delete each Pod, and the StatefulSet controller recreates it with the new spec. Gives you full control over the update order and timing.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

## Complete YAML Example

A MySQL StatefulSet with persistent storage:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless          # must match the headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  podManagementPolicy: OrderedReady    # OrderedReady (default) | Parallel
  template:
    metadata:
      labels:
        app: mysql
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
          readinessProbe:
            exec:
              command:
                - mysqladmin
                - ping
                - -h
                - localhost
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            exec:
              command:
                - mysqladmin
                - ping
                - -h
                - localhost
            initialDelaySeconds: 30
            periodSeconds: 10
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

### podManagementPolicy

| Policy | Behavior |
|---|---|
| **OrderedReady** (default) | Pods are created/deleted one at a time in order. Each must be Ready before the next starts. |
| **Parallel** | All Pods are created or deleted simultaneously. Use this when ordering is not required but stable identity and storage are. |

## Limitations and Caveats

- **Storage is not auto-deleted.** PVCs created by `volumeClaimTemplates` persist after StatefulSet deletion. You must clean them up manually. This is intentional to prevent data loss.
- **Deleting a StatefulSet does not guarantee ordered Pod termination.** Scale to 0 first if you need ordered shutdown: `kubectl scale statefulset mysql --replicas=0`, then delete.
- **Headless Service must exist before the StatefulSet.** The StatefulSet controller does not create it.
- **Node failures are tricky.** If a node becomes unreachable, Kubernetes does not automatically delete the Pod (it remains in `Terminating` state). You may need to manually delete the Pod or the node object to trigger rescheduling. This prevents split-brain scenarios where two Pods with the same identity run simultaneously.
- **Rolling updates can stall.** If an updated Pod never becomes Ready, the rollout halts (there is no automatic rollback like Deployments). You must fix the issue or manually roll back.
- **Cannot change `spec.selector` after creation.** Same restriction as Deployments and ReplicaSets.
- **`volumeClaimTemplates` cannot be modified after creation.** To change storage size, you must manually resize the PVCs (if the StorageClass supports it) or recreate the StatefulSet.
