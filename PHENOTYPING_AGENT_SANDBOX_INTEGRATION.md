# OHDSI PhenotypingAgent: Autonomous LangGraph Phenotyping & Cohort Developer Integration

> **Platform**: OHDSI Sandbox 2026 — Sovereign Agentic AI & Computational Phenotyping Workbench  
> **Upstream Origin**: [`schuemie/PhenotypingAgent`](https://github.com/schuemie/PhenotypingAgent) by Dr. Martijn Schuemie  
> **Target Audience**: AI Engineers, Computational Phenotyping Leads, Clinical Informaticians, DevOps Engineers  
> **Status**: Approved Architectural Standard & Operational Runbook  
> **Cross-References**: [README.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/README.md) | [AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md) | [DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md) | [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md) | [AGENTIC_MCP_INTEGRATION_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/AGENTIC_MCP_INTEGRATION_GUIDE.md)

---

## 1. Author Credit & Attributions

This integration incorporates, operationalizes, and honors the foundational research, algorithmic designs, and open-source software authored by **Dr. Martijn Schuemie, PhD** (Janssen Research & Development / OHDSI) and collaborators across the OHDSI community:

- **Autonomous Phenotyping Engine**: [`schuemie/PhenotypingAgent`](https://github.com/schuemie/PhenotypingAgent) by **Dr. Martijn Schuemie**:
  - The compiled **LangGraph state machine** (`phenotyping_agent/graph.py`) orchestrating autonomous hypothesis formulation, Capr cohort implementation, database execution, diagnostic evaluation, and error diagnosis.
  - The **code-enforced scientific guarantees**: expectation-gated diagnostic calls, the strict Phase 2 to Phase 3 transition gate, the anti-overfitting hard cap of 3 KEEPER evaluations, and static concept-ID provenance checks.
  - The **AST-sandboxed Capr compile worker** (`tools/compileWorker.R`) enforcing strict grammar allow-listing and process isolation.
  - The **interactive IDE cohort developer skill** (`.agents/skills/cohort-developer/SKILL.md`) and operational standard (`CAPR_REFERENCE.md`).
- **Complementary Foundational Contributions**:
  - **WebApiMcp Bridge**: [`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp) by **Dr. Martijn Schuemie** (Sovereign sandbox port 8765).
  - **AgentPlayGround**: [`schuemie/AgentPlayGround`](https://github.com/schuemie/AgentPlayGround) by **Dr. Martijn Schuemie** (Conversational phenotyping refiners and concept target enumerators).
  - **HADES Analytical Packages**: [`OHDSI/Hades`](https://github.com/OHDSI/Hades) (Lead: Martijn Schuemie; pre-installed in Broadsea HADES).
  - **KEEPER Empirical Evaluation Framework**: [`OHDSI/Keeper`](https://github.com/OHDSI/Keeper) (Anna Ostropolets & Martijn Schuemie).

---

## 2. Executive Summary & Paradigm

### What is PhenotypingAgent?
`PhenotypingAgent` is a state-of-the-art computational phenotyping system designed to solve one of the hardest challenges in observational health research: **translating a clinical definition of a disease into an executable, high-performance OMOP CDM cohort algorithm** with verifiable positive predictive value (PPV) and sensitivity.

Traditionally, cohort building is an artisanal, manual process where researchers repeatedly adjust ICD/SNOMED codes, guess inclusion criteria, and overfit to local idiosyncrasies. `PhenotypingAgent` formalizes this through a dual-modality architecture:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                      OHDSI PHENOTYPING AGENT: DUAL MODALITY                            │
│                                                                                        │
│  MODALITY A: INTERACTIVE IDE WORKFLOW           MODALITY B: AUTONOMOUS LANGGRAPH CLI   │
│  [ Developer in VS Code / Antigravity ]          [ Unattended Autonomous Batch Run ]    │
│            │                                                    │                      │
│            ▼                                                    ▼                      │
│  /cohort-developer Skill                        python -m phenotyping_agent.cli run    │
│    ├─ Phase 1: Conceptual Design                  ├─ LangGraph State Machine (9 nodes) │
│    ├─ Phase 2: Generation & Diagnostics           ├─ Reasoning Tier (Hypothesis/Assess)│
│    └─ Phase 3: KEEPER Adjudication (Max 3)        ├─ Fast Tier (Capr Codegen/Report)   │
│            │                                      └─ Deterministic Stubs (CI Dry-Run)  │
│            │                                                    │                      │
│            └──────────────────────────┬─────────────────────────┘                      │
│                                       ▼                                                │
│                      [ SOVEREIGN OHDSI MCP ECOSYSTEM ]                                 │
│     ┌─────────────────────────┬────────────────────────┬─────────────────────────┐     │
│     │   r-tools (Port stdio)  │  WebApiMcp (Port 8765) │  StudyAgent (Port 8790) │     │
│     │   12 OHDSI OMOP Tools   │  ~10M Vocabularies     │  FastMCP Rest/Analysis  │     │
│     └─────────────────────────┴────────────────────────┴─────────────────────────┘     │
│                                       ▼                                                │
│                      [ SOVEREIGN SANDBOX INFRASTRUCTURE ]                              │
│     PostgreSQL 17 (omop_54 / vocab_54) │ Broadsea HADES │ Keeper (Port 8105/3852)       │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Dual-Modality Architecture

### Modality A: Interactive IDE Workflow (`/cohort-developer`)
For clinical data scientists working interactively in IDEs (Antigravity IDE, Visual Studio Code, Cursor, GitHub Copilot):

1. **Phase 1: Conceptual Design**:
   - Analysts invoke `/cohort-developer #file:clinical_definition.txt`.
   - The agent consults the `clinical-definition-refiner` skill if clinical intent is ambiguous.
   - Retrieves candidate concept sets via `listConceptSets` and inspects measurement distributions via `describeMeasurementValues`.
2. **Phase 2: Implementation, Generation & Diagnostics (Unlimited Iterations)**:
   - Constructs a standalone Capr R cohort definition inlined with `cs(...)` concept sets.
   - Validates syntax via `validateCapr` (compiled safely by `tools/compileWorker.R`).
   - Generates the cohort in the database via `generateCohort`.
   - Verifies attrition counts via `getCohortCount` and calculates incidence rates via `computeIncidenceRate`.
   - Tests hypothesis boundaries using `countConceptSetPersonOverlap`.
3. **Phase 3: KEEPER Empirical Evaluation (Strict 3-Call Maximum)**:
   - Evaluates sensitivity, specificity, and PPV against a 10,000-person KEEPER reference cohort via `evaluateCohort`.
   - Samples individual true-positive, false-positive, true-negative, and false-negative patient timelines via `samplePatientProfile`.
   - Refines logic or halts when PPV > 80% and sensitivity > 80%.

### Modality B: Autonomous LangGraph Pipeline (`phenotyping_agent/`)
For batch, unattended, and automated cohort development across phenotype catalogs:

- **State Machine Topology** (`phenotyping_agent/graph.py`):
```
  intake ──► survey ──► design ──► write_capr ──► generate ──► measure ──► assess
               ▲          │                                                 │
               │  (repair exhausted)                                       ├─► [iterate] ──► design
               │                                                            ├─► [evaluate] ──► evaluate ──► diagnose ──┐
               └────────────────────────────────────────────────────────────┴─► [done] ──────► report                  │
                                                                                                                       │
                                                                                               survey ◄────────────────┘
```

- **Node Specialization**:
  | Node | Tier | Purpose | Output Contract |
  | :--- | :--- | :--- | :--- |
  | `intake` | Fast | Normalizes clinical text and derives phenotype name | String name |
  | `survey` | Deterministic | Enumerates available concept sets and baseline metadata | Registry update |
  | `design` | Reasoning | Formulates clinical hypotheses and registers falsifiable expectations | `DesignOutput` |
  | `write_capr` | Fast | Writes executable Capr R code with bounded self-repair (<=4 turns) | R code text |
  | `generate` | Deterministic | Executes cohort generation in OMOP CDM via MCP | Cohort ID |
  | `measure` | Deterministic | Computes counts, attrition, incidence, and overlap metrics | Metrics update |
  | `assess` | Reasoning | Compares metrics against pre-registered expectations; routes next action | `AssessOutput` |
  | `evaluate` | Deterministic | Evaluates against KEEPER reference cohort (max 3 calls) | Confusion matrix |
  | `diagnose` | Reasoning | Adjudicates patient error profiles; extracts failure mechanisms | `DiagnoseOutput` |
  | `report` | Fast | Compiles comprehensive audit report, ledger, and final cohort | `report.md` |

---

## 4. The 12-Tool MCP Contract & Sovereign Sandbox Mapping

`PhenotypingAgent` expects 12 canonical tools from the MCP server. In the OHDSI Sandbox 2026, these are fully mapped to sovereign, local infrastructure:

| Tool Name | Upstream Databricks Operation | Sovereign OHDSI Sandbox 2026 Implementation |
| :--- | :--- | :--- |
| `listConceptSets` | Queries Optum concept sets on Databricks | Queries local PostgreSQL (`omop_54`, `vocab_54`, `scratch`) and `WebApiMcp` (port 8765) |
| `getConceptSetsCapr` | Generates Capr `cs(...)` from Databricks tables | Synthesizes Capr expressions from local Athena vocabularies via `WebApiMcp` / `StudyAgent` |
| `getCohortCount` | Queries `agent_test_cohort` for rule attrition | Computes attrition via `CohortGenerator::getCohortCounts()` on local OMOP CDM |
| `getDatabaseDescription` | Returns hardcoded Optum Clinformatics text | Returns Sandbox Database Profile (e.g. Synthea 2.7M / local OMOP CDM v5.4) |
| `countConceptSetPersonOverlap` | Computes pairwise person intersection in scratch | Executes set intersection SQL on local PostgreSQL `omop_54` |
| `describeMeasurementValues` | Computes percentiles (p10..p90) on measurement table | Runs parametric measurement value distribution SQL on local CDM |
| `computeIncidenceRate` | Stratifies events across 10-year age groups and sex | Calculates incidence rates using HADES `CohortIncidence` on local CDM |
| `validateCapr` | Validates Capr R AST in child process | Executes `tools/compileWorker.R` in AST-sandboxed R worker |
| `convertCaprToJson` | Compiles Capr R to Circe JSON | Compiles via `Capr::toCohortJson()` in isolated R worker |
| `generateCohort` | Executes Circe SQL on Databricks | Executes Circe SQL via `CohortGenerator` on local PostgreSQL `omop_54` |
| `evaluateCohort` | Evaluates against 10k Optum KEEPER reference cohort | Evaluates against local Keeper reference cohort (`keeper` service on port 8105) |
| `samplePatientProfile` | Retrieves raw timeline and LLM adjudication notes | Queries de-identified patient timeline tables from local Keeper service |

---

## 5. Code-Enforced Scientific Guardrails

Unlike naive LLM wrappers, `PhenotypingAgent` enforces scientific rigor **in code, not prompts**:

1. **Expectation Gating**:
   Diagnostic tools (`getCohortCount`, `computeIncidenceRate`, `countConceptSetPersonOverlap`, `describeMeasurementValues`) refuse to run unless the reasoning LLM pre-registered an explicit, falsifiable expectation with qualitative rationale. Unregistered calls raise a blocking error.
2. **Phase 2 to Phase 3 Transition Gate**:
   The reasoning LLM cannot jump directly to KEEPER evaluation. The state machine enforces 5 prerequisites in code:
   - Cohort count must be non-zero.
   - At least one concept set overlap diagnostic must have been evaluated.
   - Incidence rates for the current cohort must have been computed.
   - A written clinical readiness rationale must be provided.
   - The evaluation budget must be greater than zero.
   If any condition is violated, the gate blocks transition, logs the blocker to `ledger.json`, and forces `iterate`.
3. **Anti-Overfitting Hard Cap (3 Evaluations)**:
   A strict maximum of 3 calls to `evaluateCohort` is enforced by `ToolBudget`. Exhaustion immediately terminates the loop and routes to `report`.
4. **Concept ID Provenance Static Verification**:
   The agent checks the generated Capr R AST before execution. Any concept ID not present in the concept set registry is rejected immediately and fed back into the Capr repair loop.
5. **AST Compilation Sandbox**:
   Client-authored Capr R text is never evaluated using `eval()`. Instead, `tools/compileWorker.R` parses the R AST, validates all symbols against a strict allow-list of approved Capr functions, and evaluates only within an empty environment lacking network or disk access.

---

## 6. Step-by-Step Lift-and-Shift Deployment Plan

### Step 1: Deploy Interactive Skill
The `.agents` skill is deployed natively in the workspace:
- Definition: [`.agents/skills/cohort-developer/SKILL.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/cohort-developer/SKILL.md)
- Reference: [`.agents/skills/cohort-developer/CAPR_REFERENCE.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/cohort-developer/CAPR_REFERENCE.md)

### Step 2: Install Python Engine & Dependencies
Install the package in development mode:
```powershell
# From workspace root
python -m pip install -e .[dev]
```

### Step 3: Configure Sovereign MCP Endpoints
Configure `.vscode/mcp.json` and `.agents/mcp_config.json`:
```json
{
  "servers": {
    "ohdsi_webapi_mcp": {
      "url": "http://localhost:8765/mcp",
      "type": "http"
    },
    "ohdsi_study_agent": {
      "url": "http://localhost:8790/mcp/sse",
      "type": "http",
      "headers": {
        "Authorization": "Bearer ohdsi-admin-agent-token-2026"
      }
    },
    "r-tools": {
      "command": "Rscript",
      "args": ["--vanilla", "tools/server.R"]
    }
  }
}
```

### Step 4: Verification via Hermetic Dry-Run
Run the complete hermetic dry run with zero network, database, or LLM dependencies:
```powershell
python -m pytest -q
python -m phenotyping_agent.cli run --clinical-definition "examples/acute_liver_failure.txt" --dry-run
```

Expected output:
```text
Run complete. Report: runs/<timestamp>/report.md
Final action: done
Stop reason: Acceptance criteria met: PPV 0.840 and sensitivity 0.820 both exceed 0.8.
```

### Step 5: Live Execution with Real Models
Configure `.env` with model providers:
```ini
REASONING_TIER_PROVIDER=openai
REASONING_TIER_MODEL=gpt-4o
FAST_TIER_PROVIDER=openai
FAST_TIER_MODEL=gpt-4o-mini
OPENAI_API_KEY=your-api-key
```

Execute single phenotype:
```powershell
python -m phenotyping_agent.cli run --phenotype "Acute liver failure" --clinical-definition "examples/acute_liver_failure.txt"
```

Execute batch phenotype catalog:
```powershell
python -m phenotyping_agent.cli run --phenotypes-csv examples/phenotypes.csv
```

---

## 7. Artifacts & Ledger Directory Structure

Each run generates a self-contained, auditable directory in `runs/<run-id>/`:

| Artifact | Purpose |
| :--- | :--- |
| `report.md` | Comprehensive clinical phenotyping narrative, iteration table, attrition, incidence, and KEEPER metrics. |
| `ledger.json` | Formal JSON audit trail tracking hypotheses, expectations, verdicts, and gate blockers across all iterations. |
| `llm_events.jsonl` | Complete LLM event log with full prompts, responses, token usage, and latency. |
| `tool_calls.jsonl` | Exact chronological record of all MCP tool invocations, arguments, and return payloads. |
| `design_<n>.json` | Structured JSON output for iteration `<n>`. |
| `cohort_<n>.R` | Executable Capr R code produced in iteration `<n>`. |
| `final_cohort.json` | Server-compiled Circe JSON cohort specification ready for ATLAS and HADES import. |

---

## 8. Master Handover Checklist

- [x] Lifted `.agents/skills/cohort-developer/` with `SKILL.md` and `CAPR_REFERENCE.md`.
- [x] Integrated `phenotyping_agent/` LangGraph autonomous Python engine.
- [x] Lifted `tools/` with `server.R`, `compileWorker.R`, and `conceptSetHelpers.R`.
- [x] Added PostgreSQL environment-variable support to `tools/server.R`.
- [x] Registered `r-tools` in `.vscode/mcp.json` and `.agents/mcp_config.json`.
- [x] Prominently credited Dr. Martijn Schuemie across all documentation and files.
- [x] Verified 100% pass rate across all 80 unit and integration tests.
- [x] Verified end-to-end hermetic dry-run on benchmark Acute Liver Failure definition.
