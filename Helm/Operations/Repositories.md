---
tags:
  - helm
  - helm/operations
topic: Operations
---

# Repositories

## What Is a Chart Repository

A chart repository is an HTTP server (or OCI registry) that hosts packaged Helm charts. It serves an `index.yaml` file that lists all available charts and their versions, along with the `.tgz` chart archives themselves. When you run `helm repo add`, Helm downloads this index and caches it locally so you can search and install charts.

Traditional chart repositories are simple: a web server serving static files. OCI-based repositories use container registries, which bring better authentication, signing, and tooling.

## Managing Repositories

### helm repo add

Adds a chart repository to your local Helm configuration.

```bash
# Add the Bitnami repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Add with a custom name
helm repo add my-charts https://charts.example.com

# Add with basic auth credentials
helm repo add private-repo https://charts.example.com \
  --username admin \
  --password s3cret

# Add with a CA certificate for self-signed TLS
helm repo add corp-repo https://charts.internal.corp \
  --ca-file /path/to/ca.crt

# Add and force update the index
helm repo add bitnami https://charts.bitnami.com/bitnami --force-update
```

### helm repo list

```bash
helm repo list
# NAME        URL
# bitnami     https://charts.bitnami.com/bitnami
# my-charts   https://charts.example.com
```

### helm repo update

Fetches the latest `index.yaml` from all configured repositories. Run this before searching or installing to ensure you see the latest chart versions.

```bash
# Update all repos
helm repo update
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "bitnami" chart repository
# Update Complete. Happy Helming!

# Update a specific repo
helm repo update bitnami
```

### helm repo remove

```bash
helm repo remove my-charts
```

### helm repo index

Generates an `index.yaml` file from chart packages in a directory. Used when hosting your own chart repository.

```bash
# Generate index.yaml for charts in the current directory
helm repo index .

# Generate with a base URL (required for remote access)
helm repo index . --url https://charts.example.com

# Merge with an existing index (preserves old entries)
helm repo index . --url https://charts.example.com --merge index.yaml
```

## Searching for Charts

### helm search repo

Searches across your locally configured repositories (based on the cached `index.yaml`).

```bash
# Search for nginx charts
helm search repo nginx
# NAME                  CHART VERSION  APP VERSION  DESCRIPTION
# bitnami/nginx         15.0.0         1.25.0       NGINX is an open source HTTP web server...
# bitnami/nginx-ingress 10.0.0         3.5.0        NGINX Ingress Controller...

# Search with version constraint
helm search repo nginx --version ">=14.0.0"

# Show all versions (not just latest)
helm search repo nginx --versions

# Search with regex
helm search repo "^bitnami/nginx$"
```

### helm search hub

Searches Artifact Hub, a centralized discovery platform that aggregates charts from many sources.

```bash
# Search Artifact Hub
helm search hub nginx
# URL                                               CHART VERSION  APP VERSION
# https://artifacthub.io/packages/helm/bitnami/nginx  15.0.0       1.25.0

# Output as YAML for scripting
helm search hub nginx --output yaml
```

Artifact Hub (https://artifacthub.io) is not a repository itself -- it is a discovery tool that indexes charts from hundreds of repositories. You still need to `helm repo add` the actual repository before installing.

## Popular Public Repositories

| Repository | URL | Notes |
|---|---|---|
| Bitnami | `https://charts.bitnami.com/bitnami` | Largest maintained chart collection |
| Ingress-NGINX | `https://kubernetes.github.io/ingress-nginx` | Official NGINX Ingress Controller |
| Jetstack | `https://charts.jetstack.io` | cert-manager |
| Prometheus Community | `https://prometheus-community.github.io/helm-charts` | Prometheus, Grafana stack |
| Hashicorp | `https://helm.releases.hashicorp.com` | Vault, Consul |

## helm pull

Downloads a chart archive (`.tgz`) to your local filesystem without installing it. Useful for inspection, vendoring, or offline installs.

```bash
# Pull a chart as a .tgz file
helm pull bitnami/nginx
# Creates nginx-15.0.0.tgz in the current directory

# Pull and extract
helm pull bitnami/nginx --untar

# Pull a specific version
helm pull bitnami/nginx --version 14.0.0

# Pull from an OCI registry
helm pull oci://registry.example.com/charts/nginx --version 1.0.0

# Pull to a specific directory
helm pull bitnami/nginx --destination ./charts/
```

## OCI Registries as Chart Repositories

Helm 3.8+ supports OCI (Open Container Initiative) registries as a first-class chart storage mechanism. OCI registries reuse the same infrastructure as container image registries (Docker Hub, ECR, GCR, ACR, Harbor, GHCR).

### Pushing and Pulling Charts via OCI

```bash
# Login to the registry
helm registry login registry.example.com \
  --username admin \
  --password s3cret

# Package the chart first
helm package ./my-chart
# Creates my-chart-1.0.0.tgz

# Push to an OCI registry
helm push my-chart-1.0.0.tgz oci://registry.example.com/charts

# Pull from an OCI registry
helm pull oci://registry.example.com/charts/my-chart --version 1.0.0

# Install directly from an OCI registry
helm install my-release oci://registry.example.com/charts/my-chart --version 1.0.0

# Show chart info from OCI registry
helm show chart oci://registry.example.com/charts/my-chart --version 1.0.0

# Logout
helm registry logout registry.example.com
```

### OCI vs Traditional HTTP Repositories

| Feature | Traditional HTTP Repo | OCI Registry |
|---|---|---|
| Discovery | `index.yaml` file | Registry API / tags |
| `helm repo add` | Required | Not needed |
| `helm search repo` | Works | Not supported (use registry UI or API) |
| Authentication | Basic auth, TLS certs | Docker-style login (token-based) |
| Signing | Provenance files (`.prov`) | OCI signatures (cosign, notation) |
| Hosting | Any HTTP server | Any OCI-compliant registry |
| Chart reference | `repo-name/chart-name` | `oci://registry/path/chart-name` |
| Immutability | Not enforced | Enforced by most registries |
| `helm repo update` | Required to refresh index | Not needed |

OCI is the recommended approach for new deployments. It eliminates the need for `helm repo add` and `helm repo update`, simplifies authentication, and leverages the mature container registry ecosystem.

## Hosting Your Own Chart Repository

### GitHub Pages

The simplest free option for public charts.

```bash
# In your GitHub repository
mkdir charts
cp my-chart-1.0.0.tgz charts/
helm repo index charts/ --url https://username.github.io/repo-name/charts
# Commit and push; enable GitHub Pages for the repo
```

### ChartMuseum

A dedicated open-source chart repository server with an upload API.

```bash
# Install ChartMuseum via Helm (meta!)
helm install chartmuseum chartmuseum/chartmuseum \
  --set env.open.STORAGE=local \
  --set persistence.enabled=true

# Upload a chart
curl --data-binary "@my-chart-1.0.0.tgz" http://chartmuseum.example.com/api/charts

# Or use the Helm push plugin
helm plugin install https://github.com/chartmuseum/helm-push
helm cm-push my-chart-1.0.0.tgz my-chartmuseum-repo
```

### Cloud Storage

Both AWS S3 and GCS can serve as chart repositories since they are HTTP servers that can host static files.

```bash
# S3: upload chart and index
aws s3 cp my-chart-1.0.0.tgz s3://my-helm-bucket/charts/
helm repo index . --url https://my-helm-bucket.s3.amazonaws.com/charts
aws s3 cp index.yaml s3://my-helm-bucket/charts/

# Add the S3-backed repo
helm repo add my-s3-repo https://my-helm-bucket.s3.amazonaws.com/charts
```

For private S3/GCS repos, use the `helm-s3` or `helm-gcs` plugins which handle authentication natively.

### Harbor

Harbor is an enterprise registry that supports both container images and Helm charts (via OCI).

```bash
# Push a chart to Harbor via OCI
helm registry login harbor.example.com
helm push my-chart-1.0.0.tgz oci://harbor.example.com/my-project/charts
```

## Private Repositories with Authentication

```bash
# Basic auth
helm repo add private https://charts.example.com \
  --username admin \
  --password s3cret

# TLS client certificate
helm repo add private https://charts.example.com \
  --cert-file /path/to/cert.pem \
  --key-file /path/to/key.pem \
  --ca-file /path/to/ca.pem

# OCI registry auth (uses Docker credential helpers)
helm registry login registry.example.com --username admin --password s3cret

# AWS ECR (OCI)
aws ecr get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com
```
