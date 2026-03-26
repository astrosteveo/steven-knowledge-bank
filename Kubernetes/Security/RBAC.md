---
tags:
  - kubernetes
  - kubernetes/security
topic: Security
---

# RBAC

## What is RBAC?

**Role-Based Access Control** (RBAC) is the standard authorization mechanism in Kubernetes. It lets you define **who** (subject) can do **what** (verbs) on **which resources** (API objects). RBAC is enabled by default on all modern Kubernetes clusters and is controlled through four API objects that work together.

```
  Subject              Binding              Role
  (who)                (glue)               (permissions)

  +--------+       +-------------+       +----------+
  | User   |       |             |       |          |
  | Group  |<------| RoleBinding |------>|  Role    |
  | SA     |       |             |       |          |
  +--------+       +-------------+       +----------+
                       (namespaced)         (namespaced)

  +--------+       +--------------------+   +-------------+
  | User   |       |                    |   |             |
  | Group  |<------| ClusterRoleBinding |-->| ClusterRole |
  | SA     |       |                    |   |             |
  +--------+       +--------------------+   +-------------+
                       (cluster-wide)         (cluster-wide)
```

## RBAC API Objects

### Role (Namespaced Permissions)

A **Role** grants permissions within a single namespace. It contains a list of **rules** that specify which verbs are allowed on which resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]           # "" = core API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]   # subresource
    verbs: ["get"]
```

### ClusterRole (Cluster-Wide Permissions)

A **ClusterRole** works like a Role but applies cluster-wide. Use it for:

- Cluster-scoped resources (nodes, namespaces, persistentvolumes)
- Non-resource endpoints (`/healthz`, `/metrics`)
- Permissions you want to reuse across multiple namespaces via RoleBindings

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/healthz", "/healthz/*"]
    verbs: ["get"]
```

### RoleBinding (Grants a Role to Subjects in a Namespace)

A **RoleBinding** connects a Role (or ClusterRole) to one or more subjects within a specific namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: ci-bot
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

A RoleBinding can reference a ClusterRole. The permissions are still scoped to the binding's namespace -- this is a common pattern for reusing a ClusterRole across multiple namespaces without granting cluster-wide access.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-staging
  namespace: staging
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole       # ClusterRole, but scoped to 'staging' namespace
  name: pod-reader-global
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding (Grants a ClusterRole Cluster-Wide)

A **ClusterRoleBinding** grants a ClusterRole across the entire cluster -- every namespace and all cluster-scoped resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-node-reader
subjects:
  - kind: Group
    name: infra-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

## Subjects

Subjects are the entities that receive permissions through bindings.

| Subject | Description | Example `name` |
|---|---|---|
| **User** | A human identity, authenticated externally (X.509 cert, OIDC token, etc.). Kubernetes has no User API object. | `alice`, `alice@example.com` |
| **Group** | A set of users, also sourced from the authenticator. Some groups are built-in. | `infra-team`, `system:masters` |
| **ServiceAccount** | An in-cluster identity for Pods. Managed by Kubernetes. Namespaced. | `ci-bot`, `default` |

Built-in groups:

| Group | Meaning |
|---|---|
| `system:authenticated` | All authenticated users |
| `system:unauthenticated` | All unauthenticated requests |
| `system:masters` | Super-admin group, bypasses all RBAC -- equivalent to root |
| `system:serviceaccounts` | All service accounts across all namespaces |
| `system:serviceaccounts:<ns>` | All service accounts in a specific namespace |

## Common Verbs

| Verb | HTTP Method | Description |
|---|---|---|
| `get` | GET (single) | Read a specific resource |
| `list` | GET (collection) | List all resources of a type |
| `watch` | GET (streaming) | Watch for real-time changes |
| `create` | POST | Create a new resource |
| `update` | PUT | Replace an entire resource |
| `patch` | PATCH | Partially modify a resource |
| `delete` | DELETE (single) | Delete a specific resource |
| `deletecollection` | DELETE (collection) | Delete all resources of a type |

Less common but useful verbs:

| Verb | Used for |
|---|---|
| `bind` | Creating RoleBindings/ClusterRoleBindings that reference a Role |
| `escalate` | Modifying Roles/ClusterRoles to add permissions you don't already have |
| `impersonate` | Acting as another user, group, or service account |
| `use` | PodSecurityPolicies (deprecated), but still relevant for some custom resources |

## Aggregated ClusterRoles

Aggregated ClusterRoles combine permissions from multiple ClusterRoles using label selectors. When a new ClusterRole matches the selector, its rules are automatically merged in. This is how the built-in `admin`, `edit`, and `view` roles stay up to date when CRDs are installed.

```yaml
# The aggregating ClusterRole (collects rules from others)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-view
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-monitoring-view: "true"
rules: []   # rules are auto-filled by the controller
---
# A ClusterRole that contributes its rules to the aggregate
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-metrics-reader
  labels:
    rbac.example.com/aggregate-to-monitoring-view: "true"
rules:
  - apiGroups: ["monitoring.coreos.com"]
    resources: ["prometheuses", "servicemonitors"]
    verbs: ["get", "list", "watch"]
```

The built-in `edit` role aggregates any ClusterRole labeled `rbac.authorization.k8s.io/aggregate-to-edit: "true"`, so installing a CRD with a matching ClusterRole automatically grants `edit` users access to the new resource type.

## Checking Access

Use `kubectl auth can-i` to test whether a subject has a specific permission.

```bash
# Can I create deployments in the dev namespace?
kubectl auth can-i create deployments --namespace dev
# yes

# Can a specific user list pods?
kubectl auth can-i list pods --namespace dev --as alice
# no

# Can a service account delete secrets?
kubectl auth can-i delete secrets --namespace prod \
  --as system:serviceaccount:prod:ci-bot
# no

# List all permissions for the current user in a namespace
kubectl auth can-i --list --namespace dev
```

The `--as` flag uses the **impersonation** API. You need impersonation permissions to use it.

## Default ClusterRoles

Kubernetes ships with several default ClusterRoles. The four most important ones form a hierarchy of decreasing privilege.

| ClusterRole | Scope | Typical use |
|---|---|---|
| **cluster-admin** | Cluster-wide, all resources, all verbs | Full superuser. Grants complete control over every resource in every namespace. |
| **admin** | Namespace-scoped (via RoleBinding) | Full control within a namespace. Can create Roles and RoleBindings. Cannot modify the namespace itself or resource quotas. |
| **edit** | Namespace-scoped (via RoleBinding) | Read/write most resources in a namespace. Cannot view or modify Roles or RoleBindings. |
| **view** | Namespace-scoped (via RoleBinding) | Read-only access to most resources. Cannot view Secrets (to prevent credential exposure) or Roles/RoleBindings. |

```
  cluster-admin
       |
       | (superset of)
       v
     admin    ── can manage Roles/RoleBindings
       |
       | (superset of)
       v
     edit     ── can read/write workloads, configmaps, secrets
       |
       | (superset of)
       v
     view     ── read-only (no Secrets)
```

Other notable default ClusterRoles:

| ClusterRole | Purpose |
|---|---|
| `system:node` | Permissions for kubelets |
| `system:kube-scheduler` | Permissions for the scheduler |
| `system:kube-controller-manager` | Permissions for the controller manager |
| `system:discovery` | Read-only access to API discovery endpoints |

## Best Practices

1. **Principle of least privilege** -- Grant only the permissions a subject actually needs. Start with `view` and add permissions incrementally.
2. **Avoid cluster-admin** -- Never bind `cluster-admin` to regular users or application service accounts. Reserve it for break-glass scenarios.
3. **Use namespaced Roles over ClusterRoles** -- Prefer `Role` + `RoleBinding` when permissions don't need to span namespaces.
4. **Dedicated service accounts per workload** -- Don't rely on the `default` service account. Create one per application so you can grant granular permissions.
5. **Audit bindings regularly** -- Review who has access to what. Use `kubectl get clusterrolebindings -o wide` and `kubectl get rolebindings -n <ns> -o wide` to list bindings.
6. **Don't modify default ClusterRoles directly** -- They are reconciled by the system. Use aggregation labels to extend them instead.
7. **Restrict `escalate` and `bind` verbs** -- These verbs allow privilege escalation. Only cluster admins should have them.
8. **Use Groups for team access** -- Bind Roles to Groups rather than individual Users. This scales better and aligns with identity provider group management.
