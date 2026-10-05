# Agentic Software & Model Context Protocol (MCP) Integration Guide

> **Target Audience**: AI Engineers, Computational Phenotyping Leads, DevOps Engineers, Agent Developers  
> **Supported Agents**: Claude Desktop, Cursor, Antigravity IDE, Cline, Roo-Code, AutoGen, CrewAI, LangGraph  
> **Status**: Approved Production Specification  
> **Cross-References**: [README.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/README.md) | [DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md) | [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md) | [AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md) | [PUBLIC_DOMAIN_HOSTING_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PUBLIC_DOMAIN_HOSTING_GUIDE.md)

---

## 1. Overview: The OHDSI Agentic Ecosystem

The OHDSI 2026 platform provides two specialized **Model Context Protocol (MCP)** gateways designed to allow external agentic AI software to safely navigate, author, and execute observational healthcare studies:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 AGENTIC SOFTWARE CLIENTS                               │
│            Claude Desktop  │  Cursor  │  Antigravity IDE  │  LangChain / AutoGen       │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ Model Context Protocol (JSON-RPC 2.0)
                                            ▼
                    ┌────────────────────────────────────────────────┐
                    │       Nginx TLS 1.3 Edge Reverse Proxy         │
                    │   - proxy_buffering off; chunked off           │
                    │   - Streamable HTTP & SSE passthrough          │
                    └───────────────┬────────────────┬───────────────┘
                                    │                │
            ┌───────────────────────┘                └────────────────────────┐
            ▼                                                                 ▼
┌──────────────────────────────────────────────┐        ┌──────────────────────────────────────────────┐
│       WebApiMcp Bridge Server (:8765)        │        │      StudyAgent FastMCP Gateway (:8790)      │
│         schuemie/WebApiMcp (Python)          │        │           OHDSI/StudyAgent (Python)          │
│   Exposed at: https://<domain>/webapi-mcp/   │        │        Exposed at: https://<domain>/mcp/     │
├──────────────────────────────────────────────┤        ├──────────────────────────────────────────────┤
│ Core Functions for Agents:                   │        │ Core Functions for Agents:                   │
│ - list_sources (retrieve OMOP data sources)  │        │ - execute_query (AST SQL validation)         │
│ - search_concepts (Athena concept lookup)    │        │ - small_cell_suppression (count < 5 -> <5)   │
│ - get_concept_set (concept set expression)   │        │ - study_pipeline_runner (Strategus dispatch) │
│ - list_cohort_definitions (browse phenotypes)│        │ - patient_timeline_sampler (De-identified)   │
│ - get_cohort_definition (fetch Circe JSON)   │        │                                              │
│ - generate_cohort (trigger generation job)   │        │                                              │
│ - get_generation_status (poll progress)      │        │                                              │
└──────────────────────┬───────────────────────┘        └──────────────────────┬───────────────────────┘
                       │                                                       │
                       ▼                                                       ▼
┌──────────────────────────────────────────────┐        ┌──────────────────────────────────────────────┐
│           OHDSI WebAPI REST Backend          │        │         PostgreSQL 16 CDM & Vocab DB         │
│               Port 8080 (:8080)              │        │              Port 5432 (:5432)               │
└──────────────────────────────────────────────┘        └──────────────────────────────────────────────┘
```

---

## 2. Standard Client Configuration Snippets

### A. Claude Desktop Configuration
Add the servers to your `claude_desktop_config.json`:
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "ohdsi-webapi": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-proxy",
        "https://research.yourdomain.org/webapi-mcp/mcp"
      ]
    },
    "ohdsi-study-agent": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-proxy",
        "https://research.yourdomain.org/mcp/sse",
        "--header",
        "Authorization: Bearer <YOUR_MCP_TOKEN>"
      ]
    }
  }
}
```

---

### B. Cursor IDE Configuration (`.cursor/mcp.json`)
Create or edit `.cursor/mcp.json` in your workspace root:

```json
{
  "mcpServers": {
    "webapi-mcp": {
      "url": "https://research.yourdomain.org/webapi-mcp/mcp",
      "transport": "http"
    },
    "study-agent": {
      "url": "https://research.yourdomain.org/mcp/sse",
      "transport": "sse",
      "headers": {
        "Authorization": "Bearer <YOUR_MCP_TOKEN>"
      }
    }
  }
}
```

---

### C. Antigravity IDE Configuration (`mcp_config.json`)
Add under `.agents/mcp_config.json` or your global agent configuration:

```json
{
  "mcpServers": {
    "webapi-mcp": {
      "url": "https://research.yourdomain.org/webapi-mcp/mcp",
      "transport": "http"
    },
    "studyagent": {
      "url": "https://research.yourdomain.org/mcp/sse",
      "transport": "sse",
      "headers": {
        "Authorization": "Bearer <YOUR_MCP_TOKEN>"
      }
    }
  }
}
```

---

## 3. Tool Catalog for Agentic Workflows

### 1. WebApiMcp Tools (`schuemie/WebApiMcp`)
These tools allow agentic software to manage phenotypes and cohort logic without writing raw SQL:

| Tool Name | Parameters | Purpose & Output |
| :--- | :--- | :--- |
| **`list_sources`** | `{}` | Returns all configured OMOP CDM databases, source keys, and daimon types (`cdm`, `vocabulary`, `results`). |
| **`search_concepts`** | `{"query": string, "sourceKey": string, "domain": optional string}` | Searches Athena vocabulary concepts matching search tokens with standard concept flags. |
| **`get_concept_set`** | `{"conceptSetId": integer}` | Fetches concept set expression details, including excluded and descendant concept inclusions. |
| **`list_cohort_definitions`** | `{"sourceKey": optional string, "limit": optional integer}` | Returns list of authored cohort definitions (IDs, names, author, created timestamp). |
| **`get_cohort_definition`** | `{"cohortDefinitionId": integer}` | Retrieves full Circe JSON definition representing inclusion rules, exit strategies, and concept expressions. |
| **`generate_cohort`** | `{"cohortDefinitionId": integer, "sourceKey": string}` | Triggers asynchronous cohort generation job on the target CDM database. |
| **`get_generation_status`** | `{"cohortDefinitionId": integer, "sourceKey": string}` | Checks cohort generation job status (`RUNNING`, `COMPLETE`, `FAILED`) and patient count. |

### 2. StudyAgent FastMCP & ACP Gateways (`OHDSI/StudyAgent`)
Created by **Dr. Richard D. Boyce, PhD** (University of Pittsburgh) and the **OHDSI Study Agent Workgroup**, StudyAgent provides a dual-service architecture for autonomous and human-in-the-loop study design. See [`STUDYAGENT_SANDBOX_INTEGRATION.md`](STUDYAGENT_SANDBOX_INTEGRATION.md) for full details.

#### A. FastMCP Tools (Port 8790 / Streamable HTTP)
| Tool Name | Parameters | Safety Guardrail & Purpose |
| :--- | :--- | :--- |
| **`phenotype_search`** | `{"query": string, "limit": integer}` | Dense FAISS + sparse BM25 retrieval over indexed OHDSI Phenotype Library. |
| **`phenotype_make_computable`** | `{"narrative": string, "scope": object}` | Validates Capr syntax, checks domain boundaries, and generates Circe JSON. |
| **`keeper_concept_sets`** | `{"phenotype": string, "domain_keys": array}` | Generates condition, symptom, and treatment concept sets for Keeper review. |
| **`run_analytical_query`** | `{"sql": string, "sourceKey": string}` | Executes read-only OMOP SQL queries. **Guardrails**: AST parser blocks destructive SQL (`DROP`, `DELETE`, `UPDATE`); Small Cell Suppression masks counts `< 5`. |
| **`fetch_data_characterization`** | `{"sourceKey": string, "table": string}` | Fetches Achilles precomputed summaries and demographic distributions. |
| **`submit_network_study`** | `{"studyPackageUrl": string, "sourceKey": string}` | Submits study execution package to local Dedicated R Server / Arachne execution queue. |

#### B. Agent Client Protocol (ACP) Flows (Port 8765 / REST APIs)
StudyAgent's ACP server (`study-agent-acp`) orchestrates multi-step flows with fail-closed privacy:
- `POST /flows/phenotype_make_computable`: 3-phase review-gated Capr R & Circe synthesis.
- `POST /flows/phenotype_recommendation`: Recommends cohorts for Target, Comparator, and Outcome roles.
- `POST /flows/phenotype_intent_split`: Deconstructs clinical questions into OHDSI study elements.
- `POST /flows/keeper_concept_sets_generate`: Generates review concept sets for phenotype validation.
- `POST /flows/phenotype_validation_review`: Adjudicates de-identified patient review rows.

---

## 4. Verification & Testing Protocol for Agents

DevOps engineers must verify that MCP endpoints respond correctly to standard JSON-RPC 2.0 requests:

### 1. WebApiMcp Healthcheck
```bash
curl -s https://research.yourdomain.org/webapi-mcp/health | jq .
# Expected: {"status": "ok", "webapi": "connected"}
```

### 2. WebApiMcp Tools Discovery (JSON-RPC `tools/list`)
```bash
curl -s -X POST https://research.yourdomain.org/webapi-mcp/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {}}' | jq .
```
**Expected**: JSON-RPC response with array containing `list_sources`, `search_concepts`, `get_cohort_definition`, etc.

### 3. StudyAgent MCP Handshake (SSE Stream)
```bash
curl -N -s https://research.yourdomain.org/mcp/sse \
  -H "Authorization: Bearer <valid-token>"
# Expected: Event-Stream connection established; event: endpoint received
```

### 4. Interactive MCP Inspector Test
```bash
npx @modelcontextprotocol/inspector https://research.yourdomain.org/webapi-mcp/mcp
```

---

## 5. AgentPlayGround Computational Phenotyping Skills Integration (Dr. Martijn Schuemie)

The platform natively integrates the computational phenotyping skills authored by **Dr. Martijn Schuemie** in [`schuemie/AgentPlayGround`](https://github.com/schuemie/AgentPlayGround) within the workspace [`.agents/skills/`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/):

- **`clinical-definition-refiner`**: Guides researchers through iterative pathophysiological definition formulation without code pollution.
- **`concept-set-target-enumerator`**: Enumerates boundary-defining targets across 6 clinical categories (`Symptom`, `Drug`, `Diagnostic procedure`, `Treatment procedure`, `Measurement`, `Alternative diagnosis`).
- **`ohdsi-question-standardizer`**: Translates natural research questions or markdown study protocols into formal OHDSI analytical templates validating against Pydantic models (`study_intent.py`).
- **`phenotype-parent-concept`**: Uses `WebApiMcp` (`search_concepts`) to map internally deduced Umbrella Terms to Standard Concept IDs.

*For complete architectural specifications, state machine definitions, and end-to-end walkthroughs, see [AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md).*

---

## 6. PhenotypingAgent Autonomous LangGraph Engine & Cohort Developer (Dr. Martijn Schuemie)

The platform natively integrates the autonomous phenotyping engine and interactive cohort developer authored by **Dr. Martijn Schuemie** in [`schuemie/PhenotypingAgent`](https://github.com/schuemie/PhenotypingAgent):

- **Interactive IDE Cohort Developer ([`.agents/skills/cohort-developer/SKILL.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/cohort-developer/SKILL.md))**:
  - Provides the `/cohort-developer` command in VS Code, Cursor, and Antigravity IDE.
  - Implements a 3-Phase lifecycle: Phase 1 (Conceptual Design), Phase 2 (Implementation, Generation, and Attrition/Incidence Diagnostics), and Phase 3 (KEEPER Empirical Evaluation with strict 3-call cap).
  - Adheres strictly to [`CAPR_REFERENCE.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/cohort-developer/CAPR_REFERENCE.md).
- **Autonomous Python LangGraph Engine ([`phenotyping_agent/`](file:///c:/files/git/github/ohdsi/ohdsi_2026/phenotyping_agent/))**:
  - Compiles an autonomous 9-node LangGraph `StateGraph` (`intake`, `survey`, `design`, `write_capr`, `generate`, `measure`, `assess`, `evaluate`, `diagnose`, `report`).
  - Supports two-tier models: Reasoning tier (`gpt-4o`, `claude-3-opus`, `o1`) for clinical design, expectation setting, and error diagnosis; Fast tier (`gpt-4o-mini`, `haiku`) for Capr codegen and narrative compilation; deterministic stubs for offline testing.
- **The 12 OHDSI MCP Tools ([`tools/server.R`](file:///c:/files/git/github/ohdsi/ohdsi_2026/tools/server.R))**:
  - Powered by local PostgreSQL OMOP CDM v5.4, Broadsea HADES, and the local Keeper service.
  - AST security compilation sandbox (`tools/compileWorker.R`) prevents code injection during Capr cohort compilation.

*For complete details, see [PHENOTYPING_AGENT_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PHENOTYPING_AGENT_SANDBOX_INTEGRATION.md).*

---

## 7. Phenelope LLM-Based Concept Set Builder (Joel N. Swerdel, Dr. Martijn Schuemie, Dr. Anna Ostropolets)

The platform natively integrates the LLM-based concept set builder authored by **Joel N. Swerdel** with major contributions from **Dr. Martijn Schuemie** and **Dr. Anna Ostropolets** in [`OHDSI/Phenelope`](https://github.com/OHDSI/Phenelope):

- **Dedicated R Server Engine (`broadsea-hades`)**:
  - `Phenelope` is installed and loadable via `library(Phenelope)`.
  - Connects to local PostgreSQL OMOP CDM v5.4 (`omop_54` / `vocab_54`) and sovereign local Ollama (`llama3.3` on port 11434) via `ellmer`.
- **Dynamic Concept Set Synthesis MCP Tool (`createNewConceptSet`)**:
  - Exposed via `tools/server.R` in the `r-tools` MCP server.
  - Takes `name` and clinical `description`, discovers seed concepts, runs PHOEBE and hierarchy expansion, prompts the LLM for YES/NO adjudication with clinical rationale, and returns inlined Capr `cs(...)` code.
- **Interactive Workspace Skill ([`.agents/skills/phenelope-concept-set-builder/SKILL.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/phenelope-concept-set-builder/SKILL.md))**:
  - Provides the `/phenelope` command for IDE agents to build concept sets with full CSV audit ledgers.

*For complete details, see [PHENELOPE_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PHENELOPE_SANDBOX_INTEGRATION.md).*


