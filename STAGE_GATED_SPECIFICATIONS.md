# OHDSI Sandbox: System Requirements Specification & Acceptance Criteria

> **Document Type**: Authoritative System Requirements Specification (SRS) & Stage-Gated Acceptance Criteria (AC)  
> **Platform Mission**: High-Resilience Developer Sandbox for Innovating, Collaborating & Testing Latest Ideas in Clinical Informatics and Data Science  
> **Target Audience**: Cloud DevOps Engineers, Site Reliability Engineers (SRE), Infrastructure Architects, Data Science & Informatics Leads  
> **Platform Target**: Dedicated Bare-Metal Host (e.g. Hetzner AX/PX) or Cloud VPC (AWS / GCP / Azure)  
> **Status**: Approved Production Specification  
> **Cross-References**: [README.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/README.md) | [DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md) | [ENGINEERING_EXECUTION_PLAN.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/ENGINEERING_EXECUTION_PLAN.md) | [ARCHITECTURAL_REVIEW_DEVELOPER_PLAYGROUND.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/ARCHITECTURAL_REVIEW_DEVELOPER_PLAYGROUND.md)

---

## 1. Specification Framework & Sandbox Governance

This document establishes the formal, binding technical requirements and verifiable acceptance criteria for deploying the **OHDSI Sandbox**.

### Core Governance Principles:
1. **Developer Playground Focus**: The platform is an open sandbox engineered to withstand aggressive developer hammering (runaway SQL queries, multi-core R causal inference pipelines, high-frequency LLM agent loops).
2. **Synthetic Benchmark Data**: Connects exclusively to synthetic, benchmark, and simulated datasets (Eunomia, Synthea 100k, CMS SynPUF 2.3M) and Athena Vocabularies. Zero real patient PHI is hosted.
3. **Sub-Minute Rollbacks (Break & Restore)**: Built on Copy-on-Write (CoW) filesystem snapshots (ZFS/Btrfs) allowing instant restoration (`< 30 seconds`) when experimental cohorts or migrations corrupt schemas.
4. **Developer Empowerment**: Data scientists receive PostgreSQL superuser credentials (`ohdsi_admin`), container `sudo` in the Dedicated R Server, and global admin roles in Atlas and WebAPI.
5. **Agentic AI Workbench**: Hosts dual MCP gateways ([`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) and [`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp)) with verbose debug telemetry for automated LLM self-correction.

*For system architecture diagrams and the DevOps Rosetta Stone glossary, see [README.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/README.md).*

---

## 2. Stage-Gated Milestone Specifications

### Stage Gate 0: Host Infrastructure, Storage & Kernel Hardening

#### A. Requirements Specification
1. **REQ-001 (Host Operating System)**: The host MUST run a clean, 64-bit installation of Ubuntu Server 24.04 LTS.
2. **REQ-002 (Copy-on-Write Storage Subsystem)**: Dedicated PCIe Gen4 NVMe storage MUST be configured with a Copy-on-Write (CoW) filesystem (ZFS or Btrfs) mounted at `/var/lib/postgresql/data` and `/home/ohdsi` with compression enabled (`lz4`) and access time disabled (`noatime,nodiratime`).
3. **REQ-003 (Kernel Sizing & Swap)**: The host MUST be configured with a dedicated 64 GB NVMe swap partition and persisted sysctl parameters at `/etc/sysctl.d/99-ohdsi.conf` configuring low swappiness (`vm.swappiness = 10`), dirty page flushing (`vm.dirty_ratio = 15`), and high socket backlog (`net.core.somaxconn = 65535`).
4. **REQ-004 (Perimeter Firewall)**: Host firewall (UFW) MUST block all external inbound ports except Port 22 (SSH), Port 80 (HTTP ACME), and Port 443 (HTTPS TLS 1.3). Database Port 5432 MUST NOT be exposed to the public internet.

#### B. Acceptance Criteria Matrix
| Requirement ID | Component | Requirement Statement | Verification Method | Pass Threshold |
| :--- | :--- | :--- | :--- | :--- |
| **AC-001** | Operating System | Host runs Ubuntu Server 24.04 LTS (x86_64). | `lsb_release -a` | `Ubuntu 24.04` reported |
| **AC-002** | Storage Subsystem | NVMe partition mounted on ZFS/Btrfs CoW with `noatime,nodiratime`. | `mount \| grep -E 'zfs\|btrfs'` | CoW filesystem active with `noatime` |
| **AC-003** | Kernel & Swap | 64GB NVMe swap active and sysctl parameters loaded. | `swapon --show` & `sysctl vm.swappiness net.core.somaxconn` | 64GB swap verified; `10` and `65535` returned |
| **AC-004** | Port Perimeter | Public interface rejects direct TCP 5432 connections. | `nmap -p 5432 <public-ip>` | Port reported `filtered` or `closed` |

---

### Stage Gate 1: Off-the-Shelf Turnkey Core (Broadsea 3.5)

#### A. Requirements Specification
1. **REQ-010 (Turnkey Orchestration)**: The deployment MUST support bootstrapping the baseline platform using official Broadsea 3.5 images (`ohdsi/broadsea-atlasdb`, `ohdsi/webapi`, `ohdsi/atlas`, `broadsea-solr-vocab`, `ohdsi/broadsea-hades`).
2. **REQ-011 (WebAPI Classic Lifecycle)**: WebAPI 2.14 MUST start successfully, complete Flyway schema migrations, and return healthy telemetry at `/WebAPI/info`.
3. **REQ-012 (Atlas Classic Frontend)**: Atlas Classic UI MUST be accessible via web browser and connect to WebAPI without CORS or Mixed Content errors.
4. **REQ-013 (Solr Vocabulary Search)**: Broadsea Solr MUST be operational on port 8983 and respond to lexical autocomplete queries in `< 100ms`.

#### B. Acceptance Criteria Matrix
| Requirement ID | Component | Requirement Statement | Verification Method | Pass Threshold |
| :--- | :--- | :--- | :--- | :--- |
| **AC-010** | Broadsea Core | All baseline Broadsea containers enter running state. | `docker compose ps` | Status `Up` / `healthy` for all services |
| **AC-011** | WebAPI Backend | WebAPI returns valid build version and active sources. | `curl -s http://localhost:8080/WebAPI/info` | HTTP 200 with JSON payload |
| **AC-012** | Atlas Frontend | Atlas Classic web interface loads cleanly. | `curl -s http://localhost:8082` | HTTP 200 with HTML title `ATLAS` |
| **AC-013** | Solr Vocabulary | Solr core responds to query ping. | `curl -s http://localhost:8983/solr/admin/info/system` | HTTP 200 with system telemetry |

---

### Stage Gate 2: Data Scaling & Dedicated R Server

#### A. Requirements Specification
1. **REQ-020 (PostgreSQL 16 OLAP Tuning)**: PostgreSQL MUST be tuned for analytical workloads with `shared_buffers` set to at least 25% of host RAM (minimum 32GB, recommended 64GB), `work_mem = 256MB`, and `random_page_cost = 1.1`.
2. **REQ-021 (Vocabulary Ingestion & Indexing)**: Full Athena vocabulary release (~10M rows) MUST be loaded into `vocab_54` with trigram GIN indexes (`gin(concept_name gin_trgm_ops)`). Autocomplete queries MUST complete in `< 50ms`.
3. **REQ-022 (Redis Task Broker)**: Redis MUST be operational on port 6379 to manage asynchronous job queues and analytical results caching.
4. **REQ-023 (Dedicated R Server Standard & Supported Package Catalog)**: All OHDSI analytical R libraries and the complete HADES suite MUST run inside a single containerized R Server (`broadsea-hades` / RStudio Server on port 8787). The implementation MUST NOT deploy fragmented Plumber microservices. The Dedicated R Server MUST maintain pre-installed support across 6 core functional domains:
   - *Database & SQL*: `DatabaseConnector`, `SqlRender`, `ParallelLogger`, `Andromeda`.
   - *Cohort & Phenotyping*: `Capr`, `CirceR`, `CohortGenerator`, `CohortConstructor`, `PhenotypeLibrary`, `Phenotyper`, `Phenelope`, `PheValuator`, `Keeper`, `ProtocolGenerator`.
   - *Characterization & Diagnostics*: `CohortDiagnostics`, `CohortIncidence`, `FeatureExtraction`, `Characterization`, `ClinicalCharacteristics`, `DbDiagnostics`, `DataQualityDashboard`.
   - *Causal Estimation*: `CohortMethod`, `SelfControlledCaseSeries`, `Cyclops`, `EvidenceSynthesis`, `EmpiricalCalibration`, `MethodEvaluation`, `CaseControl`, `CaseCrossover`.
   - *Patient Prediction*: `PatientLevelPrediction`, `DeepPatientLevelPrediction`, `BigKnn`.
   - *Results & Studies*: `Strategus`, `ResultModelManager`, `ROhdsiWebApi`, `OhdsiShinyModules`, `ShinyAppBuilder`, `Eunomia`, `Taxis`.
5. **REQ-024 (External Study Packages)**: Network studies such as [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis) MUST be executed directly within the Dedicated R Server using standard `DatabaseConnector` queries.
6. **REQ-025 (R Server Connection to OMOP CDM & Vocabulary Database)**: The Dedicated R Server MUST have direct internal network connectivity and JDBC drivers to query both the OMOP CDM event tables (`cdm_synthea100k`) and Athena Vocabularies (`vocab_54`), as well as REST access to WebAPI via `ROhdsiWebApi`.
7. **REQ-026 (PostgreSQL Database Connected to Atlas Instance)**: The PostgreSQL database cluster MUST be connected to WebAPI (schemas `webapi`, `cdm`, `vocab`, `results`), and the Atlas web application instance MUST be connected to WebAPI:
   - Atlas users MUST be able to browse data sources, search standardized vocabularies, construct cohort definitions, and view generation counts from the shared PostgreSQL database.
8. **REQ-027 (OHDSI Study Shiny Apps & Report Deployment)**: The platform MUST provide a containerized Shiny runtime (`ohdsi-shiny` on port 3838) mounting `/srv/shiny-server/` for interactive dashboards (`CohortDiagnostics`, `Taxis`) and `/srv/reports/` for static analytical Quarto / RMarkdown HTML reports.
9. **REQ-028 (RStudio Server Free / Open-Source Edition Hosting & Verification)**: The Dedicated R Server MUST deploy the free, open-source edition of RStudio Server (AGPL v3, e.g. `ohdsi/broadsea-hades:1.19.0`), requiring zero commercial licenses:
   - Accessible over the unified public domain URL at `https://<domain>/rstudio/` with WebSocket upgrades.
   - Configured with PAM user credentials (`HADES_USER` and `HADES_PASSWORD`) and passwordless `sudo` for package installations.

#### B. Acceptance Criteria Matrix
| Requirement ID | Component | Requirement Statement | Verification Method | Pass Threshold |
| :--- | :--- | :--- | :--- | :--- |
| **AC-020** | Database Tuning | PostgreSQL `shared_buffers` configured >= 32GB. | `psql -c "SHOW shared_buffers;"` | Value `>= 32GB` |
| **AC-021** | Vocabulary Search | Autocomplete lookup completes under threshold. | `psql -c "EXPLAIN ANALYZE SELECT * FROM vocab_54.concept WHERE concept_name ILIKE '%diabetes%' LIMIT 20;"` | Execution time `< 50ms` |
| **AC-022** | Redis Broker | Redis responds to PING command. | `redis-cli ping` | Returns `PONG` |
| **AC-023** | Dedicated R Server | RStudio Server is live with complete HADES & OHDSI package suite. | Inside R Server: `sapply(c('DatabaseConnector', 'Capr', 'CohortMethod', 'PatientLevelPrediction', 'Strategus', 'ROhdsiWebApi', 'Keeper'), requireNamespace, quietly=TRUE)` | All required packages return `TRUE` |
| **AC-024** | Zero Microservices | Zero custom Plumber microservice containers running. | `docker ps --filter "name=plumber"` | Returns 0 running containers |
| **AC-025** | OMOP & Vocab Connect | R Server connects to both OMOP CDM and Vocabulary schemas. | Inside R Server: `DatabaseConnector::querySql(conn, "SELECT COUNT(*) FROM cdm_synthea100k.person")` & `querySql(conn, "SELECT COUNT(*) FROM vocab_54.concept")` | Both queries return non-zero counts (> 0) |
| **AC-026** | Atlas DB Connection | Atlas instance connects to PostgreSQL via WebAPI. | `curl -s http://localhost:8080/WebAPI/source/sources` & `curl -s http://localhost:8080/WebAPI/vocabulary/vocab_54/search/aspirin` | Returns configured data sources and vocabulary search results |
| **AC-027** | Shiny Apps & Reports | Shiny server is running and mounts study apps directory. | `curl -s http://localhost:3838/` | HTTP 200 with Shiny Server response |
| **AC-028** | RStudio Free Edition Test | Hosted RStudio Server Open Source loads, authenticates, and executes test query. | `curl -s -k -L https://<domain>/rstudio/auth-sign-in` & interactive test query | HTTP 200 with `RStudio` HTML title; R console executes `SELECT 1` |

---

### Stage Gate 3: Atlas 3.0 Next-Gen Frontend, WebAPI 3.0 & Public URL Ingress

#### A. Requirements Specification
1. **REQ-030 (Atlas 3.0 Micro-Frontend)**: Atlas 3.0 (Vue 3 / single-spa) MUST be deployed side-by-side with Atlas Classic, served at the root URL path (`/`).
2. **REQ-031 (WebAPI 3.0 Backend)**: WebAPI 3.0 (Spring Boot 3, Java 21) MUST be deployed with TrexSQL DuckDB query caching enabled.
3. **REQ-032 (Single-Domain Edge Ingress on Public URL)**: Nginx reverse proxy MUST route all platform components under ONE public URL domain over TLS 1.3:
   - `/` -> Atlas 3.0 Frontend
   - `/atlas/` -> Atlas Classic Frontend
   - `/WebAPI/` -> WebAPI Backend Engine
   - `/rstudio/` -> Dedicated R Server (RStudio Server Web IDE with WebSockets)
   - `/shiny/` -> Interactive OHDSI Study Shiny Apps & Dashboards (with WebSockets)
   - `/reports/` -> Static & Interactive OHDSI Study Analytical HTML Reports
   - `/mcp/` -> Model Context Protocol (FastMCP) AI Gateway
4. **REQ-033 (WebSocket Protocol Ingress)**: The reverse proxy MUST support WebSocket upgrades (`Upgrade $http_upgrade`, `Connection "upgrade"`) on `/rstudio/` and `/shiny/` paths to support interactive reactive UI updates.
5. **REQ-034 (Public Shiny Apps & Analytical Reports Ingress)**: Interactive Shiny applications and static HTML reports MUST be directly accessible over public HTTPS ingress at `/shiny/` and `/reports/` respectively.

#### B. Acceptance Criteria Matrix
| Requirement ID | Component | Requirement Statement | Verification Method | Pass Threshold |
| :--- | :--- | :--- | :--- | :--- |
| **AC-030** | Atlas 3.0 UI | Atlas 3.0 loads at root path. | `curl -s https://<domain>/` | HTTP 200 with single-spa entrypoint |
| **AC-031** | Modern WebAPI | WebAPI 3.0 responds with TrexSQL enabled. | `curl -s https://<domain>/WebAPI/info` | HTTP 200; TrexSQL status active |
| **AC-032** | RStudio Ingress | Nginx proxies RStudio over HTTPS with WebSockets. | `curl -s -k https://<domain>/rstudio/` | HTTP 200 / 302 redirect to RStudio auth |
| **AC-033** | Single Ingress | All endpoints resolve under the single domain name. | Browser navigation across `/`, `/atlas/`, `/WebAPI/`, `/rstudio/`, `/shiny/` | Zero cross-origin or port redirect errors |
| **AC-034** | Public Shiny & Reports | OHDSI Study Shiny apps and reports load via public URL. | `curl -s -k https://<domain>/shiny/` & `curl -s -k https://<domain>/reports/` | HTTP 200 with Shiny index and report index |

---

### Stage Gate 4: Sovereign Agentic Tier, Federated Network & Developer Playground

#### A. Requirements Specification
1. **REQ-040 (Official StudyAgent Container)**: The AI agent gateway MUST use the official [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) container (`ohdsi/study-agent:latest`), configured via `studyagent/config.yaml`.
2. **REQ-041 (Multi-Transport MCP Standards)**: The gateway MUST expose Model Context Protocol (MCP) endpoints supporting Server-Sent Events (SSE) at `/mcp/sse` and Streamable HTTP JSON-RPC at `/mcp/messages`.
3. **REQ-042 (Authentication & RBAC)**: Requests to MCP tool endpoints MUST enforce scoped Bearer API tokens (`admin`, `study_designer`, `readonly`). Requests lacking valid tokens MUST be rejected with HTTP 401.
4. **REQ-043 (AST SQL Guardrail)**: Incoming SQL queries from external AI agents MUST be validated via an Abstract Syntax Tree (AST) parser to block destructive commands (`DROP`, `ALTER`, `TRUNCATE`, `DELETE`).
5. **REQ-044 (Small Cell Suppression)**: The platform MUST support Small Cell Suppression (masking counts `< 5` as `"< 5"`), with a configurable bypass (`ENFORCE_SMALL_CELL_SUPPRESSION=false`) for synthetic benchmark datasets so researchers can inspect raw distributions.
6. **REQ-045 (Local Sovereign LLM Inference)**: The environment MUST support local, zero-data-egress LLM inference via containerized Ollama (`ollama/ollama`) on port 11434 serving open-weights models (`llama3.3:70b`, `qwen2.5:32b`).
7. **REQ-046 (WebApiMcp Bridge Server)**: The platform MUST deploy Martijn Schuemie's WebAPI Model Context Protocol server ([`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp)) as a containerized service (`webapi-mcp` on internal port 8765, routed at `https://<domain>/webapi-mcp/`), exposing cohort definition, concept set, and WebAPI tools to LLMs.
8. **REQ-047 (OHDSI Arachne Distributed Research Network Node)**: The platform MUST support the OHDSI Arachne federated study execution framework ([`OHDSI/ArachneDataNode`](https://github.com/OHDSI/ArachneDataNode) on port 8880 routed at `/arachne/` & [`OHDSI/ArachneExecutionEngine`](https://github.com/OHDSI/ArachneExecutionEngine) on port 8888) to execute multi-site network studies with aggregate export.
9. **REQ-048 (Agentic Software Interoperability & Standard MCP Client Integration)**: The platform MUST ensure full interoperability with external agentic AI software (Claude Desktop, Cursor, Antigravity IDE) adhering to Model Context Protocol (MCP) JSON-RPC 2.0 specifications.
10. **REQ-049 (Fast Storage Snapshot & Rollback Architecture)**: The host environment MUST implement filesystem-level Copy-on-Write (CoW) snapshots (ZFS or Btrfs) for the PostgreSQL data directory (`/var/lib/postgresql/data`) and R workspace volume (`/home/ohdsi`), providing sub-minute rollback (`< 30 seconds`) when experimentation corrupts databases. Additionally, a compressed golden baseline dump (`cdm_golden_baseline.dump.gz`) MUST be maintained in `/opt/ohdsi/seeds/` with an automated restoration script restoring the baseline in `< 5 minutes`.
11. **REQ-050 (Developer Superuser & Admin Access Permissions)**: The platform MUST grant full superuser and administrative privileges to data science and informatics researchers:
    - Dedicated PostgreSQL superuser credentials (`ohdsi_admin` with `SUPERUSER`, `CREATEDB`, `CREATEROLE`) and private scratch schemas (`scratch_<username>`).
    - Passwordless `sudo` inside the Dedicated R Server (`broadsea-hades`) allowing dynamic OS package installations (`apt-get install`) and GitHub package compilations.
    - Global administrator role in Atlas and WebAPI for custom Source Daimon management and cohort generation queue control.
12. **REQ-051 (Agentic AI Testing Sandbox & Verbose Debug Telemetry)**: The MCP gateways MUST support high-concurrency automated agentic execution with verbose compiler and AST error payloads returned in JSON-RPC responses when SQL transpilation fails, enabling autonomous LLMs to self-correct during experiment loops.
13. **REQ-052 (System Resilience, cgroups & Crash Isolation)**: The host and container runtime MUST withstand aggressive developer and agent hammering without taking down host or database services:
    - Strict cgroup memory clamping on compute containers (`broadsea-hades`: `mem_limit: 48g`, `shm_size: 16g`; `ohdsi-postgres`: `mem_limit: 48g`).
    - Dedicated 64 GB swap partition on NVMe PCIe Gen4 storage with `vm.swappiness = 10` to absorb sudden analytical memory spikes without kernel panics.
    - PostgreSQL query guardrails: `statement_timeout = '15min'`, `idle_in_transaction_session_timeout = '10min'`, and `temp_file_limit = '50GB'`.
14. **REQ-053 (Developer Web SQL Studio & MinIO S3 Object Store)**: The platform MUST provide browser-based developer tooling:
    - Containerized Web SQL Studio (`cloudbeaver:latest`) accessible at `https://<domain>/sql/` for instant schema inspection, visual explain plans, and query drafting.
    - Containerized S3-compatible object storage (`minio:latest`) accessible at `https://<domain>/s3/` for local Strategus study artifact storage, Parquet exports, and pipeline cache.

#### B. Acceptance Criteria Matrix
| Requirement ID | Component | Requirement Statement | Verification Method | Pass Threshold |
| :--- | :--- | :--- | :--- | :--- |
| **AC-040** | StudyAgent Image | Deployment uses official `ohdsi/study-agent:latest`. | `docker inspect study-agent-mcp` | Image matches `ohdsi/study-agent` |
| **AC-041** | MCP Discovery | MCP endpoint exposes available tools list. | `curl -s -H "Authorization: Bearer <token>" https://<domain>/api/v1/tools` | HTTP 200 with tools array |
| **AC-042** | Auth Rejection | Unauthenticated request rejected. | `curl -s -I https://<domain>/api/v1/tools` | HTTP 401 Unauthorized |
| **AC-043** | AST Guardrail | Destructive SQL rejected with 400. | Submit `DROP TABLE person;` to query tool | HTTP 400 with guardrail violation error |
| **AC-044** | Small Cell Filter | Cell count between 1 and 4 is masked (unless bypassed). | Query count returning 3 persons | Output explicitly formatted as `"< 5"` |
| **AC-045** | Local Ollama | Ollama instance live with local model loaded. | `curl -s http://localhost:11434/api/tags` | HTTP 200 with installed model tags |
| **AC-046** | WebApiMcp Bridge | WebApiMcp operational on port 8765 routing to WebAPI. | `curl -s http://localhost:8765/health` & MCP handshake at `/webapi-mcp/mcp` | HTTP 200 with healthy status; tools list returns cohort capabilities |
| **AC-047** | Arachne Data Node | Arachne Data Node & Execution Engine operational and connected to CDM. | `curl -s http://localhost:8880/api/v1/build-number` & Execution Engine ping | HTTP 200 with Data Node build info; status healthy |
| **AC-048** | Agentic Client Interop | Standard MCP client (Claude/Cursor/Antigravity) connects, discovers tools, and executes queries. | Connect MCP client via SSE/HTTP and invoke `list_sources` or `search_concepts` | Successful JSON-RPC handshake; valid tool response returned |
| **AC-049** | Fast Snapshot & Rollback | CoW storage snapshot created and restored in `< 30s`; golden baseline restore verified. | Run `ohdsi-snapshot create test-snap && ohdsi-snapshot rollback test-snap` | Sub-minute rollback success; CDM tables intact |
| **AC-050** | Developer Admin Access | Developers authenticate as PostgreSQL superuser, RStudio sudo, and Atlas admin. | `psql -U ohdsi_admin -c "SELECT rolsuper FROM pg_roles WHERE rolname='ohdsi_admin'"` & `docker exec -it broadsea-hades sudo whoami` | Returns `true` for superuser; returns `root` for RStudio sudo |
| **AC-051** | Agentic Sandbox Telemetry | MCP gateways return verbose JSON-RPC error telemetry and honor suppression bypass. | Submit malformed SQL to `/mcp/` and check debug output; toggle suppression flag | JSON-RPC returns detailed compiler traceback; unmasked counts returned |
| **AC-052** | Crash Isolation & cgroups | Host enforces container memory limits and PostgreSQL timeouts under heavy hammering. | Trigger test memory-spike R process inside R Server (`matrix(rnorm(1e8), 1e4, 1e4)`) | Process terminated by cgroup OOM killer; PostgreSQL & Docker daemon remain 100% operational |
| **AC-053** | Web SQL Studio & MinIO | CloudBeaver Web SQL IDE and MinIO S3 console operational over HTTPS. | `curl -s -k -I https://<domain>/sql/` & `curl -s -k -I https://<domain>/s3/` | HTTP 200/302 for both developer services |

---

## 3. Master Acceptance Criteria Sign-Off Checklist

DevOps engineers must audit and verify each criterion prior to final handover to the Data Science & Informatics leads:

### Stage Gate 0: Infrastructure & Host Hardening
- [ ] **AC-001**: Ubuntu Server 24.04 LTS verified on host.
- [ ] **AC-002**: NVMe storage configured with ZFS/Btrfs CoW datasets (`noatime,nodiratime`).
- [ ] **AC-003**: 64GB NVMe swap active; sysctl parameters verified in `/etc/sysctl.d/99-ohdsi.conf`.
- [ ] **AC-004**: UFW firewall active; Port 5432 unreachable from public internet.

### Stage Gate 1: Off-the-Shelf Broadsea Turnkey Core
- [ ] **AC-010**: Broadsea 3.5 containers booted via `docker-compose.broadsea.yml`.
- [ ] **AC-011**: WebAPI Classic responds with status `UP` at `/WebAPI/info`.
- [ ] **AC-012**: Atlas Classic accessible via browser without CORS issues.
- [ ] **AC-013**: Solr search core responds to vocabulary queries in `< 100ms`.

### Stage Gate 2: Data Scaling & Dedicated R Server
- [ ] **AC-020**: PostgreSQL 16 tuned with `shared_buffers >= 32GB`.
- [ ] **AC-021**: Full Athena vocabularies loaded; trigram GIN autocomplete completes in `< 50ms`.
- [ ] **AC-022**: Redis cache responds with `PONG` on port 6379.
- [ ] **AC-023**: Dedicated R Server (`broadsea-hades`) live on port 8787 with HADES packages installed.
- [ ] **AC-024**: Zero custom Plumber microservice containers running.
- [ ] **AC-025**: R Server connects directly to OMOP CDM and Vocabulary tables via JDBC and WebAPI via REST.
- [ ] **AC-026**: PostgreSQL database connects to WebAPI and Atlas instance (data sources and vocab search functional).
- [ ] **AC-027**: OHDSI Study Shiny server is operational, mounting study apps and reports.
- [ ] **AC-028**: RStudio Server Open Source (free edition) deployed, tested, and accessible at `https://<domain>/rstudio/`.

### Stage Gate 3: Atlas 3.0 Next-Gen, WebAPI 3.0 & Public URL Ingress
- [ ] **AC-030**: Atlas 3.0 Vue 3 single-spa frontend accessible at root path (`/`).
- [ ] **AC-031**: Modern WebAPI 3.0 operational with TrexSQL DuckDB caching.
- [ ] **AC-032**: Nginx edge proxy terminates TLS 1.3 and routes RStudio WebSockets at `/rstudio/`.
- [ ] **AC-033**: Single unified public domain routing verified for all endpoints.
- [ ] **AC-034**: OHDSI Study Shiny apps and published reports accessible via public URL (`/shiny/`, `/reports/`).

### Stage Gate 4: Sovereign Agentic Tier, Federated Network & Developer Playground
- [ ] **AC-040**: Official `ohdsi/study-agent:latest` image running from `OHDSI/StudyAgent`.
- [ ] **AC-041**: MCP endpoints `/mcp/sse` and `/mcp/messages` operational with live SSE stream.
- [ ] **AC-042**: Scoped Bearer authentication enforced; 401 returned on invalid/missing tokens.
- [ ] **AC-043**: AST SQL guardrail blocks destructive queries and raw patient-level SELECTs.
- [ ] **AC-044**: Small Cell Suppression verified (masking counts `< 5`, with configurable bypass for synthetic benchmarks).
- [ ] **AC-045**: Local sovereign Ollama instance operational on port 11434.
- [ ] **AC-046**: WebApiMcp bridge server operational on port 8765 connected to WebAPI.
- [ ] **AC-047**: OHDSI Arachne Data Node & Execution Engine operational on ports 8880/8888 and connected to OMOP CDM.
- [ ] **AC-048**: Agentic software interoperability verified with MCP client (tool discovery and execution pass 100%).
- [ ] **AC-049**: Fast snapshot and sub-minute rollback verified via ZFS/Btrfs CoW and golden baseline restore.
- [ ] **AC-050**: Developer superuser credentials (`ohdsi_admin`), RStudio passwordless sudo, and Atlas admin roles verified.
- [ ] **AC-051**: Agentic testing sandbox verified with verbose compiler tracebacks in JSON-RPC errors.
- [ ] **AC-052**: System resilience verified: cgroup memory clamps and PostgreSQL timeouts isolate crashes.
- [ ] **AC-053**: Developer Web SQL Studio (`/sql/`) and MinIO S3 object store (`/s3/`) verified.
