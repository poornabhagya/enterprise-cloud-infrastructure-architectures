# ADR-001: Selection of Dedicated Cloud VM & Infrastructure Topology over Shared Hosting

- **Status:** Accepted / Implemented
- **Date:** 2026-09-14
- **Deciders:** Cloud Platform Engineering Team
- **Project Context:** Canmee Dairies ERP (Production Workload Deployment)

---

## 1. Context and Problem Statement

The Canmee Dairies ERP is a business-critical system handling farmer payroll, daily milk collection records, financial payslips, and operational reporting[cite: 2]. During deployment evaluation, the system presented several complex structural, architectural, and asynchronous requirements:

1. **Background & Asynchronous Processing:** The application requires ongoing background task execution (heavy PDF financial reports, data aggregation, and asynchronous calculations)[cite: 2]. Running these directly inside synchronous web workers freezes web threads and causes gateway request timeouts[cite: 2].
2. **Specialized Message Queuing & In-Memory State:** Execution of task queues necessitates a dedicated message broker (Redis running on port 6379) paired with systemd-managed worker daemons (Celery)[cite: 2].
3. **Database Concurrency & Schema Integrity:** The database requires strict ACID transaction compliance under PostgreSQL, along with shell-level terminal access to resolve schema mismatches, sequential migrations, and foreign-key constraint dependencies[cite: 2].
4. **Cost Constraints:** The cloud operational budget required aggressive optimization, eliminating recurring service overheads such as AWS Route 53 zone fees ($0.50+/month per tenant) while ensuring high edge performance[cite: 2].

Standard shared hosting environments (such as legacy cPanel packages) lack root access, forbid persistent background daemons (Celery/Redis), cap SSD storage at restrictive limits (e.g., 5 GB), and prevent custom process supervision.

---

## 2. Decision Drivers

- **Process Autonomy:** Full systemd daemonization capability for WSGI (Gunicorn), task workers (Celery), and in-memory key-value stores (Redis)[cite: 2].
- **Network & Ingress Control:** Fine-grained firewall rule enforcement (Security Groups / UFW) to restrict internal services (PostgreSQL 5432, Redis 6379, Gunicorn) strictly to loopback access while exposing only Ports 80 and 443[cite: 2].
- **Cost Engineering:** Total monthly infrastructure operational costs capped at under $13.00/month per tenant.
- **Environment Separation:** Complete isolation between Staging (`test.canmeedairies.lk`) and Production endpoints[cite: 2].
- **Verification & Debuggability:** Terminal-level debugging tools (Django RequestFactory harness, shell-level regex/sed tools) to inspect and bulk-resolve production issues[cite: 2].

---

## 3. Considered Options

- **Option 1: Legacy cPanel / Shared Hosting**
  - _Pros:_ Low upfront cost (~$15/year).
  - _Cons:_ No root privileges, no background daemon support (cannot run Celery workers or Redis brokers), 5GB storage cap, inability to run Django collectstatic with automated Nginx reverse proxy caching[cite: 2].
- **Option 2: AWS Lightsail Flat-Bundle Instance ($5.00/mo)**
  - _Pros:_ Fixed predictable cost, bundled SSD storage and static IPv4 address.
  - _Cons:_ Lacks multi-tier custom VPC subnet isolation, basic firewall controls, limited enterprise compliance integration.
- **Option 3: Dedicated Single-Node Cloud Instance with Hardened Perimeter & S3 Archival (Selected)**
  - _Pros:_ Full root systemd management[cite: 2], customized Nginx reverse proxy with Gzip compression[cite: 2], internal Redis/Celery worker daemonization[cite: 2], automated external DNS routing bypass[cite: 2], and offsite disaster recovery lifecycle.

---

## 4. Decision Outcome

**Chosen Decision:** Option 3 — Deployment on a dedicated Linux VPS/EC2 node running Ubuntu LTS with external DNS routing and local system daemonization[cite: 2].

### Architectural Justification:

1. **Daemon Orchestration via systemd:**  
   Created isolated system services:
   - `gunicorn-canmee.service`: Manages WSGI worker pooling communicating over a secure Unix socket[cite: 2].
   - `redis-server.service`: In-memory broker bound to localhost (`127.0.0.1:6379`)[cite: 2].
   - `celery-canmee.service`: Asynchronous task consumer ensuring PDF generation and payroll exports do not degrade HTTP request latency[cite: 2].

2. **Zero Route 53 DNS Overhead:**  
   To eliminate AWS Route 53 monthly hosted zone fees, ingress was configured by pointing the sub-domain (`test.canmeedairies.lk`) directly from the registrar/Cloudflare DNS dashboard to the static public IPv4 address using standard A Records[cite: 2].

3. **Perimeter Hardening:**  
   Security Groups and Linux UFW were hardened to expose strictly Port 80 (HTTP) and Port 443 (HTTPS)[cite: 2]. Ports 5432 (PostgreSQL) and 6379 (Redis) are completely blocked from public network interfaces[cite: 2].

4. **Automated TLS & Reverse Proxy Pipeline:**  
   Nginx acts as the perimeter reverse proxy, handling Let's Encrypt automated SSL certificate renewals via Certbot, enforcing HTTP-to-HTTPS 301 redirection, and directly offloading static/media files with Gzip compression enabled[cite: 2].

---

## 5. Implementation Roadmap & Technical Validation

The rollout was executed and certified across 9 structured implementation phases[cite: 2]:

- **Phase 1 (Schema & Runbook):** Addressed model definition mismatches, duplicate table constraints, and sequential migration dependencies using Django ORM[cite: 2].
- **Phase 2 & 3 (Compute, Security & DNS):** Provisioned the base compute instance, applied least-privilege security groups, and bound DNS records without managed zone charges[cite: 2].
- **Phase 4 & 5 (Application Stack & Proxy):** Isolated the runtime into `/var/www/canmee_dairies/venv`, configured systemd units, secured environment credentials inside `/etc/canmee/canmee.env` (`chmod 600`), and mounted Nginx TLS termination[cite: 2].
- **Phase 6 (Bulk 500 Defect Remediation):**
  - Rectified missing `{% load loan_extras %}` (`money` filter) across 175 templates using automated Bash shell scripting (`sed`)[cite: 2].
  - Re-ordered template tag inheritance hierarchies where `{% extends %}` was violated using Python automation[cite: 2].
  - Registered missing libraries (`widget_tweaks`) into Django `INSTALLED_APPS`[cite: 2].
- **Phase 7 (Automated Route Scanning):** Built a headless automated test harness using Django `RequestFactory` and `URLResolver`[cite: 2]. Executed all 210 system static routes under simulated superuser sessions; 207 passed with `200 OK`, while the remaining 3 routes correctly returned `403 PermissionDenied` validating strict RBAC policies[cite: 2].
- **Phase 8 & 9 (Smoke Tests & Operational Sign-off):** Verified SSL TLS 1.3 handshakes, verified cookie propagation (`sessionid`, `csrftoken`), and confirmed live operational generation of Collection Point Payment Sheets, Bulk Payslip calculations, and PDF/Excel exports[cite: 2].

---

## 6. Pros and Cons of the Chosen Architecture

### Advantages:

- **Zero Worker Freezes:** Offloading long-running reporting routines to Celery/Redis preserves instant response times for active field clerks[cite: 2].
- **Cost Efficiency:** Running the entire application core, broker, and database within a single optimized compute node maintains total spend at ~$12.62/month.
- **Security & Isolation:** Sensitive configurations (`SECRET_KEY`, database passwords) are decoupled from Git and stored with locked file permissions[cite: 2]. Internal ports are blocked from ingress[cite: 2].
- **Automated Reliability:** Route integrity is systematically verified prior to production sign-off[cite: 2].

### Trade-offs & Mitigations:

- **Single Compute Failure Domain:** Because web, broker, and database layers reside on one node, an instance failure affects all services.
  - _Mitigation:_ Nightly automated database dumps (`pg_dump`) synced offsite to Amazon S3 with lifecycle migration to S3 Glacier Flexible Retrieval.
- **Manual Horizontal Scaling:** Scaling compute independently from the database is not supported without splitting the layers.
  - _Mitigation:_ The workload profile of Canmee Dairies (internal operational ERP with limited concurrent clerks) operates well within single-node memory and CPU headroom[cite: 2].
