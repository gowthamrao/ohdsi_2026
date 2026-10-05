# OHDSI AgentPlayGround: Sandbox Integration & Lift-and-Shift Architectural Guide

> **Platform**: OHDSI Sandbox 2026 — Agentic AI & Computational Phenotyping Workbench  
> **Upstream Origin**: [`schuemie/AgentPlayGround`](https://github.com/schuemie/AgentPlayGround) by Martijn Schuemie  
> **Target Audience**: AI Engineers, Computational Phenotyping Leads, Clinical Informaticians, DevOps Engineers  
> **Status**: Approved Architectural Standard & Operational Guide  
> **Cross-References**: [README.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/README.md) | [DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md) | [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md) | [AGENTIC_MCP_INTEGRATION_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/AGENTIC_MCP_INTEGRATION_GUIDE.md)

---

## 1. Author Credit & Attributions

This integration directly incorporates, operationalizes, and honors the pioneering research and open-source specifications created by **Dr. Martijn Schuemie** (Janssen Research & Development / OHDSI) and collaborators across the OHDSI community:

- **Agentic Phenotyping Framework**: [`schuemie/AgentPlayGround`](https://github.com/schuemie/AgentPlayGround) by **Dr. Martijn Schuemie**:
  - The iterative conversational clinical phenotyping state machine (`clinical-definition-refiner`).
  - The 6-category boundary-defining concept target enumerator (`concept-set-target-enumerator`).
  - The OHDSI standardized analysis template standardizer (`ohdsi-question-standardizer`).
  - The ontological umbrella term reasoning engine (`phenotype-parent-concept`).
  - The Pydantic study intent models (`study_intent.py`) and Circe cohort directives (`ohdsi-cohorts.md`).
- **WebAPI Model Context Protocol Bridge**: [`schuemie/WebApiMcp`](https://github.com/schuemie/WebApiMcp) by **Dr. Martijn Schuemie**.
- **Foundational Treatment Pathways Protocol**: The benchmark network study protocol in [`examples/OHDSI treatment patterns 30nov2014.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/examples/OHDSI%20treatment%20patterns%2030nov2014.md) was authored by:
  - **Patrick Ryan, PhD** (Janssen Research & Development)
  - **Jon Duke, MD** (Regenstrief Institute)
  - **Martijn Schuemie, PhD** (Janssen Research & Development)
  - **George Hripcsak, MD** (Columbia University)
  - **Nigam Shah, PhD** (Stanford University)
- **Open-Source Attribution**: All lifted skills, schemas, and context documents are attributed to their original creators and preserved under their original open-source community terms.

---

## 2. Executive Summary & Paradigm

### What is AgentPlayGround?
[`schuemie/AgentPlayGround`](https://github.com/schuemie/AgentPlayGround) is an open-source agentic research framework authored by Dr. Martijn Schuemie exploring standard agentic workflows for observational health data science using the emerging `.agents` specification.

### The Problem It Solves
Designing clinical phenotypes and observational study protocols is traditionally an error-prone, cognitively demanding process:
- **Conflating Clinical Intent with Database Codes**: Analysts frequently jump straight to ICD-10 or SNOMED billing codes, skipping the clinical reality of the disease (pathophysiology, disease severity, diagnostic criteria).
- **Pathological Drift**: When selecting parent groupings or umbrella terms, models and humans frequently drift across disease categories (e.g. classifying acute liver injury under acute hepatitis, confusing mechanical injury with infectious inflammation).
- **Fragmented Concept Sets**: Drug classes (e.g. "statins") are used instead of distinct active ingredients, or combination products are conflated with single-ingredient formulations.
- **Ambiguous Study Intent**: Research questions (e.g. "does drug X cause condition Y?") lack formal temporal definitions (Time At Risk, target cohort, active comparator, nesting cohort).

### How It Lifts and Shifts into the OHDSI Sandbox
In its upstream standalone repository, `AgentPlayGround` relied on external cloud MCP endpoints (such as `hecate.pantheon-hds.com`). In the **OHDSI Sandbox 2026**, we have lifted and shifted this framework directly into the sovereign platform:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        AGENTPLAYGROUND IN THE OHDSI SANDBOX                            │
│                                                                                        │
│   [ Developer / Informatician / Agent ]                                                │
│         │                                                                              │
│         ▼                                                                              │
│   [ .agents/ Workspace Skills ]                                                        │
│     1. ohdsi-question-standardizer  ──> Generates study_intent_<id>.json               │
│     2. clinical-definition-refiner  ──> Generates ### Final Clinical Definition        │
│     3. concept-set-target-enumerator──> Generates CSV concept targets                   │
│     4. phenotype-parent-concept     ──> Identifies ontological parent Umbrella Term     │
│         │                                                                              │
│         ▼                                                                              │
│   [ Sovereign Sandbox MCP Gateways ]                                                   │
│     - WebApiMcp Bridge (:8765 / /webapi-mcp/mcp)                                       │
│     - StudyAgent FastMCP (:8790 / /mcp/sse)                                            │
│         │                                                                              │
│         ▼                                                                              │
│   [ Sovereign Data & Compute Tier ]                                                    │
│     - PostgreSQL 16 (vocab_54: 10M Athena concepts with GIN trigram indexes)           │
│     - WebAPI Classic & 3.0 (Circe JSON generation & cohort daimon routing)             │
│     - Dedicated R Server (broadsea-hades: Capr, CohortDiagnostics, HADES)              │
│     - Local Ollama LLM (llama3.3:70b, qwen2.5-coder with GPU passthrough)              │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

1. **Sovereign Local Execution**: All vocabulary searches, concept expressions, and cohort generations execute against the sandbox's local PostgreSQL database (`vocab_54` ~10M concepts with GIN trigram indexes) via `WebApiMcp` and `StudyAgent FastMCP`.
2. **Zero-Egress Sovereign LLM**: Agentic workflows can be driven either by local sovereign LLMs (Ollama `llama3.3:70b` on port 11434) or external frontier models (Claude 3.5 Sonnet, GPT-4o).
3. **Direct HADES & Capr Compilation**: Once concept targets and study intents are finalized, they transpile directly into `Capr` cohort expressions inside the Dedicated R Server (`broadsea-hades`) and instantiate cohorts on the connected OMOP CDM.

---

## 2. The 5-Stage Agentic Phenotyping Pipeline

The lifted skills form an end-to-end computational phenotyping and study design pipeline:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           THE 5-STAGE PHENOTYPING PIPELINE                              │
├───────┬─────────────────────────────────┬──────────────────────┬────────────────────────┤
│ Stage │ Skill Name                      │ Input                │ Output Artifact        │
├───────┼─────────────────────────────────┼──────────────────────┼────────────────────────┤
│ **1** │ `ohdsi-question-standardizer`   │ Natural query / study│ `study_intent_<id>.json`│
│       │                                 │ protocol markdown    │ (Pydantic model)       │
├───────┼─────────────────────────────────┼──────────────────────┼────────────────────────┤
│ **2** │ `clinical-definition-refiner`   │ Disease / phenotype  │ `### Final Clinical    │
│       │                                 │ concept name         │ Definition` (Plaintext)│
├───────┼─────────────────────────────────┼──────────────────────┼────────────────────────┤
│ **3** │ `concept-set-target-enumerator` │ Refined clinical     │ CSV: category, term,   │
│       │                                 │ definition           │ boundary definition    │
├───────┼─────────────────────────────────┼──────────────────────┼────────────────────────┤
│ **4** │ `phenotype-parent-concept`      │ Medical condition    │ JSON: Umbrella Term &  │
│       │                                 │ term                 │ Standard Concept ID    │
├───────┼─────────────────────────────────┼──────────────────────┼────────────────────────┤
│ **5** │ `Capr` / `WebApiMcp` Execution  │ Concept sets & Circe │ Generated OMOP cohort  │
│       │                                 │ rules                │ table in PostgreSQL    │
└───────┴─────────────────────────────────┴──────────────────────┴────────────────────────┘
```

### Stage 1: Question Standardization (`ohdsi-question-standardizer`)
- **Location**: [`.agents/skills/ohdsi-question-standardizer/SKILL.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/ohdsi-question-standardizer/SKILL.md)
- **State Machine Operation**:
  - *State 1 (Clarification)*: If essential study parameters are missing, the agent asks one targeted question at a time with 2–3 suggested options.
  - *State 2 (Final Output)*: Once parameters are unambiguous, the agent generates `study_intent_<id>.json` validating against the Pydantic schema in [`.agents/schemas/study_intent.py`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/schemas/study_intent.py).
- **Supported Study Templates**:
  - *Characterization*: `patient_characterization`, `treatment_patterns`, `outcome_incidence`.
  - *Effect Estimation*: `cohort_method` (new-user active comparator cohort), `self_controlled_case_series` (within-person design).
  - *Prediction*: `patient_level_prediction` (baseline covariates predicting future outcome during Time At Risk).

### Stage 2: Clinical Definition Refinement (`clinical-definition-refiner`)
- **Location**: [`.agents/skills/clinical-definition-refiner/SKILL.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/clinical-definition-refiner/SKILL.md)
- **Facilitation Technique**:
  - Strictly separates clinical intent ("what") from operational database codes ("how"). If the user mentions ICD-10 or SNOMED codes, the agent guides them back to clinical reality.
  - Asks one question at a time with 2–3 clinically relevant answers explaining definitional consequences.
  - Evaluates across 3 dimensions: Core Pathology, Severity/Modifiers, and Conceptual Boundaries (distinct etiologies, secondary pathophysiology, competing states).
  - Outputs a single concise paragraph under `### Final Clinical Definition`.

### Stage 3: Concept Set Target Enumeration (`concept-set-target-enumerator`)
- **Location**: [`.agents/skills/concept-set-target-enumerator/SKILL.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/concept-set-target-enumerator/SKILL.md)
- **Ontological Enumeration**:
  - Takes the refined definition and enumerates concrete targets across 6 clinical categories: `Symptom`, `Drug` (individual active ingredients only), `Diagnostic procedure`, `Treatment procedure`, `Measurement` (specific analytes), and `Alternative diagnosis`.
  - Enforces boundary definitions (3–4 sentences describing intrinsic chemical, anatomical, or physiological properties without mentioning the primary disease).
  - Outputs strictly formatted 3-column CSV: `"category","term","definition"`.

### Stage 4: Parent Concept & Vocabulary Mapping (`phenotype-parent-concept`)
- **Location**: [`.agents/skills/phenotype-parent-concept/SKILL.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/phenotype-parent-concept/SKILL.md)
- **Ontological Discipline**:
  - Deduces the exact immediate Umbrella Term (1–2 semantic levels up) *internally before* invoking any tool.
  - Prevents pathological drift (e.g. distinguishes injury from inflammation).
  - Uses the sandbox's `WebApiMcp` tool (`search_concepts`) to find the Standard Concept ID matching the deduced Umbrella Term.
  - Outputs JSON with explicit ontological reasoning.

### Stage 5: Cohort Operationalization & Execution
- **Context Directive**: [`.agents/context/ohdsi-cohorts.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/context/ohdsi-cohorts.md)
- **Execution Mechanism**:
  - The mapped concept IDs are combined with Circe cohort entry, inclusion criteria, and exit rules.
  - The cohort expression is compiled to SQL via `Capr` / `CirceR` inside `broadsea-hades` or submitted to WebAPI via `WebApiMcp` (`generate_cohort`).
  - Output tables are generated in the synthetic PostgreSQL database (`ohdsi.results.cohort`).

---

## 3. Workspace File Layout

The lifted components reside natively in the repository root:

```
ohdsi_2026/
├── .agents/
│   ├── AGENTS.md                                   # Global agent persona & OHDSI standards
│   ├── mcp_config.json                             # Sandbox MCP server definitions
│   ├── context/
│   │   └── ohdsi-cohorts.md                        # Circe cohort rules & temporal logic
│   ├── schemas/
│   │   └── study_intent.py                         # Pydantic schema for study intent JSON
│   └── skills/
│       ├── clinical-definition-refiner/
│       │   └── SKILL.md                            # Conversational clinical refiner
│       ├── concept-set-target-enumerator/
│       │   └── SKILL.md                            # 6-category concept set enumerator
│       ├── ohdsi-question-standardizer/
│       │   └── SKILL.md                            # OHDSI analytical template standardizer
│       └── phenotype-parent-concept/
│           └── SKILL.md                            # Ontological parent concept mapper
├── .vscode/
│   └── mcp.json                                    # VS Code & Copilot MCP config
├── examples/
│   └── OHDSI treatment patterns 30nov2014.md       # Benchmark OHDSI network study protocol
├── AGENTIC_MCP_INTEGRATION_GUIDE.md                # MCP client configuration guide
├── AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md         # This architectural guide
└── docker-compose.yml                              # 20-container sandbox stack
```

---

## 4. IDE & Agent Client Integration

Developers and AI models can invoke these skills across diverse development environments:

### A. Antigravity IDE
The Antigravity IDE natively detects the `.agents/` directory:
- Type `/clinical-definition-refiner <disease_name>` in chat to launch the interactive facilitator.
- Type `/ohdsi-question-standardizer <protocol_text>` to standardize study intent.
- Tools from `.agents/mcp_config.json` are automatically discovered and bound to the agent.

### B. VS Code with GitHub Copilot
- VS Code automatically reads `.vscode/mcp.json`.
- In Copilot chat (agent mode), invoke skills via:
  ```
  /ohdsi-question-standardizer #OHDSI treatment patterns 30nov2014.md
  /clinical-definition-refiner Acute liver injury
  ```

### C. Cursor IDE
Add to `.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "webapi-mcp": {
      "url": "http://localhost:8765/mcp",
      "transport": "http"
    },
    "studyagent": {
      "url": "http://localhost:8790/mcp/sse",
      "transport": "sse",
      "headers": {
        "Authorization": "Bearer ohdsi-admin-agent-token-2026"
      }
    }
  }
}
```

### D. Claude Desktop
Add to `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "ohdsi-webapi": {
      "command": "npx",
      "args": ["-y", "mcp-proxy", "http://localhost:8765/mcp"]
    },
    "ohdsi-study-agent": {
      "command": "npx",
      "args": ["-y", "mcp-proxy", "http://localhost:8790/mcp/sse", "--header", "Authorization: Bearer ohdsi-admin-agent-token-2026"]
    }
  }
}
```

---

## 5. End-to-End Verification Walkthrough

To verify that the lifted AgentPlayGround skills operate correctly within the sandbox:

### Step 1: Standardize a Benchmark Protocol
In your agent chat, run:
```
/ohdsi-question-standardizer @examples/OHDSI treatment patterns 30nov2014.md
```
**Verification**: The agent analyzes the 30-page protocol, identifies the 3 chronic disease target cohorts (hypertension, type 2 diabetes mellitus, depression), identifies treatment cohorts, and outputs `study_intent_treatment_patterns.json` validating against `StudyIntent`.

### Step 2: Refine a Clinical Concept
In your agent chat, run:
```
/clinical-definition-refiner Acute myocardial infarction
```
**Verification**: The agent asks one targeted question regarding symptom duration, STEMI vs NSTEMI inclusion, and troponin thresholds, presenting 3 numbered choices. Upon completion, it outputs `### Final Clinical Definition`.

### Step 3: Enumerate Concept Set Targets
Pass the refined clinical definition to:
```
/concept-set-target-enumerator <clinical definition>
```
**Verification**: The agent outputs a CSV block with columns `"category"`, `"term"`, `"definition"`, cleanly separating symptoms (chest pain), diagnostic procedures (coronary angiography), measurements (cardiac troponin I/T), and acute treatments (aspirin, clopidogrel, alteplase).

### Step 4: Map Umbrella Concepts via WebApiMcp
Run:
```
/phenotype-parent-concept Acute liver injury
```
**Verification**: The agent reasons internally that the umbrella concept is "Liver injury" (1 level up, preserving pathology), invokes `search_concepts` against the sandbox's `WebApiMcp` endpoint, and returns the standard OHDSI concept ID (`4128378` - Injury of liver).
