---
tags:
  - kubernetes
  - kubernetes/configuration
topic: Configuration
---

# Secrets

## What is a Secret?

A Secret is a Kubernetes object designed to store **sensitive data** — passwords, API keys, TLS certificates, SSH keys, and tokens. Secrets are structurally similar to ConfigMaps (both store key-value pairs), but Kubernetes treats them differently:

| | ConfigMap | Secret |
|---|---|---|
| **Purpose** | Non-sensitive configuration | Sensitive data |
| **Storage** | Stored as plain text in etcd | Base64-encoded in etcd (encrypted at rest if configured) |
| **Default size limit** | 1 MiB | 1 MiB |
| **Memory behavior** | Written to disk by default | Stored in tmpfs (RAM) when mounted as volumes — never written to node disk |
| **RBAC** | Standard RBAC | Should have tighter RBAC restrictions |
| **kubectl output** | Values shown in plain text | Values shown as base64 by default |

**Critical understanding:** base64 encoding is **not** encryption. Anyone with `kubectl get secret -o yaml` access can decode the values trivially. Encryption at rest in etcd must be configured separately.

## Secret Types

Kubernetes supports several built-in Secret types, each with specific validation rules:

| Type | Purpose | Required Keys |
|---|---|---|
| `Opaque` | Generic, user-defined data (default) | None |
| `kubernetes.io/tls` | TLS certificate and private key | `tls.crt`, `tls.key` |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials | `.dockerconfigjson` |
| `kubernetes.io/service-account-token` | ServiceAccount token (auto-created) | `token`, `ca.crt`, `namespace` |
| `kubernetes.io/basic-auth` | Basic authentication credentials | `username`, `password` |
| `kubernetes.io/ssh-auth` | SSH private key | `ssh-privatekey` |

### Opaque (Default)

General-purpose Secret for any arbitrary data:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
type: Opaque
data:
  username: YWRtaW4=        # base64 of "admin"
  password: cEBzc3cwcmQ=    # base64 of "p@ssw0rd"
```

### TLS Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-cert
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

Or create it from files:

```bash
kubectl create secret tls tls-cert \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

### Docker Registry Secret

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com
```

### Basic Auth Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: t0p-Secret
```

### SSH Auth Secret

```bash
kubectl create secret generic ssh-key \
  --type=kubernetes.io/ssh-auth \
  --from-file=ssh-privatekey=path/to/.ssh/id_rsa
```

## Creating Secrets

### From Literals

```bash
kubectl create secret generic app-credentials \
  --from-literal=username=admin \
  --from-literal=password='p@ssw0rd'
```

### From Files

```bash
# File contents become the Secret value
kubectl create secret generic app-credentials \
  --from-file=ssh-privatekey=./id_rsa \
  --from-file=config=./app-config.json
```

### From a YAML Manifest

Two ways to provide values in YAML:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
type: Opaque
# Option 1: base64-encoded values (use `data`)
data:
  username: YWRtaW4=
  password: cEBzc3cwcmQ=
---
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials-v2
type: Opaque
# Option 2: plain-text values (use `stringData`) — Kubernetes encodes them for you
stringData:
  username: admin
  password: "p@ssw0rd"
```

`stringData` is a write-only convenience field. After creation, `kubectl get secret -o yaml` always shows values under `data` in base64. If both `data` and `stringData` specify the same key, `stringData` takes precedence.

## Base64 Encoding

```bash
# Encode
echo -n 'admin' | base64
# Output: YWRtaW4=

# Decode
echo 'YWRtaW4=' | base64 --decode
# Output: admin
```

**Always use `echo -n`** (no trailing newline) when encoding. A trailing newline changes the encoded value and can cause authentication failures that are painful to debug.

Base64 is a reversible encoding, not a security mechanism. It exists so Secrets can store binary data (certificates, keys) in a text-based format.

## Using Secrets

### As Environment Variables

#### All Keys (envFrom)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:1.0
      envFrom:
        - secretRef:
            name: app-credentials
```

#### Specific Keys (env with valueFrom)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-credentials
              key: password
              optional: false  # Pod won't start if key is missing
```

**Caveat:** environment variables are visible in `/proc/<pid>/environ`, in crash dumps, and in logs if your app prints its environment. For high-sensitivity secrets, prefer volume mounts.

### As Mounted Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: app-credentials
        defaultMode: 0400  # restrict file permissions
```

Each key becomes a file under `/etc/secrets/`. The files are backed by tmpfs (in-memory filesystem), so they never touch the node's disk.

#### Mounting Specific Keys

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: tls-cert
      items:
        - key: tls.crt
          path: server.crt
        - key: tls.key
          path: server.key
          mode: 0400
```

### As imagePullSecrets

Used to authenticate with private container registries:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: registry.example.com/myapp:1.0
```

You can also attach imagePullSecrets to a ServiceAccount so all Pods using that account automatically get registry credentials:

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

## Immutable Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
immutable: true
type: Opaque
stringData:
  username: admin
  password: "p@ssw0rd"
```

Same behavior as immutable ConfigMaps — cannot be modified after creation. Delete and recreate to change values. Reduces API server watch load.

## Security Considerations

### What Kubernetes Does NOT Do by Default

- Secrets are stored **unencrypted** in etcd by default
- Any user with API access to a namespace can read its Secrets
- Any user who can create a Pod in a namespace can mount any Secret in that namespace

### Hardening Secrets

| Layer | Action |
|---|---|
| **etcd encryption** | Enable `EncryptionConfiguration` to encrypt Secret data at rest in etcd |
| **RBAC** | Restrict `get`, `list`, and `watch` on Secrets to only the service accounts and users that need them |
| **Namespace isolation** | Use separate namespaces and RBAC to limit Secret access per team |
| **Audit logging** | Enable audit logging to track who accesses Secrets |
| **Short-lived Secrets** | Rotate credentials regularly; use projected service account tokens with expiry |
| **Avoid env vars for high-sensitivity data** | Volume mounts are slightly more secure since env vars appear in process listings |

### Encryption at Rest

Configure the API server with an `EncryptionConfiguration` file:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}  # fallback for reading unencrypted Secrets
```

## External Secret Management

For production environments, consider managing secrets outside of Kubernetes entirely and syncing them in.

### Sealed Secrets (Bitnami)

Sealed Secrets lets you safely store encrypted secrets in Git. Only the controller running in-cluster can decrypt them:

```
┌─────────────────┐    kubeseal     ┌──────────────────┐
│   Secret YAML    │ ─────────────► │ SealedSecret YAML │
│   (plain text)   │   (encrypts    │ (safe for Git)     │
└─────────────────┘    with public  └────────┬───────────┘
                       key)                   │
                                              │ kubectl apply
                                              ▼
                                   ┌──────────────────────┐
                                   │ Sealed Secrets        │
                                   │ Controller (in-cluster)│
                                   │ decrypts → creates    │
                                   │ regular Secret        │
                                   └──────────────────────┘
```

```bash
# Encrypt a Secret for safe storage in version control
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

### External Secrets Operator (ESO)

ESO syncs secrets from external providers (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, Azure Key Vault) into Kubernetes Secrets:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-credentials  # the K8s Secret that gets created
  data:
    - secretKey: username
      remoteRef:
        key: prod/app/credentials
        property: username
    - secretKey: password
      remoteRef:
        key: prod/app/credentials
        property: password
```

ESO continuously reconciles, so rotating a secret in your external provider automatically updates the Kubernetes Secret.

## Best Practices

1. **Enable encryption at rest** — without it, Secrets are plain text in etcd. This is the single most important hardening step.

2. **Use RBAC to restrict Secret access** — not every developer or service account needs to read Secrets. Follow least privilege.

3. **Prefer volume mounts over environment variables** — env vars leak into process listings, crash dumps, and logs more easily than files.

4. **Use `stringData` in manifests** — avoids manual base64 encoding and the bugs that come with it. Never commit plain-text Secret manifests to Git.

5. **Use an external secret manager for production** — Sealed Secrets or ESO keep secrets out of your Git history and provide rotation, audit trails, and centralized management.

6. **Set `immutable: true` for static credentials** — prevents accidental modification and reduces API server load.

7. **Restrict file permissions on volume mounts** — use `defaultMode: 0400` so only the container's user can read the secret files.

8. **Rotate secrets regularly** — treat credentials as ephemeral. Automate rotation with your external secret manager.
