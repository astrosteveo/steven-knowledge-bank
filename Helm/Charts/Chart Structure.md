---
tags:
  - helm
  - helm/charts
topic: Charts
---

# Chart Structure

## Standard Directory Layout

Every Helm chart follows a well-defined directory structure. When you run `helm create my-chart`, you get the scaffolding below. Understanding each file's role is essential for authoring and debugging charts.

```
my-chart/
├── Chart.yaml          # Chart metadata (name, version, dependencies)
├── Chart.lock          # Dependency lock file (auto-generated)
├── values.yaml         # Default configuration values
├── values.schema.json  # Optional JSON Schema for values validation
├── .helmignore         # Patterns to exclude from chart packaging
├── README.md           # Human-readable documentation
├── LICENSE             # License for the chart
├── charts/             # Dependency charts (tarballs or directories)
├── crds/               # Custom Resource Definitions
└── templates/          # Kubernetes manifest templates
    ├── _helpers.tpl        # Named template definitions (partials)
    ├── NOTES.txt           # Post-install/upgrade user-facing notes
    ├── deployment.yaml     # Deployment manifest template
    ├── service.yaml        # Service manifest template
    ├── serviceaccount.yaml # ServiceAccount template
    ├── ingress.yaml        # Ingress template
    ├── hpa.yaml            # HorizontalPodAutoscaler template
    └── tests/              # Helm test definitions
        └── test-connection.yaml
```

## Chart.yaml in Detail

The `Chart.yaml` file is the chart's identity card. It declares metadata, dependencies, and versioning.

### Required vs Optional Fields

| Field           | Required | Description                                                                 |
| --------------- | -------- | --------------------------------------------------------------------------- |
| `apiVersion`    | Yes      | Always `v2` for Helm 3 charts (`v1` was Helm 2)                            |
| `name`          | Yes      | The name of the chart — must match the directory name                       |
| `version`       | Yes      | SemVer chart version — bumped on every chart change                         |
| `appVersion`    | No       | Version of the application being deployed (informational only)               |
| `description`   | No       | One-line description of the chart                                            |
| `type`          | No       | `application` (default) or `library`                                         |
| `dependencies`  | No       | List of charts this chart depends on                                         |
| `maintainers`   | No       | List of maintainers with name, email, url                                    |
| `keywords`      | No       | Searchable keywords for Artifact Hub discovery                               |
| `sources`       | No       | URLs to source code for the chart or application                             |
| `home`          | No       | URL of the project homepage                                                  |
| `icon`          | No       | URL to an SVG or PNG icon for Artifact Hub                                   |
| `deprecated`    | No       | If `true`, marks the chart as deprecated                                     |
| `annotations`   | No       | Arbitrary key-value metadata (used by Artifact Hub, CI/CD, etc.)             |
| `kubeVersion`   | No       | SemVer constraint on supported Kubernetes versions (e.g., `>=1.25.0-0`)      |

### Complete Example Chart.yaml

```yaml
apiVersion: v2
name: my-web-app
version: 2.1.0
appVersion: "4.3.1"
description: A production-ready web application with caching and background workers
type: application

# Kubernetes version constraint — Helm will refuse to install on incompatible clusters
kubeVersion: ">=1.26.0-0"

# Metadata for Artifact Hub and chart discovery
keywords:
  - web
  - application
  - nodejs
home: https://github.com/myorg/my-web-app
icon: https://raw.githubusercontent.com/myorg/my-web-app/main/icon.png
sources:
  - https://github.com/myorg/my-web-app
  - https://github.com/myorg/my-web-app-chart

maintainers:
  - name: Platform Team
    email: platform@myorg.com
    url: https://myorg.com/platform

# Dependencies — resolved via `helm dependency update`
dependencies:
  - name: redis
    version: "~20.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
  - name: postgresql
    version: "~16.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: common
    version: "2.x.x"
    repository: https://charts.bitnami.com/bitnami
    tags:
      - bitnami-common

annotations:
  artifacthub.io/changes: |
    - kind: added
      description: Support for horizontal pod autoscaling
    - kind: fixed
      description: Ingress TLS configuration not applied correctly
  artifacthub.io/license: Apache-2.0
```

## .helmignore

The `.helmignore` file tells `helm package` which files and directories to exclude from the chart archive. It uses the same syntax as `.gitignore`:

```
# Common .helmignore patterns
.git
.gitignore
.vscode/
.idea/
*.swp
*.bak
*.tmp
*.orig
*.DS_Store

# CI/CD files — not needed in the chart
.github/
.gitlab-ci.yml
Makefile
Dockerfile
Jenkinsfile

# Test and development files
tests/
ci/
*.test.yaml
```

If no `.helmignore` file exists, Helm includes every file in the chart directory in the packaged archive. Always include one to keep chart archives lean.

## The templates/ Directory

This is where the actual Kubernetes manifests live, written as Go templates that Helm renders with values at install time.

### Key Files

| File                     | Purpose                                                                                               |
| ------------------------ | ----------------------------------------------------------------------------------------------------- |
| `_helpers.tpl`           | Named template definitions (partials) shared across all templates — labels, names, selectors, etc.    |
| `NOTES.txt`              | Rendered and displayed to the user after `helm install` or `helm upgrade` — connection instructions    |
| `deployment.yaml`        | The main application Deployment                                                                       |
| `service.yaml`           | ClusterIP/NodePort/LoadBalancer Service exposing the application                                      |
| `serviceaccount.yaml`    | ServiceAccount for the application Pods                                                               |
| `ingress.yaml`           | Ingress resource for HTTP routing — typically wrapped in a conditional                                 |
| `hpa.yaml`               | HorizontalPodAutoscaler — usually conditional on `autoscaling.enabled`                                |
| `configmap.yaml`         | Application configuration (optional, not in default scaffold)                                         |
| `secret.yaml`            | Sensitive configuration (optional, not in default scaffold)                                           |
| `pvc.yaml`               | PersistentVolumeClaim for stateful data (optional)                                                    |
| `tests/test-*.yaml`      | Helm test Pods that verify the release works — run with `helm test`                                   |

Files with a leading underscore (`_helpers.tpl`) are **not rendered** as Kubernetes manifests. They exist only to define reusable template fragments.

The `NOTES.txt` file is also a Go template — it can use `.Values`, `.Release`, and all the same functions as other templates to generate dynamic post-install instructions.

## charts/ Directory

The `charts/` directory stores dependency charts. Dependencies are placed here in one of two ways:

1. **Automatically** via `helm dependency update` — downloads tarballs from the repository URLs declared in `Chart.yaml`
2. **Manually** — you can place an unpacked chart directory here for local development

```
charts/
├── redis-20.6.2.tgz       # Downloaded dependency
├── postgresql-16.4.1.tgz   # Downloaded dependency
└── my-internal-lib/         # Local/vendored chart directory
    ├── Chart.yaml
    ├── templates/
    └── ...
```

Charts in this directory are installed as subcharts. Their resources are rendered with the parent chart's values (scoped under the subchart name) and deployed alongside the parent.

## crds/ Directory

Files in the `crds/` directory are treated specially by Helm:

- **Installed** on `helm install` before any templates are rendered (so templates can reference the CRDs)
- **Never upgraded** — Helm will not modify CRDs on `helm upgrade`
- **Never deleted** — Helm will not remove CRDs on `helm uninstall`

This behavior exists because CRDs are cluster-scoped and deleting them would destroy all Custom Resources across every namespace. Helm takes the safe route: install once, then hands-off.

```yaml
# crds/myresource-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.myorg.io
spec:
  group: myorg.io
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
  scope: Namespaced
  names:
    plural: myresources
    singular: myresource
    kind: MyResource
```

If you need CRDs to be upgradable or deletable, do not use the `crds/` directory. Instead, place them as regular templates in `templates/` and manage them explicitly. Many production charts (e.g., cert-manager, Prometheus Operator) use this approach.

## Library Charts vs Application Charts

The `type` field in `Chart.yaml` distinguishes two chart types:

| Aspect                | Application Chart (`type: application`)           | Library Chart (`type: library`)                        |
| --------------------- | ------------------------------------------------- | ------------------------------------------------------ |
| **Default**           | Yes — this is the default if `type` is omitted     | Must be explicitly declared                             |
| **Renderable**        | Yes — templates are rendered into K8s manifests     | No — templates are not rendered directly                |
| **Installable**       | Yes — can be installed with `helm install`          | No — cannot be installed standalone                     |
| **Purpose**           | Deploys a workload to Kubernetes                   | Provides reusable template helpers to other charts      |
| **Contains**          | Templates, values, helpers, tests                  | Only `_helpers.tpl`-style named templates                |
| **Used via**          | `helm install`, `helm upgrade`                     | Declared as a dependency by application charts           |

Library charts are a DRY mechanism. If your organization has ten microservice charts that all need the same label structure, resource naming convention, or boilerplate templates, extract those into a library chart and import it as a dependency.

```yaml
# Library chart — Chart.yaml
apiVersion: v2
name: myorg-common
version: 1.0.0
type: library
description: Shared template helpers for all myorg charts
```

```yaml
# Application chart using the library — Chart.yaml
apiVersion: v2
name: my-service
version: 1.0.0
type: application
dependencies:
  - name: myorg-common
    version: "1.x.x"
    repository: "https://charts.myorg.com"
```

The application chart can then use `{{ include "myorg-common.labels" . }}` and other named templates defined in the library.
