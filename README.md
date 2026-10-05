# OHDSI Sandbox 2026: DevOps Platform Requirements & Acceptance Criteria Specification

> **Platform Mission**: High-Resilience Developer Sandbox for Innovating, Collaborating & Testing Latest Ideas in Healthcare Informatics and Data Science  
> **Repository Type**: Production System Requirements Specification (SRS) & Stage-Gated Acceptance Criteria (AC)  
> **Target Audience**: Cloud DevOps Engineers, Site Reliability Engineers (SRE), Infrastructure Architects, Data Science & Informatics Leads  
> **Platform Target**: Dedicated Bare-Metal Host (Hetzner AX/PX) or Cloud VPC (AWS / GCP / Azure)  
> **Status**: Approved Production Specification

---

## 1. Repository Purpose & Scope: The OHDSI Sandbox

This repository is **strictly a requirements and acceptance criteria specification repository** for platform engineering and DevOps teams.

It specifies how DevOps must build, harden, and maintain the **OHDSI Sandbox**—a shared developer playground where data scientists, clinical informaticians, and AI researchers can innovate, collaborate, and test the latest ideas in observational research:

- **An Open Developer Playground**: Designed to withstand the aggressive "hammering" of data science and informatics developer teams—massive recursive SQL queries, multi-core HADES causal inference pipelines, rapid iterative Shiny app development, and high-frequency multi-agent LLM tool loops.
- **Zero Real Person-Level Data**: The environment connects exclusively to synthetic, benchmark, and simulated datasets (Eunomia, Synthea 100k, CMS SynPUF 2.3M, OMOP CDM v5.4 sample benchmarks) and Athena Vocabularies. Developers have freedom to inspect raw counts without privacy restrictions.
- **Break It and Restore It (Sub-Minute Rollbacks)**: Failure is embraced as normal experimentation. The platform implements Copy-on-Write (CoW) filesystem snapshots (ZFS or Btrfs) enabling instant `< 60-second` rollbacks (`ohdsi-snapshot rollback`) and a pre-seeded golden baseline dump (`cdm_golden_baseline.dump.gz`) restored in `< 5 minutes`.
- **Developer Superuser & Admin Access**: Developers receive PostgreSQL superuser credentials (`ohdsi_admin`), passwordless `sudo` in the Dedicated R Server (`broadsea-hades`), and global administrator privileges in Atlas and WebAPI.
- **Agentic AI Testbed**: Pre-configured dual MCP gateways ([`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) and [`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp)) with verbose debug telemetry, enabling external AI agents (Claude, Cursor, Antigravity) to self-correct during automated experiment loops.
- **Pure Specifications**: No custom implementation code, Terraform IaC, or pytest suites belong in this repository; all deliverables are specified via deterministic acceptance criteria, CLI checks, and verification checklists.

---

## 2. System Overview & The DevOps Mental Model

To a systems or DevOps engineer, the OHDSI Sandbox is a standard **3-tier analytical data warehouse, compute, and agentic AI ecosystem**:

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
     │      - Single-Domain Path Routing (/atlas, /WebAPI, /mcp, /sql, /s3)    │
     │      - Public URLs for Study Shiny Apps (/shiny/) & Reports (/reports/) │
     └───────┬────────────────────┬────────────────────┬───────────────────────┘
             │                    │                    │
             ▼                    ▼                    ▼
   ┌───────────────────┐┌───────────────────┐┌─────────────────────────────────┐
   │ Frontend Web Apps ││  Data & R Engine  ││  Agentic AI & Federated Network │
   │ - Atlas 3.0 (Vue3)││ - WebAPI Classic  ││ - FastMCP Server (:8790)        │
   │ - Atlas Classic   ││ - WebAPI 3.0      ││ - WebApiMcp Bridge (:8765)      │
   │ - Study Shiny Apps││ - Dedicated R Svr ││ - Arachne Data Node (:8880)     │
   │ - RStudio (:8787) ││   (broadsea-hades)││ - Local Ollama LLM (:11434)     │
   │ - Web SQL (:8978) ││ - MinIO S3 (:9001)││ - Redis Task Broker (:6379)     │
   └─────────┬─────────┘└─────────┬─────────┘└────────────────┬────────────────┘
             │                    │                           │
             └────────────────────┼───────────────────────────┘
                                  ▼
     ┌─────────────────────────────────────────────────────────────────────────┐
     │            PostgreSQL 16 High-Throughput RDBMS Cluster                  │
     │   - 64GB shared_buffers, NVMe Gen4 Storage (ZFS/Btrfs CoW Snapshots)    │
     │   - Dedicated 64GB NVMe Swap Partition (Anti-Panic OOM Absorption)      │
     │   - Master Lookup Schema (vocab_54: ~10M records with GIN indexes)      │
     │   - Structured Synthetic Data Schemas (cdm_synthea100k, cdm_synpuf_23m) │
     │   - Precomputed Aggregate Cache Schemas (results)                       │
     │   - Shared Connection: Connected to WebAPI, Atlas, and Dedicated R Svr  │
     │   - Port 5432 (Quarantined to Private Docker Network / VPN / Tailscale) │
     └─────────────────────────────────────────────────────────────────────────┘
```

### The DevOps Rosetta Stone (Terminology Translation)
Translating clinical and sandbox terms into standard software engineering concepts:

| Domain Term | Software Engineering Equivalent | Technical Function |
| :--- | :--- | :--- |
| **OMOP CDM** | Relational Data Warehouse Schema | Standardized PostgreSQL relational schema storing synthetic event records (`person`, `visit_occurrence`, `condition_occurrence`). |
| **Athena / Vocabularies** | Master Lookup Dictionary (~10M rows) | Lookup table mapping disparate coding systems to uniform integer primary keys (`concept_id`). Uses trigram GIN indexes. |
| **Source Daimon** | Schema Routing Registry Table | Configuration table in PostgreSQL mapping logical database aliases to physical schemas (`cdm`, `vocab`, `results`). |
| **WebAPI** | Java Spring Boot REST Backend | Core API service providing data source management, SQL transpilation, security integration, and query execution. |
| **Atlas (Classic & 3.0)** | Web Application Frontend | Single-Page Application (Classic: Knockout.js; 3.0: Vue 3 / single-spa) providing query authoring and visualization. |
| **Dedicated R Server** | Dedicated R Compute Server | Containerized RStudio Server (`broadsea-hades`, port 8787) hosting all OHDSI R libraries natively with passwordless `sudo`. |
| **Study Shiny Apps** | Interactive Analytical Dashboards | Containerized Shiny Server (`ohdsi-shiny`, port 3838) publishing interactive study apps (`CohortDiagnostics`, `Taxis`) on public URL. |
| **Study Reports** | Static Analytical HTML Reports | Quarto / RMarkdown compiled study reports served on public URL under `/reports/`. |
| **Web SQL Studio** | Web Database IDE (CloudBeaver) | Browser-based visual SQL editor, table autocomplete, and ERD browser at `/sql/`. |
| **MinIO S3 Mock** | Local Object Store (S3-Compatible) | Local bucket storage at `/s3/` for Strategus study artifacts, Parquet exports, and caches. |
| **CoW Snapshots** | Sub-Minute Filesystem Rollback | ZFS or Btrfs snapshots allowing instant rollback when experiments corrupt data. |
| **StudyAgent / MCP** | FastMCP Tool-Calling Gateway | Official [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) server exposing platform capabilities to AI models via Model Context Protocol. |
| **WebApiMcp** | WebAPI MCP Bridge Server | Dedicated MCP bridge ([`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp)) exposing cohort definitions and concept sets directly to LLMs. |
| **Arachne** | Federated Research Network Node | Distributed study execution node ([`OHDSI/ArachneDataNode`](https://github.com/OHDSI/ArachneDataNode)) enabling multi-site studies with non-PHI aggregate export. |
| **Small Cell Suppression** | Privacy Masking Middleware | Re-identification filter automatically masking counts `< 5` (configurable/bypassed in synthetic sandbox). |

---

## 3. Dedicated R Server, Atlas & Sandbox Interconnectivity

The platform integrates compute, storage, applications, and public hosting as a cohesive ecosystem:

1. **R Server Database Connectivity (OMOP CDM & Vocabularies)**:
   - The **Dedicated R Server** (`broadsea-hades`) connects directly to the PostgreSQL database cluster via JDBC (`DatabaseConnector`).
   - It queries both the clinical event tables (`person`, `condition_occurrence`, `visit_occurrence` in `cdm`) and the master Athena Vocabulary lookup tables (`concept`, `concept_ancestor`, `concept_relationship` in `vocab_54`).
2. **PostgreSQL Connected to Atlas Instance**:
   - The PostgreSQL database is connected to the WebAPI backend, which is connected to the Atlas web application instance.
   - Researchers can search vocabularies, inspect data source characterizations, and author cohort definitions from the Atlas UI against the shared PostgreSQL database.
3. **Deploying OHDSI Study Shiny Apps & Analytical Reports**:
   - The platform provides a containerized Shiny runtime (`ohdsi-shiny` on port 3838) with volumes mounted at `/srv/shiny-server/` and `/srv/reports/`.
   - Researchers can deploy interactive Shiny dashboards (`CohortDiagnostics`, `CohortIncidence`, `Characterization`, `OhdsiShinyModules`, [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis)) and published HTML study reports.
4. **Unified Public URL Exposure**:
   - All tools and dashboards are served over HTTPS under **ONE public URL domain** via Nginx TLS 1.3:
     - `https://<domain>/` -> Atlas 3.0 Next-Gen Frontend
     - `https://<domain>/atlas/` -> Atlas Classic Frontend
     - `https://<domain>/WebAPI/` -> WebAPI REST Backend
     - `https://<domain>/rstudio/` -> Dedicated R Server (RStudio Server Web IDE)
     - `https://<domain>/shiny/` -> Interactive OHDSI Study Shiny Apps
     - `https://<domain>/reports/` -> Static OHDSI Study Analytical HTML Reports
     - `https://<domain>/mcp/` -> Model Context Protocol (FastMCP) AI Agent Gateway
     - `https://<domain>/webapi-mcp/` -> WebApiMcp Bridge Server
     - `https://<domain>/arachne/` -> OHDSI Arachne Data Node
     - `https://<domain>/sql/` -> CloudBeaver Web SQL Studio
     - `https://<domain>/s3/` -> MinIO S3 Object Store Console

*For complete architectural specifications, see [ARCHITECTURAL_REVIEW_DEVELOPER_PLAYGROUND.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/ARCHITECTURAL_REVIEW_DEVELOPER_PLAYGROUND.md), [DEDICATED_R_SERVER_ARCHITECTURE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEDICATED_R_SERVER_ARCHITECTURE.md), [PUBLIC_DOMAIN_HOSTING_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PUBLIC_DOMAIN_HOSTING_GUIDE.md), and [AGENTIC_MCP_INTEGRATION_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/AGENTIC_MCP_INTEGRATION_GUIDE.md).*

---

## 4. The 5-Stage Gated Milestone Progression

All deployment requirements and acceptance criteria are organized into 5 sequential stage gates:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           STAGE-GATED ROADMAP                                          │
├─────────┬──────────────────────────────┬───────────────────────────────────────────┬───────────────────┤
│ Stage   │ Milestone Focus              │ Core Technologies                         │ Exit Verification │
├─────────┼──────────────────────────────┼───────────────────────────────────────────┼───────────────────┤
│ Gate 0  │ Host & Kernel Hardening      │ Ubuntu 24.04 LTS, NVMe CoW, UFW, Swap     │ Ports closed      │
│ Gate 1  │ Turnkey Broadsea Core        │ broadsea-atlasdb, webapi, atlas, hades    │ WebAPI /info UP   │
│ Gate 2  │ High-Capacity Data & Vocab   │ Postgres 16 (64GB shared_buffers), Redis  │ Vocab query <150ms│
│ Gate 3  │ Atlas 3.0 & WebAPI 3.0       │ Atlas 3.0 (Vue 3), WebAPI 3.0, R Server   │ Atlas 3.0 live    │
│ Gate 4  │ Agentic AI & Sandbox Tools   │ FastMCP, WebApiMcp, Arachne, CoW Snaps    │ Sandbox suite 100%│
└─────────┴──────────────────────────────┴───────────────────────────────────────────┴───────────────────┘
```

*For complete requirement IDs (REQ-001 through REQ-053) and acceptance criteria matrices (AC-001 through AC-053), see [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md).*

---

## 5. Specification Document Index

DevOps and infrastructure teams should reference the following dedicated specification documents:

| Specification Document | Focus Area & Content |
| :--- | :--- |
| **[DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md)** | **Sandbox Handover & Quickstart Runbook**: Master handover card (URLs, default credentials), developer workspace provisioning, sub-minute break-and-restore CoW snapshot procedures, hammering guardrails, and 10-point handover test. |
| **[ENGINEERING_EXECUTION_PLAN.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/ENGINEERING_EXECUTION_PLAN.md)** | **DevOps Deployment Execution Runbook**: Step-by-step rollout sequence across 4 deployment phases with single-command verifications, snapshot rollback validation, and handover sign-off. |
| **[STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md)** | **Authoritative System Requirements Specification (SRS)**: Complete RFC 2119 requirements (REQ-001 to REQ-053), acceptance criteria matrices (AC-001 to AC-053), and master sign-off checklist. |
| **[ARCHITECTURAL_REVIEW_DEVELOPER_PLAYGROUND.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/ARCHITECTURAL_REVIEW_DEVELOPER_PLAYGROUND.md)** | **Architectural Review & Systems Engineering Assessment**: Analysis of developer hammering, CoW snapshot/rollback architecture, superuser privileges, and DevOps delivery contract. |
| **[OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md)** | **Hardware & Infrastructure Specification**: Compute, RAM, NVMe ZFS/Btrfs CoW mount, 64GB NVMe swap, cgroup memory clamping, network port rules, and 20-container roster. |
| **[PUBLIC_DOMAIN_HOSTING_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PUBLIC_DOMAIN_HOSTING_GUIDE.md)** | **Ingress & Perimeter Guide**: Single public domain reverse proxy specification, subpath routing for all 10 services, TLS 1.3, automated Let's Encrypt renewals, Web SQL Studio, and MinIO S3 routing. |
| **[AGENTIC_MCP_INTEGRATION_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/AGENTIC_MCP_INTEGRATION_GUIDE.md)** | **Agentic Software & MCP Guide**: Configuration snippets for Claude Desktop, Cursor, Antigravity IDE, tool catalog, JSON-RPC schemas, and verification testing. |
| **[DEDICATED_R_SERVER_ARCHITECTURE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEDICATED_R_SERVER_ARCHITECTURE.md)** | **Architectural Decision Record (ADR)**: Justification for the Dedicated R Server model over fragmented microservices, including CDM and WebAPI connection code patterns. |

---

## 6. Official Upstream Reference Repositories

This specification directly references and relies upon official upstream OHDSI releases and container images:

- **Broadsea Core**: [`OHDSI/Broadsea`](https://github.com/OHDSI/Broadsea) (Broadsea 3.5 deployment profile standards).
- **StudyAgent FastMCP Gateway**: [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) (`ohdsi/study-agent:latest`).
- **WebApiMcp Bridge**: [`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp) (Dedicated WebAPI MCP bridge server).
- **OHDSI Arachne Data Node & Engine**: [`OHDSI/ArachneDataNode`](https://github.com/OHDSI/ArachneDataNode), [`OHDSI/ArachneExecutionEngine`](https://github.com/OHDSI/ArachneExecutionEngine), and [`OHDSI/ArachneCentral`](https://github.com/OHDSI/ArachneCentral).
- **HADES Analytical Packages**: [`OHDSI/Hades`](https://github.com/OHDSI/Hades) (Pre-installed in `ohdsi/broadsea-hades:1.19.0`).
- **WebAPI**: [`OHDSI/WebAPI`](https://github.com/OHDSI/WebAPI) (`ohdsi/webapi:2.14.0`).
- **Atlas**: [`OHDSI/Atlas`](https://github.com/OHDSI/Atlas) (`ohdsi/atlas:2.14.0`).
- **Network Studies**: [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis) (Association mining engine).
