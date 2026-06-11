# Terraform Infrastructure

This directory contains the Terraform configuration responsible for provisioning
all virtual machines, disks, networking, and initial cloud-init configuration
using **libvirt + QEMU/KVM**.

Terraform is responsible only for **infrastructure creation**.
All system configuration, hardening, and application deployment is handled by
Ansible after provisioning.

---

## Overview

Terraform provisions **7 virtual machines**, each with a predefined role:

| VM Role          | Hostname        | Static IP           |
|------------------|-----------------|---------------------|
| Application      | app-server      | 192.168.122.100     |
| Web Server 01    | web-server-01   | 192.168.122.110     |
| Web Server 02    | web-server-02   | 192.168.122.120     |
| Load Balancer    | load-balancer   | 192.168.122.130     |
| Backup Server    | backup          | 192.168.122.150     |
| CI/CD Server     | ci-server       | 192.168.122.170     |
| Monitoring       | monitoring      | 192.168.122.200     |

All VMs are connected to the default libvirt network (`192.168.122.0/24`) and use
static IP addressing to ensure stable Ansible inventory generation.

---

## Default Resource Allocation

The infrastructure is intentionally sized to simulate a small production-like
environment.

Approximate total host usage:
- **RAM:** ~21.5 GB
- **vCPUs:** 13
- **Disk:** ~120 GB (thin-provisioned qcow2)

Per-VM defaults:

| VM Role        | vCPUs | RAM (MB) | Disk (GB) |
|----------------|-------|----------|-----------|
| App Server     | 3     | 3072     | 15        |
| Web Server 01  | 2     | 3072     | 15        |
| Web Server 02  | 2     | 3072     | 15        |
| Load Balancer  | 1     | 2048     | 15        |
| Monitoring     | 1     | 2048     | 15        |
| Backup         | 1     | 2048     | 20        |
| CI/CD          | 3     | 6144     | 20        |

---

## Reducing Resource Usage

This setup can be safely scaled down for systems with limited resources
(e.g. laptops with 16 GB RAM).

### Recommended adjustments
- Reduce **CI/CD server RAM** first (largest consumer)
- Reduce **web servers** to `1 vCPU / 1024 MB`
- Reduce **monitoring server RAM** if Prometheus retention is low
- Keep the load balancer small (1 vCPU is sufficient)

### Avoid
- Changing VM hostnames (used by Ansible inventory generation)
- Removing VMs without updating Ansible roles and inventory
- Changing static IPs without adjusting firewall and WireGuard configs

All resource values can be modified directly in `main.tf` per module.

---

## File Structure and Responsibilities

### `main.tf`
- Defines the libvirt provider
- Imports the Ubuntu cloud image as a base volume
- Instantiates one VM module per server role
- Controls per-VM resources (CPU, RAM, disk size, IP)

### `locals.tf`
- Collects VM IPs and hostnames into maps
- Generates a hostname → IP mapping used for Ansible integration

### `outputs.tf`
- Exposes VM IPs, hostnames, and MAC addresses
- Automatically generates Ansible `host_vars/*/ansible_host.yml`
  using a template

---

## VM Module (`modules/vm/`)

The `vm` module encapsulates all logic required to create a virtual machine.

### Key features
- Thin-provisioned qcow2 disks cloned from a base image
- Cloud-init for:
  - Hostname configuration
  - User creation
  - SSH enablement
  - Static networking
- Deterministic MAC address generation (or override support)

### Important files

- `vm.tf` - Defines disks, cloud-init ISO, and the libvirt domain
- `variables.tf` - Configurable VM parameters (CPU, RAM, disk, IP, etc.)
- `outputs.tf` - Exposes hostname, IP, and MAC
- `cloudinit.tpl` - Initial OS bootstrap configuration
- `network.tpl` - Netplan static network configuration

---

## Cloud-Init Behavior

Each VM is provisioned with:
- A non-root user (`devops`)
- Password-based SSH enabled initially (hardened later by Ansible)
- Static IP configuration via Netplan
- Hostname and `/etc/hosts` management

Security hardening is intentionally deferred to Ansible to maintain separation
between provisioning and configuration.

---

## Notes and Limitations

- This Terraform configuration targets **libvirt/QEMU only**
- Cloud providers (AWS, GCP, etc.) are not supported
- Terraform state is stored locally for simplicity
- Static IPs assume the default libvirt network (`192.168.122.0/24`)

---

## Relationship to Ansible

Terraform automatically generates Ansible host variables under:

```
ansible/host_vars/<hostname>/ansible_host.yml
```

This ensures:
- No manual inventory management
- Consistent handoff between provisioning and configuration
- Idempotent re-runs without breaking Ansible targeting

