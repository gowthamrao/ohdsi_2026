# OHDSI 2026: Stage-Gated Milestone Requirements & Acceptance Criteria

> **Document Type**: Production System Requirements Specification (SRS) & Acceptance Criteria (AC)  
> **Target Audience**: Cloud DevOps Engineers, Site Reliability Engineers (SRE), Infrastructure Architects  
> **Implementation Platform**: Dedicated Bare-Metal Host (Hetzner AX/PX) or Cloud VPC (AWS / GCP / Azure)  
> **Status**: Approved Production Standard

---

## 1. Architecture Overview & Mental Model

This specification defines the infrastructure, container roster, ingress routing, and verification standards for deploying the **OHDSI Analytical Research Platform**.

To a systems or DevOps engineer, the platform is a standard **3-tier data warehouse and analytics system**:

```
                         SYSTEM ARCHITECTURE & INGRESS TOPOLOGY
                                          │
                     [ Public Internet / Researchers / AI Agents ]
                                          │
                            [ HTTPS: 443 / TLS 1.3 AEAD ]
                                          │
                                          ▼
     ┌─────────────────────────────────────────────────────────────────────────┐
     │                  Nginx Reverse Proxy & Edge Gateway                     │
     │      - TLS 1.3 Termination, HSTS 2-Year, Multi-Zone Rate Limiting       │
     │      - Small Cell Privacy Suppression Filter (MIN_CELL_COUNT >= 5)      │
     │      - Single-Domain Path Routing (/atlas, /WebAPI, /mcp, /rstudio, /)  │
     └───────┬────────────────────┬────────────────────┬───────────────────────┘
             │                    │                    │
             ▼                    ▼                    ▼
   ┌───────────────────┐┌───────────────────┐┌─────────────────────────────────┐
   │ Frontend Web Apps ││  Data & R Engine  ││  Agentic AI & MCP Gateway       │
   │ - Atlas 3.0 (Vue3)││ - WebAPI Classic  ││ - FastMCP Server (:8790)        │
   │ - Atlas Classic   ││ - WebAPI 3.0      ││   (SSE, Streamable HTTP, Stdio) │
   │ - Shiny Studios   ││ - Dedicated R Svr ││ - Local Ollama LLM (:11434)     │
   │ - RStudio (:8787) ││   (broadsea-hades)││ - Redis Task Broker (:6379)     │
   └─────────┬─────────┘└─────────┬─────────┘└────────────────┬────────────────┘
             │                    │                           │
             └────────────────────┼───────────────────────────┘
                                  ▼
     ┌─────────────────────────────────────────────────────────────────────────┐
     │            PostgreSQL 16 High-Throughput RDBMS Cluster                  │
     │   - 64GB shared_buffers, NVMe Gen4 Storage (noatime,nodiratime)         │
     │   - Master Lookup Schema (vocab_54: ~10M records with GIN indexes)      │
     │   - Structured Data Warehouse Schemas (cdm_synthea100k, cdm_synpuf_23m) │
     │   - Precomputed Aggregate Cache Schemas (results)                       │
     │   - Port 5432 (Quarantined to Private Docker Network / VPN / Tailscale) │
     └─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. The DevOps Rosetta Stone (Terminology Translation)

| Domain Term | Software Engineering Equivalent | Technical Function |
| :--- | :--- | :--- |
| **OMOP CDM** | Relational Data Warehouse Schema | Standardized PostgreSQL relational schema storing subject event tables (`person`, `visit_occurrence`, `condition_occurrence`). |
| **Athena / Vocabularies** | Master Lookup Dictionary (~10M rows) | Lookup table mapping disparate coding systems to uniform integer primary keys (`concept_id`). Uses trigram GIN indexes. |
| **Source Daimon** | Schema Routing Registry Table | Configuration table in PostgreSQL mapping logical database aliases to physical schemas (`cdm`, `vocab`, `results`). |
| **WebAPI** | Java Spring Boot REST Backend | Core API service providing data source management, SQL transpilation, security integration, and query execution. |
| **Atlas (Classic & 3.0)** | Web Application Frontend | Single-Page Application (Classic: Knockout.js; 3.0: Vue 3 / single-spa) providing query authoring and visualization. |
| **HADES / Dedicated R Server** | Dedicated R Compute Server | Containerized RStudio Server (`broadsea-hades`, port 8787) hosting all OHDSI R libraries natively. Replaces 50+ fragmented microservices. |
| **Achilles** | Precomputed Aggregate Cache | Batch processing job that precomputes table counts and distributions, storing them in the `results` schema for instant dashboard retrieval. |
| **Circe / Capr** | Query Transpiler & AST Builder | Compiles JSON or R criteria into target database SQL dialects (PostgreSQL, Snowflake, BigQuery). |
| **Cohort / Phenotype** | Entity Segment / Filter Query | Specific criteria defining a population slice (e.g. subjects meeting specific criteria within a date window). |
| **StudyAgent / MCP** | FastMCP Tool-Calling Gateway | Official [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) server exposing platform capabilities to AI models via Model Context Protocol. |
| **Small Cell Suppression** | Data Privacy Masking Filter | Re-identification protection filter automatically masking query count results where `0 < count < 5` as `"< 5"`. |

---

## 3. Stage-Gated Milestone Specifications

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           STAGE-GATED ROADMAP                                          │
├─────────┬──────────────────────────────┬───────────────────────────────────────────┬───────────────────┤
│ Stage   │ Milestone Focus              │ Core Technologies                         │ Exit Verification │
├─────────┼──────────────────────────────┼───────────────────────────────────────────┼───────────────────┤
│ Gate 0  │ Host & Kernel Hardening      │ Ubuntu 24.04 LTS, NVMe noatime, UFW       │ Ports closed      │
│ Gate 1  │ Turnkey Broadsea Core        │ broadsea-atlasdb, webapi, atlas, hades    │ WebAPI /info UP   │
│ Gate 2  │ High-Capacity Data & Vocab   │ Postgres 16 (64GB shared_buffers), Redis  │ Vocab query <150ms│
│ Gate 3  │ Atlas 3.0 & WebAPI 3.0       │ Atlas 3.0 (Vue 3), WebAPI 3.0, R Server   │ Atlas 3.0 live    │
│ Gate 4  │ MCP AI & Federated Network   │ StudyAgent, WebApiMcp, Arachne, TLS 1.3   │ Gate 4 suite 100% │
└─────────┴──────────────────────────────┴───────────────────────────────────────────┴───────────────────┘
```

---

### Stage Gate 0: Host Infrastructure, Storage & Kernel Hardening

#### A. Requirements Specification
1. **REQ-001 (Host Operating System)**: The host MUST run a clean, 64-bit installation of Ubuntu Server 24.04 LTS.
2. **REQ-002 (Storage Mount)**: Dedicated PCIe Gen4 NVMe storage MUST be mounted at `/var/lib/postgresql/data` with filesystem options `noatime,nodiratime,data=writeback`.
3. **REQ-003 (Kernel Tuning)**: Kernel sysctl parameters MUST be persisted at `/etc/sysctl.d/99-ohdsi.conf` configuring low swappiness (`vm.swappiness = 10`), dirty page flushing (`vm.dirty_ratio = 15`), and high socket backlog (`net.core.somaxconn = 65535`).
4. **REQ-004 (Perimeter Security)**: Host firewall (UFW) MUST block all external ports except Port 22 (SSH), Port 80 (HTTP ACME), and Port 443 (HTTPS TLS 1.3). Database Port 5432 MUST NOT be exposed to the public internet.

#### B. Acceptance Criteria Matrix
| Requirement ID | Component | Requirement Statement | Verification Method | Pass Threshold |
| :--- | :--- | :--- | :--- | :--- |
| **AC-001** | Operating System | Host runs Ubuntu Server 24.04 LTS (x86_64). | `lsb_release -a` | `Ubuntu 24.04` reported |
| **AC-002** | Storage Subsystem | NVMe partition mounted with `noatime,nodiratime`. | `mount \| grep postgresql` | `noatime` and `nodiratime` present |
| **AC-003** | Kernel Parameters | Sysctl parameters active in running kernel. | `sysctl vm.swappiness net.core.somaxconn` | `10` and `65535` returned |
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
| **AC-010** | Broadsea Core | All 5 baseline Broadsea containers enter running state. | `docker compose ps` | Status `Up` / `healthy` for all 5 |
| **AC-011** | WebAPI Backend | WebAPI returns valid build version and active sources. | `curl -s http://localhost:8080/WebAPI/info` | HTTP 200 with JSON payload |
| **AC-012** | Atlas Frontend | Atlas Classic web interface loads cleanly. | `curl -s http://localhost:8082` | HTTP 200 with HTML title `ATLAS` |
| **AC-013** | Solr Vocabulary | Solr core responds to query ping. | `curl -s http://localhost:8983/solr/admin/info/system` | HTTP 200 with system telemetry |

---

### Stage Gate 2: Data Scaling & Dedicated R Server

#### A. Requirements Specification
1. **REQ-020 (PostgreSQL 16 OLAP Tuning)**: PostgreSQL MUST be tuned for analytical workloads with `shared_buffers` set to at least 25% of host RAM (minimum 32GB, recommended 64GB), `work_mem = 256MB`, and `random_page_cost = 1.1`.
2. **REQ-021 (Vocabulary Ingestion & Indexing)**: Full Athena vocabulary release (~10M rows) MUST be loaded into `vocab_54` with trigram GIN indexes (`gin(concept_name gin_trgm_ops)`). Autocomplete queries MUST complete in `< 50ms`.
3. **REQ-022 (Redis Task Broker)**: Redis MUST be operational on port 6379 to manage asynchronous job queues and analytical results caching.
4. **REQ-023 (Dedicated R Server Standard & Supported Package Catalog)**: All OHDSI analytical R libraries and the complete HADES suite MUST run inside a single containerized R Server (`broadsea-hades` / RStudio Server on port 8787). The implementation MUST NOT deploy fragmented Plumber microservices. The Dedicated R Server MUST maintain pre-installed support for the following packages across 6 core functional domains:

   | Tier / Functional Domain | Supported Packages | Capabilities & Upstream Repository |
   | :--- | :--- | :--- |
   | **1. Database & SQL Infrastructure** | `DatabaseConnector`<br>`SqlRender`<br>`ParallelLogger`<br>`Andromeda` | JDBC connectivity across RDBMS dialects, parameterized SQL transpilation, high-throughput in-memory/disk data frames ([OHDSI/Hades](https://github.com/OHDSI/Hades)). |
   | **2. Cohort Definition & Phenotyping** | `Capr`<br>`CirceR`<br>`CohortGenerator`<br>`CohortConstructor`<br>`PhenotypeLibrary`<br>`Phenotyper`<br>`Phenelope`<br>`PheValuator`<br>`Keeper`<br>`ProtocolGenerator` | Programmatic cohort authoring, Circe JSON compiler, phenotype extraction, semi-supervised phenotype evaluation (PPV/sensitivity), and timeline adjudication ([OHDSI/Capr](https://github.com/OHDSI/Capr), [OHDSI/Keeper](https://github.com/OHDSI/Keeper)). |
   | **3. Characterization & Diagnostics** | `CohortDiagnostics`<br>`CohortIncidence`<br>`FeatureExtraction`<br>`Characterization`<br>`ClinicalCharacteristics`<br>`DbDiagnostics`<br>`DataQualityDashboard` | Phenotype characterization, incidence rate computation, covariate extraction, table shell generation, 24-point study feasibility, and data quality checks ([OHDSI/CohortDiagnostics](https://github.com/OHDSI/CohortDiagnostics)). |
   | **4. Population-Level Causal Estimation** | `CohortMethod`<br>`SelfControlledCaseSeries`<br>`Cyclops`<br>`EvidenceSynthesis`<br>`EmpiricalCalibration`<br>`MethodEvaluation`<br>`CaseControl`<br>`CaseCrossover` | New-user active comparator cohort designs, within-person self-controlled designs, large-scale L1/L2 regularized regression, and empirical calibration using negative controls ([OHDSI/CohortMethod](https://github.com/OHDSI/CohortMethod)). |
   | **5. Patient-Level Prediction (ML/DL)** | `PatientLevelPrediction`<br>`DeepPatientLevelPrediction`<br>`BigKnn` | Machine learning (LASSO, Random Forest, XGBoost) and deep learning clinical prediction pipelines on OMOP CDM ([OHDSI/PatientLevelPrediction](https://github.com/OHDSI/PatientLevelPrediction)). |
   | **6. Execution, Results & Visualization** | `Strategus`<br>`ResultModelManager`<br>`ROhdsiWebApi`<br>`OhdsiShinyModules`<br>`ShinyAppBuilder`<br>`Eunomia`<br>`Taxis` | Multi-analysis pipeline orchestration, Results Data Model DDL manager, WebAPI REST integration, interactive Shiny modules, synthetic CDM testbed, and association mining ([ohdsi-studies/Taxis](https://github.com/ohdsi-studies/Taxis)). |

5. **REQ-024 (External Study Packages)**: Network studies such as [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis) MUST be executed directly within the Dedicated R Server using standard `DatabaseConnector` queries.
6. **REQ-025 (R Server Connection to OMOP CDM & Vocabulary Database)**: The Dedicated R Server MUST have direct internal network connectivity to the PostgreSQL database housing both the OMOP CDM event tables (`person`, `condition_occurrence`, `visit_occurrence`, etc.) and the standardized Athena Vocabulary tables (`concept`, `concept_ancestor`, `concept_relationship`, `concept_synonym`):
   - **Direct JDBC Access**: The R Server MUST be provisioned with JDBC drivers (`DATABASECONNECTOR_JAR_FOLDER`) and environment connection parameters (`CDM_SERVER`, `CDM_PORT`, `CDM_DATABASE`, `CDM_SCHEMA`, `VOCAB_SCHEMA`, `RESULTS_SCHEMA`, `CDM_USER`, `CDM_PASSWORD`) to execute queries via `DatabaseConnector`.
   - **Full Vocabulary Exploration**: The R Server MUST be able to perform concept ancestor lookups and concept set expressions directly against the vocabulary schema.
   - **WebAPI Integration**: The R Server MUST be configured with `WEBAPI_URL` allowing `ROhdsiWebApi` to fetch cohort definitions, export concept sets, and execute cohort generation directly against WebAPI.
7. **REQ-026 (PostgreSQL Database Connected to Atlas Instance)**: The PostgreSQL database cluster MUST be connected to WebAPI (schemas `webapi`, `cdm`, `vocab`, `results`), and the Atlas web application instance MUST be connected to WebAPI:
   - Atlas users MUST be able to browse data sources (`source`, `source_daimon`), search standardized vocabularies, construct cohort definitions, and view precomputed cohort generation counts from the shared PostgreSQL database.
8. **REQ-027 (OHDSI Study Shiny Apps & Report Deployment)**: The platform MUST provide a containerized Shiny runtime (`ohdsi-shiny` / `broadsea-open-shiny-server` on port 3838) capable of deploying interactive Shiny applications and published reports from OHDSI network studies:
   - Supported interactive apps include `CohortDiagnostics` viewer, `CohortIncidence` viewer, `Characterization` viewer, `PheValuator` viewer, `OhdsiShinyModules`, `ShinyAppBuilder`, and study-specific results dashboards (e.g. [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis)).
   - The Shiny runtime MUST mount `/srv/shiny-server/` for interactive dashboards and `/srv/reports/` for static analytical Quarto / RMarkdown reports.
9. **REQ-028 (RStudio Server Free / Open-Source Edition Hosting & Verification)**: The Dedicated R Server MUST deploy the free, open-source edition of RStudio Server (AGPL v3, e.g. Rocker / `ohdsi/broadsea-hades:1.19.0`), requiring zero commercial licenses:
   - **Hosted Public Web IDE**: Accessible over the unified public domain URL at `https://<domain>/rstudio/`.
   - **Authentication & Security**: Configured with PAM user credentials (`HADES_USER` and `HADES_PASSWORD`).
   - **End-to-End Testing**: Must be tested by verifying: (1) public URL loads RStudio login prompt (`/rstudio/auth-sign-in`), (2) authenticated session establishes WebSocket connection, and (3) interactive R console executes queries against the connected OMOP CDM and Vocabulary database.

#### B. Acceptance Criteria Matrix
| Requirement ID | Component | Requirement Statement | Verification Method | Pass Threshold |
| :--- | :--- | :--- | :--- | :--- |
| **AC-020** | Database Tuning | PostgreSQL `shared_buffers` configured >= 32GB. | `psql -c "SHOW shared_buffers;"` | Value `>= 32GB` |
| **AC-021** | Vocabulary Search | Autocomplete lookup completes under threshold. | `psql -c "EXPLAIN ANALYZE SELECT * FROM vocab_54.concept WHERE concept_name ILIKE '%diabetes%' LIMIT 20;"` | Execution time `< 50ms` |
| **AC-022** | Redis Broker | Redis responds to PING command. | `redis-cli ping` | Returns `PONG` |
| **AC-023** | Dedicated R Server | RStudio Server is live with complete HADES & OHDSI package suite. | Inside R Server: `sapply(c('DatabaseConnector', 'Capr', 'CohortMethod', 'PatientLevelPrediction', 'Strategus', 'ROhdsiWebApi', 'Keeper'), requireNamespace, quietly=TRUE)` | All required packages return `TRUE` |
| **AC-024** | Zero Microservices | Zero custom Plumber microservice containers running. | `docker ps --filter "name=plumber"` | Returns 0 running containers |
| **AC-025** | OMOP & Vocab Connect | R Server connects to both OMOP CDM and Vocabulary schemas. | Inside R Server: `DatabaseConnector::querySql(conn, "SELECT COUNT(*) FROM cdm.person")` & `querySql(conn, "SELECT COUNT(*) FROM vocab_54.concept")` | Both queries return non-zero counts (> 0) |
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

#### B. Acceptance Criteria Matrix
| Requirement ID | Component | Requirement Statement | Verification Method | Pass Threshold |
| :--- | :--- | :--- | :--- | :--- |
| **AC-030** | Atlas 3.0 UI | Atlas 3.0 loads at root path. | `curl -s https://<domain>/` | HTTP 200 with single-spa entrypoint |
| **AC-031** | Modern WebAPI | WebAPI 3.0 responds with TrexSQL enabled. | `curl -s https://<domain>/WebAPI/info` | HTTP 200; TrexSQL status active |
| **AC-032** | RStudio Ingress | Nginx proxies RStudio over HTTPS with WebSockets. | `curl -s -k https://<domain>/rstudio/` | HTTP 200 / 302 redirect to RStudio auth |
| **AC-033** | Single Ingress | All endpoints resolve under the single domain name. | Browser navigation across `/`, `/atlas/`, `/WebAPI/`, `/rstudio/`, `/shiny/` | Zero cross-origin or port redirect errors |
| **AC-034** | Public Shiny & Reports | OHDSI Study Shiny apps and reports load via public URL. | `curl -s -k https://<domain>/shiny/` & `curl -s -k https://<domain>/reports/` | HTTP 200 with Shiny index and report index |

---

### Stage Gate 4: Sovereign Agentic Tier (Hosted FastMCP & BYO-Agent)

#### A. Requirements Specification
1. **REQ-040 (Official StudyAgent Container)**: The AI agent gateway MUST use the official [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) container (`ohdsi/study-agent:latest`), configured via `studyagent/config.yaml`.
2. **REQ-041 (Multi-Transport MCP Standards)**: The gateway MUST expose Model Context Protocol (MCP) endpoints supporting:
   - Server-Sent Events (SSE) at `/mcp/sse`.
   - Streamable HTTP JSON-RPC at `/mcp/messages`.
3. **REQ-042 (Authentication & RBAC)**: Requests to MCP tool endpoints MUST enforce scoped Bearer API tokens (`admin`, `study_designer`, `readonly`). Requests lacking valid tokens MUST be rejected with HTTP 401.
4. **REQ-043 (AST SQL Guardrail)**: Incoming SQL queries from external AI agents MUST be validated via an Abstract Syntax Tree (AST) parser:
   - Destructive commands (`DROP`, `ALTER`, `TRUNCATE`, `DELETE`, `UPDATE`, `INSERT`) MUST be blocked with HTTP 400.
   - Raw queries selecting individual patient records without aggregation MUST be blocked.
5. **REQ-044 (Small Cell Suppression)**: All aggregate query responses MUST enforce Small Cell Suppression: any person count `0 < count < 5` MUST be masked as `"< 5"`.
6. **REQ-045 (Local Sovereign LLM Inference)**: The environment MUST support local, zero-data-egress LLM inference via containerized Ollama (`ollama/ollama`) on port 11434 serving open-weights models (`llama3.3:70b`, `qwen2.5:32b`).
7. **REQ-046 (WebApiMcp Bridge Server)**: The platform MUST deploy Martijn Schuemie's WebAPI Model Context Protocol server ([`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp)) as a containerized service (`webapi-mcp` on internal port 8765):
   - **Upstream Source**: [`https://github.com/schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp).
   - **Configuration**: Must be configured with `WEBAPI_MCP_WEBAPI_BASE_URL` pointing directly to WebAPI (e.g. `http://webapi-classic:8080/WebAPI`).
   - **MCP Tool Surface**: Exposes native WebAPI capabilities to LLM clients (Claude, Cursor, Antigravity) via JSON-RPC, including cohort definition inspection, generation triggering, concept set expression extraction, and data source lookups.
   - **Endpoints**: Health check at `http://localhost:8765/health` and MCP bridge at `http://localhost:8765/mcp`.
   - **Public Ingress**: Routed via Nginx at `https://<domain>/webapi-mcp/` with proxy buffering and caching disabled.
8. **REQ-047 (OHDSI Arachne Distributed Research Network Node)**: The platform MUST support the OHDSI Arachne federated study execution framework ([`OHDSI/ArachneDataNode`](https://github.com/OHDSI/ArachneDataNode) & [`OHDSI/ArachneExecutionEngine`](https://github.com/OHDSI/ArachneExecutionEngine)) to participate in distributed network studies:
   - **Arachne Data Node Container**: Deploys `ohdsi/arachne-data-node:latest` on internal port 8880, accessible on public HTTPS ingress at `https://<domain>/arachne/`.
   - **Execution Engine Container**: Deploys `ohdsi/arachne-execution-engine:latest` on internal port 8888 (isolated on private Docker network).
   - **Database & WebAPI Interconnectivity**: Connects directly to the PostgreSQL OMOP CDM database for study package execution and to WebAPI for study metadata synchronization.
   - **Data Privacy & Governance**: Adheres to OHDSI federated execution security standards—study code runs locally against patient data, and strictly aggregated, non-PHI summary results are returned. Supports both Standalone Mode (local execution) and Network Mode (federation with Arachne Central).
9. **REQ-048 (Agentic Software Interoperability & Standard MCP Client Integration)**: The platform MUST ensure full interoperability with external agentic AI software (Claude Desktop, Cursor, Antigravity IDE, Cline, LangChain, AutoGen):
   - **Protocol Compliance**: Endpoints MUST strictly adhere to Model Context Protocol (MCP) JSON-RPC 2.0 specifications over SSE (`/mcp/sse`, `/webapi-mcp/mcp`) and streamable HTTP.
   - **Client Configuration Artifacts**: Standard `mcpServers` JSON connection definitions MUST be provided for both StudyAgent and WebApiMcp.
   - **Tool Schema Discoverability**: Tools MUST expose self-describing JSON Schema parameter definitions for cohort management (`list_cohort_definitions`, `get_cohort_definition`, `generate_cohort`), concept exploration (`search_concepts`, `get_concept_set`), and CDM database queries with small-cell privacy guards.
   - **Edge Gateway Streaming**: The Nginx reverse proxy MUST explicitly disable proxy buffering (`proxy_buffering off`), disable chunking delays, and pass `text/event-stream` headers without timeout truncation to maintain active persistent sessions with agentic runtimes.
10. **REQ-049 (Fast Storage Snapshot & Rollback Architecture)**: The host environment MUST implement filesystem-level Copy-on-Write (CoW) snapshots (ZFS or Btrfs) for the PostgreSQL data directory (`/var/lib/postgresql/data`) and R workspace volume (`/home/ohdsi`), providing sub-minute rollback capability when experimentation corrupts databases or schemas. Additionally, a compressed golden baseline dump (`cdm_golden_baseline.dump.gz`) MUST be maintained in `/opt/ohdsi/seeds/` with an automated restoration script restoring the baseline in `< 5 minutes`.
11. **REQ-050 (Developer Superuser & Admin Access Permissions)**: The platform MUST grant full superuser and administrative privileges to data science and informatics researchers:
    - Dedicated PostgreSQL superuser credentials (`ohdsi_admin` with `SUPERUSER`, `CREATEDB`, `CREATEROLE`) and private scratch schemas (`scratch_<username>`).
    - Passwordless `sudo` inside the Dedicated R Server (`broadsea-hades`) allowing dynamic OS package installations (`apt-get install`) and GitHub package compilations.
    - Global administrator role in Atlas and WebAPI for custom Source Daimon management and cohort generation queue control.
    - Direct SSH and VS Code Remote Containers integration into compute containers.
12. **REQ-051 (Agentic AI Testing Sandbox & Verbose Debug Telemetry)**: The MCP gateways MUST support high-concurrency automated agentic execution:
    - Pre-configured Developer Mode (`DEV_MODE=true`) permitting pre-shared developer keys or local loopback authentication.
    - Verbose compiler and AST error payloads returned in JSON-RPC responses when SQL transpilation fails, enabling autonomous LLMs to self-correct during experiment loops.
    - Configurable Small Cell Suppression (`ENFORCE_SMALL_CELL_SUPPRESSION=false`) for benchmark/synthetic datasets so developers can inspect raw patient count distributions without masking.
13. **REQ-052 (System Resilience, cgroups & Crash Isolation)**: The host and container runtime MUST withstand aggressive developer and agent hammering without taking down host or database services:
    - Strict cgroup memory clamping on compute containers (`broadsea-hades`: `mem_limit: 48g`, `shm_size: 16g`; `ohdsi-postgres`: `mem_limit: 48g`).
    - Dedicated 32 GB–64 GB swap partition on NVMe PCIe Gen4 storage with `vm.swappiness = 10` to absorb sudden analytical memory spikes without kernel panics.
    - PostgreSQL query guardrails: `statement_timeout = '15min'` (overridable per-session for long batch studies), `idle_in_transaction_session_timeout = '10min'` to prevent abandoned zombie locks, and `temp_file_limit = '50GB'` to prevent disk exhaustion.
14. **REQ-053 (Developer Web SQL Studio & MinIO S3 Object Store)**: The platform MUST provide browser-based developer tooling:
    - Containerized Web SQL Studio (`cloudbeaver:latest` or `pgadmin4`) accessible at `https://<domain>/sql/` for instant schema inspection, visual explain plans, and query drafting.
    - Containerized S3-compatible object storage (`minio:latest`) accessible at `https://<domain>/s3/` for local Strategus study artifact storage, Parquet exports, and pipeline cache.

#### B. Acceptance Criteria Matrix
| Requirement ID | Component | Requirement Statement | Verification Method | Pass Threshold |
| :--- | :--- | :--- | :--- | :--- |
| **AC-040** | StudyAgent Image | Deployment uses official `ohdsi/study-agent:latest`. | `docker inspect study-agent-mcp` | Image matches `ohdsi/study-agent` |
| **AC-041** | MCP Discovery | MCP endpoint exposes available tools list. | `curl -s -H "Authorization: Bearer <token>" https://<domain>/api/v1/tools` | HTTP 200 with tools array |
| **AC-042** | Auth Rejection | Unauthenticated request rejected. | `curl -s -I https://<domain>/api/v1/tools` | HTTP 401 Unauthorized |
| **AC-043** | AST Guardrail | Destructive SQL rejected with 400. | Submit `DROP TABLE person;` to query tool | HTTP 400 with guardrail violation error |
| **AC-044** | Small Cell Filter | Cell count between 1 and 4 is masked. | Query count returning 3 persons | Output explicitly formatted as `"< 5"` |
| **AC-045** | Local Ollama | Ollama instance live with local model loaded. | `curl -s http://localhost:11434/api/tags` | HTTP 200 with installed model tags |
| **AC-046** | WebApiMcp Bridge | WebApiMcp server operational on port 8765 and routes to WebAPI. | `curl -s http://localhost:8765/health` & MCP handshake at `/webapi-mcp/mcp` | HTTP 200 with healthy status; tools list returns cohort and concept set capabilities |
| **AC-047** | Arachne Data Node | Arachne Data Node & Execution Engine operational and connected to CDM. | `curl -s http://localhost:8880/api/v1/build-number` & Execution Engine ping | HTTP 200 with Data Node build info; status healthy |
| **AC-048** | Agentic Client Interop | Standard MCP client (Claude/Cursor/Antigravity) connects, discovers tools, and executes queries. | Connect MCP client via SSE/HTTP and invoke `list_sources` or `search_concepts` | Successful JSON-RPC handshake; valid tool response returned |
| **AC-049** | Fast Snapshot & Rollback | CoW storage snapshot created and restored in `< 60s`; golden baseline restore verified. | Run `ohdsi-snapshot create test-snap && ohdsi-snapshot rollback test-snap` | Sub-minute rollback success; CDM tables intact |
| **AC-050** | Developer Admin Access | Developers authenticate as PostgreSQL superuser, RStudio sudo, and Atlas admin. | `psql -U ohdsi_admin -c "SELECT rolsuper FROM pg_roles WHERE rolname='ohdsi_admin'"` & `docker exec -it broadsea-hades sudo whoami` | Returns `true` for superuser; returns `root` for RStudio sudo |
| **AC-051** | Agentic Sandbox Telemetry | MCP gateways return verbose JSON-RPC error telemetry and honor suppression bypass. | Submit malformed SQL to `/mcp/` and check debug output; toggle suppression flag | JSON-RPC returns detailed compiler traceback; unmasked counts returned |
| **AC-052** | Crash Isolation & cgroups | Host enforces container memory limits and PostgreSQL timeouts under heavy hammering. | Trigger test memory-spike R process inside R Server (`matrix(rnorm(1e8), 1e4, 1e4)`) | Process terminated by cgroup OOM killer; PostgreSQL & Docker daemon remain 100% operational |
| **AC-053** | Web SQL Studio & MinIO | CloudBeaver Web SQL IDE and MinIO S3 console operational over HTTPS. | `curl -s -k -I https://<domain>/sql/` & `curl -s -k -I https://<domain>/s3/` | HTTP 200/302 for both developer services |

---

## 4. Self-Contained Verification Sign-Off Checklist

DevOps engineers must audit and verify each stage gate prior to production sign-off:

### Stage Gate 0: Infrastructure & Host Hardening
- [ ] **AC-001**: Ubuntu Server 24.04 LTS verified on host.
- [ ] **AC-002**: NVMe data partition mounted with `noatime,nodiratime` at `/var/lib/postgresql/data`.
- [ ] **AC-003**: Kernel sysctl parameters verified in `/etc/sysctl.d/99-ohdsi.conf`.
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
- [ ] **AC-044**: Small Cell Suppression verified: cell counts `< 5` masked across all queries (configurable for synthetic benchmarks).
- [ ] **AC-045**: Local sovereign Ollama instance operational on port 11434.
- [ ] **AC-046**: WebApiMcp bridge server operational on port 8765 connected to WebAPI.
- [ ] **AC-047**: OHDSI Arachne Data Node & Execution Engine operational on ports 8880/8888 and connected to OMOP CDM.
- [ ] **AC-048**: Agentic software interoperability verified with MCP client (tool discovery and execution pass 100%).
- [ ] **AC-049**: Fast snapshot and sub-minute rollback verified via ZFS/Btrfs CoW and golden baseline restore.
- [ ] **AC-050**: Developer superuser credentials, RStudio sudo, and Atlas admin roles verified.
- [ ] **AC-051**: Agentic testing sandbox verified with verbose compiler tracebacks in JSON-RPC errors.
- [ ] **AC-052**: System resilience verified: cgroup memory clamps and PostgreSQL timeouts isolate crashes.
- [ ] **AC-053**: Developer Web SQL Studio (`/sql/`) and MinIO S3 object store (`/s3/`) verified.
