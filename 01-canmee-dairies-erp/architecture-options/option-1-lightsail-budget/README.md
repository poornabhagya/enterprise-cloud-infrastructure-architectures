# Option 1: Ultra-Low Budget Architecture (AWS Lightsail)

**Cost-Optimized Blueprint (~$5.00/mo) for Micro-Tier Workloads**

---

## 1. Overview & Positioning

This option provides an ultra-low-cost, predictable pricing model tailored for cost-constrained operational scenarios, pre-revenue internal tools, or micro-businesses where monthly cloud hosting spend must be minimized.

By packaging the entire operational stack—compute, fast SSD persistence, static networking, and data egress—into a single flat-rate AWS Lightsail bundle, the total infrastructure spend is contained at **~$5.00/month** while overcoming the architectural limitations of legacy shared hosting environments.

---

## 2. Strategic Trade-offs (Lightsail vs. Dedicated VPC)

| Evaluation Factor          | Option 1: AWS Lightsail Budget     | Option 2: Dedicated EC2 VPC (Selected)             |
| :------------------------- | :--------------------------------- | :------------------------------------------------- |
| **Monthly Cost**           | **Flat ~$5.00 / month**            | **~$12.62 / month**                                |
| **Public IPv4 Cost**       | **Included in bundle ($0.00)**     | $3.65 / month billed separately[cite: 1]           |
| **Storage Allocation**     | **40 GB SSD Included ($0.00)**     | 20 GB gp3 EBS ($1.60/mo)                           |
| **Data Transfer**          | **1 TB Outbound / mo Included**    | CloudFront Free Tier / Standard Egress             |
| **Network Architecture**   | Flat VPC / Basic Firewall Rules    | Multi-Tier Custom Subnets, Security Groups & NACLs |
| **Scalability Horizon**    | Vertical upgrade via bundle resize | Flexible decoupled scale (RDS, ElastiCache, ALB)   |
| **Compliance & Isolation** | Shared VPC boundary                | True multi-tenant VPC isolation (`10.0.0.0/16`)    |

---

## 3. High-Level Architecture Design

![Canmee Dairies AWS Lightsail Architecture](canmee_lightsail-archi.png)

---

## 4. Cost Breakdown (Lightsail Bundle Model)

By taking advantage of the bundled static IP and block storage, extra AWS fees are avoided:

| Component                     | Sizing & Configuration                             | Cost (USD / Month) |
| :---------------------------- | :------------------------------------------------- | :----------------- |
| **Lightsail Instance Bundle** | 1 vCPU, 2 GB RAM, 40 GB SSD, 1 TB Data Transfer    | $5.00              |
| **Static IPv4 Address**       | 1 Dedicated In-use Public IP                       | $0.00 _(Included)_ |
| **DNS Resolution**            | External Registrar / Cloudflare A-Record           | $0.00              |
| **SSL / TLS Certificate**     | Let's Encrypt automated via Certbot                | $0.00              |
| **S3 Offsite Backups**        | Encrypted `pg_dump` retention + Glacier Transition | ~$0.66             |
| **Total Monthly Spend**       | **Self-Contained Production Node**                 | **~$5.66 / mo**    |

---

## 5. Deployment & Runtime Guidelines

1. **Memory Safeguards (Swap Allocation):**  
   Because the compute instance operates on 2 GB of physical memory, running PostgreSQL 16, Redis, Gunicorn, and Celery concurrently risks encountering Linux OOM (Out Of Memory) process termination. A mandatory **2 GB Swapfile** must be created:
   ```bash
   sudo fallocate -l 2G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```
