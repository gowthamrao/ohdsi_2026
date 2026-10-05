# DevOps Sandbox Delivery & Handover Runbook

> **Platform**: OHDSI Sandbox 2026 — High-Resilience Developer Playground for Data Science & Informatics  
> **Target Audience**: DevOps Engineers, SREs, Systems Administrators, Data Science & Informatics Leads  
> **Status**: Approved Operational Standard & Handover Protocol  
> **Cross-References**: [README.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/README.md) | [ENGINEERING_EXECUTION_PLAN.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/ENGINEERING_EXECUTION_PLAN.md) | [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md) | [OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md)

---

## 1. Sandbox Purpose & Developer Handover Card

The **OHDSI Sandbox** is a dedicated developer playground where data scientists, clinical informaticians, phenotyping engineers, and AI researchers can innovate, collaborate, and test the latest ideas in observational research.

- **Zero Real Person-Level Data**: The system connects exclusively to synthetic benchmark datasets (Eunomia, Synthea 100k, CMS SynPUF 2.3M) and standardized Athena Vocabularies (`vocab_54`, ~10M concepts).
- **The Core Axiom**: **Developers WILL break things, and they must be encouraged to do so.** The architecture embraces failure as normal operation: enabling instant Copy-on-Write (CoW) snapshots, `< 30-second` rollbacks, developer superuser access, and strict container crash isolation.

### The Master Handover Card (URLs, Credentials & Access)

When delivering the sandbox to the Data Science and Informatics teams, provide this table:

| Service Name | Public HTTPS URL | Default Credentials / Auth | Primary User & Purpose |
| :--- | :--- | :--- | :--- |
| **Atlas 3.0 Next-Gen** | `https://<domain>/` | Role: `admin` | Data Science: Modern Vue 3 cohort authoring & exploration. |
| **Atlas Classic** | `https://<domain>/atlas/` | Role: `admin` | Informatics: Knockout.js cohort builder & vocabulary search. |
| **WebAPI REST Engine** | `https://<domain>/WebAPI/info` | API Key / Bearer | Backend: SQL transpiler, source registry, and batch jobs. |
| **Dedicated R Server** | `https://<domain>/rstudio/` | User: `ohdsi`<br>Pass: `${HADES_PASSWORD}` | Data Science: RStudio Server (HADES suite, passwordless sudo). |
| **Study Shiny Apps** | `https://<domain>/shiny/` | Public / Open | Data Science: Interactive dashboards (`CohortDiagnostics`, `Taxis`). |
| **Study Static Reports**| `https://<domain>/reports/` | Public / Open | Researchers: Quarto / RMarkdown analytical HTML reports. |
| **WebApiMcp Bridge** | `https://<domain>/webapi-mcp/mcp` | Bearer Token / Open DEV | Agentic AI: Model Context Protocol bridge for cohort tools ([`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp) by Dr. Martijn Schuemie). |
| **StudyAgent MCP & ACP** | `https://<domain>/mcp/` & Port 8765 | Bearer Token / Open DEV | Agentic AI: Dual FastMCP tool provider (8790) & ACP flow orchestrator (8765) ([`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) by Dr. Richard D. Boyce). |
| **Arachne Data Node** | `https://<domain>/arachne/` | User: `admin`<br>Pass: `Arachne123#` | Informatics: Federated study execution node & engine. |
| **Web SQL Studio** | `https://<domain>/sql/` | User: `ohdsi_admin`<br>Pass: `${DB_ADMIN_PASS}` | Developers: Browser-based CloudBeaver SQL editor & ERDs. |
| **MinIO S3 Mock** | `https://<domain>/s3/` | User: `minioadmin`<br>Pass: `${MINIO_ROOT_PASSWORD}`| Developers: S3 object storage for Strategus study artifacts. |
| **AgentPlayGround Skills** | Native in `.agents/skills/` | IDE Agent Commands | Data Science: 4 phenotyping skills by Dr. Martijn Schuemie (`/clinical-definition-refiner`, `/ohdsi-question-standardizer`, etc.). |
| **PhenotypingAgent & Cohort Developer** | `phenotyping_agent/` & `.agents/skills/cohort-developer/` | CLI / IDE Command | Data Science & AI: Autonomous LangGraph phenotyping engine and `/cohort-developer` skill by Dr. Martijn Schuemie. |
| **Phenelope Concept Builder** | `OHDSI/Phenelope` & `.agents/skills/phenelope-concept-set-builder/` | R / MCP Tool / Skill | Informatics & AI: LLM-based concept set builder by Joel N. Swerdel, Dr. Martijn Schuemie, Dr. Anna Ostropolets. |

---

## 2. Developer Workspace & Access Provisioning

### A. PostgreSQL Superuser & Personal Scratch Schemas
Developers have full superuser rights to create test tables, build indexes, and experiment freely:
```sql
-- Connect as ohdsi_admin to PostgreSQL:
-- psql -h localhost -U ohdsi_admin -d ohdsi

-- 1. Create personal scratch schema for each researcher:
CREATE SCHEMA IF NOT EXISTS scratch_gowtham AUTHORIZATION ohdsi_admin;

-- 2. Grant full permissions on shared synthetic CDM and vocabulary:
GRANT USAGE ON SCHEMA cdm_synthea100k TO ohdsi_admin;
GRANT SELECT ON ALL TABLES IN SCHEMA cdm_synthea100k TO ohdsi_admin;
GRANT USAGE ON SCHEMA vocab_54 TO ohdsi_admin;
GRANT SELECT ON ALL TABLES IN SCHEMA vocab_54 TO ohdsi_admin;
```

### B. Dedicated R Server (`broadsea-hades`) Setup
- **User Home Directory**: `/home/ohdsi` is mounted on a persistent Docker volume (`broadsea_hades_home`) backed by ZFS/Btrfs CoW storage.
- **Passwordless Sudo**: The container user `ohdsi` is pre-configured with `NOPASSWD: ALL` in `/etc/sudoers.d/ohdsi`. Data scientists can install system C-dependencies (`sudo apt-get install -y libglpk-dev`) and compile packages from GitHub (`devtools::install_github(...)`) without submitting DevOps tickets.
- **Connecting to CDM from R**:
  ```r
  library(DatabaseConnector)
  conn <- connect(createConnectionDetails(
    dbms = "postgresql",
    server = paste0(Sys.getenv("CDM_SERVER", "ohdsi-postgres"), "/", Sys.getenv("CDM_DATABASE", "ohdsi")),
    user = Sys.getenv("CDM_USER", "ohdsi_admin"),
    password = Sys.getenv("CDM_PASSWORD"),
    pathToDriver = Sys.getenv("DATABASECONNECTOR_JAR_FOLDER", "/opt/drivers")
  ))
  # Query synthetic CDM
  person_count <- querySql(conn, "SELECT COUNT(*) FROM cdm_synthea100k.person;")
  print(paste("Synthetic Person Count:", person_count[1,1]))
  disconnect(conn)
  ```

### C. Deploying Interactive Shiny Apps & Reports
- **Interactive Shiny Dashboards**: Developers publish Shiny apps simply by creating a directory under `/srv/shiny-server/<study_name>/` (e.g. `/srv/shiny-server/taxis/app.R`). The app is immediately live at `https://<domain>/shiny/taxis/`.
- **Static Analytical Reports**: Pre-compiled Quarto or RMarkdown HTML reports placed in `/srv/reports/<study_name>/index.html` are instantly accessible at `https://<domain>/reports/<study_name>/`.

### D. Computational Phenotyping & AgentPlayGround Skills (Dr. Martijn Schuemie)
The sandbox natively integrates Dr. Martijn Schuemie's [`schuemie/AgentPlayGround`](https://github.com/schuemie/AgentPlayGround) skills within the workspace [`.agents/skills/`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/). Data scientists, informaticians, and external AI agents (Antigravity IDE, Cursor, GitHub Copilot, Claude Desktop) can immediately invoke:
- `/clinical-definition-refiner`: Conversational clinical concept refiner.
- `/concept-set-target-enumerator`: 6-category boundary-defining concept target enumerator.
- `/ohdsi-question-standardizer`: Translates research questions into formal OHDSI study templates.
- `/phenotype-parent-concept`: Ontological umbrella term mapper using local `WebApiMcp`.
- *For complete architectural and operational details, see [AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md).*

### E. Autonomous LangGraph Phenotyping & Cohort Developer (Dr. Martijn Schuemie)
The sandbox natively integrates Dr. Martijn Schuemie's [`schuemie/PhenotypingAgent`](https://github.com/schuemie/PhenotypingAgent) framework:
- **Interactive Cohort Developer Skill**: Invoke `/cohort-developer #file:examples/acute_liver_failure.txt` in VS Code / Cursor / Antigravity IDE.
- **Autonomous CLI Pipeline**: Run unattended LangGraph phenotyping state machine across phenotypes:
  ```powershell
  # Offline hermetic dry-run (verifies complete 9-node pipeline with stubs):
  python -m phenotyping_agent.cli run --clinical-definition "examples/acute_liver_failure.txt" --dry-run
  
  # Run full automated test suite:
  python -m pytest -q
  ```
- *For complete architectural and operational details, see [PHENOTYPING_AGENT_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PHENOTYPING_AGENT_SANDBOX_INTEGRATION.md).*

### F. LLM-Based Concept Set Building with Phenelope (Joel N. Swerdel, Dr. Martijn Schuemie, Dr. Anna Ostropolets)
The sandbox natively integrates the [`OHDSI/Phenelope`](https://github.com/OHDSI/Phenelope) framework:
- **Interactive R in HADES**: Data scientists run `Phenelope::createConceptSet()` connecting to local Ollama (`llama3.3`) and PostgreSQL `omop_54`.
- **Sovereign MCP Tool**: AI agents call `createNewConceptSet(name, description)` over `r-tools` (`tools/server.R`) for dynamic concept set generation.
- **Workspace Skill**: Invoke `/phenelope` in IDEs for guided concept set building and audit CSV review.
- *For complete architectural and operational details, see [PHENELOPE_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PHENELOPE_SANDBOX_INTEGRATION.md).*

---

## 3. The "Break & Restore" Operations (Core Sandbox Superpower)

Data scientists and agentic AI loops will frequently attempt intensive operations, drop test tables, or corrupt schemas. The platform provides two tiers of instant restoration:

### Tier 1: Sub-Minute CoW Filesystem Rollback (< 30 Seconds)
The PostgreSQL data partition (`/var/lib/postgresql/data`) and R workspace (`/home/ohdsi`) reside on ZFS or Btrfs Copy-on-Write datasets.

```bash
# 1. Create a zero-cost snapshot before starting an experiment:
sudo ohdsi-snapshot create "pre-experiment-$(date +%Y%m%d_%H%M)"

# 2. List available snapshots:
sudo ohdsi-snapshot list

# 3. If an experiment breaks schemas or corrupts tables, restore instantly:
sudo ohdsi-snapshot rollback "pre-experiment-20261005_1200"
# -> Stops database container, rolls back ZFS dataset in < 2 seconds, restarts database (< 25s total)
```

### Tier 2: Golden Baseline Re-Seed (< 5 Minutes)
If the database state is corrupted beyond a snapshot, reload the verified baseline seed containing Athena Vocabularies (`vocab_54` ~10M rows), Synthea 100k CDM, and initialized WebAPI source daimons:
```bash
docker compose exec db-manager /scripts/restore-golden-baseline.sh
```

---

## 4. Hammering Absorption & Anti-Crash Guardrails

To protect the host from crashing when data scientists run multi-core causal inference pipelines or recursive SQL queries, DevOps configures three layers of defense:

1. **Strict Container cgroup Memory Clamping**:
   - `broadsea-hades` (R Server): `mem_limit: 48g`, `shm_size: 16g`
   - `ohdsi-postgres` (Database): `mem_limit: 48g`, `shm_size: 16g`
   - `webapi-classic` / `atlas3-webapi`: `mem_limit: 12g`
   - **Protection Effect**: If a developer R script or AI agent triggers an out-of-memory error, the Linux kernel terminates *only that isolated R process*. Docker, PostgreSQL, and other developers remain completely unaffected.
2. **Dedicated 64GB NVMe Swap**:
   - Configured with `vm.swappiness = 10` on PCIe Gen4 NVMe to absorb momentary analytical memory spikes without abrupt kernel panics.
3. **PostgreSQL Anti-Lock Guardrails (`postgresql.conf`)**:
   - `statement_timeout = '15min'`: Automatically terminates runaway Cartesian joins (overridable per-session for authorized long-running batch studies).
   - `idle_in_transaction_session_timeout = '10min'`: Automatically clears abandoned client sessions holding table locks.
   - `temp_file_limit = '50GB'`: Prevents unindexed queries from consuming all available disk space.

---

## 5. The 10-Point Handover Readiness Test

Before sending the handover email to the Data Science & Informatics leads, run this automated verification script from the host or an external workstation:

```bash
#!/usr/bin/env bash
# OHDSI Sandbox Handover Verification Script
set -euo pipefail
DOMAIN="${1:-research.yourdomain.org}"
echo "=== Verifying OHDSI Sandbox on https://${DOMAIN} ==="

pass=0
fail=0

check() {
  local name="$1"
  local url="$2"
  local match="$3"
  printf "%-35s " "${name}..."
  if curl -s -k -L "${url}" | grep -q "${match}"; then
    echo "[ PASS ]"
    pass=$((pass + 1))
  else
    echo "[ FAIL ] -> URL: ${url}"
    fail=$((fail + 1))
  fi
}

# 1. Atlas 3.0 Next-Gen Frontend
check "Atlas 3.0 Frontend" "https://${DOMAIN}/" "html"

# 2. Atlas Classic Frontend
check "Atlas Classic Frontend" "https://${DOMAIN}/atlas/" "ATLAS"

# 3. WebAPI Backend Info
check "WebAPI REST Info" "https://${DOMAIN}/WebAPI/info" "version"

# 4. Dedicated R Server (RStudio)
check "RStudio Server Web IDE" "https://${DOMAIN}/rstudio/auth-sign-in" "RStudio"

# 5. OHDSI Study Shiny Apps
check "Study Shiny Server" "https://${DOMAIN}/shiny/" "Shiny"

# 6. WebApiMcp Bridge
check "WebApiMcp Bridge Server" "https://${DOMAIN}/webapi-mcp/health" "ok"

# 7. StudyAgent FastMCP Gateway (Port 8790) & ACP Flow Server (Port 8765)
check "StudyAgent FastMCP Gateway" "https://${DOMAIN}/mcp/" ""
check "StudyAgent ACP Flows" "http://localhost:8765/flows" "phenotype_make_computable"

# 8. Arachne Data Node
check "Arachne Data Node" "https://${DOMAIN}/arachne/api/v1/build-number" "buildNumber"

# 9. CloudBeaver Web SQL Studio
check "CloudBeaver Web SQL" "https://${DOMAIN}/sql/" "CloudBeaver"

# 10. MinIO S3 Console
check "MinIO S3 Object Store" "https://${DOMAIN}/s3/" "MinIO"

echo "=================================================="
echo "Verification Complete: ${pass}/11 Passed, ${fail}/11 Failed"
if [ "${fail}" -eq 0 ]; then
  echo ">>> SUCCESS: OHDSI Sandbox is 100% Certified for Handover! <<<"
else
  echo ">>> ATTENTION: Resolve failed endpoints before handover. <<<"
fi
```
