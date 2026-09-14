# Enterprise Multi-Tenant Cloud Infrastructure Library

**Standardized Architecture Blueprints, Dedicated VPC Multi-Tenancy & Zero-Trust Cloud Portfolio**

![Enterprise Multi-Tenant Cloud Architecture](assets/overall-architecture.png)

---

## 1. Executive Summary

This repository hosts the centralized cloud infrastructure design, automation frameworks, and deployment specifications for an enterprise multi-tenant software ecosystem. Operating on AWS, the architecture transitions mission-critical production workloads from unmanaged, shared hosting environments to **fully isolated, cost-optimized, Dedicated VPC topologies**.

The platform is designed to service heterogeneous business workloads—ranging from transactional ERPs to real-time asynchronous engines—while maintaining consistent security governance, containerized releases via Amazon ECR, zero Route 53 DNS costs, and automated disaster recovery pipelines.

---

## 2. Core Architectural Principles

- **True Workload Isolation (Dedicated VPC per Tenant):**  
  Each client tenant runs inside a dedicated Virtual Private Cloud (VPC) with non-overlapping RFC 1918 CIDR blocks (`10.0.0.0/16`, `10.1.0.0/16`, `10.2.0.0/16`). A fatal breach, traffic spike, or memory leak in one tenant domain cannot cross tenant boundaries.
- **Cost-Engineered Compute Layer:**  
  Workloads utilize ARM64 Graviton-optimized compute instances (`t4g.medium` / `t4g.small`) pairing production application processes, asynchronous task workers (Celery/Python), and local ACID relational databases (PostgreSQL) within a single hardened node. This achieves an enterprise baseline cost of **~$12.62/month per tenant**.
- **Zero-Cost Edge Routing & Ingress:**  
  DNS resolution is terminated externally via Registrar/Cloudflare CNAME/ALIAS records to **AWS CloudFront Edge Distributions**. This mitigates DNS hosted zone fees ($0.50+/mo per zone) while providing global TLS offloading, DDoS absorption, and static asset caching under the AWS Always-Free Tier.
- **Immutable Containerized Releases:**  
  Centralized CI/CD builds container artifacts and registers them into **Amazon Elastic Container Registry (ECR)**. Compute nodes pull version-tagged images directly over internal AWS backbone endpoints.
- **Air-Gapped Disaster Recovery Pipeline:**  
  All stateful database engines execute encrypted snapshot runs (`pg_dump`) nightly. Backups are shipped to **Amazon S3 Standard** (SSE-S3 AES-256) and transitioned to **S3 Glacier Flexible Retrieval** after 30 days via strict automated lifecycle policies.

---

## 3. High-Level Portfolio & Workload Matrix

| Identifier    | Workload Name                 | Primary Technology Stack      | Subnet Topology                                 | Data Persistence & Async Tier                   | Status               |
| :------------ | :---------------------------- | :---------------------------- | :---------------------------------------------- | :---------------------------------------------- | :------------------- |
| **Tenant 01** | **Canmee Dairies ERP**        | Django WSGI, Gunicorn, Nginx  | Staging (`10.0.2.0/24`)<br>Prod (`10.0.1.0/24`) | PostgreSQL 16, Redis 7.x, Celery, S3 + Glacier  | **Production Ready** |
| **Tenant 02** | **Hardware Renting Platform** | Node.js / Express, Nginx      | Staging (`10.1.2.0/24`)<br>Prod (`10.1.1.0/24`) | PostgreSQL 16, EBS Volume Snapshots, S3 Glacier | _Blueprint Defined_  |
| **Tenant 03** | **Restaurant POS Engine**     | FastAPI, WebSocket Hub, Nginx | Staging (`10.2.2.0/24`)<br>Prod (`10.2.1.0/24`) | PostgreSQL 16, Redis Cache Store, S3 Glacier    | _Blueprint Defined_  |

---

## 4. Multi-Environment Promotion Lifecycle

Each tenant repository maintains a strictly mirrored two-tier environment workflow:
