# AWS Lightsail Budget Cost Breakdown

**Option 1: Optimized Flat-Rate Compute & Storage Tier (~$12.43/mo)**

---

## 1. Executive Cost Summary

- **Deployment Region:** Asia Pacific (Mumbai) `ap-south-1`[cite: 2]
- **Upfront Cost:** **$0.00 USD**[cite: 2]
- **Original Estimate (with Route 53):** $13.09 USD / month[cite: 2]
- **Optimized Monthly Cost (Route 53 Eliminated):** **$12.43 USD / month**
- **Projected 12-Month Total Commitment:** **$149.16 USD**

By eliminating the recurring Amazon Route 53 Hosted Zone charge ($0.66/month)[cite: 2] through external Registrar / Cloudflare DNS management, the recurring infrastructure spend is reduced without sacrificing performance or edge reliability.

---

## 2. Detailed Service Cost Breakdown

| Service Component                | Configuration & Operational Sizing                                                                                                                                                                                                                                                                                                                                                                              | Monthly Cost (USD)                 |
| :------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------- |
| **Amazon Lightsail**[cite: 2]    | **Canmee Dairies Budget All-in-One VPS Tier**[cite: 2]<br>• Operating System: Linux (Ubuntu LTS)[cite: 2]<br>• Server Count: 1 instance[cite: 2]<br>• Instance Type: **Bundle: 2GB** (1 vCPU, 2GB RAM, 40GB NVMe SSD, 1TB Transfer)[cite: 2]<br>• In-use Dedicated Static IPv4: Bundled ($0.00)                                                                                                                 | **$11.77**[cite: 2]                |
| **Amazon S3**[cite: 2]           | **Canmee Dairies Offsite Backup & Archival Tier**[cite: 2]<br>• S3 Standard Storage: 20 GB/month[cite: 2]<br>• S3 Operations: 20,000 PUT/COPY/POST/LIST & 100,000 GET/SELECT requests[cite: 2]<br>• S3 Glacier Flexible Retrieval: 2 GB/month storage[cite: 2]<br>• Lifecycle Transitions: 100 transitions to Glacier (30-day policy)[cite: 2]<br>• Standard Restore/Retrievals: 1 GB/month allocation[cite: 2] | **$0.66**[cite: 2]                 |
| **Amazon Route 53**[cite: 2]     | **Domain DNS Management**[cite: 2]<br>• _Eliminated:_ Hosted Zone routing moved to External Registrar / Cloudflare DNS                                                                                                                                                                                                                                                                                          | **$0.00** _(Saved $0.66)_[cite: 2] |
| **Amazon CloudFront**[cite: 2]   | **Global Edge Caching & TLS Offload**[cite: 2]<br>• Plan: AWS Free Tier allocation (up to 1TB data egress & 10M requests)[cite: 2]                                                                                                                                                                                                                                                                              | **$0.00**[cite: 2]                 |
| **Amazon SNS**[cite: 2]          | **Canmee Dairies Telemetry Alerts SNS Topic**[cite: 2]<br>• 10,000 requests/mo, 1,000 Email alerts, 10,000 HTTP/S notifications[cite: 2]                                                                                                                                                                                                                                                                        | **$0.00**[cite: 2]                 |
| **AWS Systems Manager**[cite: 2] | **Canmee Dairies Secrets & Session Manager**[cite: 2]<br>• 10 Standard parameters with continuous API interactions[cite: 2]                                                                                                                                                                                                                                                                                     | **$0.00**[cite: 2]                 |
| **Total Monthly Spend**          | **Fully Functional All-in-One Lightsail Workload**                                                                                                                                                                                                                                                                                                                                                              | **$12.43 / mo**                    |

---

## 3. Route 53 Elimination Justification

1. **Avoided Hosted Zone Fee:** Amazon Route 53 charges a base rate of $0.50 + applicable taxes per hosted zone monthly regardless of traffic volume[cite: 2].
2. **External DNS Routing Strategy:** The root domain and `test.canmeedairies.lk` sub-domain point directly to the Lightsail instance's Static Public IPv4 address using standard A-Records configured on the domain registrar or Cloudflare dashboard at zero additional cost.
3. **SSL Handling:** Let's Encrypt automated TLS certificates are issued via Nginx Certbot directly on the instance, bypassing Route 53 ACM DNS-validation dependencies.

---

## 4. Workload Sustainability & Scaling Thresholds

- **Compute Capacity:** The 2GB RAM Lightsail bundle comfortably hosts Gunicorn (2 workers), PostgreSQL 16, Redis 7, and Celery when buffered by an active 2GB Linux swap partition.
- **Storage Runway:** 40GB local SSD provides ample headroom for database tablespaces and application assets, with historical snapshots continuously exported off-host into the S3/Glacier lifecycle pipeline[cite: 2].
