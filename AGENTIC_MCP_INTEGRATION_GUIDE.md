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

### 2. StudyAgent FastMCP Tools (`OHDSI/StudyAgent`)
These tools allow agentic software to run analytical jobs and inspect data securely:

| Tool Name | Parameters | Safety Guardrail & Purpose |
| :--- | :--- | :--- |
| **`run_analytical_query`** | `{"sql": string, "sourceKey": string}` | Executes read-only OMOP SQL queries. **Guardrails**: AST parser blocks destructive SQL (`DROP`, `DELETE`, `UPDATE`); Small Cell Suppression masks counts `< 5`. |
| **`fetch_data_characterization`** | `{"sourceKey": string, "table": string}` | Fetches Achilles precomputed summaries and demographic distributions. |
| **`submit_network_study`** | `{"studyPackageUrl": string, "sourceKey": string}` | Submits study execution package to local Dedicated R Server / Arachne execution queue. |

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
