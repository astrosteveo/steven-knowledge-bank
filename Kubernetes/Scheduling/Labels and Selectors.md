---
tags:
  - kubernetes
  - kubernetes/scheduling
topic: Scheduling
---

# Labels and Selectors

## What Are Labels?

Labels are **key-value pairs** attached to Kubernetes objects (Pods, Nodes, Services, Deployments, etc.). They are the primary mechanism for organizing, grouping, and selecting resources. Unlike names or UIDs, labels are not unique -- many objects can share the same label.

```yaml
metadata:
  labels:
    app: web-frontend
    env: production
    tier: frontend
    version: v2.1.0
```

Labels serve two core purposes:

1. **Organization** -- Group and categorize objects in ways meaningful to your team.
2. **Selection** -- Other objects and controllers use label selectors to find and act on specific resources (e.g., a Service routing traffic to Pods with `app: web-frontend`).

## Label Syntax Rules

### Key Format

A label key has two parts: an optional **prefix** and a required **name**, separated by a `/`.

```
<prefix>/<name>

Examples:
  app                          # name only (no prefix)
  app.kubernetes.io/name       # prefix + name
  company.com/team             # custom prefix
```

| Part | Rules |
|---|---|
| **Prefix** | Optional. Must be a DNS subdomain (max 253 characters). If omitted, the label is assumed to be private to the user. The `kubernetes.io/` and `k8s.io/` prefixes are reserved for Kubernetes core components. |
| **Name** | Required. Max 63 characters. Must start and end with an alphanumeric character. May contain dashes (`-`), underscores (`_`), dots (`.`), and alphanumerics between. |

### Value Format

| Rule | Detail |
|---|---|
| Max length | 63 characters |
| Allowed characters | Alphanumerics, dashes (`-`), underscores (`_`), dots (`.`) |
| Start/end | Must begin and end with an alphanumeric character |
| Empty string | Allowed (a key with an empty value is valid) |

## Common Labeling Conventions

Kubernetes recommends a set of standard labels under the `app.kubernetes.io` prefix. Using these consistently makes tools like Helm, ArgoCD, and dashboards work better out of the box.

| Label | Purpose | Example Value |
|---|---|---|
| `app.kubernetes.io/name` | The name of the application | `mysql` |
| `app.kubernetes.io/instance` | A unique name identifying the instance of an application | `mysql-prod` |
| `app.kubernetes.io/version` | The current version of the application | `5.7.21` |
| `app.kubernetes.io/component` | The component within the architecture | `database` |
| `app.kubernetes.io/part-of` | The name of a higher-level application this is part of | `wordpress` |
| `app.kubernetes.io/managed-by` | The tool managing the operation of the application | `helm` |

```yaml
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-prod
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
```

## Adding and Removing Labels Imperatively

```bash
# Add a label to a pod
kubectl label pod my-pod env=production

# Add a label to a node
kubectl label node worker-1 disktype=ssd

# Overwrite an existing label (--overwrite required)
kubectl label pod my-pod env=staging --overwrite

# Remove a label (trailing minus sign)
kubectl label pod my-pod env-

# Add labels to all pods in a namespace
kubectl label pods --all tier=backend -n my-namespace

# Show labels in output
kubectl get pods --show-labels
kubectl get pods -L app,env    # show specific labels as columns
```

## Selectors

Selectors are queries that filter objects by their labels. There are two types.

### Equality-Based Selectors

Use `=`, `==` (both mean equal), and `!=` (not equal).

```bash
# Select pods where env equals production
kubectl get pods -l env=production

# Select pods where env is NOT staging
kubectl get pods -l env!=staging

# Multiple conditions (comma = AND)
kubectl get pods -l env=production,tier=frontend
```

### Set-Based Selectors

Use `in`, `notin`, and `exists` (key exists, regardless of value).

```bash
# Select pods where env is either production or staging
kubectl get pods -l 'env in (production, staging)'

# Select pods where env is NOT dev or test
kubectl get pods -l 'env notin (dev, test)'

# Select pods that HAVE the "tier" label (any value)
kubectl get pods -l tier

# Select pods that DO NOT have the "tier" label
kubectl get pods -l '!tier'

# Combine set-based and equality-based
kubectl get pods -l 'env in (production),tier=frontend'
```

## matchLabels vs matchExpressions

In manifests (Deployments, Services, Jobs, etc.), selectors appear in two forms. Both go under the `selector` field and can be combined -- when combined, all conditions must match (AND logic).

### matchLabels

Simple equality matching. Each entry is an implicit `key = value` check.

```yaml
selector:
  matchLabels:
    app: web-frontend
    env: production
```

### matchExpressions

More flexible. Supports set-based operators: `In`, `NotIn`, `Exists`, `DoesNotExist`.

```yaml
selector:
  matchExpressions:
    - key: env
      operator: In
      values:
        - production
        - staging
    - key: tier
      operator: Exists
```

### Combined Example

```yaml
selector:
  matchLabels:
    app: web-frontend
  matchExpressions:
    - key: env
      operator: NotIn
      values:
        - dev
```

## How Services Use Selectors

A Service finds its target Pods using a label selector defined in `spec.selector`. The Service continuously watches for Pods matching the selector and updates its Endpoints list.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: web-frontend        # routes traffic to all Pods with this label
    tier: frontend
  ports:
    - port: 80
      targetPort: 8080
```

```
  +-----------------+
  |  frontend-svc   |
  |  (ClusterIP)    |
  +--------+--------+
           |  selector: app=web-frontend, tier=frontend
           |
    +------+------+------+
    |      |      |      |
  +---+  +---+  +---+  +---+
  |Pod|  |Pod|  |Pod|  |Pod|   <-- all have labels app=web-frontend, tier=frontend
  +---+  +---+  +---+  +---+
```

Services only support `matchLabels`-style selectors (simple equality). They do not support `matchExpressions`.

## How Deployments and ReplicaSets Use Selectors

A Deployment uses `spec.selector` to identify which Pods it manages. The selector must match the labels in `spec.template.metadata.labels`. This is a hard requirement -- Kubernetes rejects the Deployment if they don't match.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:              # must match template labels below
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend     # these labels must satisfy the selector
        version: v2.1.0       # extra labels are fine
    spec:
      containers:
        - name: app
          image: frontend:2.1.0
```

Key rules:

- The `selector` is **immutable** after creation. You cannot change a Deployment's selector without recreating it.
- The template labels can be a superset of the selector labels but must contain all of them.
- The ReplicaSet created by the Deployment inherits the selector and uses it to count running Pods.

## Annotations vs Labels

Both are key-value metadata on objects, but they serve different purposes.

| | Labels | Annotations |
|---|---|---|
| **Purpose** | Identify and select objects | Attach non-identifying metadata |
| **Used by selectors** | Yes | No |
| **Size limit** | 63-char values, 253-char prefixes | Values can be much larger (up to 256 KB total annotation data) |
| **Typical consumers** | Kubernetes controllers, Services, kubectl | External tools, operators, humans |
| **Queryable** | Yes (`-l` flag, field selectors) | No (must read the object to inspect) |

**Rule of thumb**: if you need to filter or select objects by a piece of metadata, use a label. If it's informational metadata that doesn't drive selection, use an annotation.

### Annotation Use Cases

```yaml
metadata:
  annotations:
    # Build and release information
    build.company.com/git-sha: "abc123def"
    build.company.com/pipeline-url: "https://ci.company.com/builds/456"

    # Configuration hints for tools
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    nginx.ingress.kubernetes.io/rewrite-target: /

    # Last-applied-configuration (managed by kubectl apply)
    kubectl.kubernetes.io/last-applied-configuration: '{"apiVersion":"v1",...}'

    # Human-readable descriptions
    description: "Frontend web server for the customer portal"
```

Common annotation patterns:

| Pattern | Example |
|---|---|
| Build/release info | `git-sha`, `build-timestamp`, `pipeline-url` |
| Tool configuration | `prometheus.io/scrape`, `nginx.ingress.kubernetes.io/*` |
| Internal Kubernetes tracking | `kubectl.kubernetes.io/last-applied-configuration` |
| Change management | `kubernetes.io/change-cause` (recorded in rollout history) |
| Contact info | `team`, `owner`, `slack-channel` |
