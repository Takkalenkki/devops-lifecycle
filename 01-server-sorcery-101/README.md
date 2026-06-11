# Server Sorcery 101

## Overview

Server Sorcery 101 is a hands-on documentation project that walks through the complete process of building, securing, and monitoring a small-scale virtualized infrastructure.
The goal is to simulate a production-like environment from scratch — covering VM creation, networking, user and security configuration, firewalling, VPN setup, intrusion prevention, and live monitoring with Prometheus + Grafana.
Each step is documented in detail to explain not only *how* things are done, but *why* they are done that way.

**DISCLAIMER:** You, the reviewer of this project, are not expected to set the VMs up. The end result will be showcased to you. You are however free to follow the guide if you are interested in DevOps or want to try the guide out.

## Requirements

The task required setting up multiple Ubuntu Server virtual machines to form an interconnected and secure network.
Each machine was assigned a clear role (such as web servers, application server, load balancer, or monitoring node) with defined network interfaces, static IPs, and user permissions.
Strong emphasis was placed on security best practices: SSH hardening, automated updates, and controlled access.
The extra challenge introduced advanced components like Fail2Ban, WireGuard VPN, and centralized system monitoring.

### Hardware Specs

| Component          | Specification                             |
| :----------------- | :---------------------------------------- |
| **CPU**            | AMD Ryzen 7 3700X (8 Cores / 16 Threads)  |
| **RAM**            | 32 GB DDR4 3200 MHz                       |
| **Storage**        | 500 GB SSD                                |
| **Virtualization** | KVM / QEMU on Ubuntu Host                 |
| **Network**        | libvirt bridge network (192.168.122.0/24) |

## Approach

The project was approached as if deploying a small production cluster.
I began by provisioning base Ubuntu Server VMs, configuring static IPs, and establishing internal communication.
From there, I iteratively layered on features: secure user management, SSH key enforcement, firewalls, automatic updates, and intrusion prevention.
After achieving a hardened baseline, I extended the setup with a WireGuard VPN for encrypted host communication and a separate monitoring VM hosting Prometheus + Grafana to visualize system metrics.
Every stage was thoroughly documented to provide a reproducible, educational workflow.

## End Result

The final outcome is a fully functional, hardened multi-VM environment resembling a real-world DevOps lab:
a cluster of virtual machines communicating securely, automatically maintained, and continuously monitored through modern open-source tooling.
It demonstrates the complete lifecycle from raw OS installation to operational observability, serving as both a technical showcase and a learning resource for infrastructure automation and Linux administration.

## Special Thanks

* My father for proof-reading the entirety of this document and giving feedback
* Stef van Carpels
* Ubuntu Forums
* Linux Forums
* Nixcraft
* Ubuntu man pages
* ...and all the other documentation pages I crawled through in this journey.

---

## **Ready to dive in?** [(Link to the first page of the documentation)](pages/00-introduction.md)
