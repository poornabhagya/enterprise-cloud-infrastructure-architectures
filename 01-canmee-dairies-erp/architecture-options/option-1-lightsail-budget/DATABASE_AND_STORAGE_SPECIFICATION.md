# Database & Storage Engineering Specification

**Workload:** Canmee Dairies ERP  
**Deployment Tier:** AWS Lightsail All-in-One Topology (Option 1)  
**Document Code:** SPEC-DB-STR-001  
**Status:** Approved / Production Baseline

---

## 1. Relational Database Engine Architecture

### 1.1 Core Engine Parameters & Hosting Topology

- **Engine Type:** PostgreSQL 16 (Relational ACID Engine)[cite: 3]
- **Hosting Model:** Localized engine co-located within the AWS Lightsail 2GB instance boundary[cite: 2, 3]
- **Binding & Interface:** Listening strictly on loopback interface (`127.0.0.1:5432`); blocked from all external network interfaces[cite: 3]
- **Authentication Method:** `scram-sha-256` password hashing with application credentials injected at runtime via `/etc/canmee/canmee.env`[cite: 3]

### 1.2 Memory Allocation & Buffer Optimization (2GB Host Sizing)

Because PostgreSQL shares physical host memory with Nginx, Gunicorn, and Celery, runtime parameters are tuned for low-memory footprint stability:

- `shared_buffers = 256MB` (25% of available free RAM baseline)
- `work_mem = 16MB` (Limits per-operation sort/hash memory to prevent OOM panics)
- `maintenance_work_mem = 64MB` (Dedicated memory for migrations and vacuuming)
- `effective_cache_size = 768MB` (Informs planner of available OS cache capacity)
- `max_connections = 50` (Connection pool bounded via Gunicorn concurrency)

### 1.3 Schema Migration, Foreign-Key Alignment & Integrity Management

- **Migration Orchestration:** Django ORM migrations applied sequentially against the clean state[cite: 3].
- **Historical Conflict Resolution:** Resolved structural schema conflicts, constraint dependencies, and duplicate table errors through model-definition synchronization and targeted sequential migration paths[cite: 3].
- **Foreign-Key Safeguards:** Foreign-key constraints and column nullability validations enforced at engine level across critical entities (Farmer balances, Milk Collections, Route Invoices, and Payroll ledgers)[cite: 3].

---

## 2. In-Memory State & Message Broker (Redis)

### 2.1 Service Configuration

- **Engine:** Redis Server (Advanced In-Memory Key-Value Store)[cite: 3]
- **Host & Port:** `127.0.0.1:6379` (Local loopback only)[cite: 3]
- **Process Supervision:** Daemonized via Linux systemd (`redis-server.service`)[cite: 3]

### 2.2 Memory Bounds & Eviction Policy

- `maxmemory = 128MB` (Strict upper memory ceiling protecting host RAM)
- `maxmemory-policy = volatile-lru` (Evicts expiring cache keys under memory pressure while retaining active Celery task queues)
- **Persistence Strategy:** RDB snapshots enabled for failure recovery; heavy append-only file (AOF) disabled to eliminate continuous disk write thrashing on local SSD.

---

## 3. Persistent Local Storage Allocation (NVMe SSD)

### 3.1 Host Disk Partitioning & Layout

The single-node workload utilizes the Lightsail 40GB NVMe SSD allocation:

| Mount / Path                     | Usage Classification                             | Quota / Expected Footprint | Storage Characteristics                     |
| :------------------------------- | :----------------------------------------------- | :------------------------- | :------------------------------------------ |
| `/` (Root Partition)             | OS Base Packages & Application Runtime           | ~8.0 GB                    | Ext4 filesystem, high IOPS                  |
| `/swapfile`                      | Linux Virtual Swap Partition                     | 2.0 GB                     | Non-fragmented allocation; OOM buffer       |
| `/var/lib/postgresql/16/main`    | PostgreSQL Relational Tablespaces & WAL          | 8.0 GB - 15.0 GB           | Write-ahead logging enabled                 |
| `/var/www/canmee_dairies/static` | Compiled Static Assets (CSS, JS, Fonts)[cite: 3] | ~500 MB                    | Served directly by Nginx with Gzip[cite: 3] |
| `/var/www/canmee_dairies/media`  | Uploaded Invoices, Slips & Binary Media          | ~5.0 GB                    | File permissions `chmod 750`                |
| `/var/backups/postgres`          | Ephemeral Daily Dump Staging Directory           | ~2.0 GB                    | Auto-purged after S3 sync                   |

---

## 4. Offsite Disaster Recovery Pipeline (Amazon S3 & Glacier)

### 4.1 Automated Nightly Backup Flow

Database backups are fully decoupled from local storage to guard against instance or filesystem failures:

1. **Local Dump Execution:**  
   A nightly scheduled cron execution triggers a compressed, consistent database dump:  
   `pg_dump -U canmee_user -h 127.0.0.1 -Fc canmee_prod > /var/backups/postgres/canmee_$(date +%F).dump`

2. **Client-Side Encryption & Transmission:**  
   The resulting binary dump is pushed to an encrypted private Amazon S3 bucket using the AWS CLI over TLS:  
   `aws s3 cp /var/backups/postgres/canmee_$(date +%F).dump s3://canmee-dairies-backups-ap-south-1/database/ --sse AES256`

3. **Local Cleanup:**  
   Upon confirmed upload, local backup artifacts older than 48 hours are automatically purged to preserve SSD capacity:  
   `find /var/backups/postgres/ -type f -name "*.dump" -mtime +2 -delete`

### 4.2 S3 Storage Lifecycle & Glacier Archival Policy

- **Bucket Sizing Baseline:** 20 GB monthly capacity allocated for daily dumps and document archives[cite: 2].
- **Hot Retention Tier (S3 Standard):** Snapshots remain in Amazon S3 Standard for 30 days to facilitate instant, zero-latency Point-in-Time operational restores[cite: 2].
- **Cold Archival Tier (S3 Glacier Flexible Retrieval):** An automated S3 Lifecycle rule transitions database snapshots older than 30 days into S3 Glacier Flexible Retrieval[cite: 2].
- **Retention Horizon:** Glacier archives are automatically expired and deleted after 365 days, keeping storage costs optimized at ~$0.66/month[cite: 2].

---

## 5. Recovery Time & Recovery Point Objectives

- **Recovery Point Objective (RPO):** Maximum 24 hours (governed by the nightly automated snapshot cron).
- **Recovery Time Objective (RTO):** Less than 45 minutes (retrieval of the latest S3 Standard dump followed by a `pg_restore` onto a newly provisioned instance).
