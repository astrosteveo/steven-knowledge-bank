---
tags:
  - kubernetes
  - kubernetes/security
topic: Security
---

# Service Accounts

## What is a Service Account?

A **service account** provides an identity for processes running inside a Pod. When a Pod makes API calls to the Kubernetes API server, it authenticates as its service account. Unlike User accounts (which are for humans and managed externally), service accounts are Kubernetes API objects -- namespaced, managed by the cluster, and intended for workloads.

```
  +------------------+
  |      Pod         |
  |                  |       authenticated as
  |  app container --+-----> system:serviceaccount:dev:my-app
  |                  |
  |  projected token |       kube-apiserver checks RBAC:
  |  (mounted vol)   |       "Can this SA do X on resource Y?"
  +------------------+
```

## Default Service Account

Every namespace has a `default` service account, created automatically when the namespace is created. If you don't specify a service account in your Pod spec, the Pod runs as `default`.

```bash
# View the default service account
kubectl get serviceaccount default -n dev

# Every namespace starts with one
kubectl get serviceaccounts --all-namespaces | grep default
```

The `default` service account has minimal permissions -- it can authenticate to the API server but can't do much unless you bind Roles to it. This is intentional, but it also means every Pod in a namespace shares the same identity if you don't create dedicated service accounts.

## Creating Service Accounts

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: dev
  labels:
    app: my-app
```

```bash
# Imperative approach
kubectl create serviceaccount my-app -n dev

# Verify
kubectl get sa my-app -n dev -o yaml
```

## Assigning Service Accounts to Pods

Specify the service account in the Pod's `.spec.serviceAccountName` field. This must be set at creation time -- you cannot change a Pod's service account after it starts.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: dev
spec:
  serviceAccountName: my-app
  containers:
    - name: app
      image: my-app:1.0
```

For workloads managed by a Deployment, set it in the Pod template:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-app
      containers:
        - name: app
          image: my-app:1.0
```

## Token Projection and the TokenRequest API

Starting in Kubernetes 1.20+, Pods receive **bound service account tokens** via the **TokenRequest API**. These tokens are projected into the Pod as a volume and have three important properties that legacy tokens did not:

1. **Time-bound** -- Tokens expire (default: 1 hour) and are automatically rotated by the kubelet.
2. **Audience-bound** -- Tokens are scoped to a specific audience (the API server by default).
3. **Object-bound** -- Tokens are tied to a specific Pod and become invalid when the Pod is deleted.

```yaml
# This is what Kubernetes mounts automatically (you rarely need to configure it manually)
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: kube-api-access
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          readOnly: true
  volumes:
    - name: kube-api-access
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: https://kubernetes.default.svc
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
          - downwardAPI:
              items:
                - path: namespace
                  fieldRef:
                    fieldPath: metadata.namespace
```

The mounted directory at `/var/run/secrets/kubernetes.io/serviceaccount` contains:

| File | Contents |
|---|---|
| `token` | A JWT token, auto-rotated before expiration |
| `ca.crt` | The cluster's CA certificate for verifying the API server |
| `namespace` | The namespace the Pod is running in |

### Requesting Tokens Manually

You can request tokens with custom audiences or expiration times using `kubectl`:

```bash
# Request a token valid for 10 minutes with a custom audience
kubectl create token my-app -n dev \
  --duration=10m \
  --audience=https://my-external-service.example.com
```

## automountServiceAccountToken

By default, Kubernetes mounts a service account token into every Pod. If your Pod doesn't need to talk to the Kubernetes API, disable this to reduce your attack surface.

Disable at the **service account** level (affects all Pods using this SA):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: dev
automountServiceAccountToken: false
```

Disable at the **Pod** level (overrides the service account setting):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false
  containers:
    - name: app
      image: my-app:1.0
```

The Pod-level setting takes precedence. If the SA says `true` but the Pod says `false`, the token is not mounted.

## Service Account Tokens in Kubernetes 1.24+

Before Kubernetes 1.24, creating a service account automatically generated a long-lived Secret containing a non-expiring token. This was a security concern because those tokens never expired and persisted even after the service account was deleted.

| Version | Behavior |
|---|---|
| Pre-1.22 | Auto-created long-lived Secret tokens, mounted into Pods |
| 1.22-1.23 | Projected tokens used by default in Pods, but long-lived Secrets still auto-created |
| 1.24+ | Long-lived Secrets **no longer auto-created**. Pods use projected tokens. You must explicitly create Secrets if you need long-lived tokens. |

If you genuinely need a long-lived token (for an external system that can't use the TokenRequest API), you can still create one manually:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-token
  namespace: dev
  annotations:
    kubernetes.io/service-account.name: my-app
type: kubernetes.io/service-account-token
```

The token controller will populate the `token` field in the Secret's data. However, **prefer projected tokens** whenever possible because they expire and are automatically rotated.

## ImagePullSecrets on Service Accounts

You can attach image pull credentials to a service account so that every Pod using that SA can pull from private registries without specifying `imagePullSecrets` in every Pod spec.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: dev
imagePullSecrets:
  - name: my-registry-creds
```

Create the registry credential Secret first:

```bash
kubectl create secret docker-registry my-registry-creds \
  -n dev \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword
```

Any Pod using the `my-app` service account in the `dev` namespace will automatically use `my-registry-creds` when pulling images, with no additional Pod spec changes needed.

## RBAC for Service Accounts

Service accounts are referenced in RBAC subjects using the format `system:serviceaccount:<namespace>:<name>`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-pod-reader
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

For cross-namespace access (a Pod in namespace A needs to read resources in namespace B):

```yaml
# Role in namespace 'prod' granting read access to configmaps
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: prod
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
# Bind the Role to a service account in namespace 'dev'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-app-reads-prod-configmaps
  namespace: prod          # binding lives in the target namespace
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: dev         # SA lives in a different namespace
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

Verify permissions:

```bash
kubectl auth can-i list configmaps -n prod \
  --as system:serviceaccount:dev:my-app
# yes
```

## When to Create Dedicated Service Accounts

**Always create a dedicated service account** when a Pod needs to interact with the Kubernetes API or any system that uses service account tokens for authentication. Specific situations:

| Scenario | Why a dedicated SA matters |
|---|---|
| App calls the Kubernetes API | You need specific RBAC permissions. Binding Roles to `default` grants those permissions to every Pod in the namespace. |
| CI/CD runners (Argo, Tekton) | Pipeline steps need permissions to create/delete resources. Isolate them per pipeline. |
| Controllers and operators | Custom controllers watch and modify resources. Give them exactly the permissions they need. |
| Pulling from private registries | Attach `imagePullSecrets` to a specific SA rather than the `default` SA. |
| External service authentication | Workload identity (GKE, EKS IRSA) maps Kubernetes SAs to cloud IAM roles. Each workload needs its own SA for granular cloud permissions. |
| Audit trail | API audit logs record the service account name. Dedicated SAs make it clear which application performed an action. |

As a rule of thumb: if you are writing a Deployment manifest, include a `serviceAccountName` field and create a matching ServiceAccount. This costs almost nothing and prevents accidental privilege sharing.
