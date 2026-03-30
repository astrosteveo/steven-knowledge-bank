---
tags:
  - helm
  - helm/reference
topic: Quick Reference
---

# Template Functions

Helm templates use Go's `text/template` language augmented with the [Sprig](https://masterminds.github.io/sprig/) function library and several Helm-specific functions. This reference covers the most important functions organized by category.

## String Functions

String manipulation is the most commonly used function category in Helm templates.

```yaml
# Case conversion
{{ upper "hello" }}             # HELLO
{{ lower "HELLO" }}             # hello
{{ title "hello world" }}       # Hello World

# Trimming
{{ trim "  hello  " }}          # hello
{{ trimPrefix "v" "v1.2.3" }}   # 1.2.3
{{ trimSuffix ".yaml" "app.yaml" }}  # app

# Replace and search
{{ replace "old" "new" "the old value" }}  # the new value
{{ contains "world" "hello world" }}       # true
{{ hasPrefix "http" "https://example.com" }}  # true
{{ hasSuffix ".com" "example.com" }}          # true

# Quoting
{{ quote "hello" }}             # "hello"
{{ squote "hello" }}            # 'hello'

# Concatenation
{{ cat "hello" "world" }}       # hello world

# Indentation — critical for YAML embedding
{{ "key: value" | indent 4 }}   # adds 4 spaces to every line (including the first)
{{ "key: value" | nindent 4 }}  # adds a newline, then indents 4 spaces

# Formatted printing
{{ printf "%s-svc" .Release.Name }}  # my-release-svc
{{ printf "%s:%d" "nginx" 80 }}      # nginx:80
```

### The toYaml | nindent Pattern

This is the single most important pattern in Helm templating. It converts a values structure to properly indented YAML.

```yaml
# values.yaml
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

nodeSelector:
  disktype: ssd
  region: us-east-1

# In a template (deployment.yaml)
spec:
  containers:
    - name: {{ .Chart.Name }}
      resources:
        {{- toYaml .Values.resources | nindent 8 }}
      # renders as:
      #   resources:
      #     limits:
      #       cpu: 100m
      #       memory: 128Mi
      #     requests:
      #       cpu: 50m
      #       memory: 64Mi

  {{- with .Values.nodeSelector }}
  nodeSelector:
    {{- toYaml . | nindent 4 }}
  {{- end }}
```

**Why nindent and not indent?** `nindent` inserts a newline before the indented content, so the `{{-` can eat the whitespace before it. This gives you clean, predictable YAML every time. Using `indent` without the leading newline causes the first line to be at the wrong indentation level.

```yaml
# WRONG — indent without dash-trimming produces broken YAML
  resources:
    {{ toYaml .Values.resources | indent 4 }}
    # First line ends up on the same line as the key with extra spaces

# CORRECT — nindent with dash-trimmed left whitespace
  resources:
    {{- toYaml .Values.resources | nindent 4 }}
    # Clean newline, then 4-space indent on every line
```

## Type Conversion Functions

Converting between types is essential for embedding YAML, generating JSON, and handling user input.

```yaml
# YAML and JSON conversion
{{ toYaml .Values.labels }}         # Go structure → YAML string
{{ toJson .Values.config }}         # Go structure → JSON string
{{ fromYaml $yamlString }}          # YAML string → Go structure
{{ fromJson $jsonString }}          # JSON string → Go structure

# Numeric/string conversions
{{ toString 42 }}                   # "42"
{{ toStrings (list 1 2 3) }}        # ["1" "2" "3"]
{{ atoi "42" }}                     # 42 (string to int)
{{ int64 42 }}                      # converts to int64
{{ float64 42 }}                    # converts to float64

# Practical example: embedding JSON in a ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  config.json: |
    {{ toJson .Values.appConfig | indent 4 }}

# Practical example: iterating over fromYaml result
{{- $config := fromYaml (include "my-chart.defaultConfig" .) -}}
{{- range $key, $val := $config }}
  {{ $key }}: {{ $val }}
{{- end }}
```

## Math Functions

```yaml
{{ add 1 2 }}           # 3
{{ sub 10 3 }}          # 7
{{ mul 2 5 }}           # 10
{{ div 10 3 }}          # 3 (integer division)
{{ mod 10 3 }}          # 1
{{ max 5 10 }}          # 10
{{ min 5 10 }}          # 5
{{ ceil 1.5 }}          # 2
{{ floor 1.9 }}         # 1
{{ round 1.555 2 }}     # 1.56

# Practical example: calculating resource limits
{{- $cpuRequest := .Values.resources.requests.cpu | default "100m" }}
{{- $replicaCount := .Values.replicaCount | default 1 }}
# Use math to compute total replicas for annotations
replicas: {{ mul $replicaCount 1 }}
```

## List Functions

```yaml
# Creating lists
{{ list "a" "b" "c" }}                # [a b c]

# Accessing elements
{{ first (list "a" "b" "c") }}        # a
{{ last (list "a" "b" "c") }}         # c
{{ rest (list "a" "b" "c") }}         # [b c]  (all but first)
{{ initial (list "a" "b" "c") }}      # [a b]  (all but last)

# Modifying lists
{{ append (list "a" "b") "c" }}       # [a b c]
{{ prepend (list "b" "c") "a" }}      # [a b c]
{{ concat (list "a" "b") (list "c" "d") }}  # [a b c d]

# Searching and filtering
{{ has "b" (list "a" "b" "c") }}      # true
{{ without (list "a" "b" "c") "b" }}  # [a c]
{{ uniq (list "a" "b" "a" "c") }}     # [a b c]
{{ compact (list "a" "" "b" "") }}    # [a b]  (removes empty strings)

# Sorting and reversing
{{ sortAlpha (list "c" "a" "b") }}    # [a b c]
{{ reverse (list "a" "b" "c") }}      # [c b a]

# Practical example: generating comma-separated list of hostnames
{{- $hosts := list -}}
{{- range .Values.ingress.hosts }}
  {{- $hosts = append $hosts .host -}}
{{- end }}
ALLOWED_HOSTS={{ join "," $hosts }}
```

## Dict (Map) Functions

```yaml
# Creating dictionaries
{{ dict "key1" "val1" "key2" "val2" }}

# Modifying dictionaries
{{ $d := dict "name" "app" }}
{{ $_ := set $d "version" "1.0" }}    # adds key; assign to $_ to suppress output
{{ $_ := unset $d "version" }}         # removes key

# Querying
{{ hasKey $d "name" }}                 # true
{{ pluck "name" $d1 $d2 $d3 }}        # returns list of values for "name" from multiple dicts
{{ keys $d }}                          # [name version]
{{ values $d }}                        # [app 1.0]

# Filtering
{{ pick $d "name" "version" }}         # only keep these keys
{{ omit $d "internal" "debug" }}       # remove these keys

# Merging (left-most wins for conflicts)
{{ merge $default $override }}          # $default values take precedence
{{ mergeOverwrite $base $override }}    # $override values take precedence

# Deep copy (avoids mutating the original)
{{ deepCopy .Values.config }}

# Practical example: building labels
{{- $labels := dict
  "app.kubernetes.io/name" (include "my-chart.name" .)
  "app.kubernetes.io/instance" .Release.Name
  "app.kubernetes.io/version" .Chart.AppVersion
  "app.kubernetes.io/managed-by" .Release.Service
-}}
labels:
  {{- toYaml $labels | nindent 2 }}
```

## Flow Control Functions

These functions handle defaults, validation, and conditional logic inside expressions.

```yaml
# Default — provide a fallback if a value is empty/nil
{{ .Values.image.tag | default .Chart.AppVersion }}
{{ .Values.storageClass | default "standard" }}

# Empty — returns true if the value is the zero value for its type
{{ empty "" }}           # true
{{ empty 0 }}            # true
{{ empty (list) }}       # true
{{ empty (dict) }}       # true

# Coalesce — returns the first non-empty value
{{ coalesce .Values.custom.image .Values.global.image "nginx:latest" }}

# Required — fail the template if the value is empty
{{ required "A valid .Values.database.host is required" .Values.database.host }}
# If database.host is empty → error: "A valid .Values.database.host is required"

# Fail — unconditionally fail with a message
{{ if gt (len .Values.servers) 10 }}
  {{ fail "No more than 10 servers are supported" }}
{{ end }}

# Ternary — inline if/else
{{ ternary "yes" "no" true }}    # yes
{{ ternary "yes" "no" false }}   # no

# Practical example: conditional service type
type: {{ ternary "LoadBalancer" "ClusterIP" .Values.service.external }}
```

## Encoding Functions

```yaml
# Base64
{{ b64enc "hello" }}              # aGVsbG8=
{{ b64dec "aGVsbG8=" }}           # hello

# SHA256 hash
{{ sha256sum "hello" }}           # 2cf24dba5fb0a30e26e83b2ac5b9e29e...

# Practical example: triggering redeployment when ConfigMap changes
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

## Date Functions

```yaml
# Current time
{{ now }}                                    # 2024-10-04 09:15:00.123456 +0000 UTC

# Format a date (Go reference time: Mon Jan 2 15:04:05 MST 2006)
{{ now | date "2006-01-02" }}                # 2024-10-04
{{ now | date "2006-01-02T15:04:05Z07:00" }} # 2024-10-04T09:15:00+00:00

# Modify a date
{{ now | dateModify "+24h" }}                 # tomorrow
{{ now | dateModify "-168h" }}                # one week ago

# Parse a date string
{{ toDate "2006-01-02" "2024-10-04" }}       # time.Time object

# Practical example: setting certificate expiry
annotations:
  cert-expiry: {{ now | dateModify "+8760h" | date "2006-01-02" }}
  # 1 year from now
```

## Path Functions

```yaml
{{ base "path/to/file.yaml" }}    # file.yaml
{{ dir "path/to/file.yaml" }}     # path/to
{{ ext "file.yaml" }}             # .yaml
{{ clean "path//to/../file" }}    # path/file
{{ isAbs "/etc/config" }}         # true
{{ isAbs "relative/path" }}       # false
```

## Regex Functions

```yaml
# Match — returns true if pattern matches
{{ regexMatch "^v[0-9]+\\.[0-9]+\\.[0-9]+$" "v1.2.3" }}  # true

# Find — returns the first match
{{ regexFind "[0-9]+" "version-123-abc" }}  # 123

# Replace
{{ regexReplaceAll "[^a-zA-Z0-9]" "hello-world_123!" "" }}  # helloworld123

# Split
{{ regexSplit "\\s+" "hello   world   foo" -1 }}  # [hello world foo]

# Practical example: sanitizing a release name for use as a DNS label
{{- define "my-chart.sanitizedName" -}}
{{ regexReplaceAll "[^a-z0-9-]" (lower .Release.Name) "" | trunc 63 | trimSuffix "-" }}
{{- end -}}
```

## Crypto Functions

```yaml
# Generate a private key
{{ genPrivateKey "rsa" }}          # PEM-encoded RSA private key
{{ genPrivateKey "ecdsa" }}        # PEM-encoded ECDSA private key

# Generate a CA certificate
{{- $ca := genCA "my-ca" 365 -}}
ca.crt: {{ $ca.Cert | b64enc }}
ca.key: {{ $ca.Key | b64enc }}

# Generate a signed certificate
{{- $cert := genSignedCert "my-service.my-namespace.svc" nil
    (list "my-service.my-namespace.svc" "my-service.my-namespace.svc.cluster.local")
    365 $ca -}}
tls.crt: {{ $cert.Cert | b64enc }}
tls.key: {{ $cert.Key | b64enc }}

# Derive a password (deterministic)
{{ derivePassword 1 "long" "master-password" "user" "example.com" }}
```

## Kubernetes-Specific Functions

### lookup

Query live cluster resources during template rendering. Returns an empty dict during `helm template` (no cluster).

```yaml
# Look up a single resource
{{ $secret := lookup "v1" "Secret" "my-namespace" "my-secret" }}

# If the secret exists, use its value; otherwise generate a new one
{{- $existing := lookup "v1" "Secret" .Release.Namespace (printf "%s-secret" .Release.Name) -}}
{{- if $existing -}}
password: {{ index $existing.data "password" }}
{{- else -}}
password: {{ randAlphaNum 32 | b64enc }}
{{- end -}}

# Look up all resources of a type (empty name = list all)
{{ $services := lookup "v1" "Service" "my-namespace" "" }}
{{- range $services.items }}
  - {{ .metadata.name }}
{{- end }}

# Check if a CRD exists before creating a resource
{{- if lookup "apiextensions.k8s.io/v1" "CustomResourceDefinition" "" "certificates.cert-manager.io" }}
# cert-manager CRD exists, safe to create Certificate resources
{{- end }}
```

### include

Call a named template and capture its output as a string (unlike `template`, which outputs directly).

```yaml
# _helpers.tpl
{{- define "my-chart.labels" -}}
app.kubernetes.io/name: {{ include "my-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

# deployment.yaml — include returns a string, so you can pipe it
metadata:
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}

# Why include instead of template?
# "template" outputs directly — you can't pipe it through nindent
# "include" returns a string — you CAN pipe it: include "name" . | nindent 4
```

### tpl

Render a string as a template. Useful when values.yaml contains template expressions.

```yaml
# values.yaml
configContent: |
  host={{ .Release.Name }}-db.{{ .Release.Namespace }}.svc
  port=5432

# template
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  app.conf: |
    {{ tpl .Values.configContent . | nindent 4 }}
    # The template expressions inside configContent are evaluated

# Another common use: templated annotations
# values.yaml
annotations:
  iam.amazonaws.com/role: "arn:aws:iam::{{ .Values.awsAccountId }}:role/{{ .Release.Name }}"

# template
metadata:
  annotations:
    {{- range $key, $value := .Values.annotations }}
    {{ $key }}: {{ tpl $value $ | quote }}
    {{- end }}
```

## Function Reference Table

| Category | Functions |
|---|---|
| **String** | `upper`, `lower`, `title`, `trim`, `trimPrefix`, `trimSuffix`, `replace`, `contains`, `hasPrefix`, `hasSuffix`, `quote`, `squote`, `cat`, `indent`, `nindent`, `printf`, `trunc`, `abbrev`, `wrap`, `repeat`, `substr` |
| **Type** | `toYaml`, `toJson`, `fromYaml`, `fromJson`, `toString`, `toStrings`, `atoi`, `int64`, `float64` |
| **Math** | `add`, `sub`, `mul`, `div`, `mod`, `max`, `min`, `ceil`, `floor`, `round` |
| **List** | `list`, `first`, `rest`, `last`, `initial`, `append`, `prepend`, `concat`, `has`, `without`, `uniq`, `sortAlpha`, `reverse`, `compact`, `join`, `seq` |
| **Dict** | `dict`, `set`, `unset`, `hasKey`, `pluck`, `merge`, `mergeOverwrite`, `keys`, `values`, `pick`, `omit`, `deepCopy` |
| **Flow** | `fail`, `required`, `default`, `empty`, `coalesce`, `ternary` |
| **Encoding** | `b64enc`, `b64dec`, `sha256sum`, `sha1sum` |
| **Date** | `now`, `date`, `dateModify`, `toDate`, `dateInZone` |
| **Path** | `base`, `dir`, `ext`, `clean`, `isAbs` |
| **Regex** | `regexMatch`, `regexFind`, `regexFindAll`, `regexReplaceAll`, `regexSplit` |
| **Crypto** | `genPrivateKey`, `derivePassword`, `genCA`, `genSignedCert`, `genSelfSignedCert` |
| **Kubernetes** | `lookup`, `include`, `tpl` |
| **Random** | `randAlphaNum`, `randAlpha`, `randNumeric`, `randAscii`, `uuidv4` |
