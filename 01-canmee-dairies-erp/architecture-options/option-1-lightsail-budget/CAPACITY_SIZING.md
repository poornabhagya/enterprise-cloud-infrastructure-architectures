# Capacity Sizing & Resource Allocation Specification

**Workload:** Canmee Dairies ERP  
**Deployment Tier:** AWS Lightsail All-in-One Topology (Option 1)  
**Document Code:** SPEC-CAP-SIZE-001  
**Status:** Approved / Production Baseline

---

## 1. Executive Summary & Host Hardware Profile

This capacity sizing specification establishes compute, memory, storage, and throughput headroom boundaries for Canmee Dairies ERP deployed on the single-node AWS Lightsail bundle tier[cite: 2, 3].

### Base Hardware Footprint

- **Compute Tier:** 1 vCPU (Burstable performance baseline)[cite: 2]
- **Memory Capacity:** 2.0 GB Physical RAM + 2.0 GB Linux Swap Allocation (4.0 GB Total Virtual Memory)[cite: 2]
- **Local Disk Storage:** 40.0 GB NVMe SSD[cite: 2]
- **Network & Transfer Allowance:** 1.0 TB monthly outbound data transfer quota bundled with static IPv4[cite: 2]

---

## 2. Memory (RAM) Allocation & Concurrency Budget

To eliminate the risk of Linux Out-Of-Memory (OOM) killer process terminations, total physical RAM (2048 MB) is budgeted strictly across system processes:

| Process / Component                | Execution Model                                        | Allocated Physical Memory   | Peak Consumption Threshold       |
| :--------------------------------- | :----------------------------------------------------- | :-------------------------- | :------------------------------- |
| **Linux Kernel & OS Core**         | Base system services, systemd, sshd, rsyslog           | ~250 MB                     | 350 MB                           |
| **Nginx Web Server**               | Master process + 2 worker processes                    | ~50 MB                      | 100 MB                           |
| **Gunicorn WSGI Master & Workers** | 2 Synchronous application workers[cite: 3]             | ~450 MB (225 MB per worker) | 650 MB                           |
| **Celery Asynchronous Worker**     | 1 Worker daemon (solo concurrency)[cite: 3]            | ~250 MB                     | 450 MB                           |
| **Redis Broker**                   | In-memory key-value queue (`maxmemory 128MB`)[cite: 3] | ~80 MB                      | 128 MB (Capped)                  |
| **PostgreSQL 16 Engine**           | `shared_buffers` + connection pooling[cite: 3]         | ~350 MB                     | 550 MB                           |
| **Buffer / Free Overhead**         | Dynamic disk cache & OS write buffers                  | ~618 MB                     | N/A                              |
| **Total Physical Host Budget**     | **Co-located Services Boundary**                       | **2048 MB**                 | **2048 MB (Overflow into Swap)** |

---

## 3. Persistent Local Disk Sizing (40 GB NVMe SSD)

The 40 GB NVMe local disk partition is allocated across operational boundaries to ensure sustained runway without manual disk expansion:

| Disk Partition / Directory       | Functional Purpose                                       | Baseline Footprint | 12-Month Projected Growth      |
| :------------------------------- | :------------------------------------------------------- | :----------------- | :----------------------------- |
| `/` (Base Operating System)      | Ubuntu LTS packages, Python venv runtime[cite: 2, 3]     | 7.5 GB             | 10.0 GB                        |
| `/swapfile`                      | Virtual Memory Buffer                                    | 2.0 GB             | 2.0 GB (Fixed allocation)      |
| `/var/lib/postgresql/16/main`    | Relational tablespaces, index B-trees, WAL logs[cite: 3] | 1.5 GB             | 6.5 GB                         |
| `/var/www/canmee_dairies/media`  | Milk collection PDFs, supplier receipts, slips[cite: 3]  | 1.0 GB             | 4.5 GB                         |
| `/var/www/canmee_dairies/static` | Compiled CSS, JS, Bootstrap, and web fonts[cite: 3]      | 350 MB             | 500 MB                         |
| `/var/log/`                      | Systemd journald, Nginx access/error, PostgreSQL logs    | 500 MB             | 1.5 GB (Managed via logrotate) |
| `/var/backups/postgres/`         | Temporary local database dump staging directory          | 500 MB             | 1.5 GB (Retained max 48 hours) |
| **Reserved Free Headroom**       | Filesystem safety buffer                                 | ~26.65 GB          | ~13.5 GB Remaining Headroom    |

---

## 4. Workload Throughput & Operational Limits

Based on the single-node architecture, the following concurrency parameters define the operational envelope:

- **Concurrent Field Clerks:** Sized for 10–25 simultaneous active users performing ledger entry, route updates, and milk intake logging[cite: 3].
- **Asynchronous Queue Handling:** Heavy payroll exports, farmer payment sheets, and bulk PDF generation are queued via Celery, offloading web worker thread execution to maintain HTTP request latency under 300ms[cite: 3].
- **Database Connection Limits:** Bounded to `max_connections = 50` in PostgreSQL, matched to Gunicorn thread concurrency to prevent connection exhaustion[cite: 3].

---

## 5. Scaling Triggers & Thresholds

If workload demands exceed the single-node profile, the following operational thresholds trigger architecture promotion:

1. **CPU Throttling Threshold:** Sustained CPU utilization above 75% for longer than 15 consecutive minutes (depleting Lightsail burstable CPU credits).
2. **Memory Saturation Threshold:** Swap usage exceeding 1.0 GB persistently, indicating heavy memory pressure and disk thrashing.
3. **Storage Ceiling:** Local SSD usage crossing 80% (32 GB utilized).

_Promotion Target:_ When thresholds are exceeded, the workload transitions from **Option 1 (Lightsail All-in-One)** to **Option 2 (Dedicated EC2 VPC Topology)** with decoupled compute and independent storage lifecycle boundaries.
