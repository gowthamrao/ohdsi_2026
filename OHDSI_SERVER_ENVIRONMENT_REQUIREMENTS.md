# OHDSI Sandbox: Server Environment & Infrastructure Requirements

> **Platform Mission**: High-Resilience Developer Sandbox for Innovating, Collaborating & Testing Latest Ideas in Clinical Informatics and Data Science  
> **Target Audience**: Infrastructure Architects, DevOps/SRE Engineers, Cloud Systems Administrators  
> **Platform Target**: Dedicated Bare-Metal Host (e.g. Hetzner AX/PX) or Cloud VM (AWS EC2 / Azure / GCP)  
> **Status**: Approved Production Specification  
> **Cross-References**: [README.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/README.md) | [DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md) | [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md) | [PUBLIC_DOMAIN_HOSTING_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PUBLIC_DOMAIN_HOSTING_GUIDE.md)

---

## 1. Hardware Specifications

| Resource | Baseline (Development Sandbox) | Heavy Hammering / Multi-Agent Sandbox |
| :--- | :--- | :--- |
| **Compute** | 16 vCPUs | 32 to 64 vCPUs (or Bare-Metal AMD EPYC / Ryzen 9) |
| **Memory (RAM)** | 64 GB RAM | 128 GB to 256 GB ECC RAM |
| **Storage Tier** | 1 TB PCIe Gen4 NVMe SSD | 2 TB to 4 TB PCIe Gen4 NVMe SSD (ZFS / Btrfs CoW) |
| **High-Speed Swap** | 32 GB NVMe Swap Partition | 64 GB NVMe Swap Partition (`vm.swappiness = 10`) |
| **Network Bandwidth** | 1 Gbps NIC | 10 Gbps redundant uplink |
| **Operating System** | Ubuntu Server 24.04 LTS (x86_64) | Ubuntu Server 24.04 LTS (x86_64) |

---

## 2. Storage, Instant Snapshots & Kernel Tuning

### Copy-on-Write (CoW) Snapshots for Instant Rollbacks
To allow researchers and agents to experiment fearlessly without fear of permanent schema or database corruption, the NVMe storage partition is configured with Copy-on-Write snapshots (ZFS or Btrfs):

```bash
# Example ZFS dataset configuration for sub-minute rollback capability:
zfs create -o mountpoint=/var/lib/postgresql/data -o compression=lz4 -o atime=off rpool/pgdata
zfs create -o mountpoint=/home/ohdsi -o compression=lz4 rpool/rstudio-workspace

# Sub-minute developer rollback workflow:
# 1. Take snapshot before running an experimental cohort or agent loop:
zfs snapshot rpool/pgdata@pre-experiment-$(date +%s)
# 2. Rollback instantly if experiment corrupts data:
zfs rollback rpool/pgdata@<snapshot_name>
```

### High-Speed NVMe Swap Partition
To absorb violent memory spikes from concurrent R causal inference jobs (`CohortMethod`, `PatientLevelPrediction`) without kernel panics, configure a 64 GB swapfile on NVMe:

```bash
fallocate -l 64G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### Kernel Sysctl Parameters (`/etc/sysctl.d/99-ohdsi.conf`):
```ini
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 8192
```

---

## 3. Network Ports & Firewall Rules

| Port | Protocol | Scope | Target Service | Security Standard |
| :--- | :--- | :--- | :--- | :--- |
| **80** | HTTP | Public | Nginx Reverse Proxy | ACME Let's Encrypt validation & HTTP-to-HTTPS 301 redirect. |
| **443** | HTTPS | Public | Nginx Reverse Proxy | Modern TLS 1.3 only; HSTS 2-Year preload; multi-tier rate limiting. |
| **22** | SSH | Restricted | Host Shell / Remote IDE | Key-based authentication only; supports VS Code Remote & JetBrains Gateway. |
| **5432** | TCP | Private | PostgreSQL 16 | **Quarantined**: Accessible via private Docker network, VPN, or Tailscale. |
| **8787** | HTTP | Private | RStudio Server (HADES) | Proxied via Nginx HTTPS at `/rstudio/`; WebSocket upgrades enabled. |
| **3838** | HTTP | Private | OHDSI Study Shiny Server | Proxied via Nginx HTTPS at `/shiny/` and `/reports/`; WebSockets enabled. |
| **8765** | HTTP | Private | WebApiMcp Bridge | Proxied via Nginx HTTPS at `/webapi-mcp/`; direct host port closed to public. |
| **8880** | HTTP | Private | Arachne Data Node | Proxied via Nginx HTTPS at `/arachne/`; direct host port closed to public. |
| **8888** | HTTP | Private | Arachne Execution Engine | Quarantined to internal Docker network; executes study packages. |
| **8978** | HTTP | Private | CloudBeaver Web SQL IDE | Proxied via Nginx HTTPS at `/sql/`; browser-based SQL & schema studio. |
| **9000** | HTTP | Private | MinIO S3 API | Proxied via Nginx HTTPS at `/s3/api/`; study artifact storage. |
| **9001** | HTTP | Private | MinIO Web Console | Proxied via Nginx HTTPS at `/s3/`; object storage management UI. |
| **6379** | TCP | Private | Redis Task Broker | Internal Docker network only; no external exposure. |

---

## 4. Production Container Roster (20 Core Services)

All services are orchestrated via standard `docker-compose.yml`:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        CORE CONTAINER PLATFORM                         │
├──────────────────────┬──────────────────────┬─────────────┬────────────┤
│ Service              │ Image                │ Internal    │ Exposed In │
├──────────────────────┼──────────────────────┼─────────────┼────────────┤
│ `reverse-proxy`      │ `nginx:alpine`       │ 80, 443     │ Public URL │
│ `certbot`            │ `certbot/certbot`    │ -           │ Internal   │
│ `ohdsi-postgres`     │ `postgres:16-alpine` │ 5432        │ Private    │
│ `webapi-classic`     │ `ohdsi/webapi:2.14`  │ 8080        │ Nginx URL  │
│ `atlas-classic`      │ `ohdsi/atlas:2.14`   │ 80          │ `/atlas/`  │
│ `atlas3-webapi`      │ `webapi:3.0-dev`     │ 8080        │ Nginx URL  │
│ `atlas3-frontend`    │ `atlas3:dev`         │ 80          │ Root `/`   │
│ `broadsea-hades`     │ `broadsea-hades:1.19`│ 8787        │ `/rstudio/`│
│ `broadsea-solr-vocab`│ `solr:8.11.1`        │ 8983        │ Internal   │
│ `ohdsi-redis`        │ `redis:7-alpine`     │ 6379        │ Internal   │
│ `ohdsi-shiny`        │ `rocker/shiny:4.3.3` │ 3838        │ `/shiny/`  │
│ `study-agent-mcp`    │ `study-agent:latest` │ 8790        │ `/mcp/`    │
│ `webapi-mcp`         │ `webapi-mcp:latest`  │ 8765        │`/webapi-mcp`│
│ `arachne-data-node`  │ `arachne-data-node`  │ 8880        │ `/arachne/`│
│ `arachne-exec-engine`│ `arachne-exec-engine`│ 8888        │ Internal   │
│ `cloudbeaver-sql`    │ `dbeaver/cloudbeaver`│ 8978        │ `/sql/`    │
│ `minio-s3`           │ `minio/minio:latest` │ 9000, 9001  │ `/s3/`     │
│ `ollama-service`     │ `ollama/ollama`      │ 11434       │ Internal   │
│ `pythia-agent`       │ `ohdsi/pythia`       │ 8080        │ Internal   │
│ `atlas3-db-init`     │ `postgres:16-alpine` │ Migration   │ Internal   │
└──────────────────────┴──────────────────────┴─────────────┴────────────┘
```

---

## 5. Dedicated R Server & Study Applications Architecture

Rather than deploying 15+ fragile Plumber microservices, all analytical R packages (`Capr`, `CohortDiagnostics`, `CohortIncidence`, `PheValuator`, `Hades`) run inside the **Dedicated R Server** (`broadsea-hades` / RStudio Server):

- **OMOP CDM & Vocabulary Database Connection**: The R Server MUST have direct internal network access and JDBC credentials to the PostgreSQL database cluster containing both the OMOP CDM event tables (`person`, `visit_occurrence`, etc.) and the master Athena Vocabulary tables (`concept`, `concept_ancestor`, `concept_relationship`, etc.).
- **Shared PostgreSQL Database with Atlas Instance**: The exact same PostgreSQL database cluster accessed by the R Server MUST be connected to WebAPI and the Atlas instance, giving researchers interactive UI access to data sources, vocabulary searches, and cohort definitions.
- **OHDSI Study Shiny Apps & Report Publishing**: The platform MUST support deploying interactive Shiny applications and published analytical reports from OHDSI studies (e.g. `CohortDiagnostics`, `CohortIncidence`, `Characterization`, `OhdsiShinyModules`, `ShinyAppBuilder`, [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis)). Interactive dashboards are published to `/srv/shiny-server/` and accessible via public URL at `/shiny/`; static reports are published to `/srv/reports/` and accessible at `/reports/`.
- **WebAPI Integration**: The R Server MUST have direct REST reachability to the WebAPI backend (`WEBAPI_URL`), allowing `ROhdsiWebApi` to import/export cohort definitions and concept sets.
- **Passwordless Sudo for Developers**: The user `ohdsi` inside `broadsea-hades` is granted passwordless `sudo` to install dynamic C-dependencies and compile experimental R packages from GitHub without DevOps friction.
- **Supported Package Suite**: The R Server pre-installs all official HADES packages and analytical libraries across 6 functional domains (Database & SQL, Cohort & Phenotyping, Characterization & Diagnostics, Causal Estimation, Patient-Level Prediction, Results & Shiny). For the exhaustive package catalog, see [DEDICATED_R_SERVER_ARCHITECTURE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEDICATED_R_SERVER_ARCHITECTURE.md) and [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md#stage-gate-2-data-scaling--dedicated-r-server).

---

## 6. Model Context Protocol (MCP) Agent Gateways

The platform deploys two specialized MCP servers to interface with external LLM agents (Claude, Cursor, Antigravity):

### A. StudyAgent FastMCP Gateway (`study-agent-mcp`)
- **Upstream**: [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) (port 8790).
- **Ingress Route**: `/mcp/sse` and `/mcp/messages` (proxied by Nginx with SSE buffering disabled).
- **Configuration**: Mounted from `studyagent/config.yaml`.
- **Developer Mode**: Verbose compiler error messages returned in JSON-RPC responses for automated agent self-correction. Small cell suppression is configurable via `ENFORCE_SMALL_CELL_SUPPRESSION=false`.

### B. WebApiMcp Bridge Server (`webapi-mcp`)
- **Upstream**: [`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp) (port 8765).
- **Ingress Route**: `/webapi-mcp/` (proxied by Nginx with chunked transfer and streaming enabled).
- **Configuration**: `WEBAPI_MCP_WEBAPI_BASE_URL=http://webapi-classic:8080/WebAPI`.
- **Function**: Enables LLM agents to manage cohort definitions, fetch Circe JSON, inspect concept sets, and query data sources directly via WebAPI.
- **Endpoints**: Health check at `/webapi-mcp/health` and MCP JSON-RPC at `/webapi-mcp/mcp`.

---

## 7. OHDSI Arachne Distributed Research Network Integration

The platform provides native support for distributed network study execution via OHDSI Arachne:

- **Upstream Standard**: [`OHDSI/ArachneDataNode`](https://github.com/OHDSI/ArachneDataNode), [`OHDSI/ArachneExecutionEngine`](https://github.com/OHDSI/ArachneExecutionEngine), and [`OHDSI/ArachneCentral`](https://github.com/OHDSI/ArachneCentral).
- **Arachne Data Node (`arachne-data-node`)**:
  - Exposes web management and REST API on internal port 8880.
  - Proxied via Nginx HTTPS at `https://<domain>/arachne/`.
  - Connects to the local PostgreSQL OMOP CDM database and WebAPI instance.
- **Arachne Execution Engine (`arachne-exec-engine`)**:
  - Internal execution daemon on port 8888 (isolated from public ingress).
  - Executes study R packages and SQL queries inside isolated Docker container runtimes.

---

## 8. Developer Sandbox Crash Isolation & cgroups

To allow aggressive developer and agent experimentation without crashing the underlying system:

```yaml
# cgroup limits applied in docker-compose.yml:
services:
  broadsea-hades:
    mem_limit: 48g
    mem_reservation: 8g
    shm_size: 16g
    restart: unless-stopped

  ohdsi-postgres:
    mem_limit: 48g
    shm_size: 16g
    restart: unless-stopped

  webapi-classic:
    mem_limit: 12g
    restart: unless-stopped
```

### PostgreSQL Anti-Lock Configuration (`postgresql.conf`):
- `statement_timeout = '15min'`: Automatically terminates runaway Cartesian joins.
- `idle_in_transaction_session_timeout = '10min'`: Automatically terminates abandoned sessions that hold exclusive table locks.
- `temp_file_limit = '50GB'`: Prevents unindexed queries from consuming all available disk space.

---

## 9. Developer Tooling (Web SQL Studio & MinIO S3 Object Store)

- **CloudBeaver Web SQL Studio (`cloudbeaver-sql`)**:
  - Exposes a web-based database management interface on port 8978, proxied at `https://<domain>/sql/`.
  - Enables instant browser-based querying, visual explain plans, table autocomplete, and ER diagram browsing without local client setup.
- **MinIO S3 Object Store (`minio-s3`)**:
  - Exposes S3-compatible API on port 9000 (`/s3/api/`) and Web Management Console on port 9001 (`/s3/`).
  - Provides local object storage for Strategus study packages, Parquet data exports, and pipeline artifacts.
