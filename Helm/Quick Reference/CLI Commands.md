---
tags:
  - helm
  - helm/reference
topic: Quick Reference
---

# CLI Commands

A comprehensive reference for the Helm CLI. Commands are grouped by workflow category with all commonly used flags documented.

## Installation and Upgrades

These commands handle the full release lifecycle — installing, upgrading, and rolling back charts.

### helm install

```bash
# Basic install
helm install my-release bitnami/nginx

# Install into a specific namespace (create it if it doesn't exist)
helm install my-release bitnami/nginx \
  --namespace web \
  --create-namespace

# Install with a custom values file
helm install my-release bitnami/nginx \
  --values custom-values.yaml

# Install with multiple values files (later files take precedence)
helm install my-release bitnami/nginx \
  --values base-values.yaml \
  --values production-values.yaml

# Override individual values on the command line
helm install my-release bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=LoadBalancer

# Set string values (avoids type coercion — useful for booleans/numbers that must stay strings)
helm install my-release bitnami/nginx \
  --set-string annotations."kubernetes\.io/ingress\.class"=nginx

# Wait for all resources to be ready before marking release as successful
helm install my-release bitnami/nginx \
  --wait \
  --timeout 5m
  # --timeout accepts Go duration strings: 5m, 300s, 1h30m

# Atomic install — automatically rolls back on failure
helm install my-release bitnami/nginx \
  --atomic \
  --timeout 5m
  # --atomic implies --wait

# Dry run — render templates and validate against the cluster without installing
helm install my-release bitnami/nginx \
  --dry-run

# Generate a release name automatically
helm install bitnami/nginx --generate-name
# Output: nginx-1696012345

# Install a specific chart version
helm install my-release bitnami/nginx --version 15.3.1

# Include development (pre-release) versions
helm install my-release bitnami/nginx --devel

# Install from a local chart directory
helm install my-release ./my-chart/

# Install from a .tgz archive
helm install my-release ./my-chart-1.0.0.tgz

# Install from an OCI registry
helm install my-release oci://registry.example.com/charts/my-chart --version 1.0.0
```

### helm upgrade

```bash
# Basic upgrade
helm upgrade my-release bitnami/nginx

# Upgrade or install if the release doesn't exist yet
helm upgrade my-release bitnami/nginx --install
# Shorthand: helm upgrade -i

# Reuse values from the previous release (only override what you specify)
helm upgrade my-release bitnami/nginx \
  --reuse-values \
  --set replicaCount=5

# Reset to chart defaults, then apply only the values you provide
helm upgrade my-release bitnami/nginx \
  --reset-values \
  --values production-values.yaml

# Force resource updates (deletes and recreates resources that can't be patched)
helm upgrade my-release bitnami/nginx --force

# Clean up new resources on failed upgrade
helm upgrade my-release bitnami/nginx --cleanup-on-fail

# Atomic upgrade — rolls back automatically on failure
helm upgrade my-release bitnami/nginx \
  --atomic \
  --timeout 5m

# Combine common production flags
helm upgrade my-release bitnami/nginx \
  --install \
  --namespace web \
  --create-namespace \
  --values production-values.yaml \
  --wait \
  --timeout 10m \
  --atomic
```

### helm rollback

```bash
# Roll back to the previous revision
helm rollback my-release

# Roll back to a specific revision number
helm rollback my-release 3

# Roll back with a timeout and wait
helm rollback my-release 3 --wait --timeout 5m

# Force rollback (delete and recreate resources)
helm rollback my-release 3 --force

# Clean up new resources created during a failed rollback
helm rollback my-release 3 --cleanup-on-fail
```

## Release Management

Commands for inspecting, listing, and removing releases.

### helm uninstall

```bash
# Remove a release
helm uninstall my-release

# Remove a release in a specific namespace
helm uninstall my-release --namespace web

# Keep the release history (allows future rollback with helm rollback)
helm uninstall my-release --keep-history

# Dry run — see what would be removed
helm uninstall my-release --dry-run
```

### helm list

```bash
# List releases in the current namespace
helm list
# NAME         NAMESPACE  REVISION  STATUS    CHART            APP VERSION
# my-release   default    3         deployed  nginx-15.3.1     1.25.0

# List releases across all namespaces
helm list --all-namespaces
helm list -A

# Filter releases by name (regex)
helm list --filter 'nginx.*'

# Include releases in all states (failed, pending, superseded)
helm list --all

# Show only failed releases
helm list --failed

# Show only pending releases
helm list --pending

# Output as JSON or YAML
helm list --output json
helm list --output yaml
helm list --output table   # default

# Sort by date, name, or revision
helm list --date         # sort by last updated (default)
helm list --reverse      # reverse sort order

# Limit the number of results
helm list --max 10
helm list --offset 5     # skip first 5 results
```

### helm status

```bash
# Show the status of a release
helm status my-release
# Output: name, namespace, revision, status, description, notes

# Show status of a specific revision
helm status my-release --revision 2

# Show status in a specific namespace
helm status my-release --namespace web

# Output as JSON
helm status my-release --output json
```

### helm history

```bash
# View release history
helm history my-release
# REVISION  UPDATED                   STATUS      CHART          APP VERSION  DESCRIPTION
# 1         Mon Oct 2 10:00:00 2024   superseded  nginx-15.2.0   1.25.0       Install complete
# 2         Tue Oct 3 14:30:00 2024   superseded  nginx-15.3.0   1.25.0       Upgrade complete
# 3         Wed Oct 4 09:15:00 2024   deployed    nginx-15.3.1   1.25.0       Upgrade complete

# Output as JSON
helm history my-release --output json

# Limit the number of revisions shown
helm history my-release --max 5
```

## Inspecting Releases

The `helm get` subcommands retrieve detailed information about a deployed release.

### helm get

```bash
# Get all information about a release (values + manifest + notes + hooks)
helm get all my-release

# Get the computed values for a release
helm get values my-release
# Output: the merged values used for the current revision

# Get only user-supplied values (not chart defaults)
helm get values my-release
# Helm 3 returns only user-supplied overrides by default

# Get all values including chart defaults
helm get values my-release --all

# Get the rendered Kubernetes manifest
helm get manifest my-release

# Get the release notes
helm get notes my-release

# Get the hooks
helm get hooks my-release

# Inspect values/manifest from a specific revision
helm get values my-release --revision 2
helm get manifest my-release --revision 2

# Output as JSON
helm get values my-release --output json
```

## Templating and Validation

Render and validate templates without touching the cluster.

### helm template

```bash
# Render templates locally (no cluster needed)
helm template my-release bitnami/nginx

# Render with custom values
helm template my-release bitnami/nginx --values production-values.yaml

# Show only a specific template file
helm template my-release bitnami/nginx --show-only templates/deployment.yaml

# Show multiple specific templates
helm template my-release bitnami/nginx \
  --show-only templates/deployment.yaml \
  --show-only templates/service.yaml

# Validate rendered templates against the Kubernetes API schema
helm template my-release bitnami/nginx --validate

# Set Kubernetes API versions available (useful for testing)
helm template my-release bitnami/nginx --api-versions networking.k8s.io/v1

# Override the release namespace in output
helm template my-release bitnami/nginx --namespace production

# Include CRDs in the output
helm template my-release bitnami/nginx --include-crds
```

### helm lint

```bash
# Lint a local chart for issues
helm lint ./my-chart/
# ==> Linting ./my-chart/
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed

# Lint with values (catches template rendering errors)
helm lint ./my-chart/ --values production-values.yaml

# Lint with --strict (treat warnings as errors)
helm lint ./my-chart/ --strict

# Set values for linting
helm lint ./my-chart/ --set replicaCount=3
```

## Repository Management

Commands for managing chart repositories.

### helm repo

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Add a repo with authentication
helm repo add private-repo https://charts.example.com \
  --username myuser \
  --password mypass

# Add a repo with a CA certificate
helm repo add private-repo https://charts.example.com \
  --ca-file ca.crt

# List configured repositories
helm repo list
# NAME      URL
# bitnami   https://charts.bitnami.com/bitnami
# stable    https://charts.helm.sh/stable

# Update all repository indexes (fetch latest chart listings)
helm repo update
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "bitnami" chart repository

# Update a specific repository
helm repo update bitnami

# Remove a repository
helm repo remove bitnami

# Generate an index file for a local chart repository
helm repo index ./my-charts-dir/

# Generate an index with a specific URL prefix
helm repo index ./my-charts-dir/ --url https://charts.example.com
```

## Searching for Charts

### helm search

```bash
# Search for charts in configured repos
helm search repo nginx
# NAME                  CHART VERSION  APP VERSION  DESCRIPTION
# bitnami/nginx         15.3.1         1.25.0       NGINX is an open source HTTP web server...

# Search with version constraints
helm search repo nginx --version ">=15.0.0"

# Include development/pre-release versions
helm search repo nginx --devel

# Show all available versions of a chart
helm search repo nginx --versions

# Search Artifact Hub (public chart hub)
helm search hub nginx
# URL                                               CHART VERSION  APP VERSION  DESCRIPTION
# https://artifacthub.io/packages/helm/bitnami/...  15.3.1         1.25.0       ...

# Output as JSON
helm search repo nginx --output json
```

## Chart Packaging and Distribution

### helm pull

```bash
# Download a chart archive (.tgz) to the current directory
helm pull bitnami/nginx

# Download a specific version
helm pull bitnami/nginx --version 15.3.1

# Download and untar in one step
helm pull bitnami/nginx --untar

# Download to a specific directory
helm pull bitnami/nginx --destination ./charts/

# Download and untar to a specific directory
helm pull bitnami/nginx --untar --untardir ./charts/

# Pull from OCI registry
helm pull oci://registry.example.com/charts/my-chart --version 1.0.0
```

### helm package

```bash
# Package a chart directory into a .tgz archive
helm package ./my-chart/
# Successfully packaged chart and saved it to: ./my-chart-1.0.0.tgz

# Package with a specific destination
helm package ./my-chart/ --destination ./dist/

# Package and sign with a GPG key
helm package ./my-chart/ --sign --key "John Doe" --keyring ~/.gnupg/pubring.gpg

# Override the version in Chart.yaml during packaging
helm package ./my-chart/ --version 2.0.0

# Override the appVersion during packaging
helm package ./my-chart/ --app-version 1.5.0
```

### helm push (OCI)

```bash
# Log in to an OCI registry
helm registry login registry.example.com \
  --username myuser \
  --password mypass

# Push a packaged chart to an OCI registry
helm push my-chart-1.0.0.tgz oci://registry.example.com/charts

# Log out from an OCI registry
helm registry logout registry.example.com
```

## Chart Development

### helm create

```bash
# Scaffold a new chart
helm create my-chart
# Creates:
#   my-chart/
#     Chart.yaml
#     values.yaml
#     charts/
#     templates/
#       deployment.yaml
#       service.yaml
#       serviceaccount.yaml
#       ingress.yaml
#       hpa.yaml
#       NOTES.txt
#       _helpers.tpl
#       tests/
#         test-connection.yaml
```

### helm dependency

```bash
# Download dependencies listed in Chart.yaml
helm dependency update ./my-chart/
# Downloads charts to my-chart/charts/ and generates Chart.lock

# Rebuild the charts/ directory from Chart.lock (deterministic)
helm dependency build ./my-chart/

# List dependencies and their status
helm dependency list ./my-chart/
# NAME        VERSION   REPOSITORY                          STATUS
# postgresql  12.1.0    https://charts.bitnami.com/bitnami  ok
# redis       17.0.0    https://charts.bitnami.com/bitnami  missing
```

### helm test

```bash
# Run test pods defined in templates/tests/
helm test my-release
# NAME: my-release
# LAST DEPLOYED: Wed Oct 4 09:15:00 2024
# ...
# Phase: Succeeded

# Run tests with a timeout
helm test my-release --timeout 5m

# Show test pod logs
helm test my-release --logs
```

## Plugin Management

### helm plugin

```bash
# Install a plugin from a URL
helm plugin install https://github.com/databus23/helm-diff

# Install a specific version of a plugin
helm plugin install https://github.com/databus23/helm-diff --version 3.8.1

# List installed plugins
helm plugin list
# NAME    VERSION   DESCRIPTION
# diff    3.8.1     Preview helm upgrade changes as a diff

# Update a plugin
helm plugin update diff

# Uninstall a plugin
helm plugin uninstall diff
```

## Environment and Diagnostics

### helm env

```bash
# Show all Helm environment variables
helm env
# HELM_BIN="helm"
# HELM_CACHE_HOME="/home/user/.cache/helm"
# HELM_CONFIG_HOME="/home/user/.config/helm"
# HELM_DATA_HOME="/home/user/.local/share/helm"
# HELM_DEBUG="false"
# HELM_KUBECONTEXT=""
# HELM_NAMESPACE="default"
# HELM_PLUGINS="/home/user/.local/share/helm/plugins"
# HELM_REGISTRY_CONFIG="/home/user/.config/helm/registry/config.json"
# HELM_REPOSITORY_CACHE="/home/user/.cache/helm/repository"
# HELM_REPOSITORY_CONFIG="/home/user/.config/helm/repositories.yaml"
```

### helm version

```bash
# Show the Helm client version
helm version
# version.BuildInfo{Version:"v3.13.1", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.21.3"}

# Short version
helm version --short
# v3.13.1+g3547a4b

# Output as JSON
helm version --template '{{ .Version }}'
# v3.13.1
```

## Common Flag Patterns

Several flags work across multiple commands. These are worth memorizing.

| Flag | Commands | Purpose |
|---|---|---|
| `--namespace` / `-n` | Most commands | Target a specific namespace |
| `--kube-context` | Most commands | Use a specific kubeconfig context |
| `--kubeconfig` | Most commands | Path to a kubeconfig file |
| `--debug` | Most commands | Verbose output for troubleshooting |
| `--dry-run` | install, upgrade, uninstall | Preview without executing |
| `--wait` | install, upgrade, rollback | Wait for readiness |
| `--timeout` | install, upgrade, rollback, test | Max wait time (default 5m0s) |
| `--atomic` | install, upgrade | Auto rollback on failure |
| `--output` / `-o` | list, status, history, search | Output format (table, json, yaml) |
| `--values` / `-f` | install, upgrade, template, lint | Path to values file |
| `--set` | install, upgrade, template, lint | Override a single value |
