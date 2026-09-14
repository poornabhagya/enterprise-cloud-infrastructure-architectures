# Security & Compliance Engineering Specification

**Workload:** Canmee Dairies ERP  
**Deployment Tier:** AWS Lightsail All-in-One Topology (Option 1)  
**Document Code:** SPEC-SEC-COMP-001  
**Status:** Approved / Production Baseline

---

## 1. Compliance Baseline & Scope

This specification establishes the security governance, runtime hardening, and data protection controls implemented for the Canmee Dairies ERP system[cite: 3]. The system processes sensitive agricultural supply-chain data, daily milk collection logs, farmer accounts, and financial payroll records[cite: 3].

The security baseline aligns with standard cloud security frameworks:

- **Confidentiality:** Protection of farmer personal identifiable information (PII), payroll ledgers, and platform credentials[cite: 3].
- **Integrity:** Enforcing database ACID guarantees and schema constraints across financial records[cite: 3].
- **Availability:** Resiliency against worker freezes and automated offsite disaster recovery pipelines[cite: 2, 3].

---

## 2. Identity, Access Management & Principle of Least Privilege

### 2.1 OS-Level Privilege Separation

- **Non-Root Execution:** The application WSGI server (Gunicorn) and background workers (Celery) execute strictly under unprivileged system accounts (`www-data`)[cite: 3].
- **Interactive Shell Access:** Superuser / administrative shell access is isolated to authorized engineering keys via SSH with `PermitRootLogin no` enforced[cite: 3].
- **Filesystem Lockdown:**
  - Application codebase located at `/var/www/canmee_dairies/` is owned by `www-data:www-data` with restrictive permissions[cite: 3].
  - Secrets configuration file at `/etc/canmee/canmee.env` is restricted via `chmod 600`, permitting read access exclusively to the application runtime owner and root[cite: 3].

### 2.2 Application-Level Role-Based Access Control (RBAC)

- **Session Security:**
  - Enforces `SESSION_COOKIE_SECURE = True` to mandate transmission strictly over HTTPS[cite: 3].
  - Enforces `SESSION_COOKIE_HTTPONLY = True` to prevent client-side JavaScript access and mitigate XSS session hijacking[cite: 3].
- **Cross-Site Request Forgery (CSRF):** Middleware enforces validated `csrftoken` tokens on all state-changing HTTP methods (`POST`, `PUT`, `DELETE`)[cite: 3].
- **Permission Verification:** Verified using automated headless harnesses across all 210 static routes, ensuring critical operational modules (such as `/assets/policies/` and `/assets/depreciation/`) throw `403 PermissionDenied` when requested without explicit role authorization[cite: 3].

---

## 3. Network Defense & Ingress Filtering

### 3.1 Perimeter & Port Hardening

- Ingress traffic is strictly controlled through AWS firewall policies and Linux UFW packet filtering[cite: 3].
- Only external web ingress ports (Port `80` HTTP and Port `443` HTTPS) and administrative SSH (Port `22`) are exposed[cite: 3].
- Internal database ports (PostgreSQL `5432`), message brokers (Redis `6379`), and internal WSGI sockets (`8000`) are blocked from all external interfaces and bound exclusively to loopback addresses (`127.0.0.1`)[cite: 3].

### 3.2 Automated Transport Security

- External communications are secured using Let's Encrypt automated TLS certificates with HTTP-to-HTTPS 301 redirection enforced at the Nginx reverse proxy layer[cite: 3].
- Legacy protocols (SSLv3, TLS 1.0, TLS 1.1) are explicitly disabled, restricting handshakes to modern TLS 1.2 and TLS 1.3 standards.

---

## 4. Cryptographic Controls & Data Protection

### 4.1 Data in Transit

- All external user traffic, staff portal sessions, and administrative interactions terminate over TLS 1.3 encryption[cite: 3].
- Edge caching and static file transfers maintain secure transport headers[cite: 2, 3].

### 4.2 Data at Rest & Offsite Backups

- **Local Persistence:** PostgreSQL database tablespaces and application static/media files reside on local NVMe SSD storage[cite: 2, 3].
- **Disaster Recovery Encryption:** Nightly automated `pg_dump` database dumps are synced to Amazon S3 Standard using Server-Side Encryption with Amazon S3 managed keys (SSE-S3 AES-256)[cite: 2].
- **Lifecycle Archival:** Data transitioned to Amazon S3 Glacier Flexible Retrieval remains encrypted at rest throughout its retention lifecycle[cite: 2].

---

## 5. Vulnerability Management & Incident Telemetry

### 5.1 System Hardening & Package Governance

- Operating system packages and security patches are maintained via scheduled Ubuntu LTS security update routines[cite: 3].
- Python application dependencies are isolated within a dedicated virtual environment at `/var/www/canmee_dairies/venv/` to eliminate host-level dependency pollution[cite: 3].

### 5.2 Telemetry & Alert Dispatch

- **Operational Monitoring:** AWS Systems Manager manages centralized operational parameters[cite: 2].
- **Emergency Alerting:** Amazon SNS topic `Canmee Dairies TelemetryAlerts` is configured to deliver email and webhook notifications for operational warnings and threshold events[cite: 2].
