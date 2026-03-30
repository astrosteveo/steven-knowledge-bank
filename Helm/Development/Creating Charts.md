---
tags:
  - helm
  - helm/development
topic: Development
---

# Creating Charts

## helm create

The `helm create` command scaffolds a new chart directory with sensible defaults. It generates a production-ready structure based on an NGINX deployment that you can modify for your application.

```bash
helm create my-chart
# Creating my-chart

# What it generates:
# my-chart/
# ├── .helmignore          # patterns to exclude when packaging
# ├── Chart.yaml           # chart metadata (name, version, dependencies)
# ├── values.yaml          # default configuration values
# ├── charts/              # dependency charts (subcharts)
# └── templates/           # Kubernetes manifest templates
#     ├── NOTES.txt         # post-install/upgrade usage notes
#     ├── _helpers.tpl      # reusable template snippets
#     ├── deployment.yaml
#     ├── hpa.yaml
#     ├── ingress.yaml
#     ├── service.yaml
#     ├── serviceaccount.yaml
#     └── tests/
#         └── test-connection.yaml
```

## What Each File Does

| File | Purpose |
|---|---|
| `Chart.yaml` | Chart metadata: name, version, appVersion, dependencies, maintainers |
| `values.yaml` | Default values that templates reference via `.Values.*` |
| `templates/` | Go-template Kubernetes manifests rendered at install time |
| `templates/_helpers.tpl` | Named template definitions (partials) reused across manifests |
| `templates/NOTES.txt` | Text displayed to the user after install/upgrade (access URLs, next steps) |
| `templates/tests/` | Test Pod definitions run by `helm test` |
| `charts/` | Vendored dependency chart archives (`.tgz`) or unpacked chart directories |
| `.helmignore` | Glob patterns for files to exclude from the chart package |

### Chart.yaml

```yaml
apiVersion: v2                    # v2 = Helm 3 charts
name: my-chart
description: A Helm chart for my application
type: application                 # "application" or "library"
version: 0.1.0                   # chart version (SemVer) -- bump this on chart changes
appVersion: "1.0.0"              # version of the app being deployed (informational only)

maintainers:
  - name: Platform Team
    email: platform@example.com

dependencies:
  - name: postgresql
    version: "~13.0"             # SemVer range
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled  # only include if this value is true

keywords:
  - web
  - api
home: https://github.com/example/my-chart
sources:
  - https://github.com/example/my-chart
```

## Building a Chart from Scratch

This walkthrough builds a chart for a simple web application with a Deployment, Service, ConfigMap, and optional Ingress.

### Step 1: Scaffold and Clean

```bash
helm create webapp
cd webapp

# Remove the auto-generated templates -- we'll write our own
rm templates/deployment.yaml templates/service.yaml templates/ingress.yaml
rm templates/serviceaccount.yaml templates/hpa.yaml
```

### Step 2: Define values.yaml

```yaml
# values.yaml

replicaCount: 2

image:
  repository: myorg/webapp
  tag: ""                         # defaults to .Chart.AppVersion if empty
  pullPolicy: IfNotPresent

imagePullSecrets: []

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: webapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

config:
  logLevel: info
  maxConnections: "100"

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

nodeSelector: {}
tolerations: []
affinity: {}

# Use an existing secret for sensitive config
existingSecret: ""
```

### Step 3: Write _helpers.tpl

```yaml
# templates/_helpers.tpl

{{/*
Chart name, truncated to 63 characters.
*/}}
{{- define "webapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Fully qualified app name. Truncated to 63 chars because some K8s fields are limited.
*/}}
{{- define "webapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "webapp.labels" -}}
helm.sh/chart: {{ include "webapp.chart" . }}
{{ include "webapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "webapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "webapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Chart label value
*/}}
{{- define "webapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Image reference with tag fallback to appVersion
*/}}
{{- define "webapp.image" -}}
{{- $tag := default .Chart.AppVersion .Values.image.tag -}}
{{- printf "%s:%s" .Values.image.repository $tag -}}
{{- end }}
```

### Step 4: Write Templates

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
data:
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
  MAX_CONNECTIONS: {{ .Values.config.maxConnections | quote }}
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: {{ include "webapp.image" . }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "webapp.fullname" . }}
          {{- if .Values.existingSecret }}
            - secretRef:
                name: {{ .Values.existingSecret }}
          {{- end }}
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "webapp.selectorLabels" . | nindent 4 }}
```

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "webapp.fullname" $ }}
                port:
                  name: http
          {{- end }}
    {{- end }}
{{- end }}
```

Note the `checksum/config` annotation on the Deployment -- this forces a Pod restart whenever the ConfigMap content changes, since Kubernetes does not automatically restart Pods when mounted ConfigMaps change.

## helm template

Renders chart templates locally without connecting to a cluster. Essential for debugging template logic and reviewing what will be sent to Kubernetes.

```bash
# Render all templates with default values
helm template my-release ./my-chart

# Render with custom values
helm template my-release ./my-chart -f production-values.yaml

# Render a single template
helm template my-release ./my-chart --show-only templates/deployment.yaml

# Render with --debug to see computed values and additional info
helm template my-release ./my-chart --debug

# Validate against a specific Kubernetes version
helm template my-release ./my-chart --kube-version 1.28.0

# Set values inline
helm template my-release ./my-chart --set ingress.enabled=true
```

`helm template` does not validate against the Kubernetes API -- it only renders Go templates. Use `helm install --dry-run` for server-side validation.

## helm lint

Validates chart structure and catches common errors like malformed YAML, missing required fields, or deprecated API versions.

```bash
helm lint ./my-chart
# ==> Linting ./my-chart
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed

# Lint with specific values
helm lint ./my-chart -f production-values.yaml

# Lint with strict mode (treats warnings as errors)
helm lint ./my-chart --strict

# Lint with --debug for verbose output
helm lint ./my-chart --debug
```

## helm package

Creates a versioned `.tgz` archive of the chart, ready for distribution or upload to a repository.

```bash
# Package the chart
helm package ./my-chart
# Successfully packaged chart and saved it to: /path/to/my-chart-0.1.0.tgz

# Package with a specific version override
helm package ./my-chart --version 1.2.3

# Package with a specific appVersion override
helm package ./my-chart --app-version 2.0.0

# Package to a specific directory
helm package ./my-chart --destination ./dist/

# Update dependencies before packaging
helm package ./my-chart --dependency-update
```

## Chart Signing and Provenance

Helm supports GPG signing for chart integrity verification.

```bash
# Package and sign a chart (requires a GPG key)
helm package ./my-chart --sign --key "Platform Team" --keyring ~/.gnupg/secring.gpg
# Creates:
#   my-chart-0.1.0.tgz       (the chart archive)
#   my-chart-0.1.0.tgz.prov  (the provenance file)

# Verify a chart's signature
helm verify my-chart-0.1.0.tgz --keyring ~/.gnupg/pubring.gpg

# Install with verification
helm install my-release my-chart-0.1.0.tgz --verify --keyring ~/.gnupg/pubring.gpg
```

The provenance file (`.prov`) contains the chart's SHA256 digest and a PGP signature. It proves the chart was created by a trusted party and has not been tampered with.

## .helmignore

Works like `.gitignore` -- patterns listed here are excluded when packaging the chart with `helm package`.

```text
# .helmignore

# Common patterns
.git
.gitignore
.vscode/
.idea/

# CI/CD files
.github/
.gitlab-ci.yml
Makefile
Dockerfile

# Test and development files
*.test.yaml
ci/
tests/

# Documentation (not needed in the package)
docs/
README.md
CHANGELOG.md

# OS files
.DS_Store
Thumbs.db
```

## Making Resources Conditional

Use `if` conditions tied to values to let users enable or disable resources.

```yaml
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "webapp.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- toYaml .Values.autoscaling.metrics | nindent 4 }}
{{- end }}
```

## Supporting Multiple Kubernetes Versions

Use `.Capabilities` to check the cluster's API versions and Kubernetes version.

```yaml
# Choose API version based on cluster capabilities
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}

# Check Kubernetes version
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.GitVersion }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: extensions/v1beta1
{{- end }}
```
