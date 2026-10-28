# Cloud Cartographer

## Summary

This project analyzes the cost of migrating a Kubernetes-based application stack from local infrastructure to AWS and Azure. It uses measured workload data rather than assumed sizing, maps each component to equivalent managed cloud services, produces provider-specific monthly cost estimates, and documents the architectural and operational trade-offs behind those estimates.

**As per the rules set by kood/Sisu organization, the fully detailed reports are not included and the final prices is redacted.**

## About This Project

This repository analyzes migrating the Sample application (React frontend, Go backend, PostgreSQL database) plus its monitoring stack (Prometheus, Grafana, Loki, Alloy) and CI/CD tooling (GitLab, ArgoCD) from a local setup to the cloud, across separate `test` and `prod` environments.

No infrastructure was actually provisioned. Every figure in these documents comes from each provider's official pricing calculator.

Two of the three major cloud providers (AWS, Azure, GCP) were required for comparison. AWS and Azure were selected because they are the two largest public cloud providers by market share. Both were priced against the closest available region to Finland - neither provider currently operates a region physically located in Finland, so Sweden (`eu-north-1` / Sweden Central) was used as the nearest equivalent for both.

## Documents

The four documents below are meant to be read in order: infrastructure requirements are established first, then mapped to each provider's services, then priced, then assessed for risk. Each document builds on the decisions and assumptions established in the previous one.

### 1. Infrastructure Analysis Report

Describes what is being migrated and establishes the baseline infrastructure requirements. Covers the application and monitoring stack's architecture, the Kubernetes cluster topology (including the node-count difference between AKS and EKS), storage and networking design, and security requirements. Grounded in real load test measurements (CPU, RAM, disk, and network usage at 500 and 1,000 concurrent users), with dependency and data-flow diagrams showing how every component connects.

### 2. Cloud Provider Comparison

Maps every infrastructure component from the report above to its AWS and Azure equivalent - compute, storage, database, Kubernetes, networking, secrets management, and more - and documents where the two providers genuinely differ rather than just differ in naming. Also covers free tier offerings for both platforms, compares on-demand/reserved/spot pricing models, and identifies hidden costs (NAT Gateway's multi-part billing, data transfer, request-metered services, public IPv4 charges) with links to official pricing documentation for every service used.

### 3. Migration Cost Analysis

Monthly cost estimates for both providers, broken down by component category (compute, storage, database, networking, security & management), including a calculated data-transfer cost range derived from the measured traffic figures rather than left as an unexamined placeholder.

### 4. Risk Assessment & Management Plan

Cost controls (budget alerts, anomaly detection, scheduled shutdown), resource optimization strategy (instance sizing rationale, free tier usage, automated cleanup policies), risk mitigation approach (testing methodology, HA verification, backup validation, least-privilege access), and the resource tagging strategy used to make all of the above enforceable rather than aspirational.

## Key Findings

A few things surfaced during this analysis that changed the final architecture, not just its price:

- **AKS and EKS need different node topologies for equivalent redundancy.** AKS mandates a dedicated system node pool (minimum 2, recommended 3 nodes) for core cluster components; EKS has no equivalent requirement. This is why final node counts differ (AWS: 3/6 nodes test/prod across three pools; Azure: 5/10 across four pools) despite both delivering the same workload-level redundancy.
- **NAT Gateway is a three-part charge on both platforms** - an hourly rate, a per-GB data-processing fee, and standard egress on top - not the single line item it first appears to be. This was the largest single source of estimation error encountered while building the AWS estimate.
- **HA has to be verified against the specific mechanism providing it, not inferred from a line item's presence.** An early AWS estimate gave `prod` the same single-node-per-pool count as `test`, which cannot satisfy a multi-availability-zone requirement - a single node lives in exactly one AZ by definition. This was only caught by checking against the task's literal wording rather than a cost-parity target.
- **Vendor recommendations should be evaluated against actual workload requirements.** GitLab recommends an 8 vCPU non-burstable VM for a single-node deployment, but this project deliberately uses a 2 vCPU burstable instance because the expected workload is a single developer with light CI activity. The difference is documented and justified.
- **AWS restructured its free tier in mid-2025**, moving from 12 months of specific per-service allowances to a $200-credit model. Any account created for this project today falls under the newer structure - worth knowing given how much free-tier advice online still describes the older system.
- **Once both providers are configured to the same HA standard, their costs converge to within single digits of each other (~7%)**, rather than the near-exact match an earlier, under-provisioned pass produced. The closer, more honest number required no shortcuts on either side to reach.

## Assumptions & Methodology

- **Region:** Stockholm (`eu-north-1`) / Sweden Central - the nearest available region to Finland on each platform, since neither operates a region physically located here.
- **Pricing basis:** on-demand (pay-as-you-go) pricing throughout. Reserved Instances, Savings Plans, and other long-term commitments were intentionally excluded because the architecture changed substantially during the analysis.
- **Currency:** AWS figures are in USD as produced by the AWS Pricing Calculator; Azure figures are in EUR as produced by the Azure Pricing Calculator. Combined totals use 1 USD = 0.876 EUR.
- **Sizing basis:** all compute and database instance choices are derived from local load-test measurements (a Minikube cluster on an AMD Ryzen 7 3700X, 500 and 1,000 simulated concurrent users), not observed cloud production traffic. The Risk Assessment document specifies a plan to re-validate these once actually deployed.
- **Account context:** priced as a genuinely new account created specifically for the upcoming migration task, so both providers' free-tier/trial-credit offers were evaluated as directly applicable rather than theoretical.

## Headline Numbers

| | AWS | Azure |
|---|---:|---:|
| Estimated monthly cost | $<`some_amount_of_dollars`> | €<`some_amount_of_euros`> |
| Converted to EUR (1 USD = 0.876 EUR) | ≈ €<`some_amount_of_dollars_converted_to_euros`> | €<`some_amount_of_euros`> |
| Region used | eu-north-1 (Stockholm) | Sweden Central |

AWS runs roughly 7% higher once both providers are configured to the same multi-AZ HA standard for `prod` - see the Migration Cost Analysis document for the full breakdown, and the Cloud Provider Comparison document for why that gap exists structurally.

## Repository Structure

```sh
.
├── documentation
│   ├── 01-infrastructure-analysis-report.md        # Infrastructure analysis
│   ├── 02-cloud-provider-comparison.md             # Provider comparison
│   ├── 03-migration-cost-analysis.md               # Migration cost analysis
│   ├── 04-risk-assessment-and-management-plan.md   # Risk assessment
│   ├── estimates
│   │   ├── AWS_Estimate.csv                        # AWS Pricing Calculator export
│   │   └── Azure_ExportedEstimate.xlsx             # Azure Pricing Calculator export
│   └── images
│       ├── 1000-users-30-min.png                   # Grafana load test (1000 users, 30 minutes)
│       └── 500-users-30-min.png                    # Grafana load test (500 users, 30 minutes)
└── README.md                                       # This document; Summary
```

## Author

Aki Heiskanen