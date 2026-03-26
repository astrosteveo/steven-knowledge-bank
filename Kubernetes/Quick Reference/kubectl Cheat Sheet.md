---
tags:
  - kubernetes
  - kubernetes/reference
aliases:
  - kubectl-commands
  - kubectl-reference
topic: Quick Reference
created: 2026-03-26
---

# kubectl Cheat Sheet

A categorized reference for the most commonly used `kubectl` commands. Each section groups related operations so you can quickly find what you need during day-to-day cluster work.

## Cluster Info

```bash
kubectl cluster-info                          # display API server and CoreDNS endpoints
kubectl get nodes                             # list all nodes and their status
kubectl get nodes -o wide                     # nodes with extra detail (IP, OS, kernel, runtime)
kubectl get componentstatuses                 # health of scheduler, controller-manager, etcd (deprecated in 1.19+)
kubectl api-resources                         # every resource type the cluster supports
kubectl api-resources --namespaced=true       # only namespaced resources
kubectl api-versions                          # all API group/version combos (e.g. apps/v1)
kubectl version --short                       # client and server versions
```

## Context and Config

Your kubeconfig (`~/.kube/config`) can hold multiple clusters, users, and contexts.

```bash
kubectl config view                           # display merged kubeconfig
kubectl config view --minify                  # only the current context's config
kubectl config get-contexts                   # list all contexts (* marks active)
kubectl config current-context                # print the active context name
kubectl config use-context staging            # switch active context
kubectl config set-context --current \
  --namespace=my-ns                           # set default namespace for current context
kubectl config set-context dev-ctx \
  --cluster=dev --user=dev-admin \
  --namespace=development                     # create or update a named context
kubectl config delete-context old-ctx         # remove a context
```

> **Tip:** Tools like `kubectx` and `kubens` make context/namespace switching faster.

## Namespace Operations

```bash
kubectl get namespaces                        # list all namespaces
kubectl create namespace staging              # create a namespace imperatively
kubectl delete namespace staging              # delete namespace and ALL resources inside it
kubectl get all -n kube-system                # list common resources in a specific namespace
kubectl get all --all-namespaces              # list common resources across every namespace (shorthand: -A)
```

## Pod Operations

| Command | Purpose |
|---------|---------|
| `kubectl get pods` | list pods in current namespace |
| `kubectl get pods -A` | list pods across all namespaces |
| `kubectl get pods -w` | watch for real-time status changes |
| `kubectl get pods --show-labels` | display all labels on each pod |
| `kubectl get pods -l app=web` | filter by label selector |
| `kubectl get pods --field-selector status.phase=Running` | filter by field |
| `kubectl describe pod <name>` | detailed info including events |
| `kubectl logs <pod>` | stdout logs from the pod |
| `kubectl logs <pod> -c <container>` | logs from a specific container |
| `kubectl logs <pod> --previous` | logs from the previous (crashed) container |
| `kubectl logs <pod> -f` | stream logs in real time |
| `kubectl logs <pod> --since=1h` | logs from the last hour |
| `kubectl logs -l app=web --all-containers` | aggregate logs by label |
| `kubectl exec -it <pod> -- /bin/sh` | interactive shell in the pod |
| `kubectl exec <pod> -- cat /etc/hosts` | run a single command |
| `kubectl exec -it <pod> -c <ctr> -- sh` | exec into a specific container |
| `kubectl port-forward <pod> 8080:80` | forward local 8080 to pod port 80 |
| `kubectl port-forward svc/my-svc 8080:80` | forward through a service |
| `kubectl cp <pod>:/path/file ./file` | copy file from pod to local |
| `kubectl cp ./file <pod>:/path/file` | copy file from local to pod |
| `kubectl attach <pod> -it` | attach to a running process's stdin/stdout |
| `kubectl top pod` | CPU/memory usage (requires metrics-server) |
| `kubectl top pod --containers` | per-container resource usage |

## Deployment Operations

```bash
kubectl create deployment nginx \
  --image=nginx:1.25 --replicas=3             # create a deployment imperatively

kubectl get deployments                       # list deployments
kubectl describe deployment nginx             # detailed info and events

kubectl scale deployment nginx --replicas=5   # scale manually
kubectl autoscale deployment nginx \
  --min=2 --max=10 --cpu-percent=80           # create an HPA

kubectl set image deployment/nginx \
  nginx=nginx:1.26                            # update container image (triggers rollout)

kubectl rollout status deployment/nginx       # watch rollout progress
kubectl rollout history deployment/nginx      # list revision history
kubectl rollout undo deployment/nginx         # rollback to previous revision
kubectl rollout undo deployment/nginx \
  --to-revision=2                             # rollback to a specific revision
kubectl rollout restart deployment/nginx      # trigger a rolling restart
kubectl rollout pause deployment/nginx        # pause a rollout mid-way
kubectl rollout resume deployment/nginx       # resume a paused rollout
```

## Service Operations

```bash
kubectl get services                          # list services (shorthand: svc)
kubectl get svc -o wide                       # services with selector info
kubectl describe svc my-svc                   # endpoints, ports, selector detail
kubectl expose deployment nginx \
  --port=80 --target-port=8080 \
  --type=ClusterIP                            # create a service from a deployment
kubectl expose deployment nginx \
  --port=80 --type=NodePort                   # expose on a high-numbered node port
kubectl get endpoints my-svc                  # which pod IPs back the service
```

## Resource Creation and Management

```bash
kubectl apply -f manifest.yaml                # declarative create-or-update (preferred)
kubectl apply -f ./manifests/                 # apply every file in a directory
kubectl apply -f https://example.com/m.yaml   # apply from a URL
kubectl apply -k ./overlays/prod/             # apply a kustomize overlay

kubectl create -f manifest.yaml               # imperative create (errors if exists)
kubectl delete -f manifest.yaml               # delete resources defined in the file
kubectl delete pod nginx --grace-period=0 \
  --force                                     # force-delete a stuck pod

kubectl replace -f manifest.yaml              # replace resource (must already exist)
kubectl replace --force -f manifest.yaml      # delete then re-create

kubectl patch deployment nginx -p \
  '{"spec":{"replicas":5}}'                   # strategic merge patch (JSON)
kubectl patch deployment nginx --type=merge \
  -p '{"spec":{"replicas":5}}'               # JSON merge patch
```

> **Best practice:** Prefer `kubectl apply` for anything tracked in version control. Use imperative commands only for one-off debugging.

## Output Formatting

```bash
kubectl get pods -o wide                      # extra columns (node, IP, etc.)
kubectl get pod nginx -o yaml                 # full resource as YAML
kubectl get pod nginx -o json                 # full resource as JSON

kubectl get pods -o jsonpath='{.items[*].metadata.name}'          # extract names
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

kubectl get pods -o custom-columns=\
NAME:.metadata.name,STATUS:.status.phase,\
NODE:.spec.nodeName                           # custom tabular output

kubectl get pods --sort-by=.status.startTime  # sort by a field
kubectl get pods --sort-by=.metadata.creationTimestamp

kubectl get events --sort-by=.lastTimestamp   # recent events first

kubectl get pods -o name                      # just resource type/name (pod/nginx)
```

## Labels and Annotations

```bash
kubectl label pod nginx env=prod              # add a label
kubectl label pod nginx env=staging \
  --overwrite                                 # update an existing label
kubectl label pod nginx env-                  # remove a label

kubectl annotate pod nginx description="web"  # add an annotation
kubectl annotate pod nginx description-       # remove an annotation

kubectl get pods -l env=prod                  # select by label equality
kubectl get pods -l 'env in (prod,staging)'   # select by set membership
kubectl get pods -l app=web,env=prod          # multiple selectors (AND)
```

## Debugging

```bash
kubectl describe pod <name>                   # events, conditions, container statuses
kubectl logs <pod> --all-containers           # logs from every container in the pod
kubectl get events -n <ns> \
  --sort-by=.lastTimestamp                    # namespace events sorted by time
kubectl get events --field-selector \
  involvedObject.name=<pod>                   # events for a specific resource

kubectl debug <pod> -it \
  --image=busybox --target=<container>        # ephemeral debug container (1.23+)
kubectl debug node/<node> -it \
  --image=ubuntu                              # debug a node via a privileged pod

kubectl run debug --rm -it \
  --image=busybox -- sh                       # disposable troubleshooting pod
kubectl run curl --rm -it \
  --image=curlimages/curl -- \
  curl -s http://my-svc.default.svc:80       # quick network test from inside cluster
```

## Advanced

```bash
kubectl diff -f manifest.yaml                 # preview changes before applying

kubectl wait --for=condition=Ready \
  pod/nginx --timeout=60s                     # block until condition is met
kubectl wait --for=delete pod/nginx \
  --timeout=120s                              # block until resource is deleted

kubectl auth can-i create deployments         # check your own permissions
kubectl auth can-i get pods \
  --as=system:serviceaccount:ns:sa            # impersonate a service account
kubectl auth can-i '*' '*'                    # check if you're cluster-admin

kubectl explain pod.spec.containers           # built-in API docs for a field
kubectl explain deployment.spec.strategy      # works for any resource field
kubectl explain --recursive pod.spec          # full field tree

kubectl api-resources --verbs=list \
  --namespaced -o name                        # resources you can list in a namespace

kubectl get pods --v=6                        # verbose output showing API calls (6-9)
```

## Quick Reference Card

```
+--------------------------+---------------------------------------+
| Task                     | Command                               |
+--------------------------+---------------------------------------+
| Which cluster am I on?   | kubectl config current-context        |
| Switch cluster           | kubectl config use-context <ctx>      |
| Set default namespace    | kubectl config set-context --current  |
|                          |   --namespace=<ns>                    |
| Create from YAML         | kubectl apply -f <file>               |
| See what changed         | kubectl diff -f <file>                |
| Watch pod status         | kubectl get pods -w                   |
| Stream logs              | kubectl logs <pod> -f                 |
| Shell into pod           | kubectl exec -it <pod> -- sh         |
| Forward a port           | kubectl port-forward <pod> L:R       |
| Rollback a deployment    | kubectl rollout undo deploy/<name>   |
| Check permissions        | kubectl auth can-i <verb> <resource> |
| Look up API fields       | kubectl explain <resource.field>     |
+--------------------------+---------------------------------------+
```
