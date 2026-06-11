# Sherlock Logs

## Project Overview

This project implements a centralized, production-grade monitoring and logging platform to support a growing, distributed infrastructure. A dedicated virtual machine will host all observability components to ensure isolation, reliability, and scalability.

Monitoring was built around Prometheus for metrics collection and Grafana for visualization. System-level metrics are gathered from VMs and Docker containers using exporters such as Node Exporter, Elasticsearch Exporter and cAdvisor. The existing infrastructure metrics application was modified to expose a Prometheus-compatible metrics endpoint. Grafana dashboards provide real-time visibility into VM performance, container health, and application behavior, supported by targeted alerting for resource exhaustion, service instability, and infrastructure failures. Advanced alerting is configured, including logging-driven alerts for repeated SSH login failures, with all alerts forwarded to Discord.

Logging is centralized using the ELK stack. Logs from VMs, Docker containers, and the application are collected via Filebeat, processed by Logstash, and indexed in Elasticsearch. The application is updated to ensure structured log delivery. Kibana dashboards enable analysis of system, application, and container logs for troubleshooting, security monitoring, and historical insight.

All components are integrated into the existing automation and CI/CD pipelines, ensuring monitoring and logging agents are deployed automatically with new infrastructure. The final system delivers continuous observability, proactive alerting, faster incident response, and data-driven operational insight across the entire environment.

## Setup
<details>

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

For installing and using the tools listed above on Windows or MacOS, refer to relevant guides [Windows](https://betanet.net/view-post/exploring-qemu-kvm-for-windows-an)/[MacOS](https://betanet.net/view-post/using-qemu-kvm-on-macos-a-comprehensive). This might require changes to the Terraform configurations.

From this point onward the setup guide assumes that a Linux distro is used.

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
</details>

## Usage

The monitoring and logging stack is intended to be used continuously during normal operations to observe system health, investigate incidents, and validate infrastructure changes. Grafana is used for real-time metrics and alerting, while Kibana is used for log exploration and forensic analysis. Prometheus is primarily used as a data source and query engine when building or validating dashboards and alerts.

- [Grafana URL](http://192.168.122.200:3000)
- [Kibana URL](http://192.168.122.200:5601)
- [Prometheus URL](http://192.168.122.200:9090)

### Alerts and Notifications

All critical alerts are evaluated in Grafana and forwarded to a Discord channel via webhook integration. Alerts are triggered for resource exhaustion, service instability, container failures, infrastructure availability issues, and repeated failed SSH login attempts detected via logs.

Alert history and current alert state can be reviewed directly in the Grafana Alerting interface.

### Common Tasks

- **Check system health**:  
  Open Grafana and review the VM Performance and Docker Container dashboards.

- **Investigate performance issues**:  
  Correlate Grafana metrics (CPU, memory, latency) with Kibana logs for the same time range.

- **Investigate security events**:  
  Use the System Logs dashboard in Kibana to analyze SSH authentication attempts and identify suspicious activity.

- **Validate alerting rules**:  
  Use the test commands in the Alerts section to intentionally trigger alerts and confirm notification delivery.

### Data Flow
<details>

```
            Scraping               Log Shipper
┌─────────────┐ ┌───────────────┐ ┌──────────┐
│    Node     │ │ Elasticsearch │ │ Filebeat │
│   Exporter  │ │   Exporter    │ └────┬─────┘
└──────┬──────┘ └───────┬───────┘      │
       │                │              │ 
       ▼     PromQL     ▼              ▼  Log Collector
      ┌──────────────────┐       ┌──────────┐      
      │    Prometheus    │       │ Logstash │      
      └─────────┬────────┘       └────┬─────┘      
                │                     │       
                ▼                     ▼ Log Storing and Indexing 
          ┌────────────┐         ┌─────────┐     
          │   Grafana  │         │ Elastic │
          │ Dashboards │         │ Search  │
          └────────────┘         └────┬────┘
                                      │
                                      ▼
                                ┌────────────┐
                                │   Kibana   │
                                │ Dashboards │
                                └────────────┘
```
</details>

### Dashboards

Dashboards were created in Grafana and Kibana.

### Grafana

Grafana is used to visualize VM performance, App performance and Container performance. Dashboards created are listed below.

#### VM Performance Dashboard
<details>

`Server Metrics` dashboard displays the following info:

##### System Info

- CPU Busy
    - Overall CPU busy percentage.
- Sys Load
    - System load over all CPU cores together.
- RAM Used
    - Percentage of RAM used excluding buffer + cache (reclaimable memory).
- Root FS Used
    - Percentage of root filesystem used.
- SWAP Used
    - Percentage of SWAP used.
- CPU Cores
- RAM Total
- SWAP Total
- RootFS Total
- Uptime

##### Disk Info

- Disk Space Used
    - Percentage of used disk space on `/`, `/boot/` and `/boot/efi` paths.
- Disk Read/Write Data
    - Number of bytes read from or written to the device per second.
- Disk Read/Write IOps
    - Number of I/O operations completed per second.
- Time Spent Doing I/Os
    - Percentage of time the disk spent actively processing I/O operations, including general I/O, discards (TRIM), and write cache flushes.

##### Network Info

- Network Traffic
    - Incoming and outgoing network traffic per interface.
- Network Traffic by Packets
    - Number of network packets received and transmitted per second, by interface.
</details>

#### Docker Container Dashboard
<details>

`Container Metrics` dashboard displays the following info:

##### Container Status

- Seconds Since Last Seen
    - Last time a container was scraped (seen).
- Container Health
    - Container health status.
- Container Restarts
    - How many times has the container started in the last 15 minutes.

##### Container Usage Metrics

- Container CPU Usage
    - Percentage of CPU a container is using.
- Memory Usage
    - How much RAM a container is using.
</details>

#### Metrics App Dashboard
<details>

`Metrics App` Dashboard displays the following info:

##### App Info

- App Uptime
    - How long the app has been running.
- App Version
    - Table displaying the app name, current version (commit hash) and the commit message.

##### Response Times

- Requests Per Second
    - How many requests are made to the app per second (i.e. how many users the app has).
- HTTP Request Latency
    - Average response time of the app.
- P95 Latency
    - Response time threshold where 95% of requests are completed in less time than P95 value.

##### Error Rates

- Error Rate (%)
    - Percentage of error responses in the last 5 minutes.
- Error Responses
    - How many requests per second resulted in error responses.
</details>

### Kibana

Kibana is used to visualize logs from across the infrastructure. Dashboards created are listed below.

#### System Logs Dashboard
<details>

System Logs dashboard display the following info:

##### System Logs

- Logs Over Time
    - Displays the `syslog` and `auth` log entries over time.
- Top Hosts by Log Volume
    - Displays how the logs are divided per host.
- Syslog Warn/Error Messages
    - Queries the system log entries for "warn" or "error" messages and displays them.

##### SSH Activity

- Top Source IPS
    - Displays the top source IPs from where an SSH connection was either made or attempted.
- SSH Events
    - Displays the SSH authentication logs.
</details>

#### Docker Logs Dashboard
<details>

Docker Logs dashboard display the following info:

##### Logs Per Container

- Logs Per Container
    - Amount of logs per Docker containers.
- Logs Over Time by Container
    - Time series of logs by container.

##### Stdout vs Stderr

- Container stdout Logs
    - Container stdout log messages.
- Container stderr Logs
    - Container stderr log messages.
</details>

#### Application Logs Dashboard
<details>

Application Logs dashboard is by default empty, since the App does not write any logs during normal use.

##### Application Logs

- Application Logs
    - Displays the amount of logs in the Application container.
- Warning And Error Messages
    - Displays the amount of warning and error messages.
- Metrics App Logs
    - Displays the full Application logs.
</details>

### Alerts

To test the alerts, open terminal using SSH or using `virt-manager` on a VM of your choice and run the following commands. The results can be seen by navigating to Grafana interface on `http://192.168.122.200:3000/alerting/list`.

#### VM Tests

##### RAM Test
<details>

```sh
stress --vm 1 --vm-bytes 1400M --vm-keep -t 360
```
</details>

##### CPU Test
<details>

```sh
stress-ng --cpu 8 --timeout 360s
```
</details>

##### Disk Space Test
<details>

```sh
# Check how much space is left on "/" directory
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# tmpfs           795M 1012K  794M   1% /run
# /dev/vda1        29G  8.7G   20G  31% /           <--- Avail 20G, Use% 31%
# tmpfs           3.9G     0  3.9G   0% /dev/shm
# tmpfs           5.0M     0  5.0M   0% /run/lock
# /dev/vda16      881M  117M  703M  15% /boot
# /dev/vda15      105M  6.2M   99M   6% /boot/efi

# Create a large file, adjust file size as needed
# e.g if free disk space is 20 GB, allocate 15 GB 
# 20% of 20 GB = 4 GB => free disk space 3.9 GB = 85% full
fallocate -l 15G /tmp/large_file.img

df -h
# Filesystem      Size  Used Avail Use% Mounted on
# tmpfs           795M 1012K  794M   1% /run
# /dev/vda1        29G   24G  4.4G  85% /          <--- Avail 4.4G, Use% 85%
# tmpfs           3.9G     0  3.9G   0% /dev/shm
# tmpfs           5.0M     0  5.0M   0% /run/lock
# /dev/vda16      881M  117M  703M  15% /boot
# /dev/vda15      105M  6.2M   99M   6% /boot/efi

# Remove the file after testing
sudo rm -rf /tmp/large_file.img
```
</details>

##### VM Unreachable Test
<details>

Stop the VM using `virt-manager`, or simulate by stopping `prometheus-node-exporter` service.

```sh
sudo systemctl stop prometheus-node-exporter.service
```

Start the VM or service after the alert has fired.

```sh
sudo systemctl start prometheus-node-exporter.service
```
</details>

##### Repeated Failed SSH Logins
<details>

Kibana is set up to alert if repeated failed SSH attempts are made.

```sh
ssh devops@<VM_IP>
```
</details>

#### Docker Container Tests

For these tests, open a terminal via SSH or using `virt-manager` on a Docker host.

##### Container Memory Usage Test
<details>

```sh
docker run -m 512m --name memory_test ubuntu /bin/bash -c "apt-get update && apt-get install -y stress-ng && stress-ng -vm 1 --vm-bytes 450M --vm-keep --timeout 360"
```
</details>

##### Container Restart Test 
<details>

```sh
docker restart <container_name> # e.g. docker restart metrics-backend
```

Monitor `Container Metrics` dashboard. Container Health dashboard displays `Starting` during startup. Changing to `Healthy` indicates that the container has successfully started. Container Restart counter is incremented after successful restart.
</details>

##### App Container Down Test
<details>

```sh
docker stop <container_name>
```

Monitor `Container Metrics` dashboard. Container Last Seen for the chosen container will increase instead of resetting. This indicates that the container has become stale.
</details>

#### App Error Rate Test
<details>

SSH into Web-server-01 or Web-server-02. Send requests to backend `/info` endpoint with query `fail=true`.

```sh
for i in $(seq 1 20); do curl http://10.0.0.10:8080/info?fail=true; sleep 1; done
```

Monitor `Metrics App Dashboard`. Error rate should go up. An alert will fire when the error rate is above 5%.
</details>

### Author

Aki Heiskanen