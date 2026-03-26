---
tags:
  - kubernetes
  - kubernetes/reference
aliases:
  - k8s-yaml
  - kubernetes-manifests
topic: Quick Reference
created: 2026-03-26
---

# YAML Manifests Reference

Minimal but practical YAML templates for every major Kubernetes resource type. Each template includes the most commonly used fields with brief inline comments.

## Pod

Pods are rarely created directly -- use Deployments or Jobs instead. Shown here as the foundational unit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
spec:
  restartPolicy: Always                   # Always | OnFailure | Never
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8080
      env:
        - name: LOG_LEVEL
          value: "info"
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          memory: 256Mi
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
      volumeMounts:
        - name: config
          mountPath: /etc/app
  volumes:
    - name: config
      configMap:
        name: app-config
```

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 10                # how many old ReplicaSets to keep
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate                   # RollingUpdate | Recreate
    rollingUpdate:
      maxSurge: 1                         # pods above desired during rollout
      maxUnavailable: 0                   # pods that can be unavailable
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: myapp:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 10
```

## StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db                         # must match a headless Service
  replicas: 3
  podManagementPolicy: OrderedReady       # OrderedReady | Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0                        # only pods with ordinal >= partition update
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:                   # one PVC per pod, retained on deletion
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

## DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      tolerations:                        # schedule on all nodes including control plane
        - effect: NoSchedule
          operator: Exists
      hostNetwork: true                   # use the node's network namespace
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          ports:
            - containerPort: 9100
              hostPort: 9100
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              memory: 128Mi
```

## Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  backoffLimit: 3                         # retries before marking failed
  activeDeadlineSeconds: 600              # hard timeout
  ttlSecondsAfterFinished: 3600          # auto-cleanup 1 hour after completion
  template:
    spec:
      restartPolicy: Never                # Never | OnFailure (not Always)
      containers:
        - name: migrate
          image: myapp:1.0
          command: ["python", "manage.py", "migrate"]
```

## CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "30 2 * * *"                  # 02:30 UTC daily
  timeZone: "America/New_York"            # optional, 1.27+
  concurrencyPolicy: Forbid               # Allow | Forbid | Replace
  startingDeadlineSeconds: 300            # skip if missed by > 5 min
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: report
              image: report-tool:1.0
              command: ["./generate-report.sh"]
```

## Service -- ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP                         # default type, internal only
  selector:
    app: web
  ports:
    - name: http
      port: 80                            # port the Service listens on
      targetPort: 8080                    # port on the pod
      protocol: TCP
```

## Service -- NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080                     # optional, range 30000-32767
```

## Service -- LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb   # cloud-specific
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

## Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx                 # ties to an IngressClass
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls                 # TLS cert stored as a Secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix              # Prefix | Exact | ImplementationSpecific
            backend:
              service:
                name: web
                port:
                  number: 80
```

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  config.yaml: |                          # multi-line file content
    server:
      host: 0.0.0.0
      port: 8080
```

**Usage in a pod:**

```yaml
# As environment variables
envFrom:
  - configMapRef:
      name: app-config

# As a mounted file
volumes:
  - name: config
    configMap:
      name: app-config
      items:
        - key: config.yaml
          path: config.yaml
```

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:                               # plain text, base64-encoded on creation
  username: admin
  password: "s3cret!"
  url: "postgres://admin:s3cret!@db:5432/mydb"
```

> **Note:** Secrets are base64-encoded, not encrypted. Use `stringData` for convenience (Kubernetes encodes it for you). For encryption at rest, enable `EncryptionConfiguration` on the API server or use an external secrets manager.

## PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce                       # RWO | ROX | RWX
  persistentVolumeReclaimPolicy: Retain   # Retain | Delete | Recycle(deprecated)
  storageClassName: standard
  hostPath:                               # for dev only; use CSI drivers in production
    path: /mnt/data
```

## PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard              # omit to use default StorageClass
  resources:
    requests:
      storage: 10Gi
```

**Usage in a pod:**

```yaml
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
```

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com             # CSI driver name
reclaimPolicy: Delete                     # Delete | Retain
volumeBindingMode: WaitForFirstConsumer   # Immediate | WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "5000"
```

## Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
```

## ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
automountServiceAccountToken: false       # opt out of auto-mounting the token
```

## Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]                       # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
```

## ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["nodes"]
    verbs: ["get", "list"]
```

## RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

## ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view-nodes
subjects:
  - kind: Group
    name: "developers"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-viewer
```

## NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              env: staging
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: db
      ports:
        - protocol: TCP
          port: 5432
    - to:                                 # allow DNS resolution
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

> **Default deny all:** Apply a NetworkPolicy with an empty `ingress: []` and `egress: []` to block all traffic, then layer on allow rules.

## HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300     # avoid flapping
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## PodDisruptionBudget

Limits how many pods can be simultaneously unavailable during voluntary disruptions (node drain, cluster upgrades).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2                         # OR use maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

| Field | Meaning |
|-------|---------|
| `minAvailable: 2` | at least 2 pods must remain running |
| `minAvailable: "50%"` | at least half the pods must remain |
| `maxUnavailable: 1` | at most 1 pod can be down at a time |

## LimitRange

Sets default and maximum resource constraints per container in a namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
    - type: Container
      default:                            # applied if no limits specified
        cpu: 500m
        memory: 256Mi
      defaultRequest:                     # applied if no requests specified
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "2"
        memory: 2Gi
      min:
        cpu: 50m
        memory: 64Mi
    - type: PersistentVolumeClaim
      max:
        storage: 50Gi
```

## ResourceQuota

Limits total resource consumption in a namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "10"                    # total CPU requests across all pods
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"                            # max number of pods
    services: "20"
    persistentvolumeclaims: "10"
    configmaps: "50"
    secrets: "50"
    services.loadbalancers: "2"
```
