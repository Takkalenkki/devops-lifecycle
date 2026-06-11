# Gitops Galaxy

A full GitOps deployment pipeline for a multi-component web application, built on Kubernetes (Minikube / KVM2). Application deployments are managed by ArgoCD, infrastructure is packaged with Helm, secrets are stored in HashiCorp Vault via the Vault Secrets Operator, and image promotion is automated by ArgoCD Image Updater. A Jenkins CI/CD pipeline ties everything together from source commit to production.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Summary](#architecture-summary)
3. [Prerequisites](#prerequisites)
4. [Repository Layout](#repository-layout)
5. [Setup and Installation](#setup-and-installation)
   - [1. Start Minikube](#1-start-minikube)
   - [2. Apply Namespaces](#2-apply-namespaces)
   - [3. Deploy the In-Cluster Registry](#3-deploy-the-in-cluster-registry)
   - [4. Install and Initialise Vault](#4-install-and-initialise-vault)
   - [5. Configure Vault Secrets](#5-configure-vault-secrets)
   - [6. Install Core Infrastructure via Helm](#6-install-core-infrastructure-via-helm)
   - [7. Apply ArgoCD Applications](#7-apply-argocd-applications)
   - [8. Apply Remaining Manifests](#8-apply-remaining-manifests)
   - [9. Verify Jenkins](#9-verify-jenkins)
6. [Usage Guide](#usage-guide)
   - [Accessing the UIs](#accessing-the-uis)
   - [Deploying the Application](#deploying-the-application)
   - [Promoting to Staging and Production](#promoting-to-staging-and-production)
   - [Helm Operations](#helm-operations)
   - [ArgoCD CLI Operations](#argocd-cli-operations)
   - [Testing the Database](#testing-the-database)
   - [Rolling Back](#rolling-back)
7. [Additional Features](#additional-features)
   - [HashiCorp Vault - External Secret Management](#hashicorp-vault--external-secret-management)
   - [ArgoCD Image Updater](#argocd-image-updater)
   - [Multi-Environment Setup](#multi-environment-setup)
   - [Automated Vulnerability Scanning](#automated-vulnerability-scanning)
   - [Horizontal Pod Autoscaler](#horizontal-pod-autoscaler)
8. [Additional Notes](#additional-notes)

---

## Project Overview

This project implements a production-style GitOps workflow for a metrics application (Go backend + React/Vite frontend) and its PostgreSQL database. The goals are:

- **Reproducibility** - every deployed state is described by files in Git; no manual `kubectl apply` for managed resources.
- **Separation of concerns** - application source code lives on `main`; deployed state lives on `gitops`. ArgoCD watches `gitops` exclusively.
- **Automated promotion** - commits to `main` trigger Jenkins, which builds, scans, and deploys to dev automatically. Tagged releases flow to staging automatically via Image Updater and to production after a manual approval gate.
- **Security by default** - secrets are stored in Vault and injected into Kubernetes only at runtime via the Vault Secrets Operator. No sensitive values are committed to Git.

---

## Architecture Summary

```
Developer
  │
  └─► Commit to main ──► Jenkins (polls every 2 min)
                              │
                    ┌─────────┴──────────┐
                    │                    │
               Regular commit       Git tag (vX.Y.Z)
                    │                    │
              Build + Scan          Build + Scan
                    │                    │
           Update dev.yaml        Push semver image
           on gitops branch             │
                    │             Image Updater detects tag
                    │             Updates staging.yaml ──────► ArgoCD deploys staging
                    │                    │
                    │             Jenkins waits (1h gate)
                    │             Admin approves
                    │                    │
                    │             Update prod.yaml ──────────► ArgoCD deploys prod
                    │
                    ▼
          ArgoCD detects gitops change
          Renders Helm chart
          Applies diff to cluster
```

**Branch model:**
- `main` - application source code, Helm chart definitions, Jenkinsfile.
- `gitops` - deployed state only. Managed by Jenkins and Image Updater. ArgoCD watches this branch.

Take a look at the [Architecture and Concepts document](/06-gitops-galaxy/documents/architecture-and-concepts.md) for more details.

---

## Prerequisites

### Required Tools

| Tool | Purpose | Install |
|---|---|---|
| `minikube` | Local Kubernetes cluster | [minikube.sigs.k8s.io](https://minikube.sigs.k8s.io) |
| `kubectl` | Kubernetes CLI | [kubernetes.io/docs/tasks/tools](https://kubernetes.io/docs/tasks/tools) |
| `helm` | Kubernetes package manager | [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install) |
| `docker` | Container runtime (local builds) | [docs.docker.com/get-docker](https://docs.docker.com/get-docker) |
| `argocd` | ArgoCD CLI | [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/en/stable/cli_installation/) |
| `vault` | Vault CLI | [developer.hashicorp.com/vault](https://developer.hashicorp.com/vault/downloads) |
| `qemu-full` + `libvirt` | KVM2 virtualisation | Available in community managed repos |

These can be installed using your distributions package manager.

For example, on Arch-based systems:

```bash
sudo pacman -S minikube kubectl helm docker qemu-full libvirt
# Enable libvirt daemon
sudo systemctl enable --now libvirtd
# Add yourself to the libvirt group
sudo usermod -aG libvirt $USER
# Re-login or run: newgrp libvirt
```

Install the ArgoCD CLI and Vault CLI from their respective upstream releases or via the AUR.

### Helm Repositories

Add the following Helm repositories before proceeding:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo add jenkins https://charts.jenkins.io
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## Repository Layout

```
.
├── helm/
│   ├── charts/metrics-app/         # Custom Helm chart (backend + frontend)
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── backend/            # Deployment, Service
│   │       ├── frontend/           # Deployment, Service, ConfigMap, Ingress, HPA
│   │       ├── _helpers.tpl        # Shared label and name templates
│   │       ├── NOTES.txt
│   │       └── tests/              # Helm test hook (test-connection.yaml)
│   ├── env/                        # Per-environment image tag files
│   │   ├── dev.yaml
│   │   ├── staging.yaml
│   │   └── prod.yaml
│   └── overrides/                  # Values overrides for infrastructure charts
│       ├── argocd-values.yaml
│       ├── argocd-image-updater-values.yaml
│       ├── jenkinsci-values.yaml
│       ├── metrics-app-{dev,staging,prod}.yaml
│       ├── postgresql-values.yaml
│       └── vault-values.yaml
├── manifests/
│   ├── argocd/                     # ArgoCD Application CRDs
│   ├── namespaces/                 # All namespace definitions
│   ├── database/                   # PostgreSQL connection test Job
│   ├── vault/                      # Vault RBAC, auth, secret store config
│   ├── jenkins/                    # Jenkins RBAC
│   ├── registry/                   # In-cluster image registry
│   ├── ingress/                    # Ingress resources for ArgoCD and Jenkins
│   └── cert-manager/               # Self-signed certificate issuer
├── source-code/metrics-app/
│   ├── backend/                    # Go service
│   └── frontend/                   # React/Vite + Nginx
├── secrets/                        # Local secrets (NOT committed to Git)
│   ├── argocd-image-updater-token
│   ├── gitea_ci_jenkins_key
│   ├── local-ca.crt
│   └── vault-init-keys.json
├── Jenkinsfile
└── README.md
```

> **Note:** The `secrets/` directory is listed in `.gitignore` and must never be committed. It holds locally generated credentials used during bootstrap.

---

## Setup and Installation

> **Order matters.** Vault must be initialised and secrets populated before any application that depends on them is deployed. Follow the steps below in sequence.

### 1. Start Minikube

```bash
minikube start \
  --memory 12288 \
  --cpus 4 \
  --kubernetes-version=v1.35.1 \
  --insecure-registry="10.0.0.0/8,192.168.0.0/16" \
  --driver kvm2
```

Enable the ingress addon:

```bash
minikube addons enable ingress
```

Verify the cluster is healthy:

```bash
kubectl get nodes
kubectl get pods -A
```

### 2. Apply Namespaces

All namespaces are defined in `manifests/namespaces/`. Apply them first so subsequent resources have a target namespace:

```bash
kubectl apply -f manifests/namespaces/
```

This creates: `argocd`, `cert-manager`, `database`, `jenkinsci`, `metrics-app-dev`, `metrics-app-staging`, `metrics-app-prod`, `registry`, and `vault`.

### 3. Deploy the In-Cluster Registry

The in-cluster registry stores application images built by Kaniko in Jenkins. It is deployed from plain manifests (not managed by ArgoCD):

```bash
kubectl apply -f manifests/registry/
```

Retrieve the registry's ClusterIP - you will need this when configuring Jenkins:

```bash
kubectl get svc registry -n registry -o jsonpath='{.spec.clusterIP}'
```

### 4. Install and Initialise Vault

Install Vault using the override values from this repository:

```bash
helm install vault hashicorp/vault \
  -n vault \
  -f helm/overrides/vault-values.yaml \
  --wait
```

Wait for the Vault pod to be running (it will show `0/1 Ready` until initialised):

```bash
kubectl get pods -n vault -w
```

**Initialise Vault** (one-time operation - store the output securely):

```bash
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=1 \
  -key-threshold=1 \
  -format=json > secrets/vault-init-keys.json
```

**Unseal Vault** using the single generated unseal key:

```bash
UNSEAL_KEY=$(jq -r '.unseal_keys_b64[0]' secrets/vault-init-keys.json)
kubectl exec -n vault vault-0 -- vault operator unseal "$UNSEAL_KEY"
```

Verify Vault is unsealed and active:

```bash
kubectl exec -n vault vault-0 -- vault status
```

Apply Vault RBAC and configure the Kubernetes auth method:

```bash
kubectl apply -f manifests/vault/rbac.yaml
kubectl apply -f manifests/vault/vault-connection.yaml
kubectl apply -f manifests/vault/vault-auth.yaml
```

### 5. Configure Vault Secrets

Log in with the root token from the initialisation output:

```bash
ROOT_TOKEN=$(jq -r '.root_token' secrets/vault-init-keys.json)
kubectl exec -n vault vault-0 -- vault login "$ROOT_TOKEN"
```

Enable the KV secrets engine and populate all required secrets. Replace the placeholder values with your actual credentials:

```bash
# Enable KV v2
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2

# PostgreSQL credentials (consumed by the database namespace)
kubectl exec -n vault vault-0 -- vault kv put secret/database/credentials \
  user-password="<db-user-password>" \
  postgres-password="<postgres-admin-password>"

# Jenkins admin credentials (consumed by the jenkinsci namespace)
kubectl exec -n vault vault-0 -- vault kv put secret/jenkins/admin-credentials \
  jenkins-admin-user="admin" \
  jenkins-admin-password="<jenkins-admin-password>"

# Jenkins SSH key for Gitea access (consumed by the jenkinsci namespace)
kubectl exec -n vault vault-0 -- vault kv put secret/jenkins/gitea-ssh \
  ssh-privatekey="$(cat secrets/gitea_ci_jenkins_key)"

# Gitea repository credentials for ArgoCD (consumed by the argocd namespace)
kubectl exec -n vault vault-0 -- vault kv put secret/argocd/gitea-repo \
  sshPrivatekey="$(cat secrets/gitea_ci_jenkins_key)" \
  type="git" \
  url="<ssh://git@gitea.url/username/repo.git>"

# ArgoCD Image Updater API token (consumed by the argocd namespace)
# Generated in step 6 - return here to fill this in after ArgoCD is installed
kubectl exec -n vault vault-0 -- vault kv put secret/argocd/image-updater-token \
  argocd.token="<argocd-image-updater-token>"
```

> **Note:** The exact Vault KV paths must match what is configured in `manifests/vault/static-secret.yaml`. Check that file to confirm path names before writing secrets.

Apply the static secret manifest (Vault policies and role bindings for the Vault Secrets Operator):

```bash
kubectl apply -f manifests/vault/static-secret.yaml
```

The following `VaultStaticSecret` resources will be created and kept in sync with Vault by the Vault Secrets Operator:

| VaultStaticSecret | Namespace | Purpose |
|---|---|---|
| `database-credentials` | `database` | PostgreSQL user password and admin password |
| `jenkins-admin-credentials` | `jenkinsci` | Jenkins admin username and password |
| `jenkins-gitea-ssh` | `jenkinsci` | SSH private key for Gitea pushes |
| `gitea-repo` | `argocd` | SSH private key and URL for ArgoCD repository access |
| `argocd-image-updater-token` | `argocd` | ArgoCD API token for Image Updater |

### 6. Install Core Infrastructure via Helm

Install all infrastructure components. These are installed directly with Helm (not via ArgoCD) to avoid circular bootstrap dependencies:

**cert-manager:**

```bash
helm install cert-manager jetstack/cert-manager \
  -n cert-manager \
  --set installCRDs=true \
  --wait

kubectl apply -f manifests/cert-manager/selfsigned-issuer.yaml
```

**ArgoCD:**

```bash
helm install argocd argo/argo-cd \
  -n argocd \
  -f helm/overrides/argocd-values.yaml \
  --wait
```

Retrieve the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

Log in via the CLI (the ArgoCD server will be accessible via Ingress - see [Accessing the UIs](#accessing-the-uis)):

```bash
argocd login <argocd-hostname> --username admin --password <password> --grpc-web --insecure
```

**Generate the ArgoCD Image Updater token:**

```bash
argocd account generate-token --account image-updater > secrets/argocd-image-updater-token
```

Now go back to step 5 and populate the `secret/argocd/image-updater-token` path in Vault with the contents of `secrets/argocd-image-updater-token`.

### 7. Apply ArgoCD Applications

With ArgoCD running, register all managed applications. ArgoCD will immediately begin reconciling them against the `gitops` branch:

```bash
kubectl apply -f manifests/argocd/
```

This registers the following applications. ArgoCD syncs them automatically - no further Helm installs are required:

| Application | Namespace | Notes |
|---|---|---|
| `metrics-app-dev` | `metrics-app-dev` | Deployed on first Jenkins build |
| `metrics-app-staging` | `metrics-app-staging` | Deployed on first semver tag |
| `metrics-app-prod` | `metrics-app-prod` | Deployed after manual approval |
| `postgresql` | `database` | Auto-synced immediately |
| `vault` | `vault` | Auto-synced immediately |
| `vault-config` | `vault` | Vault policies and auth config |
| `jenkinsci` | `jenkinsci` | Auto-synced; fully configured via JCasC |
| `argocd-image-updater` | `argocd` | Auto-synced immediately |

Check sync status:

```bash
argocd app list
```

### 8. Apply Remaining Manifests

Apply ingress rules for ArgoCD and Jenkins:

```bash
kubectl apply -f manifests/ingress/
```

### 9. Verify Jenkins

Jenkins is deployed and fully configured by ArgoCD via the `jenkinsci` Application. All setup - plugins, Kubernetes cloud, and credentials - is applied automatically through Jenkins Configuration as Code (JCasC) defined in `helm/overrides/jenkinsci-values.yaml`. No manual UI configuration is required.

**Admin credentials** are sourced from the `jenkins-admin-credentials` Vault secret, synced into the `jenkinsci` namespace as a Kubernetes Secret by the Vault Secrets Operator.

**The Gitea SSH key** for Git write access is mounted from the `jenkins-gitea-ssh` Vault secret and referenced by JCasC as the `gitea-ssh` credential.

Wait for Jenkins to finish starting:

```bash
kubectl get pods -n jenkinsci -w
```

Retrieve the admin password if you need to log into the UI:

```bash
kubectl get secret jenkins-admin-credentials -n jenkinsci \
  -o jsonpath='{.data.jenkins-admin-password}' | base64 -d && echo
```

Once Jenkins is up, navigate to the UI and verify the Kubernetes cloud is connected. The first build will trigger automatically on the next poll of the `main` branch (within two minutes) after the pipeline job is created.

#### Creating the Pipeline Job

1. Create a new **Multibranch Pipeline** job.
2. Add **Branch Source**:
   - **Project Repository**: `git@gitea.url:username/repo.git`
   - **Credentials**: Git
   - **Behaviors**: Discover branches, Discover tags
   - **Property strategy**: All branches get the same properties
   - **Build strategies**: Ignore tags older than: 7
3. **Build Configuration**:
   - **Mode**: by Jenkinsfile
   - **Script Path**: `Jenkinsfile`
4. **Scan Multibranch Pipeline Triggers**:
   - **Interval**: 2 minutes

---

## Usage Guide

### Accessing the UIs

All UIs are exposed via Minikube's ingress. Add the following entries to `/etc/hosts`, substituting your Minikube IP:

```bash
# Get Minikube IP
minikube ip
```

```
# /etc/hosts
<minikube-ip>  metrics.lan
<minikube-ip>  metrics-dev.lan
<minikube-ip>  metrics-staging.lan
<minikube-ip>  jenkins.lan
<minikube-ip>  argocd.lan
<minikube-ip>  vault.lan
```

| Service | URL | Default credentials |
|---|---|---|
| ArgoCD | `https://argocd.lan` | `admin` / see step 6 |
| Jenkins | `https://jenkins.lan` | `admin` / from `jenkins-admin-credentials` Vault secret |

The CA certificate for the self-signed TLS issuer is saved at `secrets/local-ca.crt`. Import it into your browser or system trust store to avoid certificate warnings:

```bash
# Arch / systemd-based systems
sudo trust anchor --store secrets/local-ca.crt
```

### Deploying the Application

Any commit pushed to `main` triggers the Jenkins pipeline within two minutes (polling interval). The pipeline:

1. Builds Docker images with Kaniko (no Docker daemon required in-cluster).
2. Scans images with Trivy for HIGH and CRITICAL CVEs - any finding blocks the pipeline.
3. Pushes the image to the in-cluster registry tagged as `dev-<git-short-hash>`.
4. Commits the new tag to `helm/env/dev.yaml` on the `gitops` branch.
5. ArgoCD detects the commit and deploys to `metrics-app-dev`.

To watch the ArgoCD sync in real time:

```bash
argocd app get metrics-app-dev --watch
```

### Promoting to Staging and Production

**Staging** is promoted automatically. Create and push a semantic version tag on `main`:

```bash
git tag v1.0.3
git push origin v1.0.3
```

Jenkins builds the release image and pushes it with the tag `1.0.3`. ArgoCD Image Updater detects the new tag (matching `^[1]+\.[0]+\.[0-9]+$` = `1.0.x`), commits the updated tag to `helm/env/staging.yaml` on `gitops`, and ArgoCD deploys to `metrics-app-staging` automatically.

**Production** requires manual approval. While the image is running in staging, Jenkins is paused at a one-hour approval gate. Log in to the Jenkins UI, review the staging deployment, and click **Approve** on the waiting pipeline step. Jenkins then commits the tag to `helm/env/prod.yaml` on `gitops` and ArgoCD deploys to `metrics-app-prod`.

> **Skipping the gate:** if no approval is given within one hour, Jenkins aborts the pipeline. The staging deployment remains unchanged; production is unaffected.

> **Major and minor versions** (e.g. `v1.1.0`, `v2.0.0`) are not automatically deployed to staging because they do not match the current Image Updater `allowTags` pattern. To enable auto-staging for a new version line, update `manifests/argocd/image-updater.yaml` to target the new pattern before tagging.

### Helm Operations

**Lint the chart** (checks syntax, no cluster required):

```bash
helm lint helm/charts/metrics-app
```

**Preview rendered manifests** (no cluster changes):

```bash
helm template metrics-app helm/charts/metrics-app \
  -f helm/charts/metrics-app/values.yaml \
  -f helm/env/dev.yaml \
  -f helm/overrides/metrics-app-dev.yaml
```

**Dry-run against the API server** (validates schema without deploying):

```bash
helm install metrics-app-dev helm/charts/metrics-app \
  -n metrics-app-dev \
  -f helm/env/dev.yaml \
  -f helm/overrides/metrics-app-dev.yaml \
  --dry-run
```

**View release history:**

```bash
helm history metrics-app-dev -n metrics-app-dev
```

**Run Helm tests** (connection smoke test):

```bash
helm test metrics-app-dev -n metrics-app-dev
```

### ArgoCD CLI Operations

```bash
# List all applications and their sync status
argocd app list

# Inspect a specific application
argocd app get metrics-app-prod

# Show the diff between desired (Git) state and live cluster state
argocd app diff metrics-app-prod

# Manually trigger a sync and wait for completion
argocd app sync metrics-app-prod --wait

# View the revision history of an application
argocd app history metrics-app-prod

# Roll back to a specific revision
argocd app rollback metrics-app-prod <revision-number>
```

### Testing the Database

A Kubernetes Job is provided to verify the PostgreSQL deployment is operational. It connects to the database, runs a test query, and exits:

```bash
kubectl apply -f manifests/database/job.yaml
kubectl logs -n database job/postgres-connection-test
```

Clean up after testing:

```bash
kubectl delete -f manifests/database/job.yaml
```

### Rolling Back

**Via Helm** (for releases not managed by ArgoCD, or for manual override):

```bash
helm rollback argocd <revision> -n argocd --wait
```

**Via ArgoCD** (for ArgoCD-managed applications - preferred):

```bash
# View available revisions
argocd app history metrics-app-prod

# Roll back to a specific revision
argocd app rollback metrics-app-prod <revision-number>
```

Rolling back via ArgoCD re-applies the full set of manifests from a previous `gitops` commit. This is the canonical rollback path and keeps Git as the source of truth.

**Via Git** (permanent rollback - updates the gitops branch):

```bash
# Revert the relevant env file to the previous image tag
git checkout gitops
git revert HEAD   # or edit helm/env/prod.yaml manually
git push origin gitops
# ArgoCD detects the new commit and re-syncs automatically
```

---

## Additional Features

### HashiCorp Vault - External Secret Management

Kubernetes `Secret` resources store values as base64 plaintext and cannot be safely committed to Git. This project uses HashiCorp Vault as the secret backend and the **Vault Secrets Operator (VSO)** to bridge Vault and Kubernetes.

The flow is:

```
Vault (secret store, not in Git)
      ▲
      │  fetch (Vault Secrets Operator)
      │
VaultStaticSecret manifest  <- committed to Git (contains only a reference, not the value)
      │
      ▼
Kubernetes Secret  <- created at runtime by VSO, exists only in the cluster
      │
      ▼
Pod  <- consumes secret as an environment variable or volume mount
```

The configuration lives in `manifests/vault/`. The Vault Secrets Operator authenticates to Vault using Kubernetes Service Account tokens via the Kubernetes auth method (configured by `vault-auth.yaml`).

**`VaultStaticSecret` resources managed by the Vault Secrets Operator:**

| Resource name | Namespace | Contents |
|---|---|---|
| `database-credentials` | `database` | PostgreSQL user password and admin password |
| `jenkins-admin-credentials` | `jenkinsci` | Jenkins admin username and password |
| `jenkins-gitea-ssh` | `jenkinsci` | SSH private key for Gitea write access |
| `gitea-repo` | `argocd` | SSH private key and URL for ArgoCD repository access |
| `argocd-image-updater-token` | `argocd` | ArgoCD API token for Image Updater |

All Helm charts that need secrets reference `existingSecret` values pointing to the Kubernetes Secrets created by VSO - no secret values appear in any Helm values file.

### ArgoCD Image Updater

Image Updater watches the in-cluster registry and automatically promotes new patch releases to the staging environment without requiring a Jenkins pipeline run.

**Configuration** (`manifests/argocd/image-updater.yaml`):

- **Method:** Git write-back - updates `helm/env/staging.yaml` on the `gitops` branch.
- **Tag pattern:** `^[1]+\.[0]+\.[0-9]+$` - matches `1.0.x` patch tags only. Ignores `latest`, `dev-*` hashes, and any other non-semver tags.
- **Scope:** `metrics-app-staging` application only.

The Image Updater uses the dedicated `image-updater` ArgoCD local user, which has the minimal RBAC permissions required: read and update on applications only. Its token is stored in Vault and injected at runtime.

To change the tracked version line (e.g. to auto-deploy `1.1.x` to staging), update the `allowTags` annotation in `manifests/argocd/image-updater.yaml` and apply it:

```bash
# Edit the allowTags value, then:
kubectl apply -f manifests/argocd/image-updater.yaml
```

### Multi-Environment Setup

Three environments run in isolated Kubernetes namespaces with independent DNS and RBAC.

| Environment | Namespace | Image source | Promotion trigger |
|---|---|---|---|
| Dev | `metrics-app-dev` | `dev-<hash>` | Every push to `main` |
| Staging | `metrics-app-staging` | `X.Y.Z` semver | Git tag pushed to `main` |
| Production | `metrics-app-prod` | `X.Y.Z` semver | Manual approval in Jenkins |

All three environments share the same Helm chart (`helm/charts/metrics-app`). Differences are expressed via layered values files:

```
helm/charts/metrics-app/values.yaml                    <- chart defaults
helm/env/{dev,staging,prod}.yaml                       <- image tags (written by CI or Image Updater)
helm/overrides/metrics-app-{dev,staging,prod}.yaml     <- env-specific config (replicas, limits, etc.)
```

To inspect the effective values for an environment without deploying:

```bash
helm template metrics-app-staging helm/charts/metrics-app \
  -f helm/charts/metrics-app/values.yaml \
  -f helm/env/staging.yaml \
  -f helm/overrides/metrics-app-staging.yaml
```

### Automated Vulnerability Scanning

Every image built by the Jenkins pipeline is scanned with [Trivy](https://trivy.dev/) before it is pushed to the registry:

- On the `main` branch: any HIGH or CRITICAL CVE causes the pipeline to fail and the image is not pushed or deployed.
- On other branches: the scan is informational and does not block the build.

This ensures that no known high-severity vulnerability reaches any environment without explicit intervention.

### Horizontal Pod Autoscaler

The frontend deployment includes an HPA (`helm/charts/metrics-app/templates/frontend/hpa.yaml`) that scales the number of replicas based on CPU utilisation. The target utilisation threshold and min/max replica counts are configurable per environment via the values files.

To view HPA status:

```bash
kubectl get hpa -n metrics-app-prod
```

## Additional Notes

- This setup uses **QEMU+KVM2** virtualization environment on **Linux**. Other virtualization softwares such as VMware or VirtualBox on Windows/MacOS are not tested and full compatibility can not be guaranteed.
- For security reasons, SSH keys to access the repository will **NOT** be provided. The nature of the project demands that the SSH key has full access to the repository. Therefore providing my _personal SSH key_ would compromise my account. In an actual production environment, each user should generate and use their own key pair for the repository. Access to repositories should be managed through your Git hosting platform's permission system rather than by sharing private keys.
- **Jenkins pod starts misbehaving after restarting the cluster**. Jenkins relies on `init` pod to run first before running the actual Jenkins pod. When the cluster is restarted, Jenkins has already marked the initialization phase as done, which in turn prevents the `init` pod from running. This creates a conflict and Jenkins gets stuck. The previous pod can be deleted using `kubectl delete pod -n jenkinsci jenkins-0`.
- **Vault must be manually unsealed after every restart.** Vault does not persist its unseal state across pod restarts. Any time the `vault-0` pod is restarted - including after `minikube stop` / `minikube start` - Vault will return to a sealed state and all dependent applications will fail to fetch secrets until it is unsealed again. Use the unseal command from step 4 to recover.
    - After the initial setup is done, you can use the provided [`start_minikube.sh`](./start_minikube.sh) script on project root to automatically start Minikube, unseal the Vault, delete secret resources to let Vault recreate them. The script also recycles the Jenkins pod.
- **The `gitops` branch must exist before applying ArgoCD Applications** (step 7). ArgoCD will immediately attempt to reconcile against it and will error if the branch is absent. Ensure the branch is pushed to the remote repository before proceeding.
- **`jq` is required** for the Vault initialisation commands in step 5 but is not listed among the installed tools. This is because it usually comes pre-bundled with Linux installations.
    - This can be checked with `sudo pacman -Ss jq` (or equivalent search on other distributions):
    ```sh
    extra/jq 1.8.1-1 [installed]
    Command-line JSON processor
    ```
    - Install it with `sudo pacman -S jq` (or with your distributions package manager) before running the init steps if it is not present.
- **ArgoCD will revert any direct** `helm upgrade` **or** `kubectl edit` **on managed resources**. The source of truth lives in Git, therefore any change to managed resources **MUST** be committed to Gitea in order for the changes to take effect and persist.

---

## Author

Aki Heiskanen