# Helm Guide — From Basics to Multi-Environment Deployments

A practical, demo-driven guide using the `test-helm-template` chart in this repo.

---

## Table of Contents

1. [What is Helm?](#1-what-is-helm)
2. [Helm Architecture](#2-helm-architecture)
3. [Chart Structure](#3-chart-structure)
4. [values.yaml — The Core of Customization](#4-valuesyaml--the-core-of-customization)
5. [Template Syntax](#5-template-syntax)
6. [Essential Helm Commands](#6-essential-helm-commands)
7. [Multi-Environment Deployments](#7-multi-environment-deployments)
8. [Helm Hooks](#8-helm-hooks)
9. [Helm Dependencies (Subcharts)](#9-helm-dependencies-subcharts)
10. [Packaging & Sharing Charts](#10-packaging--sharing-charts)

---

## 1. What is Helm?

Helm is the **package manager for Kubernetes**. It solves a core problem: Kubernetes apps require many YAML manifests (Deployment, Service, ConfigMap, Ingress, etc.), and managing them by hand across multiple environments is error-prone and repetitive.

Helm lets you:
- Package all those manifests into a single **chart**
- Customize deployments with **values** (variables) instead of editing raw YAML
- Install, upgrade, and rollback with single commands
- Share and reuse charts via **repositories**

### Without Helm vs With Helm

```
Without Helm:
  kubectl apply -f deployment.yaml
  kubectl apply -f service.yaml
  kubectl apply -f ingress.yaml
  kubectl apply -f configmap.yaml
  # ... repeat for every environment, manually editing each file

With Helm:
  helm install my-app ./my-chart -f values-prod.yaml
  # One command. All manifests rendered and applied.
```

---

## 2. Helm Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Helm Client (CLI)                 │
│  helm install / upgrade / rollback / uninstall ...  │
└────────────────────────┬────────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │   Chart Repository  │  (Artifact Hub, OCI, local)
              │  bitnami, stable... │
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │   Chart (package)   │
              │  Chart.yaml         │
              │  values.yaml        │
              │  templates/         │
              └──────────┬──────────┘
                         │  rendered with values
              ┌──────────▼──────────┐
              │  Kubernetes Cluster │
              │  (Release stored    │
              │   as a Secret)      │
              └─────────────────────┘
```

**Key concepts:**

| Term | Description |
|---|---|
| **Chart** | A package of Kubernetes YAML templates + default values |
| **Release** | A specific deployed instance of a chart (you can install the same chart multiple times) |
| **Repository** | A collection of charts, hosted on a server |
| **Values** | Variables that customize a chart's templates at render time |
| **Revision** | Each `helm upgrade` creates a new revision; you can rollback to any previous one |

---

## 3. Chart Structure

Looking at `test-helm-template/` in this repo:

```
test-helm-template/
├── Chart.yaml              # Chart metadata (name, version, description)
├── values.yaml             # Default values (overridable by users)
├── .helmignore             # Files to exclude when packaging (like .gitignore)
└── templates/
    ├── _helpers.tpl        # Named templates / helper functions (not rendered directly)
    ├── deployment.yaml     # Kubernetes Deployment template
    ├── service.yaml        # Kubernetes Service template
    ├── ingress.yaml        # Kubernetes Ingress template
    ├── hpa.yaml            # HorizontalPodAutoscaler template
    ├── serviceaccount.yaml # ServiceAccount template
    ├── httproute.yaml      # Gateway API HTTPRoute template
    ├── NOTES.txt           # Post-install message shown to user
    └── tests/
        └── test-connection.yaml  # Helm test pod
```

### Chart.yaml explained

```yaml
apiVersion: v2           # Always v2 for Helm 3
name: test-helm-template # Chart name
description: A Helm chart for Kubernetes
type: application        # 'application' (deployable) or 'library' (helpers only)
version: 0.1.0           # Chart version — increment when you change the chart
appVersion: "1.16.0"     # Version of the app being deployed (informational)
```

> **Rule of thumb:** `version` is for the chart itself. `appVersion` is for the application (e.g., your Docker image tag).

---

## 4. values.yaml — The Core of Customization

`values.yaml` defines **default values**. Every key here can be referenced in templates and overridden by users.

### How values are accessed in templates

```yaml
# values.yaml
replicaCount: 1
image:
  repository: nginx
  tag: ""
```

```yaml
# templates/deployment.yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

The dot (`.`) is the **current context**. `.Values` accesses `values.yaml`. `.Chart` accesses `Chart.yaml`. `.Release` accesses release metadata.

### Override methods (in order of precedence, highest wins)

```
1. --set key=value          (CLI flag, highest priority)
2. --set-string key=value   (forces string type)
3. -f custom-values.yaml    (values file, can stack multiple)
4. values.yaml              (chart defaults, lowest priority)
```

**Examples:**

```bash
# Use a custom values file
helm install my-app ./test-helm-template -f my-values.yaml

# Override a single value inline
helm install my-app ./test-helm-template --set replicaCount=3

# Stack multiple values files (later files win)
helm install my-app ./test-helm-template -f values-base.yaml -f values-prod.yaml

# Mix file and inline override
helm install my-app ./test-helm-template -f values-prod.yaml --set image.tag=v2.1.0
```

### Inspect what values are in effect

```bash
# Only your overrides (what you passed)
helm get values my-app

# Full merged result including all chart defaults
helm get values my-app --all
```

---

## 5. Template Syntax

Helm uses the **Go template** language with Sprig functions added.

### 5.1 Basic value injection

```yaml
name: {{ .Values.image.repository }}
```

### 5.2 Default values

```yaml
# If image.tag is empty, fall back to Chart.AppVersion
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

### 5.3 if / else

```yaml
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
```

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- else }}
# Ingress is disabled, nothing rendered
{{- end }}
```

### 5.4 with (scoped context)

```yaml
# Instead of .Values.podAnnotations.foo, use .foo inside with block
{{- with .Values.podAnnotations }}
annotations:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

`with` also acts as a nil-check — the block only renders if the value is non-empty.

### 5.5 range (loops)

```yaml
# values.yaml
env:
  - name: ENV
    value: production
  - name: DEBUG
    value: "false"

# template
env:
  {{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
  {{- end }}
```

### 5.6 Named templates (_helpers.tpl)

Named templates avoid repetition. They are defined with `define` and called with `include`.

```yaml
# _helpers.tpl
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

# deployment.yaml — use it with nindent for proper indentation
labels:
  {{- include "myapp.labels" . | nindent 4 }}
```

### 5.7 toYaml, quote, nindent

| Function | Use |
|---|---|
| `toYaml` | Convert a values map/list to YAML string |
| `quote` | Wrap value in double quotes (important for strings that look like numbers) |
| `nindent N` | Add newline + N spaces indent (use after pipe) |
| `indent N` | Add N spaces indent without leading newline |
| `upper` / `lower` | String case conversion |
| `trunc 63` | Truncate string (DNS label max is 63 chars) |

### 5.8 Release and Chart built-in objects

```yaml
{{ .Release.Name }}        # e.g. "my-wordpress"
{{ .Release.Namespace }}   # e.g. "default"
{{ .Release.IsInstall }}   # true on first install
{{ .Release.IsUpgrade }}   # true on upgrade
{{ .Chart.Name }}          # e.g. "test-helm-template"
{{ .Chart.Version }}       # e.g. "0.1.0"
{{ .Chart.AppVersion }}    # e.g. "1.16.0"
```

### 5.9 Dry-run to see rendered templates

```bash
# Render templates locally without connecting to cluster
helm template my-app ./test-helm-template -f my-values.yaml

# Dry-run against cluster (validates against API server)
helm install my-app ./test-helm-template --dry-run

# Show only a specific template
helm template my-app ./test-helm-template -s templates/deployment.yaml
```

---

## 6. Essential Helm Commands

### Repository management

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update local repo cache
helm repo update

# Search for a chart
helm search repo wordpress

# Search Artifact Hub (public)
helm search hub nginx
```

### Install / Upgrade / Rollback

```bash
# Install a chart
helm install <release-name> <chart> [flags]
helm install my-app ./test-helm-template
helm install my-app ./test-helm-template -f values-dev.yaml
helm install my-app bitnami/wordpress -f wordpress-values.yaml

# Upgrade (update running release)
helm upgrade my-app ./test-helm-template -f values-dev.yaml

# Install OR upgrade if already exists (idempotent)
helm upgrade --install my-app ./test-helm-template -f values-dev.yaml

# Rollback to previous revision
helm rollback my-app        # rolls back to previous revision
helm rollback my-app 2      # rolls back to revision 2

# Uninstall
helm uninstall my-app
```

### Inspect a release

```bash
helm list                          # all releases in current namespace
helm list -A                       # all namespaces
helm status my-app                 # current status
helm history my-app                # all revisions with timestamps
helm get values my-app             # overridden values only
helm get values my-app --all       # all merged values
helm get manifest my-app           # rendered YAML currently deployed
```

### Debugging & Validation

```bash
helm lint ./test-helm-template              # check for errors
helm template my-app ./test-helm-template  # render templates locally
helm diff upgrade my-app ./chart -f new-values.yaml  # requires helm-diff plugin
```

---

## 7. Multi-Environment Deployments

This is one of Helm's most powerful features. You maintain **one chart** and use **different values files** per environment.

### Strategy

```
test-helm-template/        ← single chart (shared)
  values.yaml              ← base defaults

helm/envs/
  values-dev.yaml          ← development overrides
  values-staging.yaml      ← staging overrides
  values-prod.yaml         ← production overrides
```

### Deploy commands per environment

```bash
# Development
helm upgrade --install myapp-dev ./test-helm-template \
  -f helm/envs/values-dev.yaml \
  -n dev --create-namespace

# Staging
helm upgrade --install myapp-staging ./test-helm-template \
  -f helm/envs/values-staging.yaml \
  -n staging --create-namespace

# Production
helm upgrade --install myapp-prod ./test-helm-template \
  -f helm/envs/values-prod.yaml \
  -n prod --create-namespace
```

> You can also stack a base file + env override: `-f values-base.yaml -f values-prod.yaml`

### See `helm/envs/` for the demo values files:
- [`values-dev.yaml`](envs/values-dev.yaml)
- [`values-staging.yaml`](envs/values-staging.yaml)
- [`values-prod.yaml`](envs/values-prod.yaml)

### Key differences across environments

| Feature | Dev | Staging | Prod |
|---|---|---|---|
| `replicaCount` | 1 | 2 | 3 |
| `image.tag` | `latest` | `v1.2.0-rc1` | `v1.2.0` |
| `resources` | minimal | medium | full |
| `autoscaling` | disabled | disabled | enabled |
| `ingress` | disabled | enabled | enabled |
| `nodeSelector` | none | none | `env: prod` |

### Simulating the render (no cluster needed)

```bash
# What will be deployed to dev?
helm template myapp-dev ./test-helm-template -f helm/envs/values-dev.yaml

# What will be deployed to prod?
helm template myapp-prod ./test-helm-template -f helm/envs/values-prod.yaml

# Diff between staging and prod
diff \
  <(helm template myapp ./test-helm-template -f helm/envs/values-staging.yaml) \
  <(helm template myapp ./test-helm-template -f helm/envs/values-prod.yaml)
```

---

## 8. Helm Hooks

Hooks let you run Jobs at specific points in the release lifecycle.

### Hook types

| Hook | Runs when |
|---|---|
| `pre-install` | Before any manifests are installed |
| `post-install` | After all manifests are installed |
| `pre-upgrade` | Before upgrade begins |
| `post-upgrade` | After upgrade completes |
| `pre-rollback` | Before rollback begins |
| `post-rollback` | After rollback completes |
| `pre-delete` | Before uninstall begins |
| `post-delete` | After uninstall completes |
| `test` | When `helm test` is run |

### Example: database migration before upgrade

```yaml
# templates/db-migrate-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"          # lower runs first
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./migrate.sh"]
```

### Hook weight

When multiple hooks run at the same point, `helm.sh/hook-weight` controls order (lowest first, can be negative).

### Hook delete policy

| Policy | Behaviour |
|---|---|
| `before-hook-creation` | Delete old hook resource before creating new one (default) |
| `hook-succeeded` | Delete after successful completion |
| `hook-failed` | Delete if hook fails |

---

## 9. Helm Dependencies (Subcharts)

Charts can declare dependencies on other charts (subcharts). The WordPress chart you deployed uses this — MariaDB is a dependency.

### Declare in Chart.yaml

```yaml
# Chart.yaml
dependencies:
  - name: mariadb
    version: "11.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: mariadb.enabled    # only pull in if mariadb.enabled=true

  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### Download dependencies

```bash
helm dependency update ./my-chart
# Creates charts/ directory with .tgz files of each dependency
```

### Override subchart values

In your `values.yaml` or custom values file, use the dependency name as the key:

```yaml
# Override mariadb subchart values from parent chart
mariadb:
  enabled: true
  auth:
    database: myapp
    username: myuser
    password: mypass
  primary:
    persistence:
      enabled: true
      size: 8Gi
```

This is exactly what `wordpress-values.yaml` does — it overrides the MariaDB subchart that ships with the WordPress chart.

---

## 10. Packaging & Sharing Charts

### Package a chart into a .tgz

```bash
helm package ./test-helm-template
# Creates: test-helm-template-0.1.0.tgz
```

### Host a simple chart repository

```bash
# Generate index from a directory of .tgz files
helm repo index ./charts-dir --url https://myorg.github.io/helm-charts

# Serve locally for testing
helm serve  # (older method, for testing only)
```

### Use OCI registries (modern approach)

```bash
# Push to an OCI registry (e.g. GHCR, ACR, ECR)
helm push test-helm-template-0.1.0.tgz oci://ghcr.io/myorg/helm-charts

# Pull and install from OCI
helm install my-app oci://ghcr.io/myorg/helm-charts/test-helm-template --version 0.1.0
```

### Helm test

Run tests defined in `templates/tests/` to verify a release works:

```bash
helm test my-app
# Runs test-connection.yaml pod and checks it exits 0
```

---

## Quick Reference

```bash
# Common workflow
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
helm show values bitnami/nginx > my-values.yaml   # dump defaults to edit
helm install my-nginx bitnami/nginx -f my-values.yaml
helm list
helm upgrade my-nginx bitnami/nginx -f my-values.yaml --set image.tag=1.25
helm history my-nginx
helm rollback my-nginx 1
helm uninstall my-nginx

# Chart development workflow
helm create my-chart
helm lint my-chart
helm template my-app my-chart -f values-dev.yaml
helm install my-app my-chart --dry-run
helm install my-app my-chart
helm test my-app
helm package my-chart
```
