# Automation Alchemy

## Objectives

Below are objectives of each task that this task aimed to automate.

### Server Sorcery 101

Server Sorcery 101 focused on documenting the complete process of building, securing and monitoring a small-scale virtualized infrastructure. The goal was to simulate a production-like environment from scratch. It covered VM creation, networking, user and security configuration, firewalling, VPN setup, intrusion prevention, and live monitoring with Prometheus + Grafana. Each step was documented in detail to explain not only _how_ things are done, but _why_ they are done that way.

### Infrastructure Insight

Infrastructure Insight involved setting up a secure, containerized infrastructure and deploying a diagnostic web application across multiple servers. It included preparing application, web, and load-balancer servers with containerization tools, configuring network/firewall rules, and ensuring inter-server communication.

The diagnostic app included a modular frontend and backend with a `/metrics` endpoint exposing system details (hostname, OS, CPU, memory, etc.) and is designed for containerized deployment.

Once built, the backend runs on the application server, while frontend container runs on two web servers behind a load balancer. The load balancer distributes traffic between the web servers, and the frontend retrieves data from the backend. The system should serve the application through the load balancer and alternate responses between servers.

It also included setting up a backup server, that takes weekly automated backups of `/home/devops`, `/etc` and app related directories.

This is a continuation of [Server Sorcery 101 (Task 1)](../01-server-sorcery-101/README.md).

### Automation Alchemy

Automation Alchemy's objective was to automate the previous tasks: [Server Sorcery 101 (Task 1)](../01-server-sorcery-101/README.md) and [Infrastructure Insight (Task 2)](../02-infrastructure-insight/README.md). It built upon those tasks to also feature a CI/CD pipeline for the app deployment, as well as comprehensive testing, operational rollback mechanism for the application and pipeline notifications.

The entire deployment from creating the VMs to full configured infrastructure has been automated via a deploy script.

## Scopes

### Server Sorcery 101

Included:
- Comprehensive documentation for easy reproducibility.
- Preparation of VMs for the next task.
- IPS and VPN setups.
- Basic monitoring.

_Did not include_:
- Further customization beyond core system configuration.
- Application development.
- Backup and recovery.

### Infrastructure Insight

Included:
- Application development
  - Modular frontend + backend diagnostic tool
  - `/metrics` endpoint showing hostname, OS, CPU, memory, and responding server.
- Deployment
  - Backend container deployed on the application server.
  - Frontend containers deployed on both web servers.
  - Nginx load balancer serving traffic and alternating responses between web servers based on the algorithm chosen.
- Backup and recovery
  - Deployment of a dedicated backup VM.
  - Automated weekly backups via `rsync`
  - Restoration procedures and support for rescue-mode recovery.
  - Backup cleanup automation.
- Enhancements beyond basic requirements
  - UI/UX improvements, such as responsive elements and various color blindness modes.
  - Load balancing algorithms.
  - Secure communication between VMs using WireGuard VPN.

_Did not include_:
- Production-grade security hardening beyond basic firewall rules.
- Multi-region deployments.
- CI/CD pipelines.
- Monitoring beyond the custom `/metrics` endpoint.

### Automation Alchemy

Included:
- CI/CD pipeline.
- One-click (script) automation.
- Idempotency.
- Easy to set up.
- Security hardening.
- Comprehensive testing.
- Pipeline and build notifications.
- Manual rollback mechanisms.

_Did not include_:
- Automated rollbacks.
- Setting up external registries.

## Requirements

### Functional Requirements

Infrastructure Automation
- The system shall automatically provision 7 virtual machines:
  - Application server
  - Two web servers
  - Load balancer
  - Backup server
  - Monitoring server with Prometheus + Grafana
  - CI/CD server
- VM provisioning shall be fully automated and executable from a single entry point.
- All VMs shall be networked according to predefined roles, including:
  - Internal private networking
  - Secure inter-VM communication
  - Restricted public exposure where applicable
- Provisioning shall be repeatable and idempotent, allowing multiple executions without breaking existing infrastructure.

System Configuration & Hardening
- Each VM shall be automatically configured with:
  - Non-root administrative users
  - SSH key-based authentication
  - Disabled password login
  - Firewall rules appropriate to its role
- Unnecessary services shall be disabled by default.
- Time synchronization and logging shall be configured consistently across all VMs.

Application Deployment
- The application shall be deployed using containerization.
- The backend shall run on the application server.
- Frontend containers shall run on both web servers.
- A load balancer shall distribute traffic between frontend instances.
- The application shall be reachable only via the load balancer.
- The `/metrics` endpoint shall expose system and application metrics.

CI/CD Pipeline
- The CI/CD system shall monitor a Git repository for changes.
- On code changes, the pipeline shall automatically:
  1. Checkout the source code
  2. Run tests and quality checks
  3. Build a deployable artifact
  4. Push the artifact to a container registry
  5. Deploy the updated application to target servers
- The CI/CD VM shall be provisioned and configured automatically as part of the infrastructure.

### Testing Requirements

- The pipeline shall include automated testing stages, including:
  - Static code analysis / linting
  - Unit or integration tests
- The pipeline shall fail on test or quality violations, preventing deployment.
- Test results shall be logged and visible in the CI/CD system.

### Rollback Requirements

- The system shall support manual rollback to a previously deployed application version.
- Rollback shall not require redeploying infrastructure.
- At least one previously built artifact shall be retained and redeployable.

### Notification & Alerting Requirements

- The system shall send notifications on:
  - Successful pipeline execution
  - Failed builds or deployments
- Notifications shall include link to the build output to identify the cause and stage of failure.

## Infrastructure Overview

This automation creates 7 virtual machines:
- Load Balancer (external access point)
- Application Server (backend container)
- Web Server 1 & 2 (frontend containers)
- CI Server (Jenkins pipeline)
- Backup Server (automated backups)
- Monitoring Server (Prometheus + Grafana)

## Setup

### Prerequisites

1. Create `./ansible/secrets/` directory for manual set up. Add the following files:

```sh
jenkins_password
gitea_token
webhook
```

2. Configure the files:
  - `jenkins_password`:
    - Set a password used to log in to Jenkins UI.
  - `gitea_token`:
    - Create an API token in Gitea:
      - User->Settings->Applications->Generate New Token with the following permissions:
        - repository: Read and Write
        - user: Read and Write
    - Copy the token to `gitea_token`.
  - `webhook`:
    - Create a server in Discord.
    - Create a text channel in the server.
    - Generate a new Webhook for the text channel.
    - Copy the webhook URL to `webhook`.

3. Copy the `gitea_ci_jenkins_key.pub` to Gitea:
  - User->Settings->SSH/GPG Keys->Add SSH Key
    - Key Name: ci-server-key
    - Content: `gitea_ci_jenkins_key.pub`
    - Add Key

**DISCLAIMER: Never share `gitea_ci_jenkins_key` and `gitea_token` to anyone, as those allow unrestricted access to your account and every repo you have created!**

### Tools

#### Virtualization & VM Management
- **QEMU** - Hardware virtualization and emulation engine used to run the virtual machines.
- **libvirt** - Virtualization management layer used by Terraform and CLI tools to define, create, and control QEMU/KVM virtual machines and networks.
- **virt-manager** - Graphical interface for managing libvirt-based virtual machines during development and troubleshooting.
- **virt-viewer** - Lightweight client for accessing VM graphical consoles via SPICE or VNC.

#### Networking & Connectivity
- **dnsmasq** - Lightweight DNS and DHCP service used by libvirt to provide IP addressing and name resolution for virtual machines.
- **bridge-utils** - Utilities for creating Linux network bridges, allowing VMs to communicate with each other and the host network.
- **vde2** - Virtual Distributed Ethernet used to simulate advanced virtual networking topologies when required.
- **openbsd-netcat (nc)** - General-purpose networking utility used for debugging, connectivity testing, and port validation.
- **firewalld** - Dynamic firewall manager used to enforce host and VM network access policies.

#### Infrastructure Automation & Configuration
- **Terraform** - Infrastructure as Code (IaC) tool used to declaratively provision virtual machines, networks, and storage via libvirt.
- **Ansible** - Configuration management and orchestration tool used to configure, harden, and deploy services across all VMs in an idempotent manner.

#### Storage & Media
- **cdrtools** - Utility suite for creating ISO images, used for generating VM installation or bootstrap media when needed.


### Tool Installation

1. Install the required tools using your preferred package manager:

```sh
# e.g. using Pacman
sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat libvirt firewalld terraform ansible cdrtools
```

2. Enable `libvirt`:

```sh
sudo systemctl enable --now libvirtd
sudo systemctl enable --now firewalld
sudo usermod -aG libvirt $(whoami)
```

3. Configure networking:

```sh
sudo firewall-cmd --permanent --new-zone=libvirt
sudo firewall-cmd --state
sudo firewall-cmd --reload
sudo firewall-cmd --get-states
# Expected: "libvirt" and "libvirt-routed"
sudo systemctl restart libvirtd
sudo virsh net-start default
sudo virsh net-autostart default
sudo virsh net-list
```

### Download Ubuntu Cloud Image

[Pre-installed image](https://cloud-images.ubuntu.com/noble/current/) that acts as the base for Ansible configuration: [`noble-server-cloudimg-amd64.img`](https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img). Download it to `/var/lib/libvirt/images/`.

### Create Storage Pool

```sh
mkdir -p /var/lib/libvirt/images/terraform
sudo virsh --connect qemu:///system pool-define-as terraform dir - - - - /var/lib/libvirt/images/terraform
sudo virsh --connect qemu:///system pool-start terraform
sudo virsh --connect qemu:///system pool-autostart terraform
```

### Create SSH Keys

1. Create a directory in the repository:

```
./ansible/secrets/
```

2. If there are no previous SSH keys, generate them without a passphrase:

```sh
ssh-keygen -t rsa -b 4096 -f ~/.ssh/<server_name>_key -C "<server-name>-access"
```

3. Lock the keys down:

```sh
chmod 600 ~/.ssh/<server_name>_key
```

4. Copy the keys from `~/.ssh/` to `./ansible/secrets/` directory:

```sh
.
└── ansible
    └── secrets
        ├── ssh_key
        └── ssh_key.pub
```

Required keys to generate and copy (**names have to be EXACT**):

```sh 
app_server_key
app_server_key.pub
backup_server_key
backup_server_key.pub
ci_jenkins_key
ci_jenkins_key.pub
ci_server_access_key
ci_server_access_key.pub
gitea_ci_jenkins_key
gitea_ci_jenkins_key.pub
load_balancer_key
load_balancer_key.pub
monitoring_server_key
monitoring_server_key.pub
web_server_1_key
web_server_1_key.pub
web_server_2_key
web_server_2_key.pub
```

## Usage

Once the setup is complete, you can use the automated deploy script [`deploy.sh` (available upon request)](deploy.sh) to automatically create and configure the infrastructure. The script runs the following commands:

```sh
cd terraform/
terraform init
terraform apply -auto-approve
# Waits for SSH to respond from each VM
cd ../ansible
ansible-playbook -i inventory.ini site.yml
```

If the infrastructure is already up, it will just run the Ansible commands.

If a clean slate is desired on an existing infrastructure, the script can be run with `bash deploy.sh --destroy`. This will destroy the current infrastructure, wait for 10 seconds, re-create the infrastructure and starts configuration.

Should any of the steps fail, it will return early from the script.

**The deployment scripts are idempotent and can be safely run multiple times without causing issues. Terraform manages infrastructure state, and Ansible tasks use handlers that only trigger on actual changes.**

You can also navigate to `./terraform/` directory and run the commands manually:

```sh
terraform init    # Initialize Terraform infrastructure
terraform plan    # Run Terraform plan
terraform apply   # Apply Terraform configuration
```

After Terraform has applied the configuration, navigate to `./ansible/` directory to run Ansible:

```sh
ansible-playbook -i inventory.ini site.yml
```

Should any of the playbooks fail, Ansible will stop the play and state the reason why the play failed.

### Ansible Roles

Ansible is capable of dividing the plays into modular roles, which can be further divided to target specific VMs.

Roles:

| Role | Targets | Purpose |
|:-----|:--------|:--------|
| **Baseline** | All VMs | Apply baseline configuration |
| **UFW** | All VMs | Apply UFW rules per each host specific list |
| **Backup** | Backup Server | Configure Backup Server to create backup directories and set up cron |
| **Backing Up** | All but Backup Server | Configure servers to accept `rsync` backup pulling from Backup Server |
| **SSH Hardening** | All VMs | Harden SSH and configure Fail2Ban IPS |
| **Wireguard** | All VMs | Install and configure WireGuard VPN |
| **Bootstrap** | Backup Server | Delete Backup Server bootstrap rules pre-WireGuard |
| **Docker** | App, Web01, Web02, CI Servers | Install and configure Docker |
| **Docker Keys** | App, Web01, Web02 | Install Docker keys |
| **Registry Creation** | CI Server | Create registry container for app images |
| **Nginx** | Load Balancer | Install and configure Nginx |
| **Balancing Algorithms** | Load Balancer | Copy balancing algorithm changing script to Load Balancer |
| **Node Exporter** | All but Monitoring Server | Install and configure Node Exporter for Prometheus scraping |
| **Prometheus** | Monitoring Server | Install Prometheus scraper |
| **Grafana** | Monitoring Server | Install Grafana and provision it |
| **Pipeline** | CI Server | Install necessary tools for Jenkins |
| **Pipeline API** | CI Server | Configure Jenkins, create pipeline job and kickstart it |
| **Final Conf** | All VMs | Modify users to require passworded sudo for devops |

### Load Balancer - Changing Balancing Algorithms

The Load Balancer supports switching between different Nginx load-balancing algorithms at runtime, without requiring redeployment or service restarts.

#### Supported Algorithms

- `round_robin` (default)
  - Requests are distributed sequentially and evenly across frontend instances.
  - Best suited for frontends with similar capacity and predictable workloads.

- `least_conn`
  - New requests are sent to the frontend with the fewest active connections.
  - Useful when requests vary in duration or frontend load is uneven.

- `ip_hash`
  - Client IP addresses are hashed to consistently route the same client to the same frontend.
  - Suitable for session-based applications that do not use shared session storage or to monitor user response over different versions of frontends.

#### Changing the Algorithm

1. Open the Load Balancer VM via virt-manager or connect over SSH:

```sh
ssh -i ./ansible/secrets/load_balancer_key devops@192.168.122.130
```

2. Run the algorithm switch script:

```sh
# Note that the script must be run as sudo to prevent any unauthorized changes
sudo ./set_balancing_algorithm [round_robin|least_conn|ip_hash]
```

3. Changes take effect immediately and persist across reboots. Note that Ansible enforces the load balancer configuration from a template; any manually selected algorithm will be reset on the next Ansible run unless the corresponding Ansible variable is updated.

> Note: The script validates input and reloads Nginx gracefully to avoid request interruption.

### Pipeline

Repository root contains `Jenkinsfile`, which dictates how the pipeline works. Due to not being able to use webhooks (since Jenkins runs in an isolated VM and Gitea is unable to send updates to it), Jenkins relies on polling the repository every 2 minutes. If it detects changes in `./metrics-app/backend/` or `./metrics-app/frontend/` directories, it will trigger the pipeline.

### Testing

Both backend and frontend feature tests run in the Docker container during building. Should any of the tests fail, the build will be halted to prevent deploying potentially broken application.

#### Backend Tests

| Package | Test Name | Description |
|:-------|:-----------|:------------|
| `main` | `TestMetricsHandler` | Verifies `/metrics` HTTP endpoint returns `200 OK`, `application/json` content type, and valid JSON |
| `structs` | `TestSystemInfoJSONShape` | Ensures `SystemInfo` marshals to JSON with all required top-level fields |
| `info` | `TestParseCPUInfo_Normal` | Parses CPU model, vendor, MHz, cores, and thread count correctly |
| `info` | `TestParseCPUInfo_MissingFields` | Missing CPU fields default to zero values |
| `info` | `TestParseCPUInfo_GarbageValues` | Ignores non-numeric CPU MHz values |
| `info` | `TestParseCPUStat_Normal` | Correctly parses CPU idle and total tick counts |
| `info` | `TestParseCPUStat_MissingFields` | Returns error for malformed CPU stat input |
| `info` | `TestParseCPUStat_NonNumeric` | Returns error when CPU stat contains non-numeric values |
| `info` | `TestParseCPUStat_Empty` | Returns error for empty CPU stat input |
| `info` | `TestComputeCPUUsage_Normal` | Computes correct CPU usage percentage |
| `info` | `TestComputeCPUUsage_BackwardsCounters` | Handles counter regression by returning zero usage |
| `info` | `TestComputeCPUUsage_ZeroDelta` | Returns zero usage when no counter delta exists |
| `info` | `TestParseDiskUsage_Normal` | Computes disk total, free, used MB and usage percentage correctly |
| `info` | `TestParseDiskUsage_ZeroBlocks` | Handles zero disk blocks without division errors |
| `info` | `TestParseDiskUsage_UsedIsGreaterThanTotal` | Prevents disk used value from exceeding total |
| `info` | `TestParseDiskUsage_EmptyValues` | Returns zeroed disk metrics when input values are empty |
| `info` | `TestComputeNetDeltas_Normal` | Computes RX/TX byte deltas for existing interfaces |
| `info` | `TestComputeNetDeltas_Regression` | Ignores interfaces with regressing counters |
| `info` | `TestComputeNetDeltas_NewInterface` | Does not compute deltas for newly discovered interfaces |
| `info` | `TestParseRAMInfo` | Parses RAM and swap totals, free, and usage values correctly |
| `info` | `TestParseRAMInfo_MissingFields` | Does not infer RAM usage when fields are missing |
| `info` | `TestParseRAMInfo_MissingMemTotal` | Prevents RAM usage calculation when total memory is missing |
| `info` | `TestParseRAMInfo_ZeroMemTotal` | Ensures zero total memory results in zero usage |
| `info` | `TestParseRAMInfo_DoesNotInferInUseWhenFreeMissing` | Avoids inferring RAM usage when free memory is absent |
| `info` | `TestParseRAMInfo_GarbageValues` | Ignores non-numeric RAM input values |
| `info` | `TestParseRAMInfo_PartialButValid` | Computes RAM usage from valid partial input |
| `info` | `TestParseRAMInfo_Swap_MissingFields` | Does not infer swap usage when swap free is missing |
| `info` | `TestParseRAMInfo_ZeroSwapTotal` | Ensures zero swap total results in zero swap usage |
| `info` | `TestParseRAMInfo_SwapGarbageValues` | Ignores non-numeric swap input values |
| `info` | `TestParseRAMInfo_SwapPartialButValid` | Computes swap usage from valid partial input |
| `info` | `TestParseUptimeInfo_Normal` | Parses uptime into hours, minutes, seconds, and formatted string |
| `info` | `TestParseUptimeInfo_Empty` | Handles empty uptime input as zero uptime |
| `info` | `TestParseUptimeInfo_NonNumeric` | Handles non-numeric uptime input as zero uptime |
| `info` | `TestParseUptimeInfo_Zero` | Correctly formats zero uptime input |

#### Frontend Tests

| Test Name | Description |
|:--------|:------------|
| `renders dashboard title` | Verifies the dashboard title renders correctly with the server name fetched from `/server-info` |
| `renders %s card` | Ensures each main metric card (OS Info, CPU Info, Uptime, RAM, Disk, Network Traffic) renders successfully |
| `renders all cards without crashing when metrics are partial` | Confirms the UI remains stable and renders all cards when backend metrics are incomplete |
| `doesn't crash when metrics is null` | Ensures the application does not crash when metrics data is `null` |
| `shows loading text when backend doesn't respond` | Displays a loading indicator when the metrics API returns a failed response |
| `setTheme is called when theme button clicked` | Verifies `setTheme` utility is called with the correct argument when a theme button is clicked |
| `theme buttons change theme properly` | Ensures theme buttons update the HTML class, persist to `localStorage`, and reflect active styling |
| `clicking RAM chart toggles unit` | Confirms clicking the RAM card toggles display units between MB and GB |

Covered UI behaviors:

- Initial data fetching and rendering
- Graceful handling of missing or partial backend data
- Theme selection and persistence
- Interactive metric unit toggling
- Chart rendering isolation via mocked chart components

### Rollback

Rolling back to a previous version of the application can be done thusly:

1. SSH into the CI Server:

```sh
ssh -i ./ansible/secrets/ci_server_access_key -o StrictHostKeyChecking=no UserKnownHostsFile=/dev/null devops@192.168.122.170
```

2. List all the images:

```sh
docker images
```

3. Identify the image you would like to re-deploy.

Example:
```sh
IMAGE                                                                      ID             DISK USAGE   CONTENT SIZE   EXTRA
10.0.0.17:5000/metrics-backend:51de3372328be063f07dcd6824ee94ffad4fd467    fae52e6568d3       25.8MB         8.59MB        
```

4. SSH into the corresponding VM.

```sh
# Example with App server
ssh -i ./ansible/secrets/app_server_key -o StrictHostKeyChecking=no UserKnownHostsFile=/dev/null devops@192.168.122.100
```

5. Stop the container, pull and build the image, and start the container:

```sh
docker stop metrics-backend || true &&
docker rm metrics-backend || true &&
docker pull 10.0.0.17:5000/metrics-backend:51de3372328be063f07dcd6824ee94ffad4fd467 &&
docker run -d --restart unless-stopped --name metrics-backend --network=host 10.0.0.17:5000/metrics-backend:51de3372328be063f07dcd6824ee94ffad4fd467
```

### Notifications

Discord is used to receive notifications.

Example of a successful build with links to Jenkins UI for details and the updated app:

![Successful build](/03-automation-alchemy/resources/build_succeeded.png)

Example of failed build with link to Jenkins UI Console Output to determine why build failed:

![Failed build](/03-automation-alchemy/resources/build_failed.png)

### Application

For details about the application, refer to relevant READMEs in `/metrics-app` directories:

[Backend README (available upon request)](/metrics-app/backend/README.md)

[Frontend README (available upon request)](/metrics-app/frontend/README.md)

Application is available [here](http://192.168.122.130) upon successful deployment.

### Jenkins UI

[Jenkins UI](http://192.168.122.170:8080) can be used to inspect build logs.

## Final Notes

- This automated infrastructure deployment has been designed to use `QEMU+libvirt`. It is not guaranteed to work with `HyperV`, `VirtualBox` or any other virtualization software.
- This project has not been tested on Windows/MacOS.
- This project requires a substantial amount of RAM. There are 7 VMs with a combined take of roughly 21.5 GB of RAM. If you desire to reduce the allocated resources, refer to [Terraform README](/03-automation-alchemy/terraform/README.md).

## Resources

[Terraform docs](https://developer.hashicorp.com/terraform/docs)

[Ansible docs](https://docs.ansible.com/projects/ansible/latest/index.html)

[Jenkins docs](https://www.jenkins.io/doc/book/getting-started/)

## Author

Aki Heiskanen