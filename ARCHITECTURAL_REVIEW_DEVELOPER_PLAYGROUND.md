# Architectural Review: OHDSI Developer & Research Playground Server

> **Document Type**: Comprehensive Architectural Review & Systems Engineering Assessment  
> **System Classification**: Developer Tool Playground / Informatics Sandbox / Agentic AI Testbed  
> **Target Audience**: Lead Infrastructure Architects, DevOps/SRE Engineers, Data Science & Informatics Leads  
> **Author**: Antigravity Autonomous Systems Engineering Team  
> **Date**: October 2026  
> **Status**: Approved Architectural Standard

---

## 1. Executive Summary & Paradigm Shift

### The Shift from Production Warehouse to Resilient Playground
Previous specifications assumed a production-oriented OHDSI deployment governed by clinical compliance: strict small-cell suppression (`count < 5`), tight perimeter lockdowns, and immutable containers.

However, the primary mission of this server is fundamentally different:
**It is an unconstrained Developer Tool Playground and Informatics Sandbox.**

In this environment:
- **Zero Real Person-Level Data**: The system connects exclusively to synthetic, simulated, and benchmark datasets (Eunomia, Synthea 100k, CMS SynPUF 2.3M, OMOP CDM v5.4 sample benchmarks) along with Athena Standardized Vocabularies.
- **The Users**: Data scientists, clinical informaticians, phenotyping engineers, and agentic AI researchers.
- **The Workload**: Highly unpredictable, experimental, and aggressive ("hammering"): runaway recursive SQL queries, multi-core HADES R causal inference pipelines, rapid iterative Shiny app development, and high-frequency multi-agent LLM tool loops.
- **The Core Axiom**: **Developers WILL break things, and they must be encouraged to do so.** The architecture must embrace failure as normal operation: enabling instant snapshotting, fast `< 60-second` rollbacks, deep admin/superuser privileges, and strict container crash isolation so a broken experiment never forces a multi-day server rebuild.

---

## 2. "Hammering" Analysis: Failure Modes in Data Science & Informatics

To build a resilient platform that DevOps can hand over to developers with confidence, we analyzed the primary failure modes when data scientists and AI agents hammer an OHDSI stack:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        FAILURE MODES & RESILIENCE ARCHITECTURE                         │
├──────────────────────┬───────────────────────────────┬─────────────────────────────────┤
│ Failure Mode         │ Mechanism of Failure          │ Architectural Countermeasure    │
├──────────────────────┼───────────────────────────────┼─────────────────────────────────┤
│ **Memory Runaways**  │ HADES `CohortMethod` /        │ Container cgroup hard limits;   │
│                      │ `PatientLevelPrediction` R    │ 64GB high-speed NVMe swap;      │
│                      │ processes consuming 80GB+ RAM │ PostgreSQL memory protection.   │
├──────────────────────┼───────────────────────────────┼─────────────────────────────────┤
│ **Database Locks &** │ Unindexed Cartesian joins or  │ `statement_timeout = 15min`;    │
│ **Zombie Sessions**  │ abandoned RStudio sessions    │ `idle_in_transaction = 10min`;  │
│                      │ holding exclusive table locks │ `temp_file_limit = 50GB`.       │
├──────────────────────┼───────────────────────────────┼─────────────────────────────────┤
│ **Schema Corruption**│ Dropped tables, bad migrations│ ZFS/Btrfs copy-on-write instant │
│                      │ polluted test schemas         │ snapshots; golden baseline dump.│
├──────────────────────┼───────────────────────────────┼─────────────────────────────────┤
│ **Agent Tool Storms**│ Recursive LLM loops flooding  │ Redis request queuing;          │
│                      │ WebAPI & MCP endpoints with   │ connection pool reservation;    │
│                      │ hundreds of queries per sec   │ detailed JSON-RPC error traces. │
├──────────────────────┼───────────────────────────────┼─────────────────────────────────┤
│ **R Package Clashes**│ `install_github` overwriting  │ Persistent user library volumes;│
│                      │ core shared C-libraries       │ Docker volume isolation; `renv`.│
└──────────────────────┴───────────────────────────────┴─────────────────────────────────┘
```

---

## 3. Core Architectural Pillars for the Playground

### Pillar 1: Instant Snapshots & Fast Rollback (Break & Restore)
Developers must be able to experiment fearlessly. If an experimental cohort generation corrupts the `results` schema or a script drops the CDM, recovery must take minutes, not hours:

1. **Storage-Level Copy-on-Write (CoW) Snapshots (ZFS / Btrfs)**:
   - The PostgreSQL NVMe data partition (`/var/lib/postgresql/data`) and Dedicated R Server workspace (`/home/ohdsi`) MUST be formatted on a CoW filesystem (ZFS or Btrfs).
   - DevOps must provide a simple CLI wrapper:
     ```bash
     # Take an instant zero-cost snapshot before an experiment:
     sudo ohdsi-snapshot create "pre-study-experiment-01"

     # Rollback instantly (< 30 seconds) if broken:
     sudo ohdsi-snapshot rollback "pre-study-experiment-01"
     ```
2. **Pre-Seeded "Golden State" Logical Dumps**:
   - The server maintains a compressed, verified baseline dump (`/opt/ohdsi/seeds/cdm_golden_baseline.dump.gz`) containing:
     - Full Athena Vocabularies (`vocab_54` ~10M concepts with GIN indexes pre-built).
     - Synthea 100k + SynPUF 2.3M CDM schemas.
     - WebAPI schema with pre-configured source daimon rows.
   - A single reset script reloads the database cleanly in `< 5 minutes`:
     ```bash
     docker compose exec db-manager /scripts/restore-golden-baseline.sh
     ```
3. **Decoupled Ephemeral Runtime**:
   - All state is strictly externalized into named Docker volumes. Any container can be forcefully killed and recreated via `docker compose up -d --force-recreate` without losing underlying data.

---

### Pillar 2: Developer & Admin Access Model
Unlike production environments that isolate users, a developer playground requires expansive privileges:

1. **PostgreSQL Superuser & Schema Freedom**:
   - Developers receive superuser credentials (`postgres` or `ohdsi_admin` with `SUPERUSER`, `CREATEDB`, `CREATEROLE`).
   - Every developer is granted an isolated personal sandbox schema (`scratch_<username>`) with full DDL/DML permissions, plus read access to shared `cdm` and `vocab_54` schemas.
2. **RStudio Server Sudo & Package Freedom**:
   - The `broadsea-hades` container user (`ohdsi`) is granted passwordless `sudo` in container:
     ```bash
     ohdsi ALL=(ALL) NOPASSWD: ALL
     ```
   - Developers can install system packages (`apt-get install -y libglpk-dev`) and compile arbitrary R packages from GitHub without requesting DevOps intervention.
3. **Atlas & WebAPI Admin Rights**:
   - The default Atlas security profile assigns developers the `admin` role with global permissions:
     - Ability to register, update, and delete Source Daimons.
     - Ability to view and cancel background generation jobs.
     - Full import/export permissions for Capr/Circe JSON and cohort definitions.
4. **Remote IDE & Workstation Connectivity**:
   - Native support for **VS Code Remote (SSH / Containers)** and **JetBrains Gateway** directly into the Dedicated R Server and Python agent environments.
   - Public HTTPS access to RStudio Server (`https://<domain>/rstudio/`) and Web SQL IDE (`https://<domain>/sql/`).

---

### Pillar 3: Agentic AI Testing & Experimentation Workbench
The playground is a primary testbed for autonomous healthcare AI agents:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        AGENTIC AI EXPERIMENTATION TOPOLOGY                             │
│                                                                                        │
│   [ External Agentic Frameworks ]              [ Local Sovereign LLM Engine ]          │
│   - Claude Code / Desktop                      - Ollama 0.5.x on Port 11434            │
│   - Cursor AI IDE                              - Models: llama3.3:70b, qwen2.5-coder   │
│   - Antigravity IDE                            - GPU Acceleration (NVIDIA Container)   │
│   - AutoGen / CrewAI / LangGraph                                                       │
└───────────────────────────┬───────────────────────────────────┬────────────────────────┘
                            │                                   │
                            ▼                                   ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                         DUAL MCP GATEWAYS (DEVELOPER MODE)                             │
├────────────────────────────────────────────┬───────────────────────────────────────────┤
│ WebApiMcp Bridge (:8765)                   │ StudyAgent FastMCP Gateway (:8790)        │
│ Upstream: schuemie/WebApiMcp               │ Upstream: OHDSI/StudyAgent                │
│ Path: /webapi-mcp/mcp                      │ Path: /mcp/sse                            │
├────────────────────────────────────────────┼───────────────────────────────────────────┤
│ - DEV_MODE=true: Token-less or pre-shared  │ - Verbose Tracebacks: Returns full SQL    │
│   developer token authentication.          │   compile error details to LLM agents.    │
│ - Schema Tools: list_sources,              │ - Configurable AST Guardrail: Allow testing│
│   search_concepts, get_cohort_definition,  │   complex analytical queries with bypass  │
│   generate_cohort, get_generation_status.  │   flag for trusted developer agents.      │
│ - Direct Circe & Capr JSON compilation.    │ - Suppression Bypass: Cell count masking  │
│                                            │   disabled/configurable for synthetic data│
└────────────────────────────────────────────┴───────────────────────────────────────────┘
```

1. **Dual MCP Standards Pre-Configured**:
   - Both [`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp) (for cohort and WebAPI operations) and [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) (for data queries and characterizations) are running and exposed over streamable HTTP and SSE.
2. **Developer-Friendly Debug Telemetry**:
   - `DEBUG=true` enabled: If an LLM agent generates an invalid SQL transpilation or Capr expression, the MCP server returns the full compiler traceback and AST parsing error rather than an opaque `400 Bad Request`. This allows autonomous agents to self-correct during experiment loops.
3. **Optional Privacy Masking**:
   - Since this is a synthetic playground, Small Cell Suppression (`MIN_CELL_COUNT >= 5`) can be toggled via `ENFORCE_SMALL_CELL_SUPPRESSION=false` in `.env` so data scientists can inspect exact raw frequency distributions.
4. **Local Sovereign LLM Support (Ollama)**:
   - Containerized Ollama on port 11434 with NVIDIA GPU passthrough (`runtime: nvidia`), enabling local multi-agent research without sending data or queries to commercial cloud APIs.

---

### Pillar 4: Developer-Centric Tooling Additions
To make this a true turnkey playground, DevOps must deploy key developer tools alongside the core OHDSI stack:

1. **Web SQL IDE / Database Studio (CloudBeaver / pgAdmin 4)**:
   - Exposed on public HTTPS at `https://<domain>/sql/`.
   - Allows instant browser-based querying, visual explain plans, table autocomplete, and ER diagram browsing without installing local database clients.
2. **Local S3 / Object Store Mock (MinIO)**:
   - Exposed at `https://<domain>/s3/` (API port 9000, Web Console port 9001).
   - Provides local S3-compatible storage for Strategus study artifacts, Parquet files, and large tabular exports without incurring AWS egress fees.
3. **Observability & Diagnostics Dashboard (Netdata or Prometheus/Grafana)**:
   - Real-time monitoring of container CPU, RAM, disk I/O, and PostgreSQL active queries.
   - When the server slows down, developers can immediately see which R script or agent loop is consuming resources.

---

### Pillar 5: System Resilience & Anti-Crash Guardrails
Because developers are actively trying complex workflows, the server must prevent one bad query from taking down the entire environment:

1. **Container cgroup Memory Clamping**:
   - `broadsea-hades` (R Server): `mem_limit: 48g`, `mem_reservation: 8g`
   - `ohdsi-postgres`: `mem_limit: 48g`, `shm_size: 16g`
   - `webapi-classic`: `mem_limit: 12g`
   - **Protection**: If an R user triggers an out-of-memory error, the Linux kernel kills only that R session; PostgreSQL and Docker remain completely healthy.
2. **NVMe Swap Partition**:
   - 32 GB to 64 GB swap partition configured on high-speed PCIe Gen4 NVMe with `vm.swappiness = 10`.
   - Prevents abrupt kernel panics during momentary memory spikes from concurrent causal inference jobs.
3. **PostgreSQL Anti-Lock Configuration**:
   - `statement_timeout = '15min'`: Kills runaway unindexed Cartesian joins automatically (can be overridden per-session for authorized long-running Strategus studies).
   - `idle_in_transaction_session_timeout = '10min'`: Automatically terminates abandoned sessions that hold locks.
   - `temp_file_limit = '50GB'`: Prevents a recursive query from filling the entire NVMe drive.

---

## 4. What DevOps Must Build and Maintain (The Delivery Contract)

The following checklist defines what the DevOps team must provision and hand over to the Data Science & Informatics team:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        DEVOPS TO DEVELOPER DELIVERY CONTRACT                           │
├────────────────────┬───────────────────────────────────────────┬───────────────────────┤
│ Component          │ Deliverable Specification                 │ Developer Access URL  │
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **Atlas 3.0**      │ Next-Gen Vue 3 UI (Admin Role configured) │ `https://<domain>/`   │
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **Atlas Classic**  │ Knockout.js UI with Vocab search          │ `https://<domain>/atlas/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **WebAPI**         │ REST Engine with TrexSQL & Sources        │ `https://<domain>/WebAPI/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **RStudio Server** │ Free AGPL-3.0 IDE with sudo & HADES pkgs  │ `https://<domain>/rstudio/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **Study Shiny**    │ Shiny Server mounting study dashboards    │ `https://<domain>/shiny/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **Study Reports**  │ Analytical HTML & Quarto Reports directory│ `https://<domain>/reports/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **WebApiMcp**      │ Schuemie WebAPI MCP Bridge for AI Agents  │ `https://<domain>/webapi-mcp/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **StudyAgent MCP** │ Official StudyAgent FastMCP Gateway       │ `https://<domain>/mcp/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **Arachne Node**   │ Federated study execution node & engine   │ `https://<domain>/arachne/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **Web SQL Studio** │ CloudBeaver / pgAdmin Web SQL interface   │ `https://<domain>/sql/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **MinIO S3 Mock**  │ S3-compatible study artifact storage      │ `https://<domain>/s3/`│
├────────────────────┼───────────────────────────────────────────┼───────────────────────┤
│ **Fast Rollback**  │ ZFS/Btrfs CLI snapshot scripts            │ `ohdsi-snapshot` CLI  │
└────────────────────┴───────────────────────────────────────────┴───────────────────────┘
```

---

## 5. Architectural Recommendation & Action Plan

1. **Codify Developer Playground Requirements** in [`STAGE_GATED_SPECIFICATIONS.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md):
   - Add **REQ-049** (Fast Snapshot & Rollback Architecture).
   - Add **REQ-050** (Developer & Admin Superuser Access Permissions).
   - Add **REQ-051** (Agentic AI Testing Sandbox & Debug Telemetry).
   - Add **REQ-052** (System Resilience, cgroups & Crash Isolation).
   - Add **REQ-053** (Developer Tooling: Web SQL IDE & MinIO S3 Mock).
2. **Update Container Roster & Ingress Specifications**:
   - Include CloudBeaver (`cloudbeaver`) on port 8978 (`/sql/`).
   - Include MinIO (`minio`) on port 9000/9001 (`/s3/`).
   - Document ZFS/Btrfs CoW snapshot mount options in [`OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md).
3. **Equip Developers with Instant Reset Scripts** in [`ENGINEERING_EXECUTION_PLAN.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/ENGINEERING_EXECUTION_PLAN.md).
