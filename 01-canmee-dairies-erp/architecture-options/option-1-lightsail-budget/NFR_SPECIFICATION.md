# Non-Functional Requirements (NFR) Specification

**Workload:** Canmee Dairies ERP  
**Deployment Tier:** AWS Lightsail All-in-One Topology (Option 1)  
**Document Code:** SPEC-NFR-001  
**Status:** Approved / Production Baseline

---

## 1. Purpose & Scope

This specification defines the Non-Functional Requirements (NFRs) governing performance, availability, data integrity, security, and recoverability for the Canmee Dairies ERP platform deployed on AWS Lightsail[cite: 2, 3]. It establishes measurable operational benchmarks ensuring the system meets business service-level expectations under cost-optimized cloud parameters[cite: 2, 3].

---

## 2. Performance & Latency Requirements

### 2.1 Web Request Responsiveness

- **Interactive UI Response Time:** 95% of standard HTTP GET/POST operations (e.g., dashboard navigation, farmer list lookups, attendance updates) must complete within **$\le$ 300 ms** under regular concurrent load[cite: 3].
- **Static Asset Loading:** CSS, JavaScript modules, fonts, and Bootstrap assets must deliver with HTTP 200 OK status in **$\le$ 150 ms**, served directly from the Nginx filesystem layer with Gzip compression enabled[cite: 3].
- **DNS Resolution Time:** External DNS resolution to the public static IP must resolve globally within **$\le$ 50 ms** via Registrar / Cloudflare Anycast networks, bypassing Route 53 managed zone overhead[cite: 2, 3].

### 2.2 Asynchronous Execution & Worker Decoupling

- **Web Thread Non-Blocking SLA:** Heavy reporting routines, bulk payment sheet calculations, and PDF generation must **never execute synchronously** within Gunicorn WSGI request threads[cite: 3].
- **Task Queuing Overhead:** Dispatch latency from Django application core to the local Redis message broker on Port 6379 must not exceed **$\le$ 10 ms**[cite: 3].
- **Background Task Processing:** Celery worker daemons must pick up and initiate heavy export tasks within **$\le$ 2 seconds** of queue entry[cite: 3].

---

## 3. Availability & Operational Reliability

### 3.1 Service Uptime

- **Target Availability:** The system targets an operational uptime of **99.5%** during active operating hours (05:00 to 22:00 IST), aligning with field milk collection and payment processing schedules[cite: 3].
- **Process Self-Healing:** Web WSGI (Gunicorn), database engines (PostgreSQL), task workers (Celery), and broker instances (Redis) must be daemonized via Linux `systemd` with `Restart=always` policies to recover automatically from unexpected process crashes[cite: 3].

### 3.2 Automated Error Detection & Route Integrity

- **Automated Route Health:** Pre-release verification mandates 100% headless validation across all 210 static routes using Django `RequestFactory`, where non-permitted routes strictly throw `403 PermissionDenied` rather than uncaught `500 Internal Server Error` exceptions[cite: 3].

---

## 4. Disaster Recovery & Data Durability (DR SLA)

### 4.1 Recovery Metrics

- **Recovery Point Objective (RPO):** **$\le$ 24 Hours.** Enforced via automated daily `pg_dump` cron jobs executing every midnight[cite: 1, 3].
- **Recovery Time Objective (RTO):** **$\le$ 45 Minutes.** In the event of catastrophic compute failure, a new instance can be provisioned, configured via operational runbooks, and populated using the latest verified dump archive[cite: 3].

### 4.2 Backup Durability & Storage Transition

- **Offsite Storage Durability:** Database snapshots shipped to Amazon S3 Standard guarantee 99.999999999% (11 9's) data durability[cite: 2].
- **Cost-Managed Retention Policy:** Snapshots must automatically transition from S3 Standard to Amazon S3 Glacier Flexible Retrieval at day 30, with automatic expiration after 365 days to maintain the storage spend within the ~$0.66/month allocation[cite: 2].

---

## 5. Security, Access & Compliance Requirements

### 5.1 Perimeter & Port Isolation

- **Zero Public Exposure of Backends:** PostgreSQL (Port 5432) and Redis (Port 6379) must be bound strictly to loopback (`127.0.0.1`) and completely blocked at the cloud firewall boundary[cite: 3].
- **Public Ingress Boundary:** Only Port 80 (HTTP redirect) and Port 443 (HTTPS) may be exposed publicly, alongside Port 22 for secure shell administration[cite: 3].

### 5.2 Transport & Cryptographic Standards

- **Transport Encryption:** 100% of external web traffic must terminate over TLS 1.2 or TLS 1.3 with automated Let's Encrypt renewal lifecycle managed via Certbot[cite: 3].
- **Data-at-Rest Protection:** S3 backup buckets must enforce default server-side encryption (SSE-S3 AES-256) for all ingested artifacts[cite: 2].

### 5.3 Secrets & Session Governance

- **Zero Secrets in Source Control:** Database credentials, Django `SECRET_KEY`, and internal tokens must reside exclusively within `/etc/canmee/canmee.env` with `chmod 600` access restricted to system users[cite: 3].
- **Session Integrity:** Cookie parameters must enforce `SESSION_COOKIE_SECURE = True` and `SESSION_COOKIE_HTTPONLY = True`, complemented by CSRF token enforcement on all state-altering requests[cite: 3].

---

## 6. Maintainability & Operability

- **Configuration Reproducibility:** Server provisioning, daemon orchestration, and Nginx reverse proxy mappings must be maintained as versioned template files and standardized runbooks[cite: 3].
- **Operational Telemetry:** System parameter updates and monitoring alerts must interface with AWS Systems Manager and Amazon SNS to dispatch email notifications whenever host alarms are tripped[cite: 2].
