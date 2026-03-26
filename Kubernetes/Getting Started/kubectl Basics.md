---
tags:
  - kubernetes
  - kubernetes/getting-started
topic: Getting Started
---

# kubectl Basics

## What is kubectl?

`kubectl` is the command-line tool for communicating with the Kubernetes API server. Every operation you perform on a cluster — creating resources, inspecting state, debugging problems, rolling out updates — goes through `kubectl`. It reads your kubeconfig to know which cluster to talk to and how to authenticate.

## Configuration and Contexts

`kubectl` determines which cluster to target using the **current context** in your kubeconfig file (`~/.kube/config`). A context binds a cluster, a user, and an optional default namespace together.

```bash
# See which cluster you're pointing at
kubectl config current-context

# List all available contexts
kubectl config get-contexts

# Switch to a different context
kubectl config use-context staging

# Set the default namespace for the active context
kubectl config set-context --current --namespace=my-app
```

## Command Structure

Every `kubectl` command follows this pattern:

```
kubectl [command] [TYPE] [NAME] [flags]
```

| Part | Description | Example |
|---|---|---|
| **command** | The operation to perform | `get`, `create`, `apply`, `delete`, `describe` |
| **TYPE** | The resource type (case-insensitive, singular/plural/abbreviated) | `pod`, `pods`, `po` |
| **NAME** | The specific resource name (optional — omit to target all of that type) | `nginx`, `my-deployment` |
| **flags** | Modifiers for the command | `-n dev`, `-o yaml`, `--all-namespaces` |

```bash
kubectl get pods                         # all Pods in the current namespace
kubectl get pod nginx                    # a specific Pod
kubectl get pods -n kube-system          # Pods in a different namespace
kubectl get pods -A                      # Pods across all namespaces
kubectl get pods -l app=nginx            # Pods matching a label selector
```

## CRUD Operations

### create — Imperatively create a resource

```bash
kubectl create namespace dev
kubectl create deployment nginx --image=nginx:1.27
kubectl create configmap app-config --from-literal=ENV=production
kubectl create secret generic db-creds --from-literal=password=s3cret
kubectl create service clusterip my-svc --tcp=80:8080
```

### get — List resources

```bash
kubectl get pods                         # Pods in current namespace
kubectl get deployments                  # Deployments
kubectl get services                     # Services
kubectl get all                          # Common resource types combined
kubectl get nodes                        # Cluster nodes
kubectl get events --sort-by=.metadata.creationTimestamp  # recent events
```

### describe — Show detailed information about a resource

```bash
kubectl describe pod nginx               # full details: events, conditions, containers
kubectl describe node worker-1           # node capacity, allocatable, conditions
kubectl describe deployment nginx        # rollout status, conditions, events
kubectl describe service my-svc          # endpoints, selector, ports
```

`describe` is your go-to debugging command — it shows events at the bottom that often explain why something is failing.

### edit — Open a resource in your editor

```bash
kubectl edit deployment nginx            # opens in $EDITOR, applies on save
kubectl edit svc my-svc                  # edit a Service
KUBE_EDITOR="code --wait" kubectl edit deployment nginx  # use VS Code
```

### delete — Remove resources

```bash
kubectl delete pod nginx                 # delete a specific Pod
kubectl delete deployment nginx          # delete a Deployment (and its Pods)
kubectl delete -f manifest.yaml          # delete everything defined in a file
kubectl delete pods --all -n dev         # delete all Pods in a namespace
kubectl delete pod nginx --grace-period=0 --force  # force-delete a stuck Pod
```

### apply — Declaratively create or update resources

```bash
kubectl apply -f deployment.yaml         # create or update from a file
kubectl apply -f ./manifests/            # apply all files in a directory
kubectl apply -f https://example.com/manifest.yaml  # apply from a URL
kubectl apply -k ./overlays/production/  # apply a Kustomize overlay
```

`apply` is the preferred method for managing resources. It tracks changes using the `kubectl.kubernetes.io/last-applied-configuration` annotation, enabling three-way merges on subsequent applies.

## Output Formats

Control how `kubectl` displays results using the `-o` flag:

| Flag | Output | Use case |
|---|---|---|
| *(default)* | Human-readable table | Quick overview |
| `-o wide` | Table with additional columns (node, IP, etc.) | See which node a Pod is on |
| `-o yaml` | Full YAML representation | Inspect the complete resource spec |
| `-o json` | Full JSON representation | Piping to `jq` or other tools |
| `-o jsonpath='{...}'` | Extract specific fields using JSONPath | Scripting and automation |
| `-o name` | Just the resource type/name | Piping to other kubectl commands |
| `-o custom-columns=...` | User-defined columns | Custom reports |

```bash
# See which node each Pod is running on
kubectl get pods -o wide

# Get the full YAML for a Deployment
kubectl get deployment nginx -o yaml

# Extract just the image for each container
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Extract Pod names and their statuses
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase'

# Get the cluster IP of a Service
kubectl get svc my-svc -o jsonpath='{.spec.clusterIP}'

# Pipe resource names into another command
kubectl get pods -o name | xargs -I {} kubectl delete {}
```

## Working with Namespaces

Namespaces scope resources within a cluster. Most `kubectl` commands default to the `default` namespace.

```bash
# List all namespaces
kubectl get namespaces

# Target a specific namespace
kubectl get pods -n kube-system

# Target all namespaces
kubectl get pods -A
kubectl get pods --all-namespaces        # equivalent

# Set the default namespace so you don't have to type -n every time
kubectl config set-context --current --namespace=my-app

# Create a namespace
kubectl create namespace staging
```

## Imperative vs Declarative Commands

| Approach | Command | When to use |
|---|---|---|
| **Imperative** | `kubectl create`, `kubectl run`, `kubectl expose`, `kubectl scale` | Quick experiments, one-off tasks, learning |
| **Declarative** | `kubectl apply -f` | Production workflows, version-controlled infrastructure, team collaboration |

```bash
# Imperative: quick and direct, but not reproducible
kubectl run nginx --image=nginx:1.27 --port=80
kubectl expose deployment nginx --type=NodePort --port=80
kubectl scale deployment nginx --replicas=5

# Declarative: reproducible and auditable
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

A useful hybrid: generate a manifest imperatively, then manage it declaratively.

```bash
# Generate a manifest without creating the resource
kubectl create deployment nginx --image=nginx:1.27 --dry-run=client -o yaml > deployment.yaml

# Review, edit, then apply
kubectl apply -f deployment.yaml
```

## Discovering API Resources

### kubectl explain

Browse the API schema directly from the terminal — shows the fields, types, and descriptions for any resource:

```bash
# Top-level fields of a Pod
kubectl explain pod

# Drill into nested fields
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources

# Recursive: show all fields at once
kubectl explain deployment --recursive

# Specific API version
kubectl explain deployment --api-version=apps/v1
```

### kubectl api-resources

List every resource type the cluster supports, along with its short name, API group, and whether it's namespaced:

```bash
# List all resource types
kubectl api-resources

# Only namespaced resources
kubectl api-resources --namespaced=true

# Only cluster-scoped resources
kubectl api-resources --namespaced=false

# Filter by API group
kubectl api-resources --api-group=apps

# Show verbs supported for each resource
kubectl api-resources -o wide
```

| Column | What it tells you |
|---|---|
| **NAME** | The resource type (e.g., `deployments`) |
| **SHORTNAMES** | Abbreviations you can use (e.g., `deploy`, `po`, `svc`, `ns`) |
| **APIVERSION** | The API group and version (e.g., `apps/v1`) |
| **NAMESPACED** | Whether the resource lives in a namespace or is cluster-wide |
| **KIND** | The object kind used in manifests (e.g., `Deployment`) |

Common short names worth memorizing:

| Short name | Resource |
|---|---|
| `po` | Pod |
| `svc` | Service |
| `deploy` | Deployment |
| `rs` | ReplicaSet |
| `ds` | DaemonSet |
| `sts` | StatefulSet |
| `cm` | ConfigMap |
| `ns` | Namespace |
| `no` | Node |
| `ing` | Ingress |
| `pv` | PersistentVolume |
| `pvc` | PersistentVolumeClaim |
| `sa` | ServiceAccount |
| `hpa` | HorizontalPodAutoscaler |
