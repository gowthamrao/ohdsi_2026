# Public Domain & Edge Ingress Hosting Guide

> **Production Specification for Hosting OHDSI on a Public URL Domain**  
> **Target Audience**: DevOps Engineers, Site Reliability Engineers (SRE), Network Administrators  
> **Applicable Environments**: Bare-Metal Dedicated (Hetzner, OVH) or Cloud VMs (AWS EC2, GCP, Azure)  
> **Status**: Approved Production Standard

---

## 1. Architecture Overview & Public URL Ingress

When hosting the OHDSI platform on a public domain (e.g. `https://research.yourdomain.org`), all traffic routes through a single hardened **Nginx Reverse Proxy Ingress** terminating TLS 1.3:

```
                            PUBLIC INTERNET / RESEARCHERS / AGENTS
                                              │
                              [ DNS: research.yourdomain.org ]
                                              │
                                        [ HTTPS: 443 ]
                                              ▼
          ┌─────────────────────────────────────────────────────────────────────────────┐
          │                      Nginx Edge Reverse Proxy Ingress                       │
          │            - TLS 1.3 Termination (Let's Encrypt / Cloudflare Origin)        │
          │            - Multi-Tier Rate Limiting (Public, Auth, Exec Zones)            │
          │            - Small Cell Privacy Suppression Filter (MIN_CELL_COUNT >= 5)    │
          │            - WebSocket Proxy Headers for RStudio Server & Shiny Apps        │
          └──────┬──────────────┬──────────────┬──────────────┬──────────────┬──────────┘
                 │              │              │              │              │
                 ▼              ▼              ▼              ▼              ▼
          ┌──────────────┐┌──────────────┐┌──────────────┐┌──────────────┐┌──────────────┐
          │  Atlas 3.0   ││    WebAPI    ││  Dedicated   ││ OHDSI Study  ││ FastMCP API  │
          │ Next-Gen UI  ││ REST Backend ││   R Server   ││  Shiny Apps  ││  AI Gateway  │
          │  Path: `/`   ││ Path:/WebAPI ││Path: /rstudio││Path: /shiny/ ││ Path: /mcp/  │
          │ (Port 3000)  ││ (Port 8080)  ││ (Port 8787)  ││ (Port 3838)  ││ (Port 8790)  │
          └──────────────┘└──────────────┘└──────────────┘└──────────────┘└──────────────┘
```

### Public URL Routing Table
Every component of the OHDSI research environment is unified under **ONE public domain URL**:

| Public Path | Internal Service | Upstream Port | Features & Protocols |
| :--- | :--- | :--- | :--- |
| **`/`** | Atlas 3.0 Next-Gen Frontend | Port 3000 | Vue 3 / single-spa micro-frontend web UI. |
| **`/atlas/`** | Atlas Classic Frontend | Port 8082 | Knockout.js classic analytical web interface. |
| **`/WebAPI/`** | WebAPI Backend Engine | Port 8080 | Java Spring Boot REST API connected to OMOP CDM & Vocabularies. |
| **`/rstudio/`** | Dedicated R Server | Port 8787 | Containerized RStudio Server with WebSocket upgrade for interactive R. |
| **`/shiny/`** | OHDSI Study Shiny Apps | Port 3838 | Interactive study reports and dashboards (e.g. `CohortDiagnostics`, `Taxis`). |
| **`/reports/`** | OHDSI Study Static Reports | Port 3838 / 80 | Pre-compiled Quarto / RMarkdown analytical HTML reports. |
| **`/mcp/`** | StudyAgent FastMCP Gateway | Port 8790 | Server-Sent Events (SSE) and HTTP JSON-RPC for external AI models. |
| **`/webapi-mcp/`** | WebApiMcp Bridge Server | Port 8765 | Dedicated WebAPI MCP gateway ([`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp)) with streaming JSON-RPC. |
| **`/arachne/`** | OHDSI Arachne Data Node | Port 8880 | Federated research network management interface & execution dispatch ([`OHDSI/ArachneDataNode`](https://github.com/OHDSI/ArachneDataNode)). |
| **`/sql/`** | CloudBeaver Web SQL Studio | Port 8978 | Browser-based SQL editor, visual explain plans, table autocomplete, and schema ERDs. |
| **`/s3/`** | MinIO S3 Object Store Console | Port 9001 | Local S3 bucket console for Strategus study packages, Parquet data exports, and artifacts. |

---

## 2. Choosing an Ingress Routing Model

| Model | Setup Complexity | DNS Requirement | Recommended Use Case |
| :--- | :--- | :--- | :--- |
| **Model A: Single-Domain Path-Based Routing** *(Recommended)* | **Lowest** (1 SSL cert, 1 DNS entry) | Single `A` Record (`research.yourdomain.org`) | **Standard Sandbox**: All tools accessible under unified prefix (`/`, `/atlas/`, `/WebAPI/`, `/rstudio/`, `/shiny/`, `/mcp/`, `/webapi-mcp/`, `/arachne/`, `/sql/`, `/s3/`). |
| **Model B: Subdomain-Based Routing** | Medium (Wildcard or multiple certs) | Multiple CNAMEs (`atlas.*`, `webapi.*`, `rstudio.*`, `shiny.*`, `sql.*`, `s3.*`) | Enterprise portals requiring independent domain isolation. |

---

## 3. Step-by-Step Public Domain Deployment

### Step 1: Configure Public DNS Records
In your DNS provider (Cloudflare, AWS Route 53, Hetzner DNS), create an `A` record pointing to your server's public IPv4 address:

```dns
Type: A
Name: research.yourdomain.org
Value: 198.51.100.42      # Server Public IPv4
TTL: Auto / 300 seconds
```

Verify DNS propagation:
```bash
dig +short research.yourdomain.org
```

### Step 2: Acquire TLS Certificate (Certbot Let's Encrypt)
Run Certbot via webroot validation:
```bash
certbot certonly --webroot -w /var/www/certbot \
  -d research.yourdomain.org \
  --email devops@yourdomain.org \
  --agree-tos --no-eff-email
```

### Step 3: Nginx Ingress Configuration Specification
Below is the reference Nginx configuration for the unified public domain ingress, including WebSocket support for Shiny and RStudio:

```nginx
server {
    listen 80;
    server_name research.yourdomain.org;
    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    location / { return 301 https://$host$request_uri; }
}

server {
    listen 443 ssl http2;
    server_name research.yourdomain.org;

    ssl_certificate /etc/letsencrypt/live/research.yourdomain.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/research.yourdomain.org/privkey.pem;
    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;

    # Security Headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    # 1. Atlas 3.0 Next-Gen Frontend (Root Path)
    location / {
        proxy_pass http://atlas3-frontend:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    # 2. Atlas Classic Frontend
    location /atlas/ {
        proxy_pass http://atlas-classic:80/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }

    # 3. WebAPI Backend REST Engine
    location /WebAPI/ {
        proxy_pass http://webapi-classic:8080/WebAPI/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
        proxy_read_timeout 600s;
    }

    # 4. Dedicated R Server (RStudio Server Web IDE with WebSockets)
    location /rstudio/ {
        proxy_pass http://broadsea-hades:8787/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;
    }

    # 5. OHDSI Study Shiny Apps & Dashboards (with WebSockets)
    location /shiny/ {
        proxy_pass http://ohdsi-shiny:3838/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 1800s;
    }

    # 6. Model Context Protocol (FastMCP) AI Agent Gateway
    location /mcp/ {
        proxy_pass http://study-agent-mcp:8790/;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
        proxy_read_timeout 3600s;
    }

    # 7. WebApiMcp Bridge Server (schuemie/WebApiMcp)
    location /webapi-mcp/ {
        proxy_pass http://webapi-mcp:8765/;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
        proxy_read_timeout 3600s;
    }

    # 8. OHDSI Arachne Data Node
    location /arachne/ {
        proxy_pass http://arachne-data-node:8880/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_read_timeout 1800s;
    }

    # 9. CloudBeaver Web SQL Studio
    location /sql/ {
        proxy_pass http://cloudbeaver-sql:8978/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 1800s;
    }

    # 10. MinIO S3 Object Store Console & API
    location /s3/ {
        proxy_pass http://minio-s3:9001/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 1800s;
    }
}
```

---

## 4. Verifying Public URL Endpoints

Test each endpoint over HTTPS from an external machine:

```bash
# 1. Verify WebAPI REST backend
curl -s -k https://research.yourdomain.org/WebAPI/info | jq .

# 2. Verify Atlas 3.0 frontend
curl -I -k https://research.yourdomain.org/

# 3. Verify Atlas Classic frontend
curl -I -k https://research.yourdomain.org/atlas/

# 4. Verify Dedicated R Server RStudio endpoint
curl -I -k https://research.yourdomain.org/rstudio/

# 5. Verify OHDSI Study Shiny Apps endpoint
curl -I -k https://research.yourdomain.org/shiny/

# 6. Verify Model Context Protocol (MCP) stream
curl -s -k https://research.yourdomain.org/mcp/sse \
  -H "Authorization: Bearer <valid-mcp-token>"

# 7. Verify WebApiMcp health endpoint
curl -s -k https://research.yourdomain.org/webapi-mcp/health | jq .

# 8. Verify Arachne Data Node build / status
curl -s -k https://research.yourdomain.org/arachne/api/v1/build-number | jq .

# 9. Verify Web SQL Studio (CloudBeaver)
curl -I -k https://research.yourdomain.org/sql/

# 10. Verify MinIO S3 Console
curl -I -k https://research.yourdomain.org/s3/
```

---

## 5. Security & Perimeter Rules

1. **Quarantine Internal Ports**:
   - Port `5432` (PostgreSQL), Port `8787` (RStudio), and Port `3838` (Shiny) must **never** be exposed directly on the public interface.
   - All client traffic MUST pass through the TLS 1.3 reverse proxy.
2. **Small Cell Suppression (`MIN_CELL_COUNT >= 5`)**:
   - Ensure privacy filters mask counts between 1 and 4 as `"< 5"` to comply with health data privacy regulations.
3. **Automated Certificate Renewal**:
   - Certbot cron executes daily renewal checks:
     ```bash
     0 3 * * * certbot renew --webroot -w /var/www/certbot --post-hook "nginx -s reload"
     ```
