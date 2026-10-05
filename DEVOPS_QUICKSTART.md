# DevOps Engineer Onboarding & Systems Quickstart Guide

> **Repository Type**: Production Requirements & Acceptance Criteria Specification  
> **Target Audience**: DevOps Engineers, SREs, Systems Architects, Infrastructure Leads  
> **Status**: Approved Production Specification

---

## 1. What is this System? (In Plain Software Terms)

From an infrastructure and systems perspective, this repository specifies requirements for an **enterprise 3-tier analytical data platform**:

1. **Database Tier (PostgreSQL 16)**:
   - Houses a relational data warehouse with event tables (`person`, `visit_occurrence`, etc.).
   - Contains a master dictionary table (`vocab_54`, ~10M rows) mapping foreign codes to primary keys.
   - Contains a precomputed aggregate results cache (`results`) used to power instant UI graphs.
2. **Backend & Compute Tier**:
   - **WebAPI (Java 21 / Spring Boot 3)**: Main REST backend that compiles abstract JSON queries into vendor-specific SQL dialects. Connects to PostgreSQL via JDBC.
   - **Dedicated R Server (`broadsea-hades`)**: Containerized RStudio Server & compute engine (port 8787) hosting all OHDSI R libraries natively. Connected directly to both the OMOP CDM database via JDBC and WebAPI via REST.
   - **FastMCP Agent Gateway**: Official [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) container exposing Model Context Protocol endpoints (SSE, HTTP, Stdio) for AI models (Claude, Cursor, Antigravity) to query data safely.
3. **Frontend & Ingress Tier**:
   - **Atlas 3.0 (Vue 3 / TypeScript)** & **Atlas Classic**: Query builder web applications.
   - **Nginx Reverse Proxy**: Enforces TLS 1.3, rate limits, single-domain path routing, and data privacy filters (`MIN_CELL_COUNT=5`).

---

## 2. The DevOps Rosetta Stone (Jargon Translation)

| Domain Term | Standard Software Term | Technical Function |
| :--- | :--- | :--- |
| **OMOP CDM** | Relational Data Warehouse Schema | PostgreSQL database schema holding structured event records. |
| **Athena / Vocabularies** | Master Lookup Dictionary (~10M rows) | Lookup table used for autocomplete and ID translation. Indexed with GIN (`pg_trgm`). |
| **Source Daimon** | Database Schema Routing Config | A row in a config table telling WebAPI which schema is which (`cdm`, `vocab`, `results`). |
| **WebAPI** | Java Spring Boot REST Backend | Exposes REST endpoints on port 8080. Connects to PostgreSQL via JDBC. |
| **Atlas** | Analytics Web Application UI | Single-page app (Classic on port 8082; Atlas 3.0 on port 3000). |
| **Dedicated R Server** | Dedicated R Compute Engine | Containerized RStudio Server (`broadsea-hades`, port 8787) hosting all OHDSI R packages. |
| **Achilles** | Precomputed Aggregate Cache | Batch processing script that populates the `results` schema for instant dashboard retrieval. |
| **Circe / Capr** | SQL Query Transpiler | Compiles JSON / R filter logic into database SQL. |
| **Cohort / Phenotype** | Population Filter / Slice | A query returning matching entity IDs meeting certain conditions within a time range. |
| **StudyAgent / MCP** | FastMCP Tool Server | Official container giving external AI agents access to database tools via MCP. |
| **Small Cell Suppression** | Privacy Masking Filter | Middleware masking any count between 1 and 4 as `"< 5"` to prevent re-identification. |

---

## 3. Dedicated R Server Interconnectivity (OMOP CDM, Vocabularies & WebAPI)

The Dedicated R Server (`broadsea-hades`) connects directly to the OMOP CDM event tables, Athena Vocabularies, and the WebAPI backend:

```r
library(DatabaseConnector)
library(ROhdsiWebApi)

# 1. Connect to OMOP CDM and Athena Vocabularies
connectionDetails <- createConnectionDetails(
  dbms = "postgresql",
  server = paste0(Sys.getenv("CDM_SERVER", "ohdsi-postgres"), "/", Sys.getenv("CDM_DATABASE", "ohdsi")),
  user = Sys.getenv("CDM_USER", "ohdsi_app_user"),
  password = Sys.getenv("CDM_PASSWORD")
)

conn <- connect(connectionDetails)
# Query CDM clinical event tables
querySql(conn, "SELECT COUNT(*) FROM cdm_synthea100k.person;")
# Query standardized Athena Vocabularies
querySql(conn, "SELECT COUNT(*) FROM vocab_54.concept;")
disconnect(conn)

# 2. Connect to WebAPI (shared with Atlas instance)
webApiUrl <- Sys.getenv("WEBAPI_URL", "http://webapi-classic:8080/WebAPI")
ROhdsiWebApi::getWebApiVersion(baseUrl = webApiUrl)
```
*For deep-dive architectural trade-offs, see [DEDICATED_R_SERVER_ARCHITECTURE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEDICATED_R_SERVER_ARCHITECTURE.md).*

---

## 4. The 5-Stage Gated Rollout

DevOps engineers must verify each stage gate sequentially:

- **Stage Gate 0**: Host & Kernel Hardening (Ubuntu 24.04 LTS, NVMe `noatime,nodiratime`, UFW firewall).
- **Stage Gate 1**: Turnkey Broadsea Core (PostgreSQL, WebAPI Classic, Atlas Classic, Solr).
- **Stage Gate 2**: High-Capacity Data, Dedicated R Server, Atlas & Shiny Apps (64GB shared buffers, full Athena vocab, Dedicated R Server connected to OMOP CDM and Vocabularies, Atlas connected to PostgreSQL via WebAPI, Shiny Server mounting study apps).
- **Stage Gate 3**: Atlas 3.0 Next-Gen Frontend, WebAPI 3.0 & Public URL Ingress (single-domain routing under TLS 1.3 for `/`, `/atlas/`, `/WebAPI/`, `/rstudio/`, `/shiny/`, `/reports/`, `/mcp/`).
- **Stage Gate 4**: Sovereign Agentic Tier (hosted FastMCP via `OHDSI/StudyAgent`, local Ollama LLM, AST guardrail).

*Detailed requirements and acceptance criteria matrices: [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md).*

---

## 5. Storage & Database Essentials

### PostgreSQL 16 Memory Allocation
Tuned for high-volume table scans:
- **`shared_buffers`**: Allocate 25% of total system RAM (minimum 32GB, recommended 64GB).
- **`work_mem`**: 256MB per operation.
- **`maintenance_work_mem`**: 4GB for index builds and VACUUM.
- **`effective_cache_size`**: 75% of total RAM.

### NVMe Mount Point
Mount your PostgreSQL data directory with `noatime,nodiratime` to reduce write wear during heavy analytical scans:
```bash
mount -o noatime,nodiratime /dev/nvme0n1p3 /var/lib/postgresql/data
```

---

## 6. The 3 Golden Rules for Operations

1. **Quarantine Database & RStudio Ports**:
   - Port `5432` (PostgreSQL) and Port `8787` (RStudio) must **never** be exposed directly to the public internet. Bind them to `127.0.0.1`, a private VPC subnet, or a VPN.
2. **Enforce Small Cell Suppression (`MIN_CELL_COUNT >= 5`)**:
   - Always ensure query filters and Nginx proxies mask counts between 1 and 4 as `"< 5"` to comply with health data privacy regulations.
3. **Use the AST SQL Parser for AI Agents**:
   - When external agents query the platform over MCP, route through the official StudyAgent container which parses SQL to prevent unaggregated extraction of raw identity tables.
