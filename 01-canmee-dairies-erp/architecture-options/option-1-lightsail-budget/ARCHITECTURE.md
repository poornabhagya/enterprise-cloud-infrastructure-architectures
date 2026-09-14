# AWS Lightsail Budget Architecture Specification

**Option 1: Single-Node Flat-Rate Topology (~$5.00/mo)**

---

## 1. Architectural Blueprint

![Canmee Dairies AWS Lightsail Architecture](canmee_lightsail-archi.png)

This architecture defines an all-in-one single-node cloud deployment model on AWS Lightsail, designed for ultra-low operational expenditure while hosting full application, broker, and database tiers on a single virtual server.

---

## 2. Component Specifications

### 2.1 Compute & OS Tier

- **Service:** AWS Lightsail Virtual Private Server (VPS)
- **Operating System:** Ubuntu 24.04 LTS (x86_64 or ARM64)
- **Instance Sizing:** 1 vCPU, 2 GB Memory, 40 GB NVMe SSD
- **Memory Management:** Mandatory 2 GB active swap space configured at `/swapfile` to prevent Out-Of-Memory (OOM) errors during simultaneous Django report generation and Celery tasks.

### 2.2 Network, Ingress & Security Plane

- **Public Ingress:** Static Public IPv4 Address (bundled at zero additional cost).
- **DNS Resolution:** External DNS Registrar / Cloudflare dashboard pointing directly to the Static Public IP via standard A-Records, bypassing AWS Route 53 zone fees ($0.00 DNS cost).
- **Perimeter Firewall (Lightsail Firewall & Linux UFW):**
  - **Port 22 (SSH):** Restricted ingress for administrator access.
  - **Port 80 (HTTP):** Public ingress (301 redirected to Port 443).
  - **Port 443 (HTTPS):** Public ingress for web application traffic.
  - **Internal Ports (5432, 6379, 8000):** Strictly bound to loopback interface (`127.0.0.1`) and blocked from external ingress.

### 2.3 Edge Acceleration & Reverse Proxy

- **Web Server:** Nginx (stable)
- **TLS / SSL Termination:** Let's Encrypt certificates managed via automated `certbot` renewal crons.
- **Performance Enhancements:**
  - Direct file-system offloading for Django static assets and user-uploaded media.
  - Gzip compression enabled for text/html, application/javascript, and text/css assets.
  - Reverse proxy routing traffic via local Unix domain socket (`/run/gunicorn.sock`) or `127.0.0.1:8000`.

### 2.4 Application & Background Worker Plane

- **Web WSGI Engine:** Gunicorn WSGI master process with 2 synchronous workers daemonized via `systemd` (`gunicorn-canmee.service`).
- **Message Broker:** Redis server (`redis-server.service`) bound to `127.0.0.1:6379`.
- **Asynchronous Processing:** Celery worker daemon (`celery-canmee.service`) consuming tasks from Redis to process heavy PDF payslips and calculation workloads off the HTTP request path.
- **Secrets Governance:** Environment variables injected at runtime from `/etc/canmee/canmee.env` with strict `chmod 600` access restricted to the application service user.

### 2.5 Relational Database & Persistence Tier

- **Database Engine:** PostgreSQL 16 server running locally.
- **Connection Interface:** Listening strictly on `127.0.0.1:5432`.
- **Tablespace Storage:** Backed by Lightsail 40 GB NVMe SSD storage.

### 2.6 Offsite Disaster Recovery Pipeline

- **Snapshot Mechanism:** Nightly automated execution of compressed, encrypted `pg_dump` runs via Linux cron.
- **Hot Storage:** Snapshots synced directly to an Amazon S3 Standard bucket using SSE-S3 (AES-256 default encryption).
- **Cold Storage Lifecycle:** S3 lifecycle transition rule migrates snapshots older than 30 days to AWS S3 Glacier Flexible Retrieval for cost-efficient long-term retention.

---

## 3. Data & Request Flow

1. **Client Request:** External clients initiate HTTPS requests to `test.canmeedairies.lk`.
2. **DNS & Firewall:** Registrar DNS resolves the hostname to the Lightsail static IP; Lightsail Firewall permits traffic on Port 443.
3. **Nginx Routing:** Nginx handles TLS termination and serves static/media files directly from disk. Dynamic requests pass via Unix domain socket to Gunicorn.
4. **App Execution:** Django executes business logic, committing transactions to PostgreSQL over `127.0.0.1:5432`.
5. **Async Offloading:** Long-running export or calculation jobs are dispatched to Redis (`127.0.0.1:6379`). Celery workers pull and process tasks in the background without blocking web responses.
6. **Disaster Recovery:** Scheduled cron jobs dump the database state and push encrypted archives offsite to Amazon S3.

---

## 4. Operational Boundaries & Constraints

- **Compute Bottlenecks:** CPU burst credits apply on low-tier Lightsail instances; sustained heavy computational processing will throttle the CPU.
- **Network Isolation:** Unlike AWS EC2 with custom VPCs, Lightsail does not support multi-tier private subnets, NAT gateways, or fine-grained Network Access Control Lists (NACLs).
- **Horizontal Scaling:** Transitioning the database or workers onto separate instances requires manual data migration or moving to full AWS EC2/RDS services.
