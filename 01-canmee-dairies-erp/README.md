# Canmee Dairies ERP — Cloud Infrastructure & Platform Engineering

**Enterprise Agricultural ERP Workload Deployment, FinOps Cost Engineering & Operational Stabilization**

---

## 1. Project Overview & Business Domain

Canmee Dairies ERP is a business-critical cloud enterprise platform built to streamline the end-to-end dairy supply chain, collection center operations, and farmer accounting workflows[cite: 4]. The platform services operational clerks, field auditors, and administrative executives handling high-volume daily transactions[cite: 4]:

- **Farmer Master & Ledger Management:** Real-time logging of morning/evening milk intake batches, density/fat metrics, and automated daily valuation[cite: 4].
- **Collection Point Financials:** Dynamic payment sheet generation with filtering by date, collection center, and branch[cite: 4].
- **Automated Payroll & Document Pipeline:** Bulk farmer payment slips, invoice calculation sheets, and asynchronous financial PDF exports[cite: 4].
- **Inventory & Asset Tracking:** Depreciation tracking and inventory valuation governed by strict Role-Based Access Control (RBAC)[cite: 4].

---

## 2. Core Technology Stack

| Layer                          | Technology Selected                   | Operational Responsibility                                                                          |
| :----------------------------- | :------------------------------------ | :-------------------------------------------------------------------------------------------------- |
| **Web Runtime & Framework**    | **Django 5.x / Python 3.12**[cite: 4] | Core business logic, ORM models, and template rendering engine[cite: 4]                             |
| **WSGI Application Server**    | **Gunicorn WSGI**[cite: 4]            | Production worker process pooling daemonized via Linux `systemd`[cite: 4]                           |
| **Edge Proxy & Web Server**    | **Nginx**[cite: 4]                    | Reverse proxying, TLS 1.3 termination, direct static/media file caching & Gzip compression[cite: 4] |
| **In-Memory Broker & Cache**   | **Redis Server (v7+)**[cite: 4]       | In-memory message broker running on loopback Port 6379[cite: 4]                                     |
| **Asynchronous Task Workers**  | **Celery**[cite: 4]                   | Offloading long-running calculations, bulk exports, and PDF rendering[cite: 4]                      |
| **Primary Relational Engine**  | **PostgreSQL 16**[cite: 4]            | Strict ACID-compliant relational data store with optimized buffer pooling[cite: 4]                  |
| **SSL / TLS Governance**       | **Let's Encrypt / Certbot**[cite: 4]  | Automated zero-touch certificate issuance and bi-daily auto-renewal crons[cite: 4]                  |
| **Disaster Recovery Pipeline** | **AWS S3 + S3 Glacier**[cite: 2]      | Nightly encrypted `pg_dump` sync with automated 30-day lifecycle transition rules[cite: 2]          |

---

## 3. Implementation Lifecycle & Engineering Milestones

The platform deployment was executed through a 9-Phase engineering roadmap to achieve high reliability and zero-cost DNS routing[cite: 4]:

1. **Phase 1: Operational Runbooks & Database Schema Rectification**  
   Formulated deployment runbooks and resolved legacy schema conflicts, duplicate table constraints, and sequential migration dependencies[cite: 4].
2. **Phase 2: Cloud Instance Provisioning & Perimeter Hardening**  
   Provisioned Ubuntu LTS base compute, configured non-root system accounts, and locked down firewall policies (exposing only Ports 22, 80, and 443)[cite: 4].
3. **Phase 3: Route 53 Bypass & DNS Ingress Stabilization**  
   Eliminated AWS Route 53 recurring fees by routing sub-domain traffic (`test.canmeedairies.lk`) directly from external DNS managers via A-records[cite: 4].
4. **Phase 4: Worker Daemonization & Secrets Isolation**  
   Configured `systemd` unit files for Gunicorn, Redis, and Celery; secured production environment variables in `/etc/canmee/canmee.env` (`chmod 600`)[cite: 4].
5. **Phase 5: Reverse Proxy, Asset Pipeline & TLS Integration**  
   Configured Nginx reverse proxying, Django `collectstatic` routing with Gzip compression, and automated Certbot SSL certificates[cite: 4].
6. **Phase 6: Bulk 500 Error Remediation**  
   Remediated missing `{% load loan_extras %}` tags across 175 templates using Bash automation, resolved `{% extends %}` hierarchy parsing errors, and registered `widget_tweaks`[cite: 4].
7. **Phase 7: System-Wide Headless Route Verification**  
   Developed a Django `RequestFactory` harness scanning all 210 static routes under superuser sessions, achieving 207 successful `200 OK` loads and 3 role-enforced `403 PermissionDenied` validations[cite: 4].
8. **Phase 8: Network Smoke Tests & Edge Validation**  
   Verified TLS 1.3 handshakes, 301 HTTPS redirections, and static asset caching via terminal network diagnostic tools (`curl -Iv`)[cite: 4].
9. **Phase 9: Operational Sign-off & Financial Smoke Tests**  
   Conducted live verification of session cookies (`sessionid`, `csrftoken`), collection point payment sheets, and bulk PDF generation workflows[cite: 4].

---

## 4. Architectural Alternatives Evaluated

To deliver the ideal balance between infrastructure cost and enterprise compliance, two formal architectural tiers were designed and documented:

- **[Option 1: AWS Lightsail Budget Tier (`architecture-options/option-1-lightsail-budget/`)](architecture-options/option-1-lightsail-budget/README.md)**  
  An all-in-one single-node blueprint utilizing AWS Lightsail's 2GB bundle ($11.77/mo) and S3 offsite backups ($0.66/mo), optimized to a **~$12.43/month** baseline by eliminating Route 53[cite: 2]. Ideal for micro-businesses and internal testing instances.
- **[Option 2: AWS EC2 Dedicated Hardened VPC (`architecture-options/option-2-ec2-dedicated-vpc-selected/`)](architecture-options/option-2-ec2-dedicated-vpc-selected/ARCHITECTURE.md)**  
  A dedicated VPC architecture (`10.0.0.0/16`) running ARM64 Graviton instances, custom subnets, and fine-grained Security Groups, engineered for multi-tenant isolation and enterprise compliance ($12.62/mo baseline).

---

## 5. Directory Structure

```text
01-canmee-dairies-erp/
├── README.md                                 <-- Master Project Documentation (This file)
├── ROADMAP.md                                <-- 9-Phase End-to-End Implementation Log
│
├── architecture-options/                     <-- Evaluated Architecture Blueprints
│   ├── option-1-lightsail-budget/            <-- Flat-Bundle Ultra-Low Cost (~$12.43/mo)
│   └── option-2-ec2-dedicated-vpc-selected/  <-- Enterprise Hardened VPC (~$12.62/mo)
│
├── environments/                             <-- Operational Runbooks & Config Templates
│   ├── staging/                              <-- Staging Environment SOPs & configs
│   └── production/                           <-- Production Deployment SOPs & configs
│
└── verification-scripts/                     <-- Quality Assurance & Scanning Harnesses
    ├── scan_urls.py                          <-- Headless 210-route validation runner
    └── trace_views.py                        <-- URL-to-View resolver analysis script
```

---

## 6. Verification & Automated Quality Assurance

All configuration files and template hierarchies are continuously validated prior to promotion:

```bash
# Execute headless verification harness across all application routes
python verification-scripts/scan_urls.py

# Trace Django URLconf routes directly to underlying view callables
python verification-scripts/trace_views.py
```
