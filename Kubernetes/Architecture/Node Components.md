---
tags:
  - kubernetes
  - kubernetes/architecture
topic: Architecture
---

# Node Components

Every worker node in a Kubernetes cluster runs three essential components: the **kubelet**, **kube-proxy**, and a **container runtime**. Together they are responsible for running Pods, managing networking, and reporting status back to the control plane.

```
  +-----------------------------------------------------------+
  |                      WORKER NODE                          |
  |                                                           |
  |  +--------+    +----------+    +--------------------+     |
  |  | kubelet |    |kube-proxy|    | container runtime  |     |
  |  |         |    |          |    | (containerd/CRI-O) |     |
  |  +----+---+    +-----+----+    +---------+----------+     |
  |       |              |                   |                |
  |       |   manages    |  configures       |  runs          |
  |       |   pod        |  network          |  containers    |
  |       |   lifecycle  |  rules            |                |
  |       v              v                   v                |
  |  +---------+   +----------+   +-----+  +-----+  +-----+  |
  |  | Pod     |   | iptables |   | ctr |  | ctr |  | ctr |  |
  |  | Specs   |   | or IPVS  |   +-----+  +-----+  +-----+  |
  |  +---------+   +----------+                               |
  +-----------------------------------------------------------+
```

## kubelet

The kubelet is the **primary node agent**. It runs on every node and is responsible for making sure containers described in PodSpecs are running and healthy.

### What It Does

1. **Registers the node** with the apiserver on startup.
2. **Watches the apiserver** for PodSpecs assigned to its node.
3. **Instructs the container runtime** (via CRI) to pull images, create containers, and manage their lifecycle.
4. **Monitors container health** using liveness, readiness, and startup probes.
5. **Reports status** (node conditions, Pod status, resource usage) back to the apiserver.
6. **Manages volumes** -- mounts ConfigMaps, Secrets, PVCs, and other volume types into Pods.
7. **Runs static Pods** defined in a local manifest directory (used for control plane bootstrapping with kubeadm).

### Pod Lifecycle Management

When the kubelet receives a new PodSpec:

```
  PodSpec received from apiserver
       |
       v
  1. Admit the Pod (check resource limits, node affinity, etc.)
       |
       v
  2. Create the Pod sandbox (network namespace via CNI)
       |
       v
  3. Pull container images (if not cached)
       |
       v
  4. Create and start init containers (sequentially)
       |
       v
  5. Create and start app containers (in parallel)
       |
       v
  6. Run probes (startup -> liveness + readiness)
       |
       v
  7. Report Pod status to apiserver
       |
       v
  8. Continue monitoring (restart on failure per restartPolicy)
```

### Health Checking (Probes)

The kubelet performs three types of health checks:

| Probe | Purpose | When It Runs | On Failure |
|---|---|---|---|
| **Startup** | Has the container finished starting? | Before liveness/readiness; stops once it succeeds | Container is killed and restarted (subject to restartPolicy) |
| **Liveness** | Is the container still working? | Periodically after startup probe succeeds | Container is killed and restarted |
| **Readiness** | Can the container serve traffic? | Periodically after startup probe succeeds | Pod removed from Service endpoints (no traffic routed to it) |

Probe mechanisms:

| Mechanism | Description | Example |
|---|---|---|
| `httpGet` | HTTP GET request; 200-399 is healthy | `httpGet: {path: /healthz, port: 8080}` |
| `tcpSocket` | TCP connection attempt; success if port is open | `tcpSocket: {port: 3306}` |
| `exec` | Runs a command inside the container; exit code 0 is healthy | `exec: {command: [cat, /tmp/healthy]}` |
| `grpc` | gRPC health check (requires gRPC health protocol) | `grpc: {port: 50051}` |

### Probe Configuration

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15    # wait before first check
  periodSeconds: 10          # check every N seconds
  timeoutSeconds: 3          # timeout per probe attempt
  failureThreshold: 3        # failures before action taken
  successThreshold: 1        # successes to be considered healthy again
```

### Static Pods

The kubelet can run Pods defined in local manifest files (default path: `/etc/kubernetes/manifests/`). These are called **static Pods**. The kubelet manages them directly, without the apiserver scheduling them. This mechanism is how kubeadm bootstraps the control plane -- the apiserver, scheduler, and controller-manager are themselves static Pods.

The kubelet creates a **mirror Pod** in the apiserver so that `kubectl get pods` shows static Pods, but the apiserver cannot control them.

---

## kube-proxy

kube-proxy runs on every node and implements **Service networking**. It watches Service and EndpointSlice objects and configures the node's network rules so that traffic to a Service's ClusterIP is forwarded to the right set of backend Pods.

### How It Works

```
  Client Pod  ──>  ClusterIP:port  ──>  kube-proxy rules  ──>  Backend Pod IP:port
                   (virtual IP)         (iptables/IPVS)        (actual Pod)
```

kube-proxy does not proxy traffic itself in the default mode. It configures kernel-level rules that handle the forwarding.

### Proxy Modes

| Mode | Mechanism | Pros | Cons |
|---|---|---|---|
| **iptables** (default) | Creates iptables NAT rules for each Service/endpoint pair | Simple, reliable, well-tested | Performance degrades with thousands of Services (linear rule evaluation) |
| **IPVS** | Uses Linux IPVS (IP Virtual Server) kernel module | O(1) connection routing, supports multiple load-balancing algorithms | Requires IPVS kernel modules; slightly more complex setup |
| **nftables** | Uses nftables (newer Linux packet filtering framework) | Better performance than iptables, modern kernel support | Newer, less battle-tested in production |

### iptables Mode Details

For each Service, kube-proxy creates:
- A DNAT rule to rewrite the destination IP from the ClusterIP to a Pod IP.
- Probability-based rules for load balancing across endpoints (random selection with equal weights).

```bash
# View the iptables rules kube-proxy creates
iptables -t nat -L KUBE-SERVICES -n
```

### IPVS Mode Details

IPVS supports multiple load-balancing algorithms:

| Algorithm | Description |
|---|---|
| `rr` (round-robin) | Cycles through backends in order |
| `lc` (least connections) | Sends to the backend with fewest connections |
| `dh` (destination hashing) | Routes based on destination IP hash |
| `sh` (source hashing) | Routes based on source IP hash (sticky sessions) |
| `sed` (shortest expected delay) | Estimates delay based on active connections and weight |

```bash
# Enable IPVS mode in kube-proxy config
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"
```

### NodePort and LoadBalancer

kube-proxy also handles:
- **NodePort** Services: Opens a port (30000-32767) on every node and forwards traffic to the Service.
- **LoadBalancer** Services: Works with the cloud-controller-manager to route traffic from an external load balancer through NodePorts to Pods.

---

## Container Runtime

The container runtime is the software responsible for pulling images and running containers. Kubernetes interacts with it through the **Container Runtime Interface (CRI)**.

### The CRI Interface

CRI is a gRPC-based API that the kubelet uses to communicate with any compatible container runtime. This abstraction means Kubernetes is not tied to any specific runtime.

```
  kubelet  ──── CRI (gRPC) ────>  Container Runtime
                                     |
                                     ├── containerd
                                     ├── CRI-O
                                     └── (any CRI-compliant runtime)
```

CRI defines two services:
- **RuntimeService** -- Container and Pod sandbox lifecycle (create, start, stop, remove, exec, attach).
- **ImageService** -- Pull, list, remove images.

### Runtime Comparison

| Runtime | Description | Used By |
|---|---|---|
| **containerd** | Industry standard, extracted from Docker. High performance, widely adopted. | EKS, GKE, AKS, most distributions |
| **CRI-O** | Purpose-built for Kubernetes. Minimal, OCI-compliant. | OpenShift (Red Hat), Fedora CoreOS |
| **Docker Engine** | Removed as a direct runtime in Kubernetes 1.24. Images built with Docker still work (OCI-compliant). | Legacy setups only |

Docker was removed because Kubernetes only needs CRI, but Docker used its own interface (dockershim). The kubelet had a shim layer that translated CRI calls to Docker API calls, adding unnecessary complexity. containerd (which Docker itself uses internally) supports CRI natively.

### OCI Standards

All modern runtimes follow OCI (Open Container Initiative) standards:
- **OCI Image Spec** -- How container images are formatted.
- **OCI Runtime Spec** -- How containers are created and run (implemented by low-level runtimes like `runc`).

```
  kubelet
    |
    v (CRI)
  containerd / CRI-O       <-- high-level runtime
    |
    v (OCI Runtime Spec)
  runc / crun / kata        <-- low-level runtime
    |
    v
  Linux namespaces, cgroups, seccomp, etc.
```

---

## Node Registration

When a node starts, it must register with the cluster:

1. The kubelet starts and reads its configuration (kubeconfig file with apiserver address and TLS credentials).
2. It sends a **Node** object creation request to the apiserver, including:
   - Node name (hostname or `--hostname-override`)
   - Labels (e.g., `kubernetes.io/os=linux`, `node.kubernetes.io/instance-type=m5.large`)
   - Capacity (total CPU, memory, pods, ephemeral storage)
   - Allocatable (capacity minus resources reserved for system daemons)
3. The apiserver stores the Node object in etcd.
4. The scheduler can now place Pods on this node.

### Self-Registration vs Manual

| Method | How | Use Case |
|---|---|---|
| **Self-registration** (default) | kubelet registers itself on startup with `--register-node=true` | Standard in almost all setups |
| **Manual** | Admin creates the Node object via `kubectl` | Rare; useful for air-gapped or specialized environments |

### Bootstrap TLS

For new nodes joining the cluster, Kubernetes supports **TLS bootstrapping**:

1. Node starts with a bootstrap token (short-lived, limited permissions).
2. kubelet sends a CertificateSigningRequest (CSR) to the apiserver.
3. A controller (or admin) approves the CSR.
4. kubelet receives its signed certificate and uses it for all future communication.

---

## Node Conditions and Status

The kubelet continuously reports node conditions to the apiserver.

### Node Conditions

| Condition | Meaning when `True` |
|---|---|
| `Ready` | Node is healthy and can accept Pods |
| `MemoryPressure` | Node is running low on memory |
| `DiskPressure` | Node is running low on disk space |
| `PIDPressure` | Too many processes running on the node |
| `NetworkUnavailable` | Node's network is not correctly configured |

```bash
# Check node conditions
kubectl describe node <node-name> | grep -A 20 "Conditions:"

# Or as JSON
kubectl get node <node-name> -o jsonpath='{.status.conditions[*].type}'
```

### What Happens When a Node Is Not Ready

1. The **node controller** in the controller-manager detects the `Ready` condition is `False` or `Unknown`.
2. After `--pod-eviction-timeout` (default: 5 minutes), it starts evicting Pods from the node.
3. Evicted Pods owned by a ReplicaSet or Deployment are rescheduled on healthy nodes.
4. Standalone Pods (not managed by a controller) are lost.

### Node Status Fields

```yaml
status:
  capacity:          # total resources on the node
    cpu: "4"
    memory: "16Gi"
    pods: "110"
  allocatable:       # resources available for Pods (capacity - system reserved)
    cpu: "3800m"
    memory: "15Gi"
    pods: "110"
  addresses:
    - type: InternalIP
      address: 10.0.1.5
    - type: Hostname
      address: worker-1
  nodeInfo:
    containerRuntimeVersion: containerd://1.7.2
    kubeletVersion: v1.29.0
    osImage: Ubuntu 22.04 LTS
    architecture: amd64
```

---

## Node Heartbeats and Lease Objects

The kubelet sends periodic **heartbeats** to inform the control plane that the node is still alive. Kubernetes uses two mechanisms for this:

### 1. NodeStatus Updates

The kubelet periodically updates the `.status` field of its Node object. By default this happens every 10 seconds (`--node-status-update-frequency`), but only if conditions have changed or a threshold time has passed.

### 2. Lease Objects (Lightweight Heartbeats)

Since Kubernetes 1.17, each node gets a **Lease** object in the `kube-node-lease` namespace. The kubelet renews this lease every 10 seconds (default). This is much lighter than full NodeStatus updates because the Lease object is small.

```bash
# View node leases
kubectl get lease -n kube-node-lease

# Example output
NAME       HOLDER     AGE
worker-1   worker-1   5d
worker-2   worker-2   5d
worker-3   worker-3   3d
```

### Why Leases Exist

Before Leases, every heartbeat was a full NodeStatus update to etcd. For large clusters (thousands of nodes), this created significant load on etcd and the apiserver. Lease objects are tiny (~300 bytes) compared to a full NodeStatus (~2-5 KB), reducing etcd write volume substantially.

### Failure Detection Timeline

```
  kubelet sends heartbeat
       |
       | (10s default interval)
       v
  Lease renewal
       |
  ...  | (node goes down)
       |
  Lease expires (40s default: node-monitor-grace-period)
       |
       v
  Node condition set to "Unknown"
       |
       | (5m default: pod-eviction-timeout)
       v
  Pod eviction begins -- Pods rescheduled to healthy nodes
```

| Parameter | Default | Description |
|---|---|---|
| `--node-status-update-frequency` | 10s | How often kubelet sends status updates |
| `--node-monitor-period` | 5s | How often the node controller checks node status |
| `--node-monitor-grace-period` | 40s | How long to wait before marking a node `Unknown` |
| `--pod-eviction-timeout` | 5m | How long to wait before evicting Pods from an `Unknown` node |
