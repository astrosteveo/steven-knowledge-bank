---
tags:
  - helm
  - helm/getting-started
topic: Getting Started
---

# Installation

## Installing Helm

Helm is a single static binary with no runtime dependencies. Choose the method that fits your platform.

### Package Managers

```bash
# macOS — Homebrew
brew install helm

# Linux — Snap
sudo snap install helm --classic

# Linux — APT (Debian/Ubuntu)
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

# Linux — DNF (Fedora/RHEL)
sudo dnf install helm

# Windows — Chocolatey
choco install kubernetes-helm

# Windows — Scoop
scoop install helm

# Windows — Winget
winget install Helm.Helm
```

### Official Install Script

The install script detects your OS and architecture, downloads the latest stable release, and places it in `/usr/local/bin`:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Manual Binary Download

For environments without internet access or when you need a specific version:

```bash
# Download a specific version
HELM_VERSION=v3.17.0
curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz" -o helm.tar.gz

# Verify the checksum
curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz.sha256sum" -o helm.sha256
sha256sum -c helm.sha256

# Extract and install
tar -zxvf helm.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
rm -rf linux-amd64 helm.tar.gz helm.sha256
```

## Verifying Installation

```bash
# Check the installed version
helm version
# version.BuildInfo{Version:"v3.17.0", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.23.4"}

# Short version output
helm version --short
# v3.17.0+g301108e
```

## Shell Completion

Helm supports tab completion for commands, flags, release names, and chart names. Set it up once and save yourself a lot of typing.

```bash
# Bash — add to ~/.bashrc
echo 'source <(helm completion bash)' >> ~/.bashrc
source ~/.bashrc

# Zsh — add to ~/.zshrc
echo 'source <(helm completion zsh)' >> ~/.zshrc
source ~/.zshrc

# Fish — add to fish config
helm completion fish | source
# To make it persist:
helm completion fish > ~/.config/fish/completions/helm.fish

# PowerShell
helm completion powershell | Out-String | Invoke-Expression
# To make it persist, add the above line to your PowerShell profile
```

If zsh completion is not working, ensure `compinit` is loaded in your `.zshrc` before the completion source line:

```bash
autoload -Uz compinit && compinit
source <(helm completion zsh)
```

## Helm Environment Variables

Helm respects several environment variables that control its behavior and file locations. These are especially useful in CI/CD pipelines and air-gapped environments.

| Variable             | Default                           | Purpose                                                                                             |
| -------------------- | --------------------------------- | --------------------------------------------------------------------------------------------------- |
| `HELM_CACHE_HOME`    | `~/.cache/helm`                   | Cached chart archives and repository indexes                                                        |
| `HELM_CONFIG_HOME`   | `~/.config/helm`                  | Helm configuration files (repository list, registry auth)                                           |
| `HELM_DATA_HOME`     | `~/.local/share/helm`             | Helm data (plugins, starters)                                                                       |
| `HELM_DRIVER`        | `secret`                          | Storage backend for release state (`secret`, `configmap`, `sql`, `memory`)                          |
| `HELM_NAMESPACE`     | (from kubeconfig context)         | Default namespace for Helm operations — equivalent to passing `--namespace` on every command          |
| `HELM_MAX_HISTORY`   | `10`                              | Maximum number of release revisions to retain                                                        |
| `HELM_NO_PLUGINS`    | (unset)                           | Disable plugins when set                                                                            |
| `HELM_KUBEAPISERVER` | (unset)                           | Override the Kubernetes API server endpoint                                                          |
| `HELM_KUBETOKEN`     | (unset)                           | Bearer token for authenticating to the API server                                                    |
| `HELM_KUBEASUSER`    | (unset)                           | Impersonate a user for Helm operations                                                               |
| `HELM_KUBEASGROUPS`  | (unset)                           | Impersonate group memberships (comma-separated)                                                      |
| `KUBECONFIG`         | `~/.kube/config`                  | Path to kubeconfig file(s) — not Helm-specific but critical for Helm to find the cluster             |
| `HELM_DEBUG`         | (unset)                           | Enable verbose debug output when set to `1` or `true`                                                |

```bash
# Example: CI/CD pipeline configuration
export HELM_CACHE_HOME=/tmp/helm/cache
export HELM_CONFIG_HOME=/tmp/helm/config
export HELM_DATA_HOME=/tmp/helm/data
export HELM_NAMESPACE=production
export HELM_MAX_HISTORY=5
export KUBECONFIG=/etc/kubernetes/ci-kubeconfig.yaml
```

To see where Helm is reading its configuration from at any time:

```bash
helm env
# HELM_BIN="helm"
# HELM_CACHE_HOME="/home/user/.cache/helm"
# HELM_CONFIG_HOME="/home/user/.config/helm"
# HELM_DATA_HOME="/home/user/.local/share/helm"
# HELM_DEBUG="false"
# HELM_KUBEAPISERVER=""
# HELM_KUBEASGROUPS=""
# HELM_KUBEASUSER=""
# HELM_KUBECONTEXT=""
# HELM_KUBETOKEN=""
# HELM_MAX_HISTORY="10"
# HELM_NAMESPACE="default"
# HELM_PLUGINS="/home/user/.local/share/helm/plugins"
# HELM_REGISTRY_CONFIG="/home/user/.config/helm/registry/config.json"
# HELM_REPOSITORY_CACHE="/home/user/.cache/helm/repository"
# HELM_REPOSITORY_CONFIG="/home/user/.config/helm/repositories.yaml"
```

## Adding Your First Repository

Helm does not ship with any repositories configured. You need to add at least one to start discovering and installing charts.

```bash
# Add the Bitnami repository (one of the most popular community chart collections)
helm repo add bitnami https://charts.bitnami.com/bitnami
# "bitnami" has been added to your repositories

# Add other common repositories
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io        # cert-manager
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update the local repository index cache
helm repo update
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "bitnami" chart repository
# Update Complete. ⎈Happy Helming!⎈

# List configured repositories
helm repo list
# NAME                   URL
# bitnami                https://charts.bitnami.com/bitnami
# ingress-nginx          https://kubernetes.github.io/ingress-nginx
# jetstack               https://charts.jetstack.io
# prometheus-community   https://prometheus-community.github.io/helm-charts

# Search for charts (local index)
helm search repo nginx
# NAME                                  CHART VERSION   APP VERSION   DESCRIPTION
# bitnami/nginx                         18.3.1          1.27.3        NGINX is a high-performance HTTP server...
# bitnami/nginx-ingress-controller      11.6.3          1.12.0        NGINX Ingress Controller is an Ingress...
# ingress-nginx/ingress-nginx           4.12.0          1.12.0        Ingress controller for Kubernetes...

# Search Artifact Hub (the public chart discovery platform)
helm search hub wordpress
# URL                                                    CHART VERSION   APP VERSION   DESCRIPTION
# https://artifacthub.io/packages/helm/bitnami/word...   24.1.2          6.7.2         WordPress is the...
```

### OCI-Based Registries

Helm 3 also supports OCI (Open Container Initiative) registries for chart storage. No `helm repo add` is needed — you pull directly from the registry:

```bash
# Pull a chart from an OCI registry
helm pull oci://registry-1.docker.io/bitnamicharts/nginx --version 18.3.1

# Install directly from an OCI registry
helm install my-nginx oci://registry-1.docker.io/bitnamicharts/nginx --version 18.3.1

# Log in to a private OCI registry
helm registry login registry.example.com
```

## Verifying Connectivity to a Cluster

Helm uses your kubeconfig to communicate with Kubernetes. Before installing charts, verify you can reach the cluster:

```bash
# Confirm kubectl can reach the cluster
kubectl cluster-info
# Kubernetes control plane is running at https://10.0.0.1:6443
# CoreDNS is running at https://10.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# Check which context Helm will use
kubectl config current-context
# my-cluster

# List available contexts
kubectl config get-contexts
# CURRENT   NAME          CLUSTER       AUTHINFO       NAMESPACE
# *         my-cluster    my-cluster    my-user        default

# Verify Helm can list releases (proves Helm + cluster connectivity)
helm list --all-namespaces
# NAME   NAMESPACE   REVISION   UPDATED   STATUS   CHART   APP VERSION

# If you need to target a specific context
helm list --kube-context my-other-cluster --all-namespaces
```

If `helm list` returns without error (even if the list is empty), Helm is properly configured and can communicate with the cluster. If you see authentication or connection errors, check your kubeconfig, context, and cluster credentials.
