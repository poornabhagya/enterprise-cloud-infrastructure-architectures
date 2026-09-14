# High-Level Architecture (HLA) Specification

**Workload:** Canmee Dairies ERP  
**Deployment Tier:** AWS Lightsail All-in-One Topology (Option 1)  
**Document Code:** SPEC-HLA-001  
**Status:** Approved / Production Baseline

---

## 1. System Overview & Architectural Objectives

The Canmee Dairies ERP architecture is designed to deliver a cost-optimized, secure, and self-contained production environment[cite: 1, 2, 3]. The high-level topology integrates secure edge routing, application processing, background task management, and offsite disaster recovery into an efficient, single-node cloud footprint[cite: 1, 2, 3].

---

## 2. High-Level Architectural Layers

### 2.1 Edge Ingress & DNS Layer

- **Domain & DNS Routing:** Bypasses Amazon Route 53 hosted zone overhead by using external domain registrars or Cloudflare dashboards to point sub-domains (`test.canmeedairies.lk`) directly to the server's static public IP via A-records[cite: 1, 3].
- **Edge Acceleration & TLS:** Utilizes Amazon CloudFront free-tier plans and Nginx with automated Let's Encrypt Certbot SSL for HTTPS redirection and secure handshakes[cite: 1, 2, 3].

### 2.2 Compute & Application Runtime Tier

- **Compute Instance:** AWS Lightsail Linux instance (Ubuntu LTS) provisioned with a 2GB bundle configuration[cite: 2, 3].
- **Perimeter Security:** Hardened using Linux UFW and security groups, opening only Ports 22 (SSH), 80 (HTTP), and 443 (HTTPS) while blocking public access to internal ports such as PostgreSQL (5432), Redis (6379), and Gunicorn (8000)[cite: 1, 3].
- **WSGI & Application Server:** Django application core served via Gunicorn WSGI processes daemonized through systemd[cite: 1, 3].
- **Static & Media Asset Pipeline:** Managed via Django `collectstatic` and served directly through Nginx reverse proxy layers with compression enabled[cite: 1, 3].

### 2.3 Asynchronous Processing & State Management Tier

- **Message Broker:** Redis server configured as an advanced in-memory key-value store running on port 6379[cite: 1, 3].
- **Background Tasks:** Celery workers daemonized via systemd to asynchronously process heavy operations (such as PDF generation, data processing, and large reports) to prevent web worker timeouts[cite: 1, 3].

### 2.4 Data Persistence & Disaster Recovery Tier

- **Relational Database:** PostgreSQL 16 engine handling core application tables, farmer records, and financial ledgers, backed by resolved schema alignments and sequential migrations[cite: 1, 3].
- **Offsite Backups & Archival:** Amazon S3 Standard storage paired with an automated lifecycle transition rule into Amazon S3 Glacier Flexible Retrieval for cost-effective long-term retention of database dumps[cite: 2].

### 2.5 Platform Governance & Telemetry Plane

- **Secret Management:** AWS Systems Manager (Parameter Store) used for managing encrypted parameters and runtime configurations injected securely via local environment files[cite: 2, 3].
- **Operational Monitoring:** Amazon SNS configured with designated alert topics to dispatch email and HTTP/S notifications for telemetry and operational events[cite: 2].
