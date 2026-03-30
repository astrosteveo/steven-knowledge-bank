---
tags:
  - helm
  - helm/development
topic: Development
---

# Best Practices

## Chart Naming Conventions

Chart names should be lowercase, hyphen-separated, and descriptive. The chart name appears in resource names, so keep it concise.

```
# Good
my-webapp
order-service
redis-cluster

# Bad
MyWebApp          # no uppercase
order_service     # no underscores
my-super-awesome-application-server   # too long
```

The chart directory name must match the `name` field in `Chart.yaml`.

## Version Conventions

Helm uses two distinct version fields in `Chart.yaml`:

| Field | Purpose | When to Bump |
|---|---|---|
| `version` | Chart version (SemVer) | Any change to the chart itself: templates, values, dependencies |
| `appVersion` | Application version (informational) | When the default app image/version changes |

```yaml
# Chart.yaml
version: 2.1.0        # chart packaging version -- bump this on every chart change
appVersion: "3.4.2"    # the app version this chart deploys by default
```

- `version` follows strict SemVer: `MAJOR.MINOR.PATCH`
  - **MAJOR**: Breaking changes to values schema or behavior
  - **MINOR**: New features (new values, new templates), backwards compatible
  - **PATCH**: Bug fixes, documentation, minor template fixes
- `appVersion` is a string and does not need to follow SemVer (it matches whatever the application uses)

## Values Best Practices

### Keep the Structure Flat When Possible

Deeply nested values are harder to override with `--set` and harder to document.

```yaml
# Prefer this
replicaCount: 3
serviceType: ClusterIP
servicePort: 80

# Over this (if the nesting doesn't add clarity)
deployment:
  scaling:
    replicas:
      count: 3
```

That said, logical grouping is fine when it improves clarity:

```yaml
# Good: grouped by concern
image:
  repository: myorg/webapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

### Document Every Value

Add comments above each value in `values.yaml`. These comments are the primary documentation for chart users.

```yaml
# -- Number of pod replicas
# @default -- 2
replicaCount: 2

# -- Container image configuration
image:
  # -- Image repository
  repository: myorg/webapp
  # -- Image tag (defaults to Chart.appVersion if empty)
  tag: ""
  # -- Image pull policy
  pullPolicy: IfNotPresent

# -- Existing secret name for database credentials.
# If set, the chart will not create its own Secret.
# The secret must contain a `DATABASE_URL` key.
existingSecret: ""
```

### Use Sensible Defaults

`values.yaml` should produce a working deployment with zero overrides on a basic cluster. Users should only need to customize values for their specific environment.

### Never Put Secrets in values.yaml

Default values are committed to version control. Never include passwords, tokens, or keys.

```yaml
# BAD: secret in values.yaml
database:
  password: "s3cret123"

# GOOD: reference an existing secret
database:
  existingSecret: ""         # user provides the secret name
  secretKey: "password"      # key within the secret
```

## Template Best Practices

### Use Named Templates (Helpers)

Avoid duplicating labels, names, and selectors across templates. Define them once in `_helpers.tpl`.

```yaml
# _helpers.tpl
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ include "myapp.chart" . }}
{{- end }}

# deployment.yaml -- use the helper
metadata:
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
```

### Whitespace Control

Go templates emit whitespace literally, which leads to messy YAML. Use `{{-` (trim left) and `-}}` (trim right) to control whitespace.

```yaml
# Without whitespace control (produces blank lines and bad indentation)
{{ if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
{{ end }}

# With whitespace control (clean output)
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
{{- end }}
```

### Quote Strings

Always quote string values in templates. This prevents YAML parsing issues with values that look like numbers, booleans, or special characters.

```yaml
# BAD: unquoted values can cause YAML parsing issues
annotations:
  description: {{ .Values.description }}       # breaks if value contains ":"

# GOOD: always quote
annotations:
  description: {{ .Values.description | quote }}

# For labels (which should not have quotes in the YAML):
labels:
  version: {{ .Chart.AppVersion | quote }}
```

### Use toYaml for Complex Structures

When a value is a map or list that should be passed through verbatim, use `toYaml` with `nindent`.

```yaml
resources:
  {{- toYaml .Values.resources | nindent 12 }}

nodeSelector:
  {{- toYaml .Values.nodeSelector | nindent 8 }}
```

## Label Conventions

Use the Kubernetes recommended labels. These are recognized by tooling and provide consistency.

| Label | Purpose | Example |
|---|---|---|
| `app.kubernetes.io/name` | Application name | `webapp` |
| `app.kubernetes.io/instance` | Release instance name | `webapp-production` |
| `app.kubernetes.io/version` | Application version | `3.4.2` |
| `app.kubernetes.io/component` | Component within the application | `frontend`, `api`, `worker` |
| `app.kubernetes.io/part-of` | Higher-level application this belongs to | `e-commerce-platform` |
| `app.kubernetes.io/managed-by` | Tool managing the resource | `Helm` |
| `helm.sh/chart` | Chart name and version | `webapp-2.1.0` |

```yaml
# _helpers.tpl standard label definitions
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

Selector labels (`matchLabels`) should be a subset of the full labels. Never include version or chart labels in selectors -- they are immutable on Deployments and would break upgrades.

## Resource Naming

Always use the `fullname` helper for resource names. This ensures uniqueness when multiple releases of the same chart exist in a namespace.

```yaml
metadata:
  name: {{ include "myapp.fullname" . }}
```

The standard `fullname` pattern is `<release-name>-<chart-name>`, truncated to 63 characters (the Kubernetes name limit). If the release name already contains the chart name, it is not duplicated.

## The existingSecret Pattern

The most common pattern for handling secrets in Helm charts. It lets users either create a new secret via the chart or reference one they manage themselves.

```yaml
# values.yaml
secret:
  # -- Create a new Secret with these values (ignored if existingSecret is set)
  create: true
  # -- Use an existing Secret instead of creating one
  existingSecret: ""
  # -- Key in the secret containing the database password
  passwordKey: "db-password"
  # -- Database password (only used if create=true; do NOT commit real values)
  password: ""
```

```yaml
# templates/secret.yaml
{{- if and .Values.secret.create (not .Values.secret.existingSecret) }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
type: Opaque
data:
  {{ .Values.secret.passwordKey }}: {{ .Values.secret.password | b64enc | quote }}
{{- end }}
```

```yaml
# templates/deployment.yaml (referencing the secret)
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ .Values.secret.existingSecret | default (include "myapp.fullname" .) }}
        key: {{ .Values.secret.passwordKey }}
```

## Multi-Environment Support Patterns

Maintain separate values files per environment, with a shared base.

```bash
# Directory structure
my-chart/
├── values.yaml                   # base defaults
├── values-dev.yaml               # development overrides
├── values-staging.yaml           # staging overrides
└── values-production.yaml        # production overrides
```

```yaml
# values-production.yaml (only override what differs)
replicaCount: 5

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi

ingress:
  enabled: true
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com
```

```bash
# Deploy to production
helm upgrade --install my-app ./my-chart \
  --namespace production \
  --values values.yaml \
  --values values-production.yaml \
  --atomic
```

Values files are merged left-to-right, so later files override earlier ones.

## Making Charts Configurable but Not Over-Engineered

Not every field needs to be a value. Expose configuration for things users will actually change.

```yaml
# GOOD: expose what matters
replicaCount: 2
image:
  repository: myorg/app
  tag: ""
resources: {}
nodeSelector: {}

# BAD: exposing every possible Kubernetes field
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      schedulerName: default-scheduler
```

A practical rule: if a field has a sensible default that works for 90% of users, hardcode it. If users will frequently need to change it, make it a value.

For flexibility without bloat, offer an `extraXxx` escape hatch:

```yaml
# values.yaml
extraEnv: []
extraVolumes: []
extraVolumeMounts: []
podAnnotations: {}
```

```yaml
# templates/deployment.yaml
spec:
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          env:
            {{- with .Values.extraEnv }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
```

## When NOT to Use Helm

Helm is not always the right tool.

| Scenario | Better Alternative |
|---|---|
| Single static manifest that rarely changes | Plain `kubectl apply -f` |
| Per-environment patches on existing manifests | Kustomize |
| One-off Jobs or debugging Pods | `kubectl run` or `kubectl create job` |
| GitOps with environment-specific overlays | Kustomize + ArgoCD/Flux |
| Application needs only image tag changes | Kustomize `images` transformer |
| Team has no Go template experience | Kustomize (pure YAML) |

Helm shines when you need **versioned, parameterized packages** distributed to multiple teams or clusters. If you just need "the same manifests with slightly different values per environment," Kustomize is simpler.

## Common Mistakes and Gotchas

| Mistake | Problem | Fix |
|---|---|---|
| Forgetting `--create-namespace` | Install fails if namespace doesn't exist | Always use `--create-namespace` in scripts |
| Using `--reuse-values` without understanding it | Carries forward stale values after chart upgrades that add new defaults | Use `--reset-values` with explicit `-f values.yaml` instead |
| Not quoting `appVersion` in Chart.yaml | `1.0` is parsed as a float by YAML | Always quote: `appVersion: "1.0"` |
| Mutable selectors in Deployments | Including version labels in `matchLabels` breaks upgrades | Keep selectors minimal: name + instance only |
| Missing `nindent` on `toYaml` | YAML indentation breaks, causing install failures | Always use `toYaml . \| nindent N` |
| Hardcoding release name in templates | Multiple releases of the same chart collide | Use `{{ include "chart.fullname" . }}` |
| Not setting `helm.sh/hook-delete-policy` | Hook Jobs accumulate and block future runs | Always set `before-hook-creation` at minimum |
| Using `helm install` in CI/CD | Fails if release already exists | Use `helm upgrade --install` for idempotency |
| Storing secrets in values.yaml | Secrets committed to git | Use `existingSecret` pattern or external secrets operator |
| Not pinning chart versions | `helm upgrade` pulls latest chart, causing unexpected changes | Always specify `--version` in production |
| Missing `resources` in production | Pods get default QoS, risking eviction and noisy-neighbor issues | Always set resource requests and limits |
| Forgetting `--atomic` in CI | Failed deploys leave broken releases that block subsequent attempts | Use `--atomic` to auto-rollback on failure |
