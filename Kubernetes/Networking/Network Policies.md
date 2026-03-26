---
tags:
  - kubernetes
  - kubernetes/networking
topic: Networking
---

# Network Policies

## What Are Network Policies?

A NetworkPolicy is a Kubernetes resource that controls **layer 3/4 traffic flow** to and from Pods. Think of them as firewall rules scoped to Pods. They let you define which Pods can talk to each other and which external IPs they can reach.

```
Without NetworkPolicy:               With NetworkPolicy:

  Pod A ◄──► Pod B                     Pod A ──► Pod B    ✓ (allowed)
  Pod A ◄──► Pod C                     Pod A ──► Pod C    ✗ (denied)
  Pod B ◄──► Pod C                     Pod B ──► Pod C    ✓ (allowed)
                                       Pod C ──► Pod A    ✗ (denied)

  All traffic allowed                  Only explicitly permitted
  by default                           traffic flows
```

## Default Behavior

By default, Kubernetes allows **all ingress and egress traffic** between all Pods. A Pod can communicate with any other Pod in any namespace, and can reach any external IP.

Once you apply **any** NetworkPolicy that selects a Pod, that Pod's traffic is restricted:

- **Ingress:** Only traffic matching an ingress rule is allowed. All other inbound traffic is denied.
- **Egress:** Only traffic matching an egress rule is allowed. All other outbound traffic is denied.

If a NetworkPolicy has `policyTypes: ["Ingress"]` but no ingress rules, all ingress is denied. The same applies to egress.

## NetworkPolicy YAML Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  # Which Pods this policy applies to
  podSelector:
    matchLabels:
      app: api

  # Which directions this policy covers
  policyTypes:
    - Ingress
    - Egress

  # Inbound rules — who can reach these Pods
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
        - namespaceSelector:
            matchLabels:
              purpose: monitoring
        - ipBlock:
            cidr: 10.0.0.0/8
            except:
              - 10.0.1.0/24
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 8443

  # Outbound rules — where these Pods can connect
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}        # any namespace
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

## Ingress Rules

Ingress rules define which sources can send traffic **to** the selected Pods.

### podSelector — Allow from Pods in the Same Namespace

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            role: frontend
    ports:
      - protocol: TCP
        port: 8080
```

Only Pods in the **same namespace** with `role: frontend` can reach port 8080.

### namespaceSelector — Allow from Pods in Other Namespaces

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            env: staging
    ports:
      - protocol: TCP
        port: 8080
```

All Pods in any namespace labeled `env: staging` can reach port 8080.

### Combined podSelector + namespaceSelector

There is a critical distinction between **AND** and **OR** logic:

```yaml
# OR — two separate selectors (two list items)
# Pods with role=frontend in the SAME namespace
# OR any Pod in namespaces with env=staging
ingress:
  - from:
      - podSelector:
          matchLabels:
            role: frontend
      - namespaceSelector:
          matchLabels:
            env: staging

# AND — combined in a single selector (one list item)
# Pods with role=frontend that are ALSO in namespaces with env=staging
ingress:
  - from:
      - podSelector:
          matchLabels:
            role: frontend
        namespaceSelector:
          matchLabels:
            env: staging
```

> **This is the single most common NetworkPolicy mistake.** Accidentally using OR when you mean AND (or vice versa) can either block legitimate traffic or open access too broadly.

### ipBlock — Allow from External IP Ranges

```yaml
ingress:
  - from:
      - ipBlock:
          cidr: 203.0.113.0/24
          except:
            - 203.0.113.128/25
    ports:
      - protocol: TCP
        port: 443
```

> **Note:** `ipBlock` matches packet source IPs. For traffic coming through a LoadBalancer or NodePort, the source IP may be the node IP (due to SNAT), not the original client IP. Set `externalTrafficPolicy: Local` on the Service to preserve client IPs.

## Egress Rules

Egress rules define which destinations the selected Pods can send traffic **to**. The selectors (`podSelector`, `namespaceSelector`, `ipBlock`) work the same as ingress.

```yaml
egress:
  # Allow database access
  - to:
      - podSelector:
          matchLabels:
            app: postgres
    ports:
      - protocol: TCP
        port: 5432

  # Allow DNS (critical — without this, name resolution breaks)
  - to:
      - namespaceSelector: {}
        podSelector:
          matchLabels:
            k8s-app: kube-dns
    ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53

  # Allow HTTPS to external APIs
  - to:
      - ipBlock:
          cidr: 0.0.0.0/0
          except:
            - 10.0.0.0/8
            - 172.16.0.0/12
            - 192.168.0.0/16
    ports:
      - protocol: TCP
        port: 443
```

> **Always allow DNS egress** when writing egress rules. Without it, Pods cannot resolve any names and most applications will break.

## Port-Level Filtering

Ports can be specified with protocol, port number, and optionally an `endPort` for port ranges:

```yaml
ports:
  # Single port
  - protocol: TCP
    port: 8080

  # Named port (matches the port name in the Pod spec)
  - protocol: TCP
    port: http

  # Port range (Kubernetes 1.25+)
  - protocol: TCP
    port: 32000
    endPort: 32768
```

If no `ports` are specified in a rule, **all ports** are allowed for that rule.

## Default Policies

### Default Deny All Ingress

Selects all Pods in the namespace and specifies Ingress with no rules, blocking all inbound traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}           # empty = all Pods in this namespace
  policyTypes:
    - Ingress
  # no ingress rules = deny all ingress
```

### Default Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # no egress rules = deny all egress
```

### Default Deny All Traffic (Ingress + Egress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Allow All Ingress (Explicit)

Useful to override a default-deny in a specific namespace or for specific Pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - {}                    # empty rule = allow from everywhere
```

### Allow All Egress (Explicit)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - {}
```

## Combining Policies — Additive Behavior

NetworkPolicies are **additive** (union). If multiple policies select the same Pod, the allowed traffic is the **combined set** of all rules across all policies. No policy can remove access granted by another policy.

```
Policy A allows ingress from Pod X on port 80
Policy B allows ingress from Pod Y on port 443

Result: Pod receives traffic from X:80 AND Y:443
```

This means:

- You cannot write a "deny" rule that overrides an "allow" in another policy.
- The only way to deny traffic is to **not have any rule that allows it** across all applicable policies.
- Start with a default-deny policy, then add specific allow policies.

## CNI Plugins That Support Network Policies

NetworkPolicies are implemented by the **CNI plugin**, not by Kubernetes itself. If your CNI does not support them, the NetworkPolicy objects are accepted by the API but **silently ignored**.

| CNI Plugin | NetworkPolicy Support | Additional Features |
|---|---|---|
| **Calico** | Full | Extended policies: global policies, DNS-based rules, application layer |
| **Cilium** | Full | L7 policies (HTTP, gRPC, Kafka), CiliumNetworkPolicy CRD |
| **Weave Net** | Full | Supports ingress and egress |
| **Antrea** | Full | Tiered policies, ClusterNetworkPolicy CRD |
| **Kube-router** | Full | Uses iptables/ipsets |
| **Flannel** | None | Does not enforce NetworkPolicy |
| **AWS VPC CNI** | Partial | Requires Calico addon for policy enforcement |
| **Azure CNI** | Depends | Needs Azure Network Policy Manager or Calico |

> **Flannel warning:** Flannel is a popular CNI for simple clusters, but it does not implement NetworkPolicy. If you need policies, pair it with Calico (canal) or switch to a policy-capable CNI.

## Common Patterns and Examples

### Allow Frontend to Backend Only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### Database Access from Specific Namespaces

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-access
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              access-db: "true"
          podSelector:
            matchLabels:
              needs-db: "true"
      ports:
        - protocol: TCP
          port: 5432
```

### Restrict Egress to Internal Network and DNS Only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-only-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      network: internal-only
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Allow traffic within the cluster Pod CIDR
    - to:
        - ipBlock:
            cidr: 10.244.0.0/16
    # Allow traffic to the Service CIDR
    - to:
        - ipBlock:
            cidr: 10.96.0.0/12
```

### Namespace Isolation — Default Deny with DNS Allowed

A practical starting point for securing a namespace — deny everything, then allow DNS so applications can still resolve names:

```yaml
# Step 1: Deny all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Step 2: Allow DNS egress for all Pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Then add targeted allow policies for each legitimate traffic flow in the namespace.
