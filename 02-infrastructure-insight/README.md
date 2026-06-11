# Infrastructure Insight

This project involves setting up a secure, containerized infrastructure and deploying a diagnostic web application across multiple servers. It includes preparing application, web, and load-balancer servers with containerization tools, configuring network/firewall rules, and ensuring inter-server communication.

The diagnostic app includes a modular frontend and backend with a `/metrics` endpoint exposing system details (hostname, OS, CPU, memory, etc.) and is designed for containerized deployment.

Once built, the backend runs on the application server, while frontend container runs on two web servers behind a load balancer. The load balancer distributes traffic between the web servers, and the frontend retrieves data from the backend. The system should serve the application through the load balancer and alternate responses between servers.

This is a continuation to the previous task, *[Server Sorcery 101](../01-server-sorcery-101/README.md)*.

## Scope

This project includes:
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
    - Restoration procedures and support for rescua-mode recovery.
    - Backup cleanup automation.
- Enhancements beyond basic requirements
    - UI/UX improvements, such as responsive elements and various color blindness modes.
    - Load balancing algorithms.
    - Secure communication between VMs using WireGuard VPN.

This project _does not include_:
- Production-grade security hardening beyond basic firewall rules.
- Multi-region deployments.
- CI/CD pipelines.
- Monitoring beyond the custom `/metrics` endpoint.

## Objectives

1. Ensure proper communication between all infrastructure components, including application, web, and load balancer servers.
2. Develop a diagnosting application (frontend + backend API) that exposes system-level metrics via a `/metrics` endpoint.
3. Deploy the application across multiple servers with load balancing.
4. Implement an automated backup system capable of weekly full backups and on-demand restoration for all VMs.
5. Enhance application usability and visual design through UI/UX improvements.
6. Support extensible load-balancing algorithms (round robin, least connections, or IP hash)

## Prerequisities

- VM environment set up using [this guide](../01-server-sorcery-101/README.md)
- [Backend requirements](/02-infrastructure-insight/backend/README.md#requirements)
- [Frontend requirements](/02-infrastructure-insight/frontend/README.md#requirements)
- [Docker on each VM](https://docs.docker.com/engine/install/ubuntu/) (excluding Load Balancer as it doesn't need Docker)

    ```sh
    # Add Docker's official GPG key:
    sudo apt update
    sudo apt install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
    Types: deb
    URIs: https://download.docker.com/linux/ubuntu
    Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
    Components: stable
    Signed-By: /etc/apt/keyrings/docker.asc
    EOF

    sudo apt update
    sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```
- Nginx on Load Balancer

## Backend Setup

- Allow access for frontend servers:
```sh
sudo ufw allow from <WEB_SERVER_1_IP> to any port 8080 proto tcp comment 'web-01-access'
sudo ufw allow from <WEB_SERVER_2_IP> to any port 8080 proto tcp comment 'web-02-access'
sudo ufw reload
```

> You can use either the static IP or VPN IP if you have one
```sh
# Static IP Example
sudo ufw allow from 192.168.122.11 to any port 8080 proto tcp comment 'web-01-access'
# VPN IP Example
sudo ufw allow from 10.0.0.12 to any port 8080 proto tcp comment 'web-02-access'
```

## Frontend Setups

- Allow access for Load Balancer:
```sh
# Same principle applies here, use either static IP or VPN IP
sudo ufw allow from <LOAD_BALANCER_IP> to any port 80 proto tcp comment 'load balancer'
sudo ufw reload
```

## Load Balancer Setup

1. Install and enable nginx:
```sh
sudo apt install nginx -y
sudo systemctl enable --now nginx
```
2. Open the necessary nginx ports and reload firewall:
```sh
sudo ufw allow from <host_PC_IP> to any port 80,443 proto tcp comment 'nginx'
sudo ufw reload
```
3. Remove `/etc/nginx/sites-enabled/default`
```sh
sudo rm /etc/nginx/sites-enabled/default
```
4. Create `load_balancer.conf`
```sh
sudo touch /etc/nginx/conf.d/load_balancer.conf 
```
5. Modify as per attached [configuration file (available upon request)](balancer-configs/load_balancer.conf).
```sh
sudo nano /etc/nginx/conf.d/load_balancer.conf
```

## Backup VM Setup

New VM called 'Backup' created using [this guide](../01-server-sorcery-101/README.md). 

This VM:
- Pulls backups via rsync
- Can restore directories and files back to the original VMs

**If you wish to continue this set up, extend the chapter below.**

<details>

### Backup VM

#### SSH Key Generation

```sh
ssh-keygen -t ed25519 -f ~/.ssh/backup_key -N "" # Make it without a passphrase in order to automate the process
sudo chmod 600 ~/.ssh/backup_key
```

#### SSH Key Copying

```sh
ssh-copy-id -i ~/.ssh/backup_key.pub devops@<IP> # Replace IP with the receiving VM IP
```

If copying fails, check:

1. Firewall

```sh
sudo ufw status numbered
```

Ensure: 
```sh
22/tcp  ALLOW   <Backup-VM-IP>
```

2. SSH Authentication

If required:

- Temporarily set in `/etc/ssh/sshd_config`:
```conf
PasswordAuthentication yes
PubKeyAuthentication no
```
- Restart `ssh`:
```sh
sudo systemctl restart ssh
```

After copying the key, revert the settings

> (PasswordAuthentication no, PubKeyAuthentication yes)

and restart SSH again.

#### Create Backup Folder Structures

```sh
sudo mkdir -p \
    /backups/app-server \
    /backups/web-server-01 \
    /backups/web-server-02 \
    /backups/load-balancer
```

### Other VMs

#### Add Sudoers Rule

```sh
sudo visudo -f /etc/sudoers.d/backup
```

- Add:
```conf
devops ALL=(root) NOPASSWD:/usr/bin/rsync
```

#### Add Firewall Rules

```sh
# Replace IP with actual Backup VM IP
sudo ufw allow from <BACKUP_VM_IP> to any port 22 proto tcp comment 'Backup SSH'
# Alternatively use VPN IP
# sudo ufw allow from <BACKUP_VPN_IP> to any port 22 proto tcp comment 'Backup SSH'
```

### Testing Backup Setup

Example: App server VM

```sh
# Replace with your SSH key and server IP if different
sudo rsync -aAXH \
    -e "ssh -i .ssh/backup_key" \
    --rsync-path="sudo rsync" \
    devops@192.168.122.10:/etc \
    /backups/app-server/$(date +%F)
```

```sh
# VPN Example
sudo rsync -aAXH \
    -e "ssh -i .ssh/backup_key" \
    --rsync-path="sudo rsync" \
    devops@10.0.0.10:/etc \
    /backups/app-server/$(date +%F)
```

The directory `/backups/app-server/<date>/etc` will contain the backup.

Repeat for other VMs.

## Automating Backup

Follow instructions to copy the [backup script (available upon request)](scripts/README.md#vms).

Use cronjob to automate weekly backups.

```sh
crontab -e
```

Add:

```sh
# At 03:00 AM, only on Monday
0 3 * * 1 /usr/local/bin/run_backup.sh >> /var/log/backup.log 2>&1
```

Run once manually:

```sh
sudo /usr/local/bin/run_backup.sh
```

This will backup `/etc`, `/home` and `/var/lib/docker` directories to the Backup VM.

### Automating Old Backup Removal

Copy the [cleanup script (available upon request)](scripts/vm-scripts/backup/clean_backups.sh). This script only runs if there are more than four backups present.

It runs 30 minutes after backing up the most recent data.

Add to crontab:
```sh
crontab -e
```

```sh
# At 03:30 AM, only on Monday
30 3 * * 1 /usr/local/bin/cleanup_backups.sh >> /var/log/backup_cleanup.log 2>&1
```

## Restoring Backups

To restore the backups from Backup VM:

```sh
# Replace with your SSH key
sudo rsync -aAXH \
    -e "ssh -i .ssh/backup_key" \
    --rsync-path="sudo rsync" \
    /backups/<server>/<date>/<directory>/ \
    devops@IP:/<directory>/
```

Restoring `/etc` or `/var/lib/docker` on a running system may require downtime or rescue mode.

### Booting a VM into Rescue Mode (Recovery Shell)

Some backup restores are safest to perform while the system is **not fully running**.

The easiest method is to boot into recovery mode from GRUB.

#### Entering Rescue Mode

1. Open the VM console
2. Restart the VM
3. When the GRUB menu appears, press `ESC` or `SHIFT` depending on your hypervisor.
4. In the GRUB menu, select "Advanced options for Ubuntu"
5. From the submenu, choose "Ubuntu, with Linux <version> (recovery mode)"
6. The system will boot into the recovery menu.
    - Select: `root Drop to shell prompt`
7. The VM is now in rescue mode and safe for tasks such as:
    - Restoring `/etc`
    - Restoring `/home`
    - Restoring `/var/lib/docker`
    - Fixing misconfigurations
    - Resetting broken services
8. When finished, reboot normally with `reboot`.

## Automation

Several scripts were created in order to reduce the repetitive tasks and commands. These can be found in [scripts/ (available upon request)](scripts) directory.

</details>

## Author

Aki Heiskanen