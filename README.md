# My DevOps Lifecycle Progression

In this repository I have bundled together the READMEs and task descriptions for the individual tasks of kood/Sisu DevOps module.

The module consists of 8 tasks; 7 mandatories and 1 optional.

Tasks in order:
1. Server Sorcery 101
2. Infrastructure Insight
3. Automation Alchemy
4. Sherlock Logs
5. Cluster Chronicles
6. GitOps Galaxy
7. Cloud Cartographer
8. Voyager (optional)

As per the kood/Sisu rules, I will only disclose the source code and configuration files to potential recruiters upon request.

## Brief Task Explanations

### Server Sorcery 101

The aim of this [task](/01-server-sorcery-101/README.md) was to install and configure virtual machines running Ubuntu. The task included VM creation, networking, user and security configuration, firewalling, VPN setup, intrusion prevention, and live monitoring with Prometheus + Grafana.

### Infrastructure Insight

The aim of this [task](/02-infrastructure-insight/README.md) was to set up a secure, containerized infrastructure and deploy a diagnostic web application across multiple servers. The task included preparing application, web, and load-balancer servers with containerization tools, configuring network/firewall rules, and ensuring inter-server communication.

The diagnostic app was written specifically for this task. It includes a modular frontend and backend with an endpoint exposing system detail (hostname, OS, CPU, memory, etc.) and it is designed for containerized deployment.

This task is a continuation of the previous task, Server Sorcery 101.

### Automation Alchemy

The aim of this [task](/03-automation-alchemy/README.md) was to automate the previous tasks: Server Sorcery 101 and Infrastructure Insight. It built upon those tasks to also feature a CI/CD pipeline for the app deployment, as well as comprehensive testing, operational rollback mechanism and pipeline notifications.

The entire deployment from creating the VMs to full configured infrastructure was automated via a deployment script.

### Sherlock Logs

The aim of this [task](/04-sherlock-logs/README.md) was to implement a centralized, production-grade monitoring and logging platform to support a growing, distributed infrastructure.

Monitoring was built around Prometheus for metrics collection and Grafana for visualization. The existing infrastructure metrics application was modified to expose a Prometheus-compatible metrics endpoint.

Logging was centralized using the ELK stack. The application was updated to ensure structured log delivery.

All components were integrated into the existing automation and CI/CD pipelines, ensuring monitoring and logging agents are depoloyed automatically with new infrastructure. The final system delivers continuous observability, proactive alerting, faster incident response, and data-driven operational insight across the entire environment.

### Cluster Chronicles

The aim of this [task](/05-cluster-chronicles/README.md) was to migrate from the previous VM-based approach to a local Kubernetes cluster on Minikube.

### GitOps Galaxy

The aim of this [task](/06-gitops-galaxy/README.md) was to implement a full GitOps deployment pipeline for a multi-component web application, built on Kubernetes (Minikube / KVM2). Application deployments were managed by ArgoCD, infrastructure was packaged with Helm, secrets were stored in HashiCorp Vault via the Vault Secrets Operator, and image promotion was automated by ArgoCD Image Updater. A Jenkins CI/CD pipeline tied everything together from source commit to production.

### Cloud Cartographer

The aim of this task is to conduct a cost analysis for migrating a _Sample application_, which consists of a React frontend, Go backend, and PostgreSQL database.

The application is deployed to two environments: `test` and `prod`.

The goal is to assess the infrastructure and calculate the cost for two out of three leading cloud providers: **Amazon Web Services**, **Google Cloud Platform** and **Microsoft Azure**.

This task is still under development, therefore no README is provided yet.

### Voyager (Optional)

The aim of this task is to use the previous cost analysis from Cloud Cartographer, and perform the cloud migration for _Sample application_.

This task is still under development, therefore no README is provided yet.

## Author

Aki Heiskanen