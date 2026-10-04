# OHDSI 2026: DevOps Platform Requirements & Acceptance Criteria Specification

> **Repository Type**: Production System Requirements Specification (SRS) & Stage-Gated Acceptance Criteria (AC)  
> **Target Audience**: Cloud DevOps Engineers, Site Reliability Engineers (SRE), Infrastructure Architects, Database Administrators  
> **Platform Target**: Dedicated Bare-Metal Host (Hetzner AX/PX) or Cloud VPC (AWS / GCP / Azure)  
> **Status**: Approved Production Standard

---

## 1. Repository Purpose & Scope

This repository is **strictly a requirements and acceptance criteria specification repository** for platform engineering and DevOps teams.

It is **NOT an implementation code repository**:
- **No Terraform or Cloud-Specific IaC**: Cloud infrastructure provisioning is implemented by downstream DevOps teams in their respective enterprise Terraform, OpenTofu, or Pulumi repositories based on the specifications defined here.
- **No Test Suites or Pytest Code**: Testing and automated CI/CD runners belong to downstream implementation pipelines; acceptance criteria in this repository are specified as deterministic verification commands (HTTP status codes, CLI checks, SQL explain plans, and sign-off checklists).
- **No Fragmented Microservices**: Bespoke custom Plumber R microservices and duplicate wrapper code have been completely eliminated in favor of official OHDSI upstream container standards.
- **Dedicated R Server Architecture**: Analytical R packages run inside a single containerized R Server (`broadsea-hades` / RStudio Server), which is directly connected to the shared OMOP CDM database and WebAPI backend.

---

## 2. System Overview & The DevOps Mental Model

To a systems or DevOps engineer, the OHDSI platform is a standard **3-tier analytical data warehouse and compute ecosystem**:

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

### The DevOps Rosetta Stone (Terminology Translation)
Translating domain-specific clinical terms into standard software engineering concepts:

| Domain Term | Software Engineering Equivalent | Technical Function |
| :--- | :--- | :--- |
| **OMOP CDM** | Relational Data Warehouse Schema | Standardized PostgreSQL relational schema storing subject event tables (`person`, `visit_occurrence`, `condition_occurrence`). |
| **Athena / Vocabularies** | Master Lookup Dictionary (~10M rows) | Lookup table mapping disparate coding systems to uniform integer primary keys (`concept_id`). Uses trigram GIN indexes. |
| **Source Daimon** | Schema Routing Registry Table | Configuration table in PostgreSQL mapping logical database aliases to physical schemas (`cdm`, `vocab`, `results`). |
| **WebAPI** | Java Spring Boot REST Backend | Core API service providing data source management, SQL transpilation, security integration, and query execution. |
| **Atlas (Classic & 3.0)** | Web Application Frontend | Single-Page Application (Classic: Knockout.js; 3.0: Vue 3 / single-spa) providing query authoring and visualization. |
| **Dedicated R Server** | Dedicated R Compute Server | Containerized RStudio Server (`broadsea-hades`, port 8787) hosting all OHDSI R libraries natively. Connected to CDM and WebAPI. |
| **Achilles** | Precomputed Aggregate Cache | Batch processing job that precomputes table counts and distributions, storing them in the `results` schema for instant dashboard retrieval. |
| **Circe / Capr** | Query Transpiler & AST Builder | Compiles JSON or R criteria into target database SQL dialects (PostgreSQL, Snowflake, BigQuery). |
| **Cohort / Phenotype** | Entity Segment / Filter Query | Specific criteria defining a population slice (e.g. subjects meeting specific criteria within a date window). |
| **StudyAgent / MCP** | FastMCP Tool-Calling Gateway | Official [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) server exposing platform capabilities to AI models via Model Context Protocol. |
| **Small Cell Suppression** | Data Privacy Masking Middleware | Re-identification protection filter automatically masking query count results where `0 < count < 5` as `"< 5"`. |

---

## 3. Dedicated R Server Interconnectivity (CDM & WebAPI)

The **Dedicated R Server** (`broadsea-hades`) operates as a first-class compute node within the internal network:
1. **Direct CDM Database Connectivity**:
   - Pre-configured with JDBC drivers and database credentials.
   - Connects directly to the PostgreSQL OMOP CDM database cluster (`ohdsi-postgres:5432/ohdsi`) via `DatabaseConnector`.
   - Accesses the exact same CDM tables (`person`, `condition_occurrence`, etc.) and vocabularies shared by Atlas and WebAPI.
2. **WebAPI Integration**:
   - Pre-configured with `WEBAPI_URL` pointing to the internal WebAPI instance.
   - Leverages `ROhdsiWebApi` to fetch cohort definitions, export concept sets, and execute cohort generation directly against WebAPI.

*For complete architectural specifications, see [DEDICATED_R_SERVER_ARCHITECTURE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEDICATED_R_SERVER_ARCHITECTURE.md).*

---

## 4. The 5-Stage Gated Milestone Progression

All deployment requirements and acceptance criteria are organized into 5 sequential stage gates:

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
│ Gate 4  │ Public Ingress & FastMCP AI  │ Nginx TLS 1.3, Let's Encrypt, StudyAgent  │ MCP suite 100%    │
└─────────┴──────────────────────────────┴───────────────────────────────────────────┴───────────────────┘
```

*For complete requirement IDs (REQ-001 through REQ-045) and acceptance criteria matrices (AC-001 through AC-045), see [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md).*

---

## 5. Specification Document Index

DevOps and infrastructure teams should reference the following dedicated specification documents:

| Specification Document | Focus Area & Content |
| :--- | :--- |
| **[STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md)** | **Authoritative System Requirements Specification (SRS)**: Complete RFC 2119 requirements (REQ-001 to REQ-045), acceptance criteria matrices, and milestone sign-off checklists. |
| **[OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md)** | **Hardware & Infrastructure Specification**: Minimum and production compute, RAM, NVMe mount options (`noatime,nodiratime`), kernel sysctl parameters, network port rules, and container roster. |
| **[ENGINEERING_EXECUTION_PLAN.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/ENGINEERING_EXECUTION_PLAN.md)** | **DevOps Execution Runbook**: Step-by-step rollout sequence across the 5 stage gates with deterministic CLI / curl verification commands. |
| **[DEDICATED_R_SERVER_ARCHITECTURE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEDICATED_R_SERVER_ARCHITECTURE.md)** | **Architectural Decision Record (ADR)**: Justification for the Dedicated R Server model over fragmented microservices, including CDM and WebAPI connection code patterns. |
| **[PUBLIC_DOMAIN_HOSTING_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PUBLIC_DOMAIN_HOSTING_GUIDE.md)** | **Ingress & Perimeter Guide**: Single public domain reverse proxy specification, subpath routing, TLS 1.3, automated Let's Encrypt renewals, and small-cell suppression. |
| **[DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md)** | **DevOps Onboarding**: System mental models, jargon translation rosetta stone, and operational best practices. |

---

## 6. Official Upstream Reference Repositories

This specification directly references and relies upon official upstream OHDSI releases and container images:

- **Broadsea Core**: [`OHDSI/Broadsea`](https://github.com/OHDSI/Broadsea) (Broadsea 3.5 deployment profile standards).
- **StudyAgent FastMCP Gateway**: [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) (`ohdsi/study-agent:latest`).
- **HADES Analytical Packages**: [`OHDSI/Hades`](https://github.com/OHDSI/Hades) (Pre-installed in `ohdsi/broadsea-hades:1.19.0`).
- **WebAPI**: [`OHDSI/WebAPI`](https://github.com/OHDSI/WebAPI) (`ohdsi/webapi:2.14.0`).
- **Atlas**: [`OHDSI/Atlas`](https://github.com/OHDSI/Atlas) (`ohdsi/atlas:2.14.0`).
- **Network Studies**: [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis) (Association mining engine).
