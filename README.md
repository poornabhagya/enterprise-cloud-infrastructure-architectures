# Enterprise Multi-Tenant Cloud Infrastructure Library

**Standardized Architecture Blueprints, Dedicated VPC Isolation & FinOps-Driven Workload Topologies**

![Enterprise Multi-Tenant Cloud Architecture](overall%20archi.png)

---

## 1. System Description & Architectural Intent

This repository establishes a standardized, enterprise-grade cloud architecture framework designed to host heterogeneous business workloads on AWS under strict multi-tenant isolation.

Rather than relying on brittle shared-hosting setups or costly unmanaged clusters, this blueprint defines a scalable multi-tier pattern:

- **True Tenant Isolation:** Dedicated VPC per tenant (`10.0.0.0/16`, `10.1.0.0/16`, `10.2.0.0/16`) ensuring zero noisy-neighbor interference, independent route tables, and strict boundary containment.
- **Dual-Tier Architectural Flexibility:** Provides two distinct delivery models:
  - **Option 1 (AWS Lightsail Budget @ ~$12.43/mo):** Predictable flat-rate compute tier for cost-sensitive or micro-tier workloads[cite: 2].
  - **Option 2 (Dedicated EC2 VPC @ ~$12.62/mo):** Production-hardened ARM64 Graviton topology for enterprise compliance and private subnet isolation.
- **FinOps-Driven Edge & Ingress:** Zero Route 53 DNS hosted zone overheads achieved via external Registrar / Cloudflare DNS management terminating on AWS CloudFront distributions and static IPv4 perimeters[cite: 4].
- **Unified Platform Governance:** Shared cross-tenant telemetry via Amazon CloudWatch, immutable image distribution via Amazon ECR, parameter encryption via AWS Systems Manager Parameter Store, and proactive operational alerts through Amazon SNS[cite: 2].
- **Automated Cold Archival:** Continuous offsite disaster recovery via automated daily snapshot transfers to Amazon S3 Standard, with policy-based lifecycle migration into Amazon S3 Glacier Flexible Retrieval[cite: 2].

---

## 2. Core Architectural Principles

- **True Workload Isolation (Dedicated VPC per Tenant):**  
  Each client tenant runs inside a dedicated Virtual Private Cloud (VPC) with non-overlapping RFC 1918 CIDR blocks. A fatal breach, traffic spike, or memory leak in one tenant domain cannot cross tenant boundaries.
- **Cost-Engineered Compute Layer:**  
  Workloads pair production WSGI/HTTP application engines, asynchronous task workers (Celery/Python), and local ACID relational databases (PostgreSQL) within a single hardened node, maintaining enterprise hosting costs below $13.00/month per tenant[cite: 4].
- **Zero-Cost Edge Routing & Ingress:**  
  DNS resolution is terminated externally via Registrar/Cloudflare CNAME/ALIAS records to AWS CloudFront Edge Distributions and public endpoints, eliminating recurring Route 53 zone fees ($0.50+/mo per zone)[cite: 2, 4].
- **Immutable Containerized Releases:**  
  Centralized CI/CD builds container artifacts and registers them into Amazon Elastic Container Registry (ECR), allowing compute nodes to pull version-tagged images directly over internal AWS backbone endpoints.
- **Air-Gapped Disaster Recovery Pipeline:**  
  All stateful database engines execute encrypted snapshot runs (`pg_dump`) nightly[cite: 4]. Backups are shipped to Amazon S3 Standard (SSE-S3 AES-256) and transitioned to S3 Glacier Flexible Retrieval after 30 days via automated lifecycle policies[cite: 2].

---

## 3. High-Level Portfolio & Workload Matrix

| Identifier    | Workload Name                 | Primary Technology Stack              | Subnet Topology                                 | Data Persistence & Async Tier                           | Status               |
| :------------ | :---------------------------- | :------------------------------------ | :---------------------------------------------- | :------------------------------------------------------ | :------------------- |
| **Tenant 01** | **Canmee Dairies ERP**        | Django WSGI, Gunicorn, Nginx[cite: 4] | Staging (`10.0.2.0/24`)<br>Prod (`10.0.1.0/24`) | PostgreSQL 16, Redis 7.x, Celery, S3 + Glacier[cite: 4] | **Production Ready** |
| **Tenant 02** | **Hardware Renting Platform** | Node.js / Express, Nginx              | Staging (`10.1.2.0/24`)<br>Prod (`10.1.1.0/24`) | PostgreSQL 16, EBS Snapshots, S3 Glacier                | _Blueprint Defined_  |
| **Tenant 03** | **Restaurant POS Engine**     | FastAPI, WebSocket Hub, Nginx         | Staging (`10.2.2.0/24`)<br>Prod (`10.2.1.0/24`) | PostgreSQL 16, Redis Cache Store, S3 Glacier            | _Blueprint Defined_  |

---

## 4. Multi-Environment Promotion Lifecycle

Each tenant workload implements a two-tier environment progression model:

```text
[ Local Developer Feature Branch ]
               |
               v
[ Staging Subnet / Instance ]  <--- (test.tenant.lk)
    - DEBUG=True
    - Mock Database & Dummy Fixtures
    - Headless Verification Harness (scan_urls.py - 210 Routes)
               |
               v  (Promoted via Validated Git Release Tag)
[ Production Subnet / Hardened Node ]  <--- (app.tenant.lk)
    - DEBUG=False
    - Production ACID PostgreSQL 16
    - Multi-Worker Gunicorn WSGI + Systemd Daemons
    - Zero-Downtime Reload (systemctl reload / SIGHUP)
```

---

## 5. Platform Governance & Operational Telemetry

All tenant workloads report into a centralized operations and telemetry plane:

1. **AWS Systems Manager (Parameter Store):**  
   Manages production environment variables, database credentials, and secret strings with KMS encryption, preventing plaintext secrets in version control[cite: 2].
2. **Amazon CloudWatch Agent:**  
   Collects host metrics (CPU utilization, physical RAM consumption, swap file activity, and disk I/O) across compute instances.
3. **Amazon SNS Telemetry:**  
   Dispatches instant operational email alerts whenever compute memory exceeds 85% utilization or when automated backup cron jobs report non-zero exit codes[cite: 2].

---

## 6. Repository Navigation & Case Studies

- **Case Study 01: [Canmee Dairies ERP — Architecture & Implementation](01-canmee-dairies-erp/README.md)**
  - **Option 1 Blueprint:** [AWS Lightsail Budget Architecture (~$12.43/mo)](01-canmee-dairies-erp/architecture-options/option-1-lightsail-budget/README.md)
  - **Option 2 Blueprint:** [AWS EC2 Dedicated Hardened VPC (~$12.62/mo)](01-canmee-dairies-erp/architecture-options/option-2-ec2-dedicated-vpc-selected/README.md)
  - **Implementation Roadmap:** [9-Phase End-to-End Delivery Roadmap](01-canmee-dairies-erp/ROADMAP.md)
  - **Automated Verification:** [Headless Route Scanner & View Tracer](01-canmee-dairies-erp/verification-scripts/)
- **Case Study 02:** [Hardware Renting Platform Blueprint](02-hardware-renting-system/README.md)
- **Case Study 03:** [Restaurant POS Engine Blueprint](03-restaurant-pos-system/README.md)

---

_Maintained by the Cloud Infrastructure & Platform Engineering Team._
