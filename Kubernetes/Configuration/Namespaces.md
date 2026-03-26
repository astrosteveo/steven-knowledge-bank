---
tags:
  - kubernetes
  - kubernetes/configuration
topic: Configuration
---

# Namespaces

## What is a Namespace?

A namespace is a **virtual cluster** within a Kubernetes cluster. It provides a scope for names — resources within a namespace must have unique names, but the same name can exist in different namespaces. Namespaces are the primary mechanism for dividing cluster resources between multiple users, teams, or environments.

```
Kubernetes Cluster
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  default      │  │  team-alpha   │  │  team-beta    │  │
│  │               │  │               │  │               │  │
│  │  Pods         │  │  Pods         │  │  Pods         │  │
│  │  Services     │  │  Services     │  │  Services     │  │
│  │  ConfigMaps   │  │  ConfigMaps   │  │  ConfigMaps   │  │
│  │  Secrets      │  │  Secrets      │  │  Secrets      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│                                                          │
│  Cluster-scoped resources (Nodes, PVs, Namespaces,       │
│  ClusterRoles, etc.) exist outside namespaces             │
└──────────────────────────────────────────────────────────┘
```

Namespaces provide **logical** isolation, not physical isolation. Pods in different namespaces still share the same underlying nodes and network unless you add Network Policies or other controls.

## Default Namespaces

Every Kubernetes cluster starts with four namespaces:

| Namespace | Purpose |
|---|---|
| `default` | Where resources go if you don't specify a namespace. Intended for getting started; avoid using it in production. |
| `kube-system` | Houses Kubernetes system components — the API server, scheduler, controller manager, kube-proxy, CoreDNS, and other infrastructure Pods. Don't deploy application workloads here. |
| `kube-public` | Readable by all users (including unauthenticated). Typically contains the `cluster-info` ConfigMap. Rarely used directly. |
| `kube-node-lease` | Contains Lease objects for each node. Kubelets send heartbeats here so the control plane can detect node failures efficiently. Don't touch this namespace. |

## Creating and Managing Namespaces

### Imperative

```bash
# Create
kubectl create namespace staging

# Delete (WARNING: deletes ALL resources in the namespace)
kubectl delete namespace staging

# List all namespaces
kubectl get namespaces

# View details
kubectl describe namespace staging
```

### Declarative

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
    team: platform
```

```bash
kubectl apply -f namespace.yaml
```

### Adding Labels to Existing Namespaces

```bash
kubectl label namespace staging env=staging team=platform
```

Labels on namespaces are important — they're used by Network Policies and admission controllers to select namespaces.

## Setting Preferred Namespace in kubeconfig

By default, `kubectl` commands operate in the `default` namespace. You can change this:

```bash
# Set the default namespace for your current context
kubectl config set-context --current --namespace=team-alpha

# Verify
kubectl config view --minify | grep namespace

# Run a command in a different namespace without changing context
kubectl get pods -n kube-system
```

For frequent namespace switching, consider the `kubens` tool (part of `kubectx`):

```bash
# Switch namespace
kubens team-alpha

# Switch back to previous namespace
kubens -
```

## Resource Quotas per Namespace

Namespaces are the boundary for ResourceQuota enforcement. Each namespace can have its own quota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
```

See the [[Resource Management]] note for full details on ResourceQuota and LimitRange.

## Network Policies per Namespace

By default, all Pods in a cluster can communicate with all other Pods across all namespaces. Network Policies let you restrict traffic at the namespace level:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: team-alpha
spec:
  podSelector: {}  # applies to all Pods in this namespace
  ingress:
    - from:
        # Only allow traffic from Pods in the same namespace
        - podSelector: {}
  policyTypes:
    - Ingress
```

This denies all ingress traffic from other namespaces while allowing Pods within `team-alpha` to talk to each other.

### Allow Traffic from a Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: team-alpha
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: monitoring
  policyTypes:
    - Ingress
```

**Note:** Network Policies require a CNI plugin that supports them (Calico, Cilium, Weave Net). The default kubenet CNI does not enforce Network Policies.

## RBAC Scoping with Namespaces

RBAC (Role-Based Access Control) uses namespaces to scope permissions:

| Resource | Scope | Use case |
|---|---|---|
| `Role` | Single namespace | Grant permissions within one namespace |
| `RoleBinding` | Single namespace | Bind a Role (or ClusterRole) to a user/group within one namespace |
| `ClusterRole` | Cluster-wide | Define permissions that apply across all namespaces |
| `ClusterRoleBinding` | Cluster-wide | Bind a ClusterRole to a user/group across the entire cluster |

### Namespace-Scoped RBAC Example

```yaml
# Role: allows reading Pods and Services in team-alpha
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: team-alpha
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]
---
# RoleBinding: grants the role to a specific user
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: team-alpha
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Alice can now read Pods and Services in `team-alpha` but has no access to other namespaces.

## Cross-Namespace Communication

Pods can communicate across namespaces using the full DNS name:

```
<service-name>.<namespace>.svc.cluster.local
```

Examples:

```bash
# From any namespace, reach the "api" service in the "backend" namespace
curl http://api.backend.svc.cluster.local:8080

# Short form (works from within the cluster)
curl http://api.backend:8080
```

Within the same namespace, the service name alone is sufficient:

```bash
# From a Pod in the "backend" namespace, reach another service in "backend"
curl http://database:5432
```

```
Pod in team-alpha          Pod in team-beta
┌─────────────┐            ┌─────────────┐
│ curl http:// │───────────►│ api:8080     │
│ api.team-beta│  DNS:      │              │
│ :8080        │  api.team- │              │
└─────────────┘  beta.svc.  └─────────────┘
                 cluster.local
```

Cross-namespace DNS works by default. Use Network Policies if you need to restrict it.

## When to Use Namespaces

### Good Use Cases

| Scenario | Example |
|---|---|
| **Multi-team clusters** | `team-alpha`, `team-beta`, `platform` — each team gets its own namespace with RBAC and resource quotas |
| **Environment separation** | `dev`, `staging`, `prod` — separate environments in the same cluster (though separate clusters for prod is often better) |
| **Application tiers** | `frontend`, `backend`, `data` — separate application layers for clearer ownership |
| **Multi-tenancy** | `customer-a`, `customer-b` — isolate tenant workloads (with Network Policies and RBAC) |
| **System vs application** | Keep your workloads out of `kube-system`; use dedicated namespaces |

### When NOT to Use Namespaces

- **Version separation** — don't use `app-v1` and `app-v2` namespaces. Use Deployments and labels for versioning.
- **Slightly different configs** — use ConfigMaps and Kustomize overlays, not separate namespaces.
- **Single-user clusters** — if you're the only user, `default` is fine for experimentation.

## Namespaced vs Cluster-Scoped Resources

Not all Kubernetes resources live in a namespace:

| Namespaced (most resources) | Cluster-Scoped |
|---|---|
| Pods | Nodes |
| Services | Namespaces |
| Deployments | PersistentVolumes |
| ConfigMaps | ClusterRoles |
| Secrets | ClusterRoleBindings |
| ReplicaSets | StorageClasses |
| Jobs, CronJobs | IngressClasses |
| NetworkPolicies | CustomResourceDefinitions |
| ServiceAccounts | PriorityClasses |
| Roles, RoleBindings | |

Check which resources are namespaced:

```bash
# List all namespaced resources
kubectl api-resources --namespaced=true

# List all cluster-scoped resources
kubectl api-resources --namespaced=false
```

## Best Practices

1. **Don't use the `default` namespace in production** — create explicit namespaces for every workload. The `default` namespace has no special properties; it just lacks the intentionality of a named namespace.

2. **Apply ResourceQuota and LimitRange to every namespace** — prevents resource starvation and ensures every Pod has resource constraints.

3. **Use labels on namespaces** — they're required for namespace-based Network Policies and useful for monitoring and cost allocation.

4. **Implement RBAC per namespace** — give each team access only to their namespace(s). Start restrictive and grant more access as needed.

5. **Add Network Policies for multi-tenant clusters** — namespaces alone don't prevent network traffic between tenants.

6. **Use a naming convention** — consistent naming (e.g., `<team>-<env>`, `<project>-<component>`) makes it easier to manage namespaces at scale.

7. **Be careful deleting namespaces** — `kubectl delete namespace X` deletes **everything** in that namespace. There is no undo.

8. **Don't over-namespace** — if you have two services that are tightly coupled and always deployed together, putting them in separate namespaces adds complexity without benefit. Namespace along organizational or isolation boundaries, not per-microservice.
