# Architecture & Concepts Reference

This document explains the design decisions, tools, and concepts behind this project.
It is a supporting reference for the main README.

---

## Table of Contents

1. [Repository Layout](#repository-layout)
2. [Helm](#helm)
   - [Why Helm](#why-helm)
   - [Chart Structure](#chart-structure)
   - [Templating and Values](#templating-and-values)
   - [Dependencies](#dependencies)
   - [Lifecycle Hooks](#lifecycle-hooks)
   - [Testing](#testing)
   - [Rollbacks](#rollbacks)
3. [GitOps Flow](#gitops-flow)
4. [ArgoCD](#argocd)
   - [Architecture](#architecture)
   - [Application CRD](#application-crd)
   - [Sync Behaviour and Drift Detection](#sync-behaviour-and-drift-detection)
   - [What ArgoCD Manages (and What It Does Not)](#what-argocd-manages-and-what-it-does-not)
   - [ArgoCD Image Updater](#argocd-image-updater)
   - [UI and CLI](#ui-and-cli)
5. [CI/CD Pipeline (Jenkins)](#cicd-pipeline-jenkins)
6. [Security](#security)
   - [Role-Based Access Control](#role-based-access-control)
   - [HashiCorp Vault and External Secrets](#hashicorp-vault-and-external-secrets)
7. [Multi-Environment Setup](#multi-environment-setup)

---

## Repository Layout

The repository uses two long-lived branches with distinct responsibilities:

- **`main`**: source of truth for application source code and Helm chart definitions.
- **`gitops`**: source of truth for the *deployed state* of the cluster. ArgoCD watches only this branch.

Changes flow in one direction: from `main` into `gitops`, either via Jenkins (for image tags and chart syncs) or by a developer committing an override file and letting Jenkins propagate it.

```
helm/
├── charts/
│   └── metrics-app/        # Custom Helm chart (backend + frontend)
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── backend/
│           │   ├── deployment.yaml
│           │   └── service.yaml
│           ├── frontend/
│           │   ├── configmap.yaml
│           │   ├── deployment.yaml
│           │   ├── hpa.yaml
│           │   ├── ingress.yaml
│           │   └── service.yaml
│           ├── _helpers.tpl
│           ├── NOTES.txt
│           └── tests/
│               └── test-connection.yaml
├── env/
│   ├── dev.yaml             # Per-environment image tag overrides
│   ├── staging.yaml
│   └── prod.yaml
└── overrides/
    ├── argocd-values.yaml
    ├── argocd-image-updater-values.yaml
    ├── jenkinsci-values.yaml
    ├── metrics-app-dev.yaml
    ├── metrics-app-staging.yaml
    ├── metrics-app-prod.yaml
    ├── postgresql-values.yaml
    └── vault-values.yaml

manifests/
├── argocd/                  # ArgoCD Application resources
├── namespaces/              # All namespace definitions
├── database/                # PostgreSQL connection job
├── vault/                   # Vault RBAC, auth, secret store config
├── registry/                # In-cluster image registry
├── jenkins/                 # Jenkins RBAC
├── ingress/
└── cert-manager/
```

---

## Helm

### Why Helm

A Kubernetes application requires many resources to run: Deployments, Services, ConfigMaps, Ingresses, HPAs, and more. Managing these as individual files creates overhead - they must be kept consistent, applied in the right order, and updated together across multiple environments.

Helm solves this by bundling all resources for an application into a single versioned package called a **chart**. Key benefits in this project:

- **One chart, multiple environments**: `metrics-app` deploys to dev, staging, and production using the same chart with different values files.
- **Release history**: every `helm install` or `helm upgrade` is recorded as a numbered revision. Helm can roll back to any previous revision with a single command.
- **Pre-existing charts for infrastructure**: PostgreSQL, ArgoCD, Jenkins, and Vault are all deployed using community charts from Helm repositories, configured via override files rather than written from scratch.
- **Dependency management**: the custom `metrics-app` chart can declare sub-charts as dependencies, resolved and bundled automatically.

### Chart Structure

```
metrics-app/
├── Chart.yaml        # Name, version, appVersion, dependencies
├── values.yaml       # Default configuration values
└── templates/
    ├── _helpers.tpl  # Shared named templates (labels, name helpers)
    ├── NOTES.txt     # Post-install message
    ├── backend/
    ├── frontend/
    └── tests/
```

**`Chart.yaml`** declares the chart identity and semver version. `appVersion` tracks the application version (used as the default image tag).

**`values.yaml`** contains defaults for every configurable parameter. Environment-specific files (`helm/env/dev.yaml`, etc.) override only what differs per environment. The override files in `helm/overrides/` serve a similar purpose for infrastructure charts (ArgoCD, Jenkins, Vault, PostgreSQL).

**`_helpers.tpl`** defines named templates (Go template partials) shared across all resource files - most importantly the full resource name and the standard label set, so these are consistent across every resource in the chart.

### Templating and Values

Helm templates are Go `text/template` files that are rendered into plain Kubernetes YAML at install time, using a context object that includes:

- `.Values` - merged values from `values.yaml` plus any override files
- `.Release` - release name and namespace
- `.Chart` - contents of `Chart.yaml`

A typical pattern in this chart:

```yaml
# templates/backend/deployment.yaml
image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
resources:
  {{- toYaml .Values.backend.resources | nindent 12 }}
```

The `|` pipe chains functions - `toYaml` converts a Go map back to YAML, and `nindent` handles indentation correctly. The `{{-` dash trims leading whitespace to avoid blank lines in output.

Conditional rendering with `if` allows features to be toggled via values:

```yaml
{{- if .Values.frontend.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

This is how the same chart deploys differently per environment without duplicating manifests.

**Values merging order** (later wins):
1. `values.yaml` (chart defaults)
2. `-f override.yaml` files (e.g. `helm/env/dev.yaml`)
3. `--set key=value` flags

### Dependencies

Dependencies are declared in `Chart.yaml` and resolved by `helm dependency update`, which downloads them into `charts/` as `.tgz` archives. The `charts/` directory and the generated `Chart.lock` (which pins exact resolved versions) are committed to the repository so the chart is fully reproducible without network access.

Values for a sub-chart are namespaced under the dependency's name in the parent `values.yaml`:

```yaml
postgresql:
  enabled: true
  auth:
    existingSecret: postgresql-credentials
  primary:
    persistence:
      enabled: true
      size: 10Gi
```

The `condition` field (`postgresql.enabled`) allows a dependency to be toggled per environment without modifying the chart.

### Lifecycle Hooks

Helm hooks run Kubernetes Jobs or Pods at specific points in the release lifecycle, outside the normal resource apply flow. They are annotated with `helm.sh/hook`:

| Hook | When it runs |
|------|-------------|
| `pre-install` | After rendering, before any resources are applied |
| `post-install` | After all resources are successfully applied |
| `pre-upgrade` | Before an upgrade |
| `post-upgrade` | After a successful upgrade |
| `test` | When `helm test` is run |

Hook weight (`helm.sh/hook-weight`) controls order when multiple hooks share the same point. Hook delete policy (`helm.sh/hook-delete-policy`) controls cleanup - `hook-succeeded` removes the resource after a successful run, keeping the namespace clean.

Common use cases: database migrations (`pre-upgrade`), post-install smoke tests (`post-install`), and connection tests (`test`).

### Testing

The `templates/tests/test-connection.yaml` file defines a Pod annotated as a `test` hook. Running `helm test <release>` creates the Pod, waits for it to complete, and reports pass/fail based on exit code.

Other validation tools used during development:

- `helm lint` - checks chart structure and template syntax without deploying.
- `helm template` - renders templates to stdout for inspection, without touching the cluster.
- `helm install --dry-run` - sends rendered manifests to the API server for validation without persisting them.

### Rollbacks

Every `helm install` and `helm upgrade` creates a numbered revision stored as a Secret in the release namespace. Rolling back re-applies the full set of manifests from a previous revision:

```bash
helm history metrics-app -n metrics-app-prod
helm rollback metrics-app 3 -n metrics-app-prod --wait
```

In the CI/CD pipeline, automated rollback is triggered if `helm upgrade --wait` does not complete within the timeout - the pipeline catches the failure and can invoke `helm rollback` before alerting.

---

## GitOps Flow

The overall flow for any change in this project:

```
Developer
  │
  ▼
Commit to main
  │
  ├─ Source code changed?
  │     │
  │     ▼
  │   Jenkins (polls main every 2 min)
  │     │  Build image (Kaniko)
  │     │  Run tests (Dockerfile + Trivy scan)
  │     │  Push to in-cluster registry
  │     │
  │     ├─ dev build -> update helm/env/dev.yaml on gitops branch
  │     └─ release tag (e.g. v1.0.3) ────────────────────────────────────────────┐
  │                                                                              │
  │                                  Image Updater detects new semver tag        │
  │                                  in registry -> writes helm/env/staging.yaml │
  │                                  on gitops -> ArgoCD deploys to staging      │
  │                                                                              │
  │                                  Meanwhile: Jenkins waits up to 1 hour       │
  │                                  for manual approval                         │
  │                                       │                                      │
  │                                       ▼ (approved)                           │
  │                                  update helm/env/prod.yaml on gitops ◄───────┘
  │                                  -> ArgoCD deploys to prod
  │
  └─ Helm chart or override changed?
        │
        ▼
      Jenkins
        │  Sync helm/charts/ and helm/overrides/ to gitops branch
        │
        ▼
      ArgoCD (watches gitops branch)
        │  Detects change
        │  Renders chart with updated values
        │  Applies diff to cluster
        ▼
      Cluster updated
```

The `gitops` branch always reflects exactly what is running in the cluster. Git is the only path to making a change - there is no direct `helm upgrade` or `kubectl apply` for ArgoCD-managed applications.

---

## ArgoCD

### Architecture

ArgoCD runs in the `argocd` namespace and is composed of several components:

**API Server**: the central gateway. Serves the REST/gRPC API used by the UI and CLI. Handles authentication (local users and OIDC), enforces RBAC, and receives Git webhooks that can trigger immediate reconciliation.

**Repository Server**: a stateless service that clones or fetches from Git and renders manifests using the appropriate tool (Helm, Kustomize, plain YAML). It returns a flat list of Kubernetes resource objects. It has no cluster access of its own.

**Application Controller**: the reconciliation engine. Watches the desired state (from the Repo Server) and the live cluster state simultaneously. Computes the diff and, when auto-sync is enabled, applies it. Updates sync and health status on every loop.

**Web UI**: a React SPA served by the API Server. Used for monitoring application status, viewing resource trees, inspecting diffs, triggering syncs, and managing rollbacks.

**Redis**: caches rendered manifests and cluster state between reconciliation loops.

### Application CRD

Each managed workload is described by an `Application` resource in `manifests/argocd/`. For example, the dev environment application:

```yaml
# manifests/argocd/metrics-app-dev-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metrics-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitea.url/username/repo.git
    targetRevision: gitops
    path: helm/charts/metrics-app
    helm:
      valueFiles:
        - values.yaml
        - ../../../helm/env/dev.yaml
        - ../../../helm/overrides/metrics-app-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: metrics-app-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ApplyOutOfSyncOnly=true
      - CreateNamespace=true
```

`targetRevision: gitops` means ArgoCD tracks the `gitops` branch exclusively. When Jenkins commits an updated image tag to `helm/env/dev.yaml` on `gitops`, ArgoCD detects the change and re-renders and applies the chart with the new tag.

`prune: true`: resources deleted from Git are also deleted from the cluster.  
`selfHeal: true`: manual changes to cluster resources are automatically reverted on the next reconciliation.  
`ApplyOutOfSyncOnly`: only changed resources are re-applied, not the entire chart on every sync.

### Sync Behaviour and Drift Detection

The Application Controller reconciles on a polling interval (default 3 minutes) and also immediately upon receiving a Git webhook. On each reconciliation:

1. Repo Server renders the chart at the current `gitops` HEAD.
2. Application Controller compares rendered manifests to live cluster state.
3. If the application is `OutOfSync` and `selfHeal` is enabled, the diff is applied.
4. Sync and health status are updated and visible in the UI.

**Drift** occurs when the live cluster state diverges from the desired state in Git - for example, if someone runs `kubectl edit` directly, or if a pod is manually deleted and comes back with different state. With `selfHeal: true`, ArgoCD corrects this automatically within the next reconciliation cycle. This enforces the GitOps invariant: the only lasting way to change a managed application is through a Git commit.

### What ArgoCD Manages (and What It Does Not)

ArgoCD manages the following applications, each with its own `Application` resource in `manifests/argocd/`:

| Application | Namespace | Config source |
|---|---|---|
| `metrics-app-dev` | `metrics-app-dev` | `helm/env/dev.yaml` + `helm/overrides/metrics-app-dev.yaml` |
| `metrics-app-staging` | `metrics-app-staging` | `helm/env/staging.yaml` + `helm/overrides/metrics-app-staging.yaml` |
| `metrics-app-prod` | `metrics-app-prod` | `helm/env/prod.yaml` + `helm/overrides/metrics-app-prod.yaml` |
| `postgresql` | `database` | `helm/overrides/postgresql-values.yaml` |
| `vault` | `vault` | `helm/overrides/vault-values.yaml` |
| `vault-config` | `vault` | `manifests/vault/` |
| `jenkinsci` | `jenkinsci` | `helm/overrides/jenkinsci-values.yaml` |
| `argocd-image-updater` | `argocd` | `helm/overrides/argocd-image-updater-values.yaml` |

**Not managed by ArgoCD:**

- **ArgoCD itself**: a circular dependency: if ArgoCD manages its own Application, a broken ArgoCD upgrade could take down the tool responsible for recovering it. ArgoCD is installed and upgraded separately.
- **cert-manager**: bootstrapped once; not subject to ongoing version changes.
- **In-cluster registry**: stateless, set up once from `manifests/registry/`. No ongoing lifecycle management needed.
- **Namespaces**: applied once from `manifests/namespaces/` as part of cluster bootstrap.

For all ArgoCD-managed applications, the correct way to make a change (e.g. updating Jenkins configuration) is to modify the corresponding override file in `helm/overrides/`, commit it to `main`, and let Jenkins sync it to `gitops`. Running `helm upgrade` directly against these releases will conflict with ArgoCD's desired state and will be reverted on the next reconciliation.

### ArgoCD Image Updater

Image Updater is responsible for automatically promoting any new release to the staging environment. It polls the in-cluster registry and, when a new tag matching the configured pattern appears, commits an updated image reference to `helm/env/staging.yaml` on the `gitops` branch. ArgoCD then detects the commit and syncs staging.

This uses the **Git write-back** method, which preserves the GitOps property: the `gitops` branch always reflects what is deployed, and every automated update is auditable as a Git commit.

Image Updater is configured for the `metrics-app-staging` application only. It watches for any clean semver tag (`^[0-9]+\.[0-9]+\.[0-9]+$`) and ignores `latest` and dev-hash tags. This means all properly tagged patch releases are automatically deployed to staging.

**The promotion model:**

| Step | Who | Mechanism |
|------|-----|-----------|
| Dev deploy | Jenkins (automatic) | Writes `dev-<hash>` to `helm/env/dev.yaml` |
| Staging deploy | Image Updater (automatic) | Detects new semver tag, writes to `helm/env/staging.yaml` |
| Prod deploy | Jenkins + admin approval | Writes semver tag to `helm/env/prod.yaml` after 1h gate |

The staging step provides a validation window: the release runs in staging while Jenkins is waiting at the approval gate. If something looks wrong in staging, the admin simply does not approve and the production environment is not updated.

Major and minor version updates are filtered out at the staging level. Only security patches are applied automatically - these are low-risk, backward-compatible changes. Major and minor versions represent significant application changes that require more thorough testing before staging exposure. To deploy a new major or minor version to staging, the `allowTags` regex in `manifests/argocd/image-updater.yaml` must be updated manually to target the new version line (e.g. `^[1]+\.[1]+\.[0-9]+$` for `1.1.x`).

Image Updater uses a dedicated local ArgoCD user (`image-updater`) with a minimal RBAC role - read and update access to applications only.

### UI and CLI

Both provide access to the same API. The UI is used for monitoring and ad-hoc operations; the CLI is used in scripts and the pipeline.

```bash
# Check application status
argocd app get metrics-app-prod

# Show diff between desired (Git) and live (cluster) state
argocd app diff metrics-app-prod

# Manually trigger sync and wait for completion
argocd app sync metrics-app-prod --wait

# View revision history
argocd app history metrics-app-prod

# Roll back to a specific revision
argocd app rollback metrics-app-prod <revision>
```

The `--wait` flag makes a sync a blocking operation with a clear success/failure result, which is used in the pipeline to gate promotion between environments.

---

## CI/CD Pipeline (Jenkins)

Jenkins runs in the `jenkinsci` namespace, managed by ArgoCD via the `jenkinsci` Application.

**Agent model.** Each build runs in an ephemeral Kubernetes Pod with three containers:
- `kaniko`: builds Docker images without requiring a Docker daemon or privileged mode.
- `kubectl`: runs Git operations and cluster queries (e.g. discovering the registry's ClusterIP).
- `trivy`: scans built images for vulnerabilities.

The Pod uses the `jenkins` service account (RBAC defined in `manifests/jenkins/rbac.yaml`) and mounts an SSH key secret (`jenkins-gitea-ssh`) for Git write access to the `gitops` branch.

**Trigger.** Jenkins polls the `main` branch every 2 minutes.

**Tag computation.** The build behaviour depends on whether the triggering commit is a Git tag:

- **Git tag** (e.g. `v1.0.3`) -> `IS_RELEASE=true`, `IMAGE_TAG=1.0.3`. Triggers the production promotion path.
- **Regular commit** -> `IS_RELEASE=false`, `IMAGE_TAG=dev-<short-hash>`. Triggers the dev deployment path.

**Pipeline stages:**

1. **Checkout**: checks out `main`.
2. **Setup**: resolves the in-cluster registry's ClusterIP dynamically via `kubectl get svc`.
3. **Compute tag**: sets `IMAGE_TAG` and `IS_RELEASE` based on whether a Git tag is present.
4. **Sync Helm Charts**: if `helm/charts/**` or `helm/overrides/**` changed on `main`, checks out `gitops` and syncs those paths. This is how infrastructure configuration changes reach the cluster.
5. **Build Backend / Frontend**: Kaniko builds the image and pushes to the in-cluster registry. Runs when source code changed, or on a release tag.
6. **Scan Backend / Frontend**: Trivy scans the freshly built image for HIGH and CRITICAL vulnerabilities. On `main`, the scan exits non-zero on any finding and blocks the pipeline. On other branches the scan is informational only.
7. **Deploy to Dev**: on a non-release build, updates `IMAGE_TAG` in `helm/env/dev.yaml` on the `gitops` branch. ArgoCD picks up the change and deploys to the `metrics-app-dev` namespace.
8. **Deploy to Prod**: on a release build, pauses for manual approval (1-hour timeout). After approval, updates `helm/env/prod.yaml` on `gitops`. ArgoCD syncs the new version to `metrics-app-prod`.

**Rollback.** Because `helm/env/prod.yaml` on `gitops` is the source of truth, rolling back production means either reverting that commit in Git, or using ArgoCD's application history to sync a previous revision.

---

## Security

### Role-Based Access Control

RBAC governs what identities are permitted to do in both Kubernetes and ArgoCD.

**Kubernetes RBAC** follows a default-deny model: nothing is permitted unless an explicit rule grants it. Permissions are composed of a `Role` or `ClusterRole` (listing allowed verbs on resources) and a `RoleBinding` or `ClusterRoleBinding` (attaching the role to a subject - user, group, or service account).

Two custom RBAC configurations exist in this project:

- `manifests/jenkins/rbac.yaml` - grants the `jenkins` service account permission to read services (to discover the registry IP) and manage pods in its own namespace.
- `manifests/vault/rbac.yaml` - allows the External Secrets Operator to authenticate with Vault and read secrets on behalf of workloads in target namespaces.

**ArgoCD RBAC** controls which users can perform which operations on ArgoCD resources (applications, repositories, clusters). Configured in the `argocd-rbac-cm` ConfigMap:

```
policy.default: role:readonly
policy.csv: |
  p, role:image-updater, applications, get, */*, allow
  p, role:image-updater, applications, update, */*, allow
  g, image-updater, role:image-updater
```

The `image-updater` account receives only what it needs: read and update access to applications. It cannot modify repositories, clusters, or any other ArgoCD resource.

The **principle of least privilege** applies throughout: each service account is granted only the permissions needed for its specific function, scoped to the relevant namespaces rather than granted cluster-wide where avoidable.

### HashiCorp Vault and Static Secrets

Kubernetes `Secret` resources store values as base64-encoded plaintext and cannot safely be committed to Git. This conflicts with the GitOps model of keeping everything in version control.

**HashiCorp Vault** stores all secret material - database credentials, registry credentials, Jenkins secrets, ArgoCD tokens, and application API keys. It is deployed to the cluster via ArgoCD and configured by the `vault-config` Application.

The **Vault Secrets Operator (VSO)** bridges Vault and Kubernetes. It watches `StaticSecret` resources - which contain only a *reference* to a secret path in Vault, not the value - fetches the actual value from Vault, and creates a native Kubernetes `Secret`. The `StaticSecret` manifest is safe to commit to Git.

```
Vault (secret store)
  ▲
  │ fetch (VSO)
  │
VaultStaticSecret manifest  <- safe to commit to Git
  │
  ▼
Kubernetes Secret  <- created by VSO, exists only in cluster
  │
  ▼
Pod (consumes as env var or volume mount)
```

The Vault connection and authentication configuration live in `manifests/vault/`. A working `vault-connection.yaml` and `vault-auth.yaml` are prerequisites for any application that relies on secrets.

**Setup order matters.** Vault must be initialised and all required secrets populated before any dependent application is deployed. This is why the main README begins with Vault initialisation - Jenkins, ArgoCD Image Updater, and the PostgreSQL all depend on secrets being present at deploy time.

---

## Multi-Environment Setup

Three environments run in isolated Kubernetes namespaces: `metrics-app-dev`, `metrics-app-staging`, and `metrics-app-prod`.

**Namespace isolation** provides independent DNS, RBAC, and resource quotas per environment. A broken deployment in `dev` cannot affect `prod` resources.

**Shared chart, separate values.** All three environments use the same `helm/charts/metrics-app` chart. Differences are expressed through layered values files:

```
helm/charts/metrics-app/values.yaml     <- chart defaults
helm/env/dev.yaml                        <- dev image tags (updated by Jenkins on every build)
helm/env/staging.yaml                    <- staging image tags
helm/env/prod.yaml                       <- prod image tags (updated by Jenkins on release)
helm/overrides/metrics-app-dev.yaml     <- dev-specific config (replicas, resource limits, etc.)
helm/overrides/metrics-app-prod.yaml    <- prod-specific config
```

Each ArgoCD `Application` in `manifests/argocd/` specifies which combination of these files applies to its environment.

**Promotion flow:**
 
- Regular commits to `main` -> Jenkins updates `helm/env/dev.yaml` on `gitops` -> ArgoCD deploys to dev automatically.
- A Git tag (e.g. `v1.0.1`) -> Jenkins builds and scans the image, pushes it to the registry -> Image Updater detects it matches the active `allowTags` pattern and updates `helm/env/staging.yaml` on `gitops` -> ArgoCD deploys to staging automatically.
- While staging is running, Jenkins is waiting at a 1-hour approval gate. If an admin approves, Jenkins updates `helm/env/prod.yaml` on `gitops` -> ArgoCD promotes to production.
 
Major and minor releases (e.g. `v1.1.0`, `v2.0.0`) do not match the current `allowTags` regex and are not automatically deployed to staging. To promote a new major or minor version, the `allowTags` pattern in `manifests/argocd/image-updater.yaml` must first be updated to target the new version line, after which the normal patch automation resumes for that line.