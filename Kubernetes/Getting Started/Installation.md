---
tags:
  - kubernetes
  - kubernetes/getting-started
topic: Getting Started
---

# Installation

## Local Development Options

| Tool | What it is | Best for |
|---|---|---|
| **minikube** | Runs a single-node cluster in a VM or container on your machine | Learning, testing, local development with add-on support |
| **kind** | "Kubernetes IN Docker" — runs cluster nodes as Docker containers | CI pipelines, testing multi-node clusters locally |
| **k3s** | Lightweight Kubernetes distribution by Rancher | Edge, IoT, resource-constrained environments, quick local clusters |
| **Docker Desktop** | Includes an optional single-node K8s cluster | macOS/Windows developers who already use Docker Desktop |

## minikube

```bash
# Install (Linux amd64)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

# Start a cluster
minikube start

# Start with a specific driver and resources
minikube start --driver=docker --cpus=4 --memory=8192

# Enable common add-ons
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard

# Open the Kubernetes dashboard
minikube dashboard

# Stop and delete
minikube stop
minikube delete
```

## kind

```bash
# Install (Linux amd64)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create a cluster
kind create cluster

# Create a named cluster with a config file
kind create cluster --name dev --config kind-config.yaml

# List clusters
kind get clusters

# Delete a cluster
kind delete cluster --name dev
```

A multi-node kind config:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

## k3s

```bash
# Install (single command)
curl -sfL https://get.k3s.io | sh -

# Check the service
sudo systemctl status k3s

# k3s bundles its own kubectl
sudo k3s kubectl get nodes

# Copy the kubeconfig to use with standard kubectl
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Uninstall
/usr/local/bin/k3s-uninstall.sh
```

## Docker Desktop

Docker Desktop includes an optional Kubernetes cluster. To enable it:

1. Open Docker Desktop Settings
2. Navigate to **Kubernetes**
3. Check **Enable Kubernetes**
4. Click **Apply & Restart**

Docker Desktop automatically configures `kubectl` and sets the context to `docker-desktop`.

## Cloud Managed Services

Managed Kubernetes offloads control plane operations (upgrades, etcd backups, high availability) to the cloud provider. You only manage worker nodes and your workloads.

| Service | Provider | Key characteristics |
|---|---|---|
| **EKS** (Elastic Kubernetes Service) | AWS | Deep AWS integration (IAM, VPC, ALB), managed node groups or Fargate for serverless Pods |
| **GKE** (Google Kubernetes Engine) | GCP | Autopilot mode for fully managed nodes, tight integration with Google Cloud services |
| **AKS** (Azure Kubernetes Service) | Azure | Free control plane, Azure AD integration, Azure CNI networking |

All three provide a managed control plane and let you interact via standard `kubectl` after configuring authentication.

## kubeadm (Production Clusters)

`kubeadm` is the official tool for bootstrapping production-grade Kubernetes clusters on your own infrastructure. It handles the control plane setup but leaves networking, load balancing, and storage to you.

```bash
# Install kubeadm, kubelet, and kubectl (Debian/Ubuntu)
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Initialize the control plane (on the control plane node)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Set up kubectl access
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin/conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install a CNI plugin (e.g., Flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Join worker nodes (command printed by kubeadm init)
# sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

A full production setup also requires: a container runtime (containerd or CRI-O), a CNI plugin, a load balancer for the API server, and an etcd backup strategy.

## Verifying Installation

Regardless of which method you used:

```bash
# Check kubectl can connect to the cluster
kubectl cluster-info

# List nodes and their status
kubectl get nodes

# Verify system Pods are running
kubectl get pods -n kube-system

# Run a quick smoke test
kubectl run test --image=nginx --restart=Never
kubectl get pod test
kubectl delete pod test
```

## kubeconfig Basics

`kubectl` reads its configuration from `~/.kube/config` (or the file pointed to by the `KUBECONFIG` environment variable). The kubeconfig file defines three things:

| Concept | What it is |
|---|---|
| **Cluster** | The API server address and CA certificate for a Kubernetes cluster |
| **User** | Credentials (certificate, token, or auth plugin) to authenticate with a cluster |
| **Context** | A named combination of cluster + user + default namespace |

```bash
# View the current kubeconfig
kubectl config view

# List all contexts
kubectl config get-contexts

# See the current context
kubectl config current-context

# Switch context
kubectl config use-context my-cluster

# Set a default namespace for the current context
kubectl config set-context --current --namespace=dev

# Use a specific kubeconfig file
KUBECONFIG=~/other-config.yaml kubectl get nodes

# Merge multiple kubeconfig files
export KUBECONFIG=~/.kube/config:~/.kube/staging-config
kubectl config view --merge --flatten > ~/.kube/merged-config
```
