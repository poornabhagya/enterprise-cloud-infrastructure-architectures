# Network & Security Engineering Specification

**Workload:** Canmee Dairies ERP  
**Deployment Tier:** AWS Lightsail All-in-One Topology (Option 1)  
**Document Code:** SPEC-NET-SEC-001  
**Status:** Approved / Production Baseline

---

## 1. Network Architecture & Ingress Strategy

### 1.1 Ingress Topology & Domain Resolution

- **Domain & Host Routing:** Managed via external DNS registrar / Cloudflare dashboard pointing directly to the instance's Static Public IPv4 address using standard A-records[cite: 3].
- **Route 53 Cost Elimination:** Eliminates monthly AWS Route 53 hosted zone recurring fees while providing zero-cost DNS resolution[cite: 2, 3].
- **Host Header Validation:** Ingress requests enforce exact `Host` header filtering in Nginx to match authorized domain endpoints (e.g., `test.canmeedairies.lk`), dropping spoofed or unmapped hostnames[cite: 3].

### 1.2 Network Interfaces & Service Binding Architecture

All internal databases, brokers, and application socket interfaces are strictly isolated to the Linux loopback adapter:

| Interface / Bind | Target Service        | Port                        | External Exposure Status                      | Enforcement Mechanism                     |
| :--------------- | :-------------------- | :-------------------------- | :-------------------------------------------- | :---------------------------------------- |
| `0.0.0.0`        | Nginx (HTTP Ingress)  | 80 / TCP                    | **Publicly Accessible** (301 Redirect to 443) | Lightsail Firewall + UFW[cite: 3]         |
| `0.0.0.0`        | Nginx (HTTPS Ingress) | 443 / TCP                   | **Publicly Accessible**                       | Lightsail Firewall + UFW[cite: 3]         |
| `0.0.0.0`        | OpenSSH Daemon        | 22 / TCP                    | **Restricted / Management Only**              | Key-pair authentication only[cite: 3]     |
| `127.0.0.1`      | Gunicorn WSGI Core    | 8000 / TCP (or Unix Socket) | **Strictly Blocked / Loopback Only**          | Application bound to `127.0.0.1`[cite: 3] |
| `127.0.0.1`      | Redis Server Broker   | 6379 / TCP                  | **Strictly Blocked / Loopback Only**          | Redis `bind 127.0.0.1 -::1`[cite: 3]      |
| `127.0.0.1`      | PostgreSQL 16 Engine  | 5432 / TCP                  | **Strictly Blocked / Loopback Only**          | `pg_hba.conf` local loopback[cite: 3]     |

---

## 2. Firewall Enforcement & Port Security

A two-tier defense-in-depth perimeter policy is enforced combining cloud-level perimeter firewalls and operating system packet filtering:

### 2.1 Cloud Perimeter Firewall (AWS Lightsail Console)

- Inbound Rules allowed:
  - Port `22` (TCP): SSH Administration (Restricted to specific administrator IP CIDRs where applicable)[cite: 3].
  - Port `80` (TCP): HTTP Ingress (Mandatory for Let's Encrypt validation and HTTP-to-HTTPS redirects)[cite: 3].
  - Port `443` (TCP): HTTPS Encrypted Traffic[cite: 3].
- Default Out-of-the-Box Inbound Policy: Implicit `DENY ALL` on all remaining TCP/UDP ports[cite: 3].

### 2.2 Host-Level Packet Filtering (Linux UFW)

Host-level firewall rules are persistently enforced across system reboots:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH Management'
sudo ufw allow 80/tcp comment 'HTTP Let’s Encrypt'
sudo ufw allow 443/tcp comment 'HTTPS Production'
sudo ufw enable
```

---

## 3. Cryptographic Controls & Transport Layer Security (TLS)

### 3.1 SSL/TLS Termination

- **Certificate Authority:** Let's Encrypt automated via Certbot Nginx plugin[cite: 3].
- **Protocol Support:** TLSv1.2 and TLSv1.3 exclusively enabled; insecure legacy protocols (SSLv3, TLSv1.0, TLSv1.1) explicitly disabled.
- **Cipher Suites:** Modern cryptographic suites configured for Perfect Forward Secrecy (PFS), prioritizing ECDHE-ECDSA-AES128-GCM-SHA256 and ECDHE-RSA-AES128-GCM-SHA256.

### 3.2 Automated Certificate Lifecycle & Verification

- **Renewal Mechanism:** Automated systemd timer executing twice daily (`certbot.timer`) testing validity and renewing within 30 days of expiration[cite: 3].
- **Renewal Hook:** Post-renewal hook reloads Nginx configuration gracefully without dropped connections (`--post-hook "systemctl reload nginx"`).
- **Handshake Verification:** Verified externally via `curl -Iv` confirming 301 redirection from HTTP to HTTPS and valid TLS certificate trust chains[cite: 3].

---

## 4. Secrets Governance & OS Access Control

### 4.1 Environment Secrets Isolation

- **Storage Path:** Isolated at `/etc/canmee/canmee.env` outside the web root `/var/www/`[cite: 3].
- **File Permissions:** Hardened using strict Linux file system permissions (`chmod 600 /etc/canmee/canmee.env`)[cite: 3].
- **Ownership:** Assigned strictly to the system user `www-data` and root; read-access is denied to unprivileged processes[cite: 3].
- **Content Boundaries:** Encapsulates Django `SECRET_KEY`, database credentials, Redis URLs, and offsite storage API tokens[cite: 3].

### 4.2 Compute Access & Privilege Separation

- Direct root SSH login disabled in `/etc/ssh/sshd_config` (`PermitRootLogin no`).
- Password-based SSH authentication disabled (`PasswordAuthentication no`), enforcing 4096-bit RSA or Ed25519 public key authentication.
- Application WSGI workers and background Celery daemons run strictly as non-privileged service users (`www-data` or dedicated service accounts)[cite: 3].

---

## 5. Application-Level Security & Access Verification

- **Cross-Site Request Forgery (CSRF):** Enforced across all POST, PUT, and DELETE forms, with validated `csrftoken` cookie transmission over HTTPS[cite: 3].
- **Session Cookie Security:** Production settings enforce `SESSION_COOKIE_SECURE = True` and `SESSION_COOKIE_HTTPONLY = True` to mitigate cookie interception and XSS extraction[cite: 3].
- **Role-Based Access Control (RBAC):** Verified using automated headless harnesses ensuring protected routes (e.g., `/assets/policies/`, `/assets/depreciation/`) throw `403 PermissionDenied` when requested outside assigned administrative permissions[cite: 3].
