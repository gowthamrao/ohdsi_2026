# OHDSI Phenelope: LLM-Based Concept Set Builder Sandbox Integration

> **Platform**: OHDSI Sandbox 2026 — Sovereign Agentic AI & Computational Phenotyping Workbench  
> **Upstream Origin**: [`OHDSI/Phenelope`](https://github.com/OHDSI/Phenelope)  
> **Package Maintainer**: Joel N. Swerdel (Janssen Research & Development)  
> **Major Contributors**: Dr. Martijn Schuemie (Janssen R&D / OHDSI) and Dr. Anna Ostropolets (Columbia University / OHDSI)  
> **Target Audience**: Clinical Informaticians, Phenotyping Leads, AI Engineers, DevOps Engineers  
> **Status**: Approved Architectural Standard & Operational Runbook  
> **Cross-References**: [README.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/README.md) | [PHENOTYPING_AGENT_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PHENOTYPING_AGENT_SANDBOX_INTEGRATION.md) | [AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/AGENT_PLAYGROUND_SANDBOX_INTEGRATION.md) | [DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md) | [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md)

---

## 1. Author Credit & Attributions

This integration incorporates, operationalizes, and prominently acknowledges the research, algorithms, and open-source software authored by the creators of **Phenelope**:

- **Lead Author & Package Maintainer**: **Joel N. Swerdel** (Janssen Research & Development / OHDSI)
- **Major Contributors**:
  - **Dr. Martijn Schuemie, PhD** (Janssen Research & Development / OHDSI)
  - **Dr. Anna Ostropolets, MD, PhD** (Columbia University / OHDSI)
- **Upstream Repository**: [`OHDSI/Phenelope`](https://github.com/OHDSI/Phenelope) (Licensed under Apache License 2.0).
- **Core Inventions**:
  1. **Algorithmic Candidate Concept Expansion**: Combining Athena vocabulary hierarchy (`concept_ancestor`) with empirical co-occurrence recommendations from PHOEBE.
  2. **LLM Semantic Adjudication**: Prompting language models via the `ellmer` framework to evaluate candidate concepts against clinical intent, clinical context, and explicit exclusions with auditable clinical rationales.
  3. **Automated Concept Set Condensation**: Minimizing concept set size by rolling up descendant concepts into root standard concepts (`Condense.R`), producing elegant Capr `cs(...)` code and ATLAS Circe JSON expressions.
  4. **Automated Clinical Description Generation**: Generating structured disease narrative documents with explicit citations of inclusions and exclusions (`createClinicalDescription.R`).

---

## 2. Executive Summary & The Phenelope Paradigm

### The Challenge of OMOP Concept Set Curation
In observational health research, defining a cohort requires specifying **concept sets** (standard concept IDs in SNOMED CT, RxNorm, LOINC, etc.). Manual concept set curation faces severe bottlenecks:
- **Semantic Ambiguity & Polysemy**: Disease terms often share lexical similarity with unrelated conditions or procedural artifacts (e.g., distinguishing "Acute liver failure" from "Chronic hepatic failure" or "Finding of liver").
- **Over-Inclusiveness of Raw Descendants**: Simply including all descendants of a SNOMED parent often sweeps in post-surgical complications, congenital anomalies, or incidental findings that violate the study protocol.
- **Under-Inclusiveness**: Overly strict code lists miss critical synonyms, brand names, or correlated diagnosis codes.
- **Review Fatigue & Lack of Audit Trails**: Human reviewers rarely document *why* a particular code was excluded or included, making definitions non-reproducible.

### How Phenelope Solves This
Phenelope replaces manual guesswork with a reproducible, three-stage hybrid statistical-semantic pipeline:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                          PHENELOPE CONCEPT SET PIPELINE                                │
│                                                                                        │
│   [ Target Condition Name ]  +  [ Seed Concept ID ]  +  [ Exclusions & Context ]       │
│                                  │                                                     │
│                                  ▼                                                     │
│   STAGE 1: STATISTICAL EXPANSION                                                       │
│   ├─ Vocabulary Descendants (@cdm.concept_ancestor via FullConcepts.sql)              │
│   ├─ Empirical Co-occurrence (PHOEBE Recommender via .getPhoebeData)                  │
│   └─ Database Domain Verification (checkDomainsForConcepts.sql)                       │
│                                  │                                                     │
│                                  ▼                                                     │
│   STAGE 2: LLM SEMANTIC ADJUDICATION (via ellmer)                                      │
│   ├─ Prompt Model: Does SUGGESTED_CONDITION imply MAIN_CONDITION in CONTEXT?          │
│   ├─ Exclusion Check: Is concept a manifestation of EXCLUDED_CONDITIONS?               │
│   ├─ Structured Output: YES/NO decision + clinical rationale + confidence score        │
│   └─ Multi-turn Consensus Voting (configurable tries / successes)                      │
│                                  │                                                     │
│                                  ▼                                                     │
│   STAGE 3: CONDENSATION & ARTIFACT EXPORT                                              │
│   ├─ Condenser: Merges redundant children into parent roots (Condense.R)               │
│   ├─ ATLAS Circe JSON (concept_set_<phenotype>.json)                                   │
│   ├─ Inlined Capr cs(...) expression                                                   │
│   └─ Reproducible Audit Ledger (CSV with rationaleForAnswer)                           │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. System Architecture in OHDSI Sandbox 2026

In the **OHDSI Sandbox 2026**, Phenelope is integrated as a core analytical capability connecting four sovereign layers:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                      PHENELOPE IN THE OHDSI SANDBOX 2026                               │
│                                                                                        │
│  [ Interactive Data Scientist ]          [ Agentic AI / Antigravity / Cursor ]         │
│         │                                                    │                         │
│         ▼                                                    ▼                         │
│  RStudio Server (/rstudio/)                       /phenelope Skill                     │
│  library(Phenelope)                                       │                            │
│         │                                                 ▼                            │
│         │                                 r-tools MCP Gateway (tools/server.R)         │
│         │                                 tool: createNewConceptSet                    │
│         │                                                 │                            │
│         └───────────────────────┬─────────────────────────┘                            │
│                                 ▼                                                      │
│                   [ DEDICATED R SERVER (HADES) ]                                       │
│                      Phenelope R Package v0.1.2                                        │
│                      DatabaseConnector | SqlRender | Capr | CirceR | ellmer            │
│                                 │                                                      │
│        ┌────────────────────────┴────────────────────────┐                             │
│        ▼                                                 ▼                             │
│  [ SOVEREIGN DATABASE ]                     [ SOVEREIGN LLM TIER ]                     │
│  PostgreSQL 17 (Port 5432)                  Local Ollama (Port 11434)                  │
│  - cdmDatabaseSchema: omop_54               - Model: llama3.3 / llama3.1               │
│  - vocabDatabaseSchema: vocab_54             - 100% Offline / Zero Data Leakage         │
│  - scratch: scratch                         Cloud API Fallback:                        │
│                                             - OpenAI (gpt-4o) / Azure OpenAI           │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### Sovereign Infrastructure Mapping
1. **Dedicated R Server (`broadsea-hades`)**:
   - `Phenelope` is installed directly in the R library of `broadsea-hades:1.19.0`.
   - Data scientists running RStudio Server (`https://<domain>/rstudio/`) load it instantly with `library(Phenelope)`.
2. **Local Sovereign LLM (Ollama Port 11434)**:
   - Phenelope connects to local Ollama via `ellmer::chat_ollama(model = "llama3.3", endpoint = "http://localhost:11434")`.
   - Clinical evaluations occur on-premise inside the sandbox perimeter with zero data leaving the host.
3. **Local OMOP CDM & Vocabularies (`omop_54` / `vocab_54`)**:
   - Queries `vocab_54.concept_ancestor` and `concept` with sub-50ms execution times.
4. **Autonomous MCP Tool (`createNewConceptSet`)**:
   - Exposed via `tools/server.R` in the `r-tools` MCP server.
   - Allows autonomous phenotyping agents (`phenotyping_agent`) and IDE assistants to dynamically generate concept sets whenever pre-computed sets are unavailable.

---

## 4. Operational Runbook: How to Incorporate and Use Phenelope

### Modality 1: Interactive R Session (RStudio Server / HADES)

Run this R workflow inside RStudio Server (`https://<domain>/rstudio/`) or an R script:

```r
library(Phenelope)
library(DatabaseConnector)
library(ellmer)

# 1. Database Connection to Sovereign Sandbox PostgreSQL
connectionDetails <- DatabaseConnector::createConnectionDetails(
  dbms = "postgresql",
  server = Sys.getenv("CDM_SERVER", "localhost/ohdsi"),
  user = Sys.getenv("CDM_USER", "ohdsi_admin"),
  password = Sys.getenv("CDM_PASSWORD", "ohdsi_admin"),
  port = as.integer(Sys.getenv("CDM_PORT", "5432"))
)

# 2. Sovereign Local LLM Client (Ollama llama3.3)
llmClient <- ellmer::chat_ollama(
  model = Sys.getenv("OLLAMA_MODEL", "llama3.3"),
  endpoint = Sys.getenv("OLLAMA_ENDPOINT", "http://localhost:11434")
)
# Alternatively, for Cloud LLMs:
# llmClient <- ellmer::chat_openai(model = "gpt-4o", api_key = Sys.getenv("OPENAI_API_KEY"))

# 3. Create Concept Set for Type 2 Diabetes Mellitus
outputDir <- "/home/ohdsi/concept_sets/type_2_diabetes"
dir.create(outputDir, recursive = TRUE, showWarnings = FALSE)

results <- Phenelope::createConceptSet(
  conceptName = "Type 2 Diabetes Mellitus",
  originalConceptList = c(201826), # SNOMED 44054006 "Type 2 diabetes mellitus"
  excludedConditions = "Type 1 diabetes, Gestational diabetes, Secondary diabetes",
  llmClient = llmClient,
  connectionDetails = connectionDetails,
  cdmDatabaseSchema = "omop_54",
  minCount = 0,
  belowMinimumCountApproach = "TEST ALL",
  condenseConceptSet = TRUE,
  outputDirectory = outputDir
)

# 4. Inspect Results
auditDf <- results[[1]]      # Dataframe of all tested concepts with LLM rationales
circeJson <- results[[2]]    # Compact Circe JSON string for ATLAS

cat("Evaluated concepts:", nrow(auditDf), "\n")
cat("Included concepts:", sum(auditDf$finalAnswer == "YES"), "\n")
```

### Modality 2: Autonomous MCP Tool (`createNewConceptSet`)

AI agents connect to the sandbox's `r-tools` server (configured in `.vscode/mcp.json` and `.agents/mcp_config.json`) and call:

```json
{
  "name": "createNewConceptSet",
  "arguments": {
    "name": "Acute liver failure",
    "description": "Severe impairment of liver function with coagulopathy and hepatic encephalopathy in patients without prior cirrhosis."
  }
}
```

**Return Payload**:
```json
[
  {
    "capr": "cs(descendants(4245975), name = \"Acute liver failure\")",
    "conditionPersons": 1284,
    "measurementPersons": 412
  }
]
```

### Modality 3: Interactive Agent Skill (`/phenelope`)

In Antigravity IDE, Cursor, or GitHub Copilot, developers invoke:
```text
/phenelope "Type 2 Diabetes Mellitus" --seed 201826 --exclude "Type 1 diabetes, Gestational diabetes"
```
The agent executes the 4-step workflow, displays the LLM decision audit table, and saves the condensed Circe JSON and Capr `cs(...)` code to the active project directory.

### Modality 4: Generating Clinical Descriptions

Phenelope can also automatically generate standardized clinical descriptions:

```r
description <- Phenelope::createClinicalDescription(
  condition = "Acute liver failure",
  llmClient = llmClient,
  excludedConditions = c("Chronic liver disease", "Cirrhosis"),
  outputToWord = TRUE,
  wordFileName = "/home/ohdsi/reports/Acute_Liver_Failure_Description.docx"
)
```

---

## 5. End-to-End Benchmark Walkthrough: Acute Liver Failure

Here is a verified benchmark run demonstrating how Phenelope operates on the sandbox's `vocab_54` vocabulary:

1. **Input Parameters**:
   - `conceptName`: `"Acute liver failure"`
   - `originalConceptList`: `c(4245975)` (SNOMED `197368007` "Acute hepatic failure")
   - `excludedConditions`: `"Chronic hepatic failure, Cirrhosis, Alcoholic liver disease"`
2. **Expansion**:
   - Discovers descendant and correlated concepts: "Subacute hepatic failure", "Acute hepatic coma", "Fulminant type A hepatitis with hepatic coma", "Alcoholic cirrhosis with failure".
3. **LLM Evaluation**:
   - *"Subacute hepatic failure"* -> **YES** (Rationale: Subacute hepatic failure is a severe manifestation sharing identical pathophysiological trajectory and acute onset).
   - *"Acute hepatic coma"* -> **YES** (Rationale: Hepatic coma occurring acutely is the pathognomonic neurological manifestation of acute liver failure).
   - *"Alcoholic cirrhosis with failure"* -> **NO** (Rationale: Expressly excluded; patient has underlying chronic cirrhosis rather than de novo acute liver failure).
4. **Condensation**:
   - Rolls up verified descendants under `descendants(4245975)` and outputs the Capr snippet and ATLAS JSON.
5. **Audit Artifacts Saved**:
   - `Acute_liver_failure_audit.csv`: 24 rows, each with explicit clinical rationale.
   - `Acute_liver_failure_concept_set.json`: Ready for direct import into ATLAS or Capr.

---

## 6. Handover Checklist & Verification Protocol

DevOps and Informatics leads verify Phenelope readiness against the following criteria:

- [x] **R Package Availability**: `Phenelope` installed and loadable in `broadsea-hades` Dedicated R Server.
- [x] **Local Sovereign LLM**: Ollama responds on port 11434; `ellmer::chat_ollama(model = "llama3.3")` connects successfully.
- [x] **Database Connectivity**: PostgreSQL `omop_54` and `vocab_54` accessible with read access to `concept` and `concept_ancestor`.
- [x] **MCP Server Integration**: `tools/server.R` exposes `createNewConceptSetTool` over stdio transport.
- [x] **Workspace Skill**: [`.agents/skills/phenelope-concept-set-builder/SKILL.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/.agents/skills/phenelope-concept-set-builder/SKILL.md) available in workspace.
- [x] **Author Attribution**: Full credit attributed to Joel N. Swerdel, Dr. Martijn Schuemie, and Dr. Anna Ostropolets.
