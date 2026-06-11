# Cluster Chronicles

A local Kubernetes cluster running a full-stack metrics application with CI/CD, monitoring, logging, and alerting - deployed on Minikube.

The application itself is a Go backend that exposes system metrics, consumed by a React frontend. See [`metrics-app/backend`](/05-cluster-chronicles/metrics-app/backend/README.md) and [`metrics-app/frontend`](/05-cluster-chronicles/metrics-app/frontend/README.md) for application-level documentation.

---

## Architecture Overview

| Namespace | Components |
|-----------|------------|
| `apps` | Backend (Go), Frontend (React + Nginx) |
| `ci` | Jenkins |
| `monitoring` | Prometheus, Alertmanager, Grafana |
| `logging` | Elasticsearch, Fluent Bit, Kibana |
| `ingress-nginx` | ingress-nginx controller |
| `cert-manager` | TLS certificate management |

External access to all services is handled through a single ingress-nginx controller. TLS certificates are issued and rotated automatically by cert-manager.

---

## Prerequisites

Install the required tools:

```sh
sudo pacman -S minikube kubectl helm docker qemu-full libvirt
```

Enable and start the `libvirtd` service if it isn't already running:

```sh
sudo systemctl enable --now libvirtd
```

Add your user to the `libvirt` and `docker` groups to avoid running commands as root:

```sh
sudo usermod -aG libvirt,docker $USER
# Log out and back in for group changes to take effect
```

**NOTICE: This setup uses QEMU+KVM2 virtualization environment on Linux. Other virtualization softwares such as VMware or VirtualBox on Windows/MacOS are not tested and full compatibility can not be guaranteed.**

---

## Configuration

### secrets.env

Sensitive values are read from a `secrets.env` file in the project root when running the script with `--env-file secrets.env`. This file is gitignored and must be created manually before running the setup scripts. Use the following as a template:

```sh
# secrets.env.example - copy to secrets.env and fill in values

# Jenkins
JENKINS_ADMIN_PASSWORD=<PASSWORD>

# Elasticsearch
ELASTICSEARCH_PASSWORD=<PASSWORD>

# Path to SSH key file
GITEA_SSH_PATH_TO_FILE=<path/to/file>

# Gitea URL
GITEA_URL=<gitea.url>

# Known hosts file
JENKINS_KNOWN_HOSTS_PATH_TO_FILE=<path/to/file>
```

> If you prefer not to use a file, `scripts/create-secrets.sh` will prompt for each value interactively when `--env-file` flag is not used.

The script creates the following secrets across namespaces:

| **Namespace** | **Secret** | **Contents** |
|---------------|------------|--------------|
| `logging` | `elasticsearch-credentials` | Elasticsearch username and password |
| `monitoring` | `elasticsearch-exporter-credentials` | Elasticsearch username and password |
| `ci` | `jenkins-admin-credentials` | Jenkins admin username and password |
| `ci` | `jenkins-gitea-ssh` | SSH private key for Gitea authentication |
| `ci` | `jenkins-known-hosts` | Pre-scanned `known_hosts` entries for Gitea |

The SSH _private key_ must already exist on the machine before running the script. The `known_hosts` file is generated automatically by the script via `ssh-keyscan` using `GITEA_URL` and written to the path specified by `JENKINS_KNOWN_HOSTS_PATH_TO_FILE`.

The corresponding SSH _public key_ must be added to the Gitea:
* **User**->**Settings**->**SSH/GPG Keys**->**Add SSH Key**
    * Key Name: set to your preferred name, e.g. jenkins_ssh_key
    * Content: `key_name.pub`
    * Add Key

**DISCLAIMER: Never share your private SSH or GPG keys. Each user should generate and use their own key pair. Access to repositories should be managed through your Git hosting platform's permission system rather than by sharing private keys.**

---

## Setup

Run the following steps in order. The scripts in steps 1, 3, and 4 are idempotent and can be re-run safely. Step 2 must be repeated in any new terminal session before building images, as `minikube docker-env` is session-scoped.

### 1. Start Minikube

```sh
bash scripts/start.sh
```

Starts Minikube with the QEMU driver, enables the required addons (ingress, metrics-server, registry), and configures Docker to allow the in-cluster registry as an insecure registry.

### 2. Create Initial App Images

`metrics-app` depends on the Minikube local images. These will need to be created before the setup can begin as the initial deployment relies on the images with the tag `bootstrap` being present. Failure to do so will result in the pods getting stuck in `ImagePullBackOff`.

```sh
eval $(minikube docker-env)

cd metrics-app/backend
docker build -t metrics-backend:bootstrap .

cd ../frontend
docker build -t metrics-frontend:bootstrap .
```

### 3. Create Secrets

```sh
bash scripts/create-secrets.sh --env-file secrets.env
```

Reads from `secrets.env` if flag is present, otherwise prompts interactively. Creates all Kubernetes `Secret` resources across the relevant namespaces.

### 4. Deploy

```sh
bash scripts/deploy.sh
```

Applies all namespace manifests, manual YAML manifests, and Helm chart releases in dependency order. Waits for each stack to become ready before proceeding.

---

## Manual Post-Setup Steps

The following steps cannot be fully automated and must be completed after the scripts finish.

### Jenkins - Pipeline Job

1. Navigate to Jenkins at `https://jenkins.lan/`
2. Log in with the admin credentials set in `secrets.env`
3. Navigate to **Manage Jenkins** -> **Security** -> **Git Host Key Verification Strategy** -> **Accept first connection**
4. Click **New Item** -> enter a name -> select **Pipeline** -> click **OK**
5. Under **Build Triggers**, enable **Poll SCM**
6. Under **Pipeline**, set **Definition** to `Pipeline script from SCM`
7. Set **SCM** to `Git` and enter your repository URL
8. Under **Credentials**, click the dropdown menu and select `git`
9. Under **Branches to build**, specify `*/main` branch
10. Set **Script Path** to `Jenkinsfile`
11. Click **Save**

### Grafana - Dashboard Import

1. Navigate to Grafana at `https://grafana.lan/`
2. Log in with the admin credentials retrieved with `kubectl get secrets -n monitoring monitoring-grafana -o jsonpath='{.data.admin-user}' | base64 -d; echo && kubectl get secrets -n monitoring monitoring-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo`
3. Go to **Dashboards -> Import**
4. Import the following files from `manifests/manual/monitoring/dashboards/`:

| File | Dashboard |
|------|-----------|
| `cluster-status.json` | Cluster Performance |
| `pod-metrics.json` | Pod and Container |
| `metrics-app.json` | Application Performance |

5. Select **Prometheus** as the data source when prompted for each import

### Kibana - Saved Objects Import

1. Navigate to Kibana at `https://kibana.lan/`
2. Log in with the Elasticsearch credentials set in `secrets.env` or the password set using manual input
3. Go to **Stack Management -> Saved Objects -> Import**
4. Select `manifests/manual/logging/kibana/saved_objects/export.ndjson`
5. Click **Import** - this will restore the three pre-configured dashboards:
   - Cluster Logs
   - Application Logs
   - Pod and Container Logs

You can always check the credentials with:
```sh
kubectl get secrets -n logging elasticsearch-credentials -o jsonpath='{.data.username}' | base64 -d; echo && kubectl get secrets -n logging elasticsearch-credentials -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## Accessing Services

All services are exposed through ingress using `/etc/hosts` DNS resolving. Get the Minikube IP first:

```sh
minikube ip
```

Add `/etc/hosts` entries:

```sh
sudo nano /etc/hosts

# Add
# <MINIKUBE_IP> metrics.lan
# <MINIKUBE_IP> jenkins.lan
# <MINIKUBE_IP> prometheus.lan      # Prometheus is only for debugging purposes
# <MINIKUBE_IP> grafana.lan
# <MINIKUBE_IP> kibana.lan
```

| Service | URL |
|---------|-----|
| Frontend | `https://metrics.lan/` |
| Jenkins | `https://jenkins.lan/` |
| Grafana | `https://grafana.lan/` |
| Kibana | `https://kibana.lan/` |
| Prometheus | `https://prometheus.lan/` |

---

## Usage

The monitoring and logging stack is intended to be used continuously during normal operations to observe system health, investigate incidents, and validate infrastructure changes. Grafana is used for real-time metrics and alerting, while Kibana is used for log exploration and forensic analysis. Prometheus is primarily used as a data source and query engine when building or validating dashboards and alerts.

- [Grafana URL](https://grafana.lan/)
- [Kibana URL](https://kibana.lan/)
- [Prometheus URL](https://prometheus.lan/)

### Alerts and Notifications

All critical alerts are evaluated in Grafana. Alerts are triggered for resource exhaustion, service instability, container failures and infrastructure availability issues.

Alert history and current alert state can be reviewed directly in the Grafana Alerting interface.

### Common Tasks

- **Check system health**:  
  Open Grafana and review the Cluster Status and Pod Metrics dashboards.

- **Investigate performance issues**:  
  Correlate Grafana metrics (CPU, memory, latency) with Kibana logs for the same time range.

- **Validate alerting rules**:  
  Use the test documentation linked in the Alerts section to intentionally trigger alerts.

### Data Flow
<details>

```
            Scraping               Log Collector
┌─────────────┐ ┌───────────────┐ ┌────────────┐
│    Node     │ │ Elasticsearch │ │ Fluent Bit │
│   Exporter  │ │   Exporter    │ └─────┬──────┘
└──────┬──────┘ └───────┬───────┘       │
       │                │               │ 
       ▼     PromQL     ▼               ▼  Log Storing and Indexing
      ┌──────────────────┐       ┌───────────────┐      
      │    Prometheus    │       │ Elasticsearch │      
      └─────────┬────────┘       └──────┬────────┘      
                │                       │       
                ▼                       ▼ 
          ┌────────────┐         ┌────────────┐     
          │   Grafana  │         │   Kibana   │
          │ Dashboards │         │ Dashboards │
          └────────────┘         └────────────┘
```
</details>

### Dashboards

Dashboards were created in Grafana and Kibana.

### Grafana

Grafana is used to visualize Cluster performance, Pod performance and App performance. Dashboards created are listed below.

#### Cluster Performance Dashboard
<details>

`Cluster Status` dashboard displays the following info:

##### Cluster Info

* Targets up/down
  * Indicates which nodes are up or down.
* Pods By Phase
  * Indicates in which phase the pods are currently
* Nodes: Ready vs NotReady
  * Indicates if there are nodes that are not ready
* API Server Availability
  * Displays the API server status

##### Disk Info

* Node Filesystem Usage Percentage
  * Displays the `/` and `/mnt/vda1` path usage percentages
* Node Filesystem Disk Info
  * Displays the sizes, used and available disk spaces of `/` and `/mnt/vda1` paths

##### System Details

* Node Memory Usage Percentage
  * Displays the current Minikube RAM usage as a percentage
* Node CPU Usage Percentage
  * Displays the current Minikube CPU usage as a percentage
* Node Memory Usage Over Time
  * Displays the RAM usage trend over time with limits
* Node CPU Usage Over Time
  * Displays the CPU usage trend over time with limits

</details>

#### Pod Performance Dashboard
<details>

`Pod Metrics` dashboard displays the following info:

##### Container Status

* Seconds Since Last Seen
  * Last time a container was scraped (seen)
* Pod Phase Status
  * Displays the current pod phase
* Pod Restarts
  * How many times has the pods started in the last 15 minutes in the current namespace
* Total Pod Restarts
  * How many times has the pod started overall in the current namespace

##### Pod Usage Metrics

* Container CPU Usage
  * Displays the CPU usage trend of the pods
* Memory Usage
  * Displays the RAM usage trend of the pods
* Pod CPU Request vs Limit
  * How much CPU pod requests and what is the limit
* Pod RAM Request vs Limit
  * How much RAM pod requests and what is the limit
* Network Activity
  * What is the current network activity of each pod (if any)
</details>

#### App Performance Dashboard
<details>

`Metrics App` Dashboard displays the following info:

##### App Info

* App Uptime
  * How long the app has been running
* App Version
  * Table displaying the app name, current version (commit hash) and the commit message

##### Response Times

* Requests Per Second
  * How many requests are made to the app per second (i.e. how many users the app has)
* HTTP Request Latency
  * Average response time of the app
* P95 Latency
  * Response time threshold where 95% of requests are completed in less time than P95 value
* HTTP Method/Status Breakdown
  * Displays the HTTP methods and responses, along with endpoints reached

##### Error Rates

- Error Rate (%)
    - Percentage of error responses in the last 5 minutes.
- Error Responses
    - How many requests per second resulted in error responses.
</details>

### Kibana

Kibana is used to visualize logs from across the cluster. Dashboards created are listed below.

#### Cluster Logs Dashboard
<details>

`Cluster Logs` dashboard display the following info:

* Log Volume Over Time
  * Displays the log volume
* Kubelet vs containerd Log Rate
  * Displays the log rates of kubelet and containerd
* Node Events by Severity
  * Breakdown of the node events, broken down by severity
* Node Events Log Level Breakdown
  * Displays the levels of the node log events on a pie graph
* CoreDNS Activity
  * Displays the CoreDNS activity
* Kube-system Log Level Breakdown
  * Displays the levels of the kube-system log events on a pie graph
* Node Logs Table
  * Log table covering the recent node log entries
* Kube Log Table
  * Log table covering the recent kube-system log entries
* Cluster Table
  * Log table covering the recent core logs entries
* Error Logs Over Time
  * Displays the error logs over time

</details>

#### Pod and Container Logs Dashboard
<details>

`Pod and Container Logs` dashboard display the following info:

* Log Volume By Namespace
  * Displays the log volumes by namespace
* Error Logs By Pod
  * Displays the error spikes
* Log Level Over Time
  * Displays the log level trend over time
* Log Volume By Container
  * Displays the log volumes by container
* stderr vs stdout Ratio
  * Pie graph displaying the ratio between stdout and stderr
* Top Pods By Log Volume
  * Displays the top pods by log volume
* Log Table
  * Table displaying the recent log messages
* Pod Log Table
  * Displays the pod log messages
</details>

#### Application Logs Dashboard
<details>

`Application Logs` dashboard display the following info:

* Error and Warning Count Over Time
  * Trend displaying the error and warning level counts over time
* Ingress-nginx Access Log Breakdown
  * Displays the Ingress related log level volumes
* Log Level Distribution
  * Displays the log level distribution on a pie graph
* Top Error Messages
  * Table displaying the top error messages
* stderr Output Over Time
  * Displays the volume of logs written to stderr
* Log Table
  * Displays the recent messages per application
* HTTP Responses
  * Displays the recent HTTP response codes, methods and paths
</details>

### Alerts

Below are the alerts created:
<details>

#### Cluster Alert

* API Server Unreachable
  * Fires when the Kubernetes API server becomes unreachable

#### Logging Alerts

* Elasticsearch Unhealthy
  * Fires when Elasticsearch cluster status changes to `yellow` or `red`
* Fluent Bit Errors
  * Fires when Fluent Bit fails to retry flushes or is dropping records

#### Node Alerts

* Node High CPU Usage
  * Fires when node CPU usage is over 80% for at least five minutes
* Node Low Disk Space
  * Fires when node disk space falls below 20%
* Node High Memory Usage
  * Fires when node RAM usage is over 90% for at least five minutes

#### Pod Alerts

* Pod Frequent Restarts
  * Fires when a pod has restarted more than 3 times in the past 15 minutes
* Container High Memory
  * Fires when a container RAM usage is over 80% of its limit
* Pod Stuck In Pending Status
  * Fires when a pod has been stuck in `Pending` status for 5 minutes

</details>

### Testing Alerts

For testing the alerts, refer to [this document (unavailable)](/documents/tests.md).

### Jenkins

Jenkins is used as a CI/CD tool for detecting changes in the `metrics-app` source code and automatically build the components, test the images for vulnerabilities and app integrity, and deploy the updated app to the respective containers.

#### Accessing Jenkins UI

Jenkins is available in [Jenkins UI](https://jenkins.lan/). Pipelines can be modified and created in the UI, inspect the app build info and steps, as well as installing additional plugins. The current installation uses only the necessary plugins, as Jenkins itself is heavy and plugins introduce a ton of overhead.

#### Jenkins Pipeline Stages

Jenkins reads the Jenkinsfile from the repository, which defines the pipeline. Below is the pipeline stages

1. Checkout SCM
2. Retrieve the Registry IP dynamically
3. Compute Tag for the app versioning
4. Build Docker images and run integrated tests
5. Push to registry
6. Images are scanned with Trivy for HIGH and CRITICAL vulnerabilities. Findings cause the pipeline to fail on the `main` branch only
7. `kubectl set image` call for the recently built images
8. Rollout verification with `kubectl rollout status`

If any of the stages fail, the pipeline is halted and the app is not updated to prevent deploying a broken/insecure application.

---

## Restarting a Stopped Cluster

If the cluster was stopped with `minikube stop`, only the first script is needed to bring it back up:

```sh
bash scripts/start.sh
```

All resources are persisted - no need to re-run `create-secrets.sh` or `deploy.sh`.

It should be noted that Jenkins pod might start to misbehave upon starting. Observe with:
```sh
kubectl get pods -n ci -w
```

If Jenkins pod doesn't start, it is best to delete the pod and wait for Minikube to recreate it:
```sh
kubectl delete pod -n ci jenkins-0
```

---

## Repository Structure

```
.
├── Jenkinsfile                  # CI/CD pipeline definition
├── manifests
│   ├── helm                     # Helm values overrides
│   │   ├── elasticsearch-exporter/
│   │   ├── fluent-bit/
│   │   ├── jenkins/
│   │   └── monitoring/
│   └── manual                   # Raw Kubernetes manifests
│       ├── apps/                # Backend and frontend deployments
│       ├── certificates/        # cert-manager issuers and certificates
│       ├── ci/                  # Jenkins RBAC
│       ├── ingress/             # ingress-nginx configuration
│       ├── logging/             # Elasticsearch, Fluent Bit, Kibana
│       ├── monitoring/          # Prometheus rules, dashboards, ServiceMonitors
│       ├── namespaces/          # Namespace definitions
│       ├── registry/            # In-cluster container registry
│       └── tests/               # Manifests for simulating failure scenarios
├── metrics-app
│   ├── backend/                 # Go metrics API
│   └── frontend/                # React + Vite frontend
├── scripts
│   ├── start.sh                 # Minikube initialisation
│   ├── create-secrets.sh        # Secret provisioning
│   └── deploy.sh                # Full cluster deployment
└── secrets.env                  # Local only, gitignored
```

---

## Additional Features

Refer to [this documentation](/05-cluster-chronicles/documents/kubernetes-principles-explained.md) for further information.

Below is a brief run down on the additional features implemented:

* **HPA**: The frontend deployment is configured with a Horizontal Pod Autoscaler that scales between 2 and 10 replicas based on CPU utilisation (target: 50%)
* **TLS**: All ingress routes are served over HTTPS via cert-manager with automatically rotated certificates
* **Image scanning**: The Jenkins pipeline runs Trivy on built images before deployment; HIGH and CRITICAL findings cause the pipeline to fail on the `main` branch
* **Network policies**: Pod-to-pod traffic is restricted by namespace; the backend's metrics endpoint is only reachable from the `monitoring` namespace

## Author

Aki Heiskanen