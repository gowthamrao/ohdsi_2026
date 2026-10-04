# Public Domain & Edge Ingress Hosting Guide

> **Production Guide for Hosting OHDSI on a Public URL Domain**  
> **Target Audience**: DevOps Engineers, Site Reliability Engineers (SRE), Network Administrators  
> **Applicable Environments**: Bare-Metal Dedicated (Hetzner, OVH) or Cloud VMs (AWS EC2, GCP, Azure)  

---

## 1. Architecture Overview

When hosting the platform on a public domain (e.g. `https://research.yourdomain.org`), all traffic routes through the hardened **Nginx Reverse Proxy Ingress** terminating TLS 1.3:

```
                            PUBLIC INTERNET / CLIENTS
                                       │
                      [ DNS: research.yourdomain.org ]
                                       │
                                [ HTTPS: 443 ]
                                       ▼
         ┌─────────────────────────────────────────────────────────────────────────┐
         │                    Nginx Edge Reverse Proxy Ingress                     │
         │          - TLS 1.3 Encryption (Let's Encrypt / Cloudflare)              │
         │          - Rate Limiting Zones (Public, Auth, Exec)                     │
         │          - Small Cell Suppression Filter (MIN_CELL_COUNT=5)             │
         └─────────────┬─────────────┬─────────────┬─────────────┬─────────────┬───┘
                       │             │             │             │             │
        ┌──────────────┘             │             │             │             └──────────────┐
        ▼                            ▼             ▼             ▼                            ▼
┌──────────────┐             ┌──────────────┐┌──────────────┐┌──────────────┐             ┌──────────────┐
│   Atlas 3.0  │             │    WebAPI    ││  FastMCP API ││  RStudio IDE │             │   Atlas 1.x  │
│  Next-Gen UI │             │ REST Backend ││  BYO-Agent   ││ Dedicated Svr│             │   Classic    │
│  Path: `/`   │             │ Path: /WebAPI││  Path: /mcp/ ││ Path:/rstudio│             │ Path: /atlas/│
│  (Port 3000) │             │ (Port 8080)  ││  (Port 8790) ││ (Port 8787)  │             │ (Port 8082)  │
└──────────────┘             └──────────────┘└──────────────┘└──────────────┘             └──────────────┘
```

---

## 2. Choosing an Ingress Routing Model

You can host the stack using either of two routing architectures:

| Model | Setup Complexity | DNS Requirement | Recommended Use Case |
| :--- | :--- | :--- | :--- |
| **Model A: Single-Domain Path-Based Routing** *(Recommended)* | **Lowest** (1 SSL cert, 1 DNS entry) | Single `A` Record (`research.yourdomain.org`) | **Most Deployments**: All tools accessible under unified prefix (`/`, `/atlas/`, `/WebAPI/`, `/mcp/`, `/shiny/`). |
| **Model B: Subdomain-Based Routing** | Medium (Wildcard or multiple certs) | Multiple CNAMEs (`atlas.*`, `webapi.*`, `studyagent.*`) | Enterprise portals requiring independent domain isolation. |

---

## 3. Step-by-Step Setup Runbook

### Step 1: Configure Public DNS Records
In your DNS provider (Cloudflare, AWS Route 53, Hetzner DNS, or GoDaddy), add an `A` record pointing to the host's public IP:

```dns
# Model A (Single Domain):
Type: A
Name: research.yourdomain.org
Value: 198.51.100.42      # Your Server Public IPv4
TTL: Auto / 300 seconds

# Model B (Wildcard Subdomains):
Type: A
Name: *.research.yourdomain.org
Value: 198.51.100.42
```

Verify DNS propagation:
```bash
dig +short research.yourdomain.org
# Should output your server's public IP
```

---

### Step 2: Automated Domain & TLS Configuration
Use the repository's configuration CLI to configure `.env`, CORS origins, and acquire TLS certificates:

```bash
# Option A: Real Let's Encrypt TLS Certificate (Production)
python scripts/setup_public_domain.py \
  --domain research.yourdomain.org \
  --email devops@yourdomain.org \
  --ssl-mode letsencrypt

# Option B: Cloudflare Origin CA (If using Cloudflare Proxy with strict SSL)
python scripts/setup_public_domain.py \
  --domain research.yourdomain.org \
  --ssl-mode cloudflare

# Option C: Self-Signed Certificate (Staging / Development)
python scripts/setup_public_domain.py \
  --domain research.yourdomain.org \
  --ssl-mode selfsigned \
  --skip-dns-check
```

---

### Step 3: Enable the Unified Ingress Template in Nginx
The repository includes [nginx/conf.d/public_unified_domain.conf](file:///c:/files/git/github/ohdsi/ohdsi_2026/nginx/conf.d/public_unified_domain.conf), which maps paths cleanly:

```bash
# Verify Nginx configuration syntax
docker compose exec reverse-proxy nginx -t

# Reload Nginx without downtime
docker compose exec reverse-proxy nginx -s reload
```

---

### Step 4: Verify Public URL Endpoints
Test each endpoint from an external machine:

```bash
# 1. Healthcheck Contract
curl -k https://research.yourdomain.org/health

# 2. WebAPI REST Info
curl -k https://research.yourdomain.org/WebAPI/info | jq .

# 3. Model Context Protocol (MCP) Tools Discovery
curl -k https://research.yourdomain.org/mcp/sse \
  -H "Authorization: Bearer ohdsi-study-designer-token-2026"

# 4. Atlas 3.0 Next-Gen Frontend
curl -I -k https://research.yourdomain.org/

# 5. Atlas Classic Frontend
curl -I -k https://research.yourdomain.org/atlas/

# 6. Dedicated RStudio Server Endpoint
curl -I -k https://research.yourdomain.org/rstudio/
```

---

## 4. Cross-Origin Resource Sharing (CORS) Configuration

When WebAPI and Atlas communicate over the public domain, CORS headers must permit the domain. The configuration CLI sets these automatically in `.env`:

```env
DOMAIN=research.yourdomain.org
WEBAPI_URL=https://research.yourdomain.org/WebAPI/
SECURITY_CORS_ALLOWED_ORIGINS=https://research.yourdomain.org,http://localhost:3000
SECURITY_ORIGIN=https://research.yourdomain.org
```

---

## 5. Security & Firewall Rules for Public Hosting

When a host has a public IP address, strict perimeter firewall rules are mandatory:

```bash
# Allow SSH management (recommended: restrict to admin IP)
ufw allow 22/tcp

# Allow HTTP for ACME Let's Encrypt validation
ufw allow 80/tcp

# Allow HTTPS for all public traffic
ufw allow 443/tcp

# CRITICAL: Drop direct external access to internal ports
# Never expose PostgreSQL (5432) or RStudio (8787) directly to 0.0.0.0
ufw default deny incoming
ufw enable
```

### Accessing Internal Services Remotely (Zero-Trust Access)
To access PostgreSQL (`5432`) or RStudio Server (`8787`) securely:
1. **Option 1 (SSH Tunnel)**:
   ```bash
   ssh -L 5432:localhost:5432 -L 8787:localhost:8787 ubuntu@research.yourdomain.org
   # Now access localhost:5432 in DBeaver / pgAdmin, and localhost:8787 in browser
   ```
2. **Option 2 (Tailscale / WireGuard)**:
   Install Tailscale on the host and bind ports `5432` and `8787` strictly to the `tailscale0` IP address.

---

## 6. Certificate Renewal & Automation

### Automated Let's Encrypt Renewal
Let's Encrypt certificates expire every 90 days. An automated renewal cron job runs inside the reverse proxy container or on the host:

```bash
# Add to host crontab (crontab -e):
0 3 * * * certbot renew --webroot -w /var/www/certbot --post-hook "docker compose -f /opt/ohdsi/docker-compose.yml exec reverse-proxy nginx -s reload"
```
