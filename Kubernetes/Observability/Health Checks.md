---
tags:
  - kubernetes
  - kubernetes/observability
topic: Observability
---

# Health Checks

## Why Health Checks Matter

Kubernetes runs containers, but running does not mean healthy. A container's process can be alive while the application inside is deadlocked, out of memory, or unable to serve requests. Without health checks, Kubernetes has no way to detect these states — it only knows whether the container process has exited.

Health checks (called **probes**) give Kubernetes visibility into application health so it can automatically:

- **Restart** containers that are stuck or broken
- **Remove** containers from Service endpoints when they can't handle traffic
- **Delay** traffic until slow-starting containers are fully initialized

Without probes, users hit broken Pods, cascading failures go undetected, and on-call engineers get paged for problems Kubernetes could have self-healed.

## The Three Probe Types

```
  Container Lifecycle with Probes
  ================================

  Container starts
       │
       ▼
  ┌─────────────┐     Failing?     Container killed
  │ startupProbe │────────────────► and restarted
  │ (optional)   │
  └──────┬──────┘
         │ Succeeds once
         ▼
  ┌──────────────┐    Failing?     Container killed
  │ livenessProbe │───────────────► and restarted
  │ (ongoing)     │
  └──────┬───────┘
         │ Runs in parallel
         ▼
  ┌───────────────┐   Failing?     Removed from
  │ readinessProbe │──────────────► Service endpoints
  │ (ongoing)      │               (no traffic)
  └───────────────┘
```

| Probe | Question it answers | On failure | Runs when |
|---|---|---|---|
| **Liveness** | Is the container alive? | Container is killed and restarted | Continuously after startup |
| **Readiness** | Can the container serve traffic? | Pod removed from Service endpoints | Continuously after startup |
| **Startup** | Has the container finished starting? | Container is killed and restarted | Only during startup; disables liveness/readiness until it succeeds |

### Liveness Probe

Detects containers that are running but broken (deadlocks, infinite loops, corrupted state). When a liveness probe fails `failureThreshold` consecutive times, the kubelet kills the container and restarts it according to the Pod's `restartPolicy`.

**Use when:** Your application can get into an irrecoverable state that only a restart can fix.

### Readiness Probe

Detects containers that are temporarily unable to serve traffic. When a readiness probe fails, the Pod's IP is removed from all matching Service endpoints. The container is **not** restarted — it stays running and can recover on its own.

**Use when:** Your application needs to load data, warm caches, or temporarily cannot handle requests (e.g., during a database migration).

### Startup Probe

Protects slow-starting containers. While the startup probe is active, liveness and readiness probes are disabled. Once the startup probe succeeds, it never runs again — the other probes take over.

**Use when:** Your application takes a long time to initialize (loading ML models, building caches, running migrations). Without a startup probe, you'd have to set a dangerously high `initialDelaySeconds` on the liveness probe.

## Probe Mechanisms

Kubernetes supports four ways to check container health:

| Mechanism | How it works | Succeeds when | Best for |
|---|---|---|---|
| `httpGet` | Sends HTTP GET to a path and port | Response status is 200-399 | Web servers, REST APIs |
| `tcpSocket` | Opens a TCP connection to a port | Connection is established | Databases, services without HTTP |
| `exec` | Runs a command inside the container | Command exits with code 0 | Custom checks, file existence |
| `grpc` | Calls a gRPC health check endpoint | Status is `SERVING` | gRPC services (K8s 1.24+) |

### httpGet

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:            # optional custom headers
      - name: X-Custom-Header
        value: probe
```

### tcpSocket

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
```

### exec

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
```

### grpc

```yaml
livenessProbe:
  grpc:
    port: 50051
    service: my.health.v1.Health   # optional, defaults to ""
```

## Probe Configuration Fields

Every probe type accepts the same timing and threshold fields:

| Field | Default | Description |
|---|---|---|
| `initialDelaySeconds` | 0 | Seconds after the container starts before the probe begins |
| `periodSeconds` | 10 | How often (in seconds) the probe is performed |
| `timeoutSeconds` | 1 | Seconds after which the probe times out |
| `successThreshold` | 1 | Consecutive successes needed to mark the probe as passing (must be 1 for liveness/startup) |
| `failureThreshold` | 3 | Consecutive failures before taking action (restart or remove from endpoints) |

The **total time before action** on failure is approximately:

```
initialDelaySeconds + (periodSeconds × failureThreshold)
```

## Complete YAML Examples

### Liveness Probe — HTTP

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http-example
spec:
  containers:
    - name: app
      image: registry.example.com/myapp:1.0
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
        timeoutSeconds: 3
        failureThreshold: 3
```

### Readiness Probe — HTTP

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-http-example
spec:
  containers:
    - name: app
      image: registry.example.com/myapp:1.0
      ports:
        - containerPort: 8080
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 2
        successThreshold: 1
        failureThreshold: 3
```

### Startup Probe — Protecting a Slow Application

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-probe-example
spec:
  containers:
    - name: app
      image: registry.example.com/slow-app:1.0
      ports:
        - containerPort: 8080
      # Startup probe: gives the app up to 5 × 60 = 300 seconds to start
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 5
        failureThreshold: 60
      # Liveness probe: only begins after startup probe succeeds
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
        failureThreshold: 3
```

### All Three Probes — TCP and Exec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-probe-example
spec:
  containers:
    - name: db
      image: postgres:16
      ports:
        - containerPort: 5432
      startupProbe:
        tcpSocket:
          port: 5432
        periodSeconds: 5
        failureThreshold: 30       # up to 150 seconds to start
      livenessProbe:
        exec:
          command:
            - pg_isready
            - -U
            - postgres
        periodSeconds: 15
        timeoutSeconds: 5
        failureThreshold: 3
      readinessProbe:
        exec:
          command:
            - pg_isready
            - -U
            - postgres
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 2
```

### gRPC Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grpc-probe-example
spec:
  containers:
    - name: grpc-server
      image: registry.example.com/grpc-app:1.0
      ports:
        - containerPort: 50051
      livenessProbe:
        grpc:
          port: 50051
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        grpc:
          port: 50051
        periodSeconds: 5
        failureThreshold: 2
```

## Best Practices

### Always set readiness probes

Every container serving traffic should have a readiness probe. Without one, Kubernetes sends traffic to the Pod as soon as the container starts — before the application may be ready. This causes request failures during deployments and scaling events.

### Liveness != readiness

These probes serve different purposes and should usually hit different endpoints:

| Probe | Endpoint should check | Example |
|---|---|---|
| Liveness (`/healthz`) | Is the process fundamentally healthy? Can it do any work? | Return 200 if the event loop is responsive |
| Readiness (`/ready`) | Can it serve user requests right now? | Return 200 if connected to DB, caches are warm, etc. |

If you point both probes at the same endpoint and that endpoint checks dependencies, a database outage will cause every Pod to be killed and restarted — making things worse, not better.

### Use startup probes for slow applications

Instead of setting a large `initialDelaySeconds` on the liveness probe (which delays failure detection after every restart), use a startup probe with a generous `failureThreshold`:

```yaml
# Bad: delays liveness detection by 120s after every restart
livenessProbe:
  initialDelaySeconds: 120
  periodSeconds: 10

# Good: startup probe handles the slow start, liveness checks begin immediately after
startupProbe:
  periodSeconds: 5
  failureThreshold: 24         # 120s budget for startup only
livenessProbe:
  periodSeconds: 10
  failureThreshold: 3
```

### Don't check dependencies in liveness probes

A liveness probe should only check the container's own health. If you check whether the database is reachable in a liveness probe, a database outage will cause Kubernetes to restart all your application Pods simultaneously — a cascading failure.

```yaml
# Bad: liveness probe checks external dependency
livenessProbe:
  httpGet:
    path: /healthz    # this endpoint pings the database
    port: 8080

# Good: liveness checks only the process, readiness checks dependencies
livenessProbe:
  httpGet:
    path: /healthz    # returns 200 if event loop is running
    port: 8080
readinessProbe:
  httpGet:
    path: /ready      # returns 200 if DB connection is healthy
    port: 8080
```

### Keep probes lightweight

Probes run frequently. Avoid expensive operations (heavy database queries, file system scans) in probe handlers. A probe that takes longer than `timeoutSeconds` counts as a failure.

### Set appropriate thresholds

- `failureThreshold: 1` is too aggressive for liveness — a single slow response kills the container
- Very long `periodSeconds` delays failure detection
- `timeoutSeconds` should be shorter than `periodSeconds`

## Common Mistakes and Debugging

### Mistake: CrashLoopBackOff from overly aggressive liveness probe

**Symptom:** Pod repeatedly restarts, status shows `CrashLoopBackOff`.

**Cause:** The liveness probe fires before the application is ready, killing it before it can finish starting.

**Fix:** Add a `startupProbe` or increase `initialDelaySeconds`.

```bash
# Check the events to see if the probe is triggering restarts
kubectl describe pod <pod-name> | grep -A 5 "Events"
```

### Mistake: Readiness probe never passes

**Symptom:** Pod is Running but shows `0/1 READY`. Service sends no traffic to it.

**Cause:** The readiness endpoint is wrong, the port is wrong, or a dependency it checks is unavailable.

```bash
# Exec into the pod and test the endpoint manually
kubectl exec -it <pod-name> -- curl -v localhost:8080/ready

# Check probe configuration
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].readinessProbe}'
```

### Mistake: Same endpoint for liveness and readiness

**Symptom:** All Pods restart when an external dependency goes down.

**Fix:** Separate the endpoints — liveness checks internal health, readiness checks ability to serve.

### Useful debugging commands

```bash
# See probe configuration and recent events
kubectl describe pod <pod-name>

# Watch pod status in real time
kubectl get pod <pod-name> -w

# Check container restart count and reason
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].restartCount}'
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].lastState}'

# View kubelet logs for probe failures (run on the node)
journalctl -u kubelet | grep <pod-name>
```
