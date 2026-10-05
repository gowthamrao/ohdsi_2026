# OHDSI Server Environment & Infrastructure Requirements

> **Target Audience**: Infrastructure Architects, DevOps Engineers, Cloud Systems Administrators  
> **Platform Target**: Dedicated Bare-Metal Host (e.g. Hetzner AX/PX) or Cloud VM (AWS EC2 / Azure / GCP)  
> **Status**: Production Standard

---

## 1. Hardware Specifications

| Resource | Baseline (Development / POC) | Production (Full Vocabulary & Large CDM) |
| :--- | :--- | :--- |
| **Compute** | 16 vCPUs | 32 to 64 vCPUs |
| **Memory (RAM)** | 64 GB RAM | 128 GB to 256 GB RAM |
| **Storage Tier** | 1 TB NVMe SSD | 2 TB to 4 TB PCIe Gen4 NVMe SSD |
| **Network Bandwidth** | 1 Gbps NIC | 10 Gbps redundant uplink |
| **Operating System** | Ubuntu Server 24.04 LTS (x86_64) | Ubuntu Server 24.04 LTS (x86_64) |

---

## 2. Storage & Kernel Tuning

Mount the NVMe data partition with `noatime,nodiratime` for high-throughput database sequential and index scans:

```bash
# Storage mount layout
mkdir -p /var/lib/postgresql/data
mount -o noatime,nodiratime /dev/nvme0n1p3 /var/lib/postgresql/data
```

Apply kernel sysctl parameters (`/etc/sysctl.d/99-ohdsi.conf`):

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
| **22** | SSH | Restricted | Host Shell | Key-based authentication only; password authentication disabled. |
| **5432** | TCP | Private | PostgreSQL 16 | **Quarantined**: Accessible only via private Docker network, VPN, or Tailscale. |
| **8787** | HTTP | Private | RStudio Server (HADES) | Proxied via Nginx HTTPS at `/rstudio/`; direct host port closed to public. |
| **3838** | HTTP | Private | OHDSI Study Shiny Server | Proxied via Nginx HTTPS at `/shiny/` and `/reports/`; direct host port closed. |
| **6379** | TCP | Private | Redis Task Broker | Internal Docker network only; no external exposure. |

---

## 4. Production Container Roster (16 Core Services)

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
│ `study-agent-acp`    │ `study-agent:latest` │ 8765        │ Internal   │
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
- **Supported Package Suite**: The R Server MUST maintain pre-installed support for all official HADES packages and platform analytical libraries:
  1. *Database & SQL*: `DatabaseConnector`, `SqlRender`, `ParallelLogger`, `Andromeda`.
  2. *Cohort & Phenotyping*: `Capr`, `CirceR`, `CohortGenerator`, `CohortConstructor`, `PhenotypeLibrary`, `Phenotyper`, `Phenelope`, `PheValuator`, `Keeper`, `ProtocolGenerator`.
  3. *Characterization & Diagnostics*: `CohortDiagnostics`, `CohortIncidence`, `FeatureExtraction`, `Characterization`, `ClinicalCharacteristics`, `DbDiagnostics`, `DataQualityDashboard`.
  4. *Causal Estimation*: `CohortMethod`, `SelfControlledCaseSeries`, `Cyclops`, `EvidenceSynthesis`, `EmpiricalCalibration`, `MethodEvaluation`, `CaseControl`, `CaseCrossover`.
  5. *Patient Prediction*: `PatientLevelPrediction`, `DeepPatientLevelPrediction`, `BigKnn`.
  6. *Results & Studies*: `Strategus`, `ResultModelManager`, `ROhdsiWebApi`, `OhdsiShinyModules`, `ShinyAppBuilder`, `Eunomia`, `Taxis`.
- **Zero Fragmented Microservices**: Custom per-package Plumber containers are strictly prohibited from baseline production deployments.
- **External Studies**: Network studies such as [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis) are executed directly within this Dedicated R Server using standard `DatabaseConnector` queries.

---

## 6. Model Context Protocol (MCP) Agent Gateway

AI agents connect via the official [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) container (`study-agent-mcp`):
- **Ingress Route**: `/mcp/sse` and `/mcp/messages` (managed by Nginx with SSE buffering disabled).
- **Configuration**: Mounted from `studyagent/config.yaml`.
- **Privacy Controls**: AST SQL query parsing and Small Cell Suppression (`MIN_CELL_COUNT >= 5`).
