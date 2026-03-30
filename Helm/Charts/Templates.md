---
tags:
  - helm
  - helm/charts
topic: Charts
---

# Templates

## Go Template Syntax

Helm templates use Go's `text/template` language, extended with the Sprig function library and a few Helm-specific additions. Template directives are enclosed in `{{ }}` delimiters. Everything outside the delimiters is emitted as-is into the rendered manifest.

```yaml
# A simple template — values are injected at render time
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config   # Outputs the release name
  labels:
    app: {{ .Values.appName }}        # Outputs a value from values.yaml
data:
  environment: {{ .Values.env | quote }}  # Pipes the value through the `quote` function
```

## Built-in Objects

Every template has access to several top-level objects. These are the data sources you use to make templates dynamic.

### .Values

The merged result of all values sources (chart defaults, parent chart overrides, `-f` files, `--set` flags). This is the primary configuration mechanism.

```yaml
# values.yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.27"

# In a template:
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

### .Release

Metadata about the current release — set by Helm at install/upgrade time, not by the user.

| Field                | Value                                                        |
| -------------------- | ------------------------------------------------------------ |
| `.Release.Name`      | The release name (`helm install my-app ...` → `my-app`)     |
| `.Release.Namespace` | The namespace the release is installed into                   |
| `.Release.IsInstall` | `true` if this is a fresh install                            |
| `.Release.IsUpgrade` | `true` if this is an upgrade                                 |
| `.Release.Revision`  | The revision number (starts at 1, incremented on each upgrade) |
| `.Release.Service`   | Always `"Helm"`                                              |

```yaml
metadata:
  name: {{ .Release.Name }}-deployment
  namespace: {{ .Release.Namespace }}
  annotations:
    helm.sh/revision: "{{ .Release.Revision }}"
```

### .Chart

Fields from `Chart.yaml`, accessed with capitalized names matching the YAML keys.

```yaml
labels:
  chart: {{ .Chart.Name }}-{{ .Chart.Version }}
  app-version: {{ .Chart.AppVersion }}
```

### .Capabilities

Information about the Kubernetes cluster Helm is deploying to. Useful for writing charts that adapt to different cluster versions.

```yaml
# Check the Kubernetes version
{{ .Capabilities.KubeVersion.Major }}.{{ .Capabilities.KubeVersion.Minor }}

# Check if a specific API is available
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
```

### .Template

Information about the current template being rendered.

```yaml
# .Template.Name — full path, e.g., "my-chart/templates/deployment.yaml"
# .Template.BasePath — directory, e.g., "my-chart/templates"
```

### .Files

Access to non-template files in the chart (covered in detail below under "Accessing Files").

## Template Actions

### if / else — Conditionals

Conditionals control whether a block of YAML is rendered. Helm considers the following values as "false": `false`, `0`, `nil`, empty string `""`, empty collection (`[]`, `{}`).

```yaml
# Conditionally create an Ingress resource
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
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
                name: {{ $.Release.Name }}-svc
                port:
                  number: {{ .port | default 80 }}
          {{- end }}
    {{- end }}
{{- end }}
```

Note the use of `$` inside a `range` block. Inside `range`, the dot (`.`) is rebound to the current iteration item. Use `$` to access the top-level scope (equivalent to the root `.` outside the loop).

### range — Iteration

`range` iterates over lists and maps:

```yaml
# Iterate over a list of ports
{{- range .Values.service.ports }}
- port: {{ .port }}
  targetPort: {{ .targetPort }}
  protocol: {{ .protocol | default "TCP" }}
  name: {{ .name }}
{{- end }}

# Iterate over a map (key-value pairs)
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}
```

### with — Scope Restriction

`with` sets the dot (`.`) to a new scope. If the value is empty/falsy, the block is skipped entirely — so it doubles as both a scope setter and a conditional.

```yaml
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}

# Equivalent to:
{{- if .Values.nodeSelector }}
nodeSelector:
  {{- toYaml .Values.nodeSelector | nindent 8 }}
{{- end }}
```

## template vs include

Both invoke a named template, but they behave differently:

| Aspect        | `{{ template "name" . }}`                          | `{{ include "name" . }}`                              |
| ------------- | -------------------------------------------------- | ----------------------------------------------------- |
| **Returns**   | Renders output inline (no return value)              | Returns a string                                       |
| **Pipeable**  | No — cannot pipe to other functions                  | Yes — can pipe to `nindent`, `quote`, `upper`, etc.    |
| **Use case**  | Rarely used in practice                              | Standard approach for all named template calls          |

```yaml
# template — output goes directly to the document, no pipeline support
{{ template "my-chart.labels" . }}

# include — output is a string you can manipulate
labels:
  {{- include "my-chart.labels" . | nindent 4 }}
```

Always use `include` over `template`. The ability to pipe to `nindent` for proper YAML indentation makes it essential for real-world charts.

## Named Templates and _helpers.tpl

Named templates are reusable fragments defined with `define` and invoked with `include`. By convention, they live in `templates/_helpers.tpl` (the underscore prefix tells Helm not to render the file as a manifest).

```yaml
# templates/_helpers.tpl

{{/*
Expand the name of the chart.
Truncated to 63 characters because Kubernetes name fields are limited.
*/}}
{{- define "my-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited.
If release name contains chart name it will be used as a full name.
*/}}
{{- define "my-chart.fullname" -}}
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
Common labels — applied to every resource in the chart.
*/}}
{{- define "my-chart.labels" -}}
helm.sh/chart: {{ include "my-chart.chart" . }}
{{ include "my-chart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels — used by Deployments and Services to match Pods.
These must be immutable across upgrades.
*/}}
{{- define "my-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Chart label value — "name-version" format.
*/}}
{{- define "my-chart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create the name of the ServiceAccount to use.
*/}}
{{- define "my-chart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-chart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

Usage in templates:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "my-chart.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          ports:
            {{- range .Values.service.ports }}
            - name: {{ .name }}
              containerPort: {{ .targetPort }}
              protocol: {{ .protocol | default "TCP" }}
            {{- end }}
```

## Whitespace Control

Go templates emit whitespace as-is, which can produce ugly YAML with blank lines and broken indentation. Use `{{-` and `-}}` to trim whitespace:

- `{{-` trims all whitespace (including newlines) **before** the directive
- `-}}` trims all whitespace (including newlines) **after** the directive

```yaml
# Without whitespace control — produces blank lines
metadata:
  labels:
    {{ if .Values.customLabel }}
    custom: {{ .Values.customLabel }}
    {{ end }}
    app: my-app

# With whitespace control — clean output
metadata:
  labels:
    {{- if .Values.customLabel }}
    custom: {{ .Values.customLabel }}
    {{- end }}
    app: my-app
```

A common mistake is over-trimming. `{{- ` eats everything to the left, including the newline from the previous line. Be careful not to collapse lines that should be separate.

```yaml
# WRONG — the dash eats the newline, merging lines
data:
  key1: value1
  {{- "key2: value2" }}
# Renders as:
# data:
#   key1: value1key2: value2

# RIGHT — trim the blank line, but keep the newline before
data:
  key1: value1
  {{ "key2: value2" }}
```

## Accessing Files

The `.Files` object lets you embed content from non-template files in the chart. This is useful for configuration files, scripts, and other assets.

```yaml
# Read a file as a string
data:
  nginx.conf: |
    {{ .Files.Get "config/nginx.conf" | nindent 4 }}

# Read a file as base64 (useful for Secrets)
data:
  tls.crt: {{ .Files.Get "certs/tls.crt" | b64enc }}

# Iterate over files matching a glob pattern
data:
  {{- range $path, $_ := .Files.Glob "config/*.yaml" }}
  {{ base $path }}: |
    {{ $.Files.Get $path | nindent 4 }}
  {{- end }}

# Read binary files
data:
  logo.png: {{ .Files.GetBytes "assets/logo.png" | b64enc }}
```

Important limitations:
- Files in `templates/` are not accessible via `.Files` (they are templates, not regular files)
- Files must be inside the chart directory
- Large files increase chart archive size — consider using ConfigMaps or external storage instead
- The chart archive cannot exceed the Kubernetes Secret size limit (1MB default) if stored as a secret

## NOTES.txt

The `templates/NOTES.txt` file is rendered and displayed to the user after `helm install` and `helm upgrade`. It is a Go template with access to all the same objects as other templates.

```
# templates/NOTES.txt
Thank you for installing {{ .Chart.Name }}!

Your release "{{ .Release.Name }}" has been deployed to namespace "{{ .Release.Namespace }}".

{{- if .Values.ingress.enabled }}

Access your application at:
{{- range .Values.ingress.hosts }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ .host }}
{{- end }}

{{- else if contains "LoadBalancer" .Values.service.type }}

Get the external IP:
  kubectl get svc {{ include "my-chart.fullname" . }} -n {{ .Release.Namespace }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

{{- else if contains "ClusterIP" .Values.service.type }}

Port-forward to access locally:
  kubectl port-forward svc/{{ include "my-chart.fullname" . }} 8080:{{ .Values.service.port }} -n {{ .Release.Namespace }}
  Then visit http://localhost:8080

{{- end }}
```

## Practical Examples

### Generating Environment Variables from a Map

```yaml
# values.yaml
env:
  LOG_LEVEL: info
  DATABASE_URL: postgres://db:5432/myapp
  CACHE_TTL: "300"

# templates/deployment.yaml (inside the container spec)
env:
  {{- range $name, $value := .Values.env }}
  - name: {{ $name }}
    value: {{ $value | quote }}
  {{- end }}
  {{- range .Values.envFromSecrets }}
  - name: {{ .name }}
    valueFrom:
      secretKeyRef:
        name: {{ .secretName }}
        key: {{ .secretKey }}
  {{- end }}
```

### Conditional Resource Creation with Multiple Checks

```yaml
# Only create a PodDisruptionBudget if we have more than 1 replica
{{- if and .Values.podDisruptionBudget.enabled (gt (int .Values.replicaCount) 1) }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  {{- if .Values.podDisruptionBudget.minAvailable }}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- else }}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable | default 1 }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
{{- end }}
```

### Adapting to Cluster API Versions

```yaml
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.GitVersion }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
kind: Ingress
```

### Helper for Image Pull Specification

```yaml
# _helpers.tpl
{{- define "my-chart.image" -}}
{{- $registry := .Values.image.registry | default "docker.io" -}}
{{- $repository := .Values.image.repository -}}
{{- $tag := .Values.image.tag | default .Chart.AppVersion -}}
{{- if .Values.image.digest }}
{{- printf "%s/%s@%s" $registry $repository .Values.image.digest }}
{{- else }}
{{- printf "%s/%s:%s" $registry $repository $tag }}
{{- end }}
{{- end }}

# Usage in deployment.yaml
image: {{ include "my-chart.image" . | quote }}
```
