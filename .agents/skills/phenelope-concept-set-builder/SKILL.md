---
name: phenelope-concept-set-builder
description: >
  Given a clinical health condition and seed concept(s), build a verified, audit-logged OMOP concept set using Phenelope (OHDSI/Phenelope) with LLM evaluation, PHOEBE recommender expansion, and automated Circe JSON condensation.
---

# Phenelope: LLM-Based OMOP Concept Set Builder

> **Attribution**: Authored by **Joel N. Swerdel** (Janssen R&D / OHDSI), with major contributions from **Dr. Martijn Schuemie** (Janssen R&D / OHDSI) and **Dr. Anna Ostropolets** (Columbia University / OHDSI). Upstream repository: [`OHDSI/Phenelope`](https://github.com/OHDSI/Phenelope). Lifted and operationalized for the **OHDSI Sandbox 2026**.

The goal of this skill is to generate a rigorous, condensed, and clinically validated OMOP concept set using an LLM to evaluate candidate concepts discovered through Athena hierarchy and PHOEBE co-occurrence recommendations.

The output consists of:
1. An **ATLAS-compatible Circe JSON** concept set definition file.
2. An inlined **Capr `cs(...)` expression** for direct use in cohort definitions.
3. A **CSV audit ledger** detailing every evaluated concept, the LLM's YES/NO decision, and the explicit clinical rationale.

---

## Prerequisites

- **Clinical Condition Name**: e.g., `"Type 2 Diabetes Mellitus"`, `"Acute liver failure"`.
- **Seed Concept ID(s)**: An appropriate SNOMED standard concept ID representing the core condition (e.g., `201826` for Type 2 Diabetes, `4245975` for Acute Hepatic Failure). If unknown, use the `ohdsi_webapi_mcp` tool `search_concepts` or Athena vocabulary lookup.
- **Clinical Context & Exclusions**:
  - Optional explicit exclusions: e.g., `"Type 1 diabetes, Gestational diabetes, Secondary diabetes"`.
  - Optional clinical context: e.g., `"in adults without pre-existing chronic liver disease"`.
- **LLM Connection**: Local sovereign Ollama on port `11434` (`model = "llama3.3"`) or configured cloud provider (OpenAI `gpt-4o`, Azure OpenAI, Anthropic).
- **Database Connection**: OHDSI Sandbox PostgreSQL (`omop_54` / `vocab_54`).

---

## 4-Step Operational Workflow

### Step 1: Clarify Clinical Intent & Seed Concepts
1. Confirm the target condition name and obtain the primary seed concept ID.
   - If starting from an umbrella concept or high-level phenotype, consult the `concept-set-target-enumerator` or `phenotype-parent-concept` skill.
2. Identify conditions that should be explicitly excluded from the concept set (e.g. secondary manifestations, acute vs chronic distinctions).

### Step 2: Configure & Execute Phenelope

Execute `Phenelope::createConceptSet()` in the Dedicated R Server (`broadsea-hades`) or via the `createNewConceptSet` MCP tool:

```r
library(Phenelope)
library(DatabaseConnector)
library(ellmer)

# 1. Establish database connection to sandbox PostgreSQL
connectionDetails <- DatabaseConnector::createConnectionDetails(
  dbms = "postgresql",
  server = Sys.getenv("CDM_SERVER", "localhost/ohdsi"),
  user = Sys.getenv("CDM_USER", "ohdsi_admin"),
  password = Sys.getenv("CDM_PASSWORD", "ohdsi_admin"),
  port = as.integer(Sys.getenv("CDM_PORT", "5432"))
)

# 2. Configure LLM Client (Local Sovereign Ollama or Cloud API)
llmClient <- tryCatch({
  ellmer::chat_ollama(
    model = Sys.getenv("OLLAMA_MODEL", "llama3.3"),
    endpoint = Sys.getenv("OLLAMA_ENDPOINT", "http://localhost:11434")
  )
}, error = function(e) {
  ellmer::chat_openai(
    model = "gpt-4o",
    api_key = Sys.getenv("OPENAI_API_KEY")
  )
})

# 3. Execute concept set building pipeline
outputDir <- file.path("/home/ohdsi/concept_sets", gsub("[^[:alnum:]]", "_", tolower(conceptName)))
dir.create(outputDir, recursive = TRUE, showWarnings = FALSE)

conceptSetResults <- Phenelope::createConceptSet(
  conceptName = "Acute liver failure",
  originalConceptList = c(4245975),
  excludedConditions = "Chronic liver failure, Cirrhosis, Alcoholic liver disease",
  llmClient = llmClient,
  connectionDetails = connectionDetails,
  cdmDatabaseSchema = "omop_54",
  minCount = 0,
  belowMinimumCountApproach = "TEST ALL",
  condenseConceptSet = TRUE,
  outputDirectory = outputDir
)
```

### Step 3: Audit LLM Rationales & Decisions

Review the generated audit CSV in `outputDirectory/`:
- **`suggestedCondition`**: Concept name evaluated.
- **`conceptId`**: OMOP Standard Concept ID.
- **`finalAnswer`**: `YES` (included) or `NO` (excluded).
- **`rationaleForAnswer`**: The pathophysiological and clinical explanation provided by the model.
- **`confidenceLevel`**: Confidence score (0% to 100%).

Verify that:
1. No excluded conditions or off-target manifestations were admitted.
2. Important subtypes and synonyms were preserved.
3. The condensation step successfully merged descendants to keep the concept set concise.

### Step 4: Export & Inlining

1. **Circe JSON**: Save `conceptSetResults[[2]]` directly to disk for import into ATLAS:
   - File: `concept_set_<phenotype>.json`.
2. **Capr R Code**: Generate the inlined `cs(...)` expression using `Capr::cs()`:
   ```r
   acute_liver_failure_cs <- cs(
     descendants(4245975), # Acute hepatic failure
     name = "Acute liver failure"
   )
   ```
3. Hand over the resulting concept set to `cohort-developer` for cohort integration.

---

## Heuristics & Guardrails

1. **Never Invent Concept IDs**: Always use concept IDs verified in Athena (`vocab_54`).
2. **Use Broad Seed Concepts Wisely**: Using a broader parent concept (e.g. SNOMED `201820` "Diabetes mellitus" instead of `201826` "Type 2 diabetes mellitus") expands candidate discovery via PHOEBE but increases LLM evaluation turns.
3. **Always Record the Audit Ledger**: Never deliver an un-audited concept set. The CSV ledger with `rationaleForAnswer` must always be retained for scientific reproducibility.
