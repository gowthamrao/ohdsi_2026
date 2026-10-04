# Architecture Decision Record (ADR): Dedicated R Server Architecture

> **Status**: APPROVED & IMPLEMENTED  
> **Target Audience**: DevOps Engineers, Site Reliability Engineers (SRE), Cloud Architects, Bioinformaticians  
> **Deciders**: OHDSI Platform Engineering Team  
> **Supersedes**: Fragmented Plumber Microservices Fleet  

---

## 1. Context & The "Microservice Anti-Pattern" in R Analytics

In observational health data platforms operating on the OMOP Common Data Model (OMOP CDM v5.4), researchers leverage dozens of specialized OHDSI R libraries:
- **Cohort Definition & Compiling**: `Capr`, `CirceR`, `CohortGenerator`
- **Validation & Diagnostics**: `CohortDiagnostics`, `CohortIncidence`
- **Feature Extraction & Phenotyping**: `FeatureExtraction`, `ClinicalCharacteristics`, `PheValuator`, `PhenotypeLibrary`
- **Causal Estimation & Prediction**: `CohortMethod`, `SelfControlledCaseSeries`, `PatientLevelPrediction`, `Strategus`
- **Association Mining & Adjudication**: `Taxis`, `Keeper`, `ResultModelManager`

### The Anti-Pattern: Decoupled Plumber Microservice Containers
In earlier designs, attempts were made to wrap every single R package into an individual HTTP microservice:
- Each package had its own Plumber container, polling worker, and separate port.
- This created **over 50 individual containers** (e.g. `capr-api`, `cohort-diagnostics-api`, etc.).

### Why This Failed Operationally:
1. **Severe Memory Bloat**: Each R Plumber container consumes 400MB–1.5GB of RAM idle. Fifty containers consumed **25–35 GB of RAM without executing a single study query**.
2. **Network Fragility & Port Proliferation**: Managing 50+ internal ports (8080–8120), cross-container DNS overhead, and complex Nginx upstream routing tables created frequent cascading connection failures.
3. **Serialization Latency**: R analytical workflows process millions of OMOP rows. Streaming large patient/event data frames as JSON payloads over HTTP introduces massive CPU serialization overhead compared to in-memory `DBI` / `Andromeda` operations.
4. **Poor Debuggability**: Plumber wrappers hid native R stack traces behind opaque HTTP 500 error payloads, making debugging syntax errors in Capr or CohortMethod exceedingly difficult.

---

## 2. The Decision: Centralize in a Dedicated R Server

### Core Architectural Decisions:
1. **R runs inside a Single Dedicated R Server** (`broadsea-hades` / RStudio Server on internal port 8787).
2. **All OHDSI R packages are installed directly** into this R Server's unified library.
3. **Direct CDM and WebAPI Connectivity**: The Dedicated R Server is colocated on the internal network with direct JDBC access to the PostgreSQL OMOP CDM database and REST connectivity to WebAPI.
4. **Default Startup (`docker compose up -d`) boots ZERO Plumber microservices**, reducing idle memory footprint from **30 GB** to **~2–4 GB**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 Nginx Edge Ingress Gateway (Port 443 / 80)                  │
│                      https://research.yourdomain.org                        │
└──────────────┬───────────────────────────────┬──────────────────────────────┘
               │                               │
               ▼                               ▼
┌──────────────────────────────┐┌─────────────────────────────────────────────┐
│  Atlas 3.0 / WebAPI Backend  ││     Dedicated R Server (`broadsea-hades`)   │
│  - Atlas UI (Vue 3 / Classic)││   Port: 8787 | Ingress Path: `/rstudio/`    │
│  - WebAPI REST (Port 8080)   ││                                             │
│  - Connects to CDM Schemas   ││  ┌───────────────────────────────────────┐  │
└──────────────┬───────────────┘│  │ Pre-Installed OHDSI R Package Suite   │  │
               │                │  │  - DatabaseConnector & SqlRender      │  │
               │ REST           │  │  - Capr & CirceR                      │  │
               │ (ROhdsiWebApi) │  │  - CohortDiagnostics & CohortIncidence│  │
               ▼                │  │  - FeatureExtraction & PheValuator    │  │
               └───────────────>│  │  - HADES (CohortMethod, SCCS, PLP)    │  │
                                │  └───────────────────────────────────────┘  │
                                └──────────────────────┬──────────────────────┘
                                                       │ JDBC (DatabaseConnector)
                                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│               PostgreSQL 16 High-Throughput CDM Database Cluster             │
│   - Port 5432 (Internal Docker Network)                                     │
│   - Tables: person, visit_occurrence, condition_occurrence, concept, etc.   │
│   - Schemas: cdm_synthea100k (data), vocab_54 (dictionary), results (cache) │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Supported OHDSI & HADES R Package Catalog

The Dedicated R Server maintains pre-installed support for all official HADES packages and platform analytical packages across 6 functional domains:

### 1. Database Connectivity & SQL Transpilation (HADES Core)
- **`DatabaseConnector`**: High-performance JDBC database connectivity abstraction supporting PostgreSQL, Snowflake, BigQuery, Redshift, Spark, and Oracle.
- **`SqlRender`**: Transpiler that translates OHDSI parameterized SQL into target DBMS dialects.
- **`ParallelLogger`**: Multi-threaded parallel processing and structured file/console logging.
- **`Andromeda`**: High-volume in-memory and disk-backed data frame storage for billion-row patient matrices.

### 2. Cohort Definition, Phenotyping & Ingestion
- **`Capr`**: Programmatic Domain-Specific Language (DSL) for authoring reproducible cohort definitions in R.
- **`CirceR`**: Compiles Circe JSON cohort expressions into executable SQL across target dialects.
- **`CohortGenerator`**: Instantiates cohorts and generates cohort table structures within OMOP CDM.
- **`CohortConstructor`**: Fast cohort transformation and logic builder.
- **`PhenotypeLibrary`**: Programmatic access to the official OHDSI phenotype repository.
- **`Phenotyper`**: Algorithmic phenotype extraction and cohort matching.
- **`Phenelope`**: Probabilistic phenotype evaluation and vocabulary reasoning.
- **`PheValuator`**: Semi-supervised machine learning evaluation of phenotype algorithms (calculates PPV, sensitivity, and specificity).
- **`Keeper`**: De-identified clinical timeline extraction and dual-hybrid adjudication.
- **`ProtocolGenerator`**: Computable study protocol and analysis specification authoring.

### 3. Characterization, Incidence & Data Diagnostics
- **`CohortDiagnostics`**: Comprehensive cohort characterization, index event breakdown, overlap, and attrition diagnostics.
- **`CohortIncidence`**: Population-level incidence rate calculations across age, gender, and calendar time strata.
- **`FeatureExtraction`**: Extracts baseline covariate matrices (demographics, conditions, drugs, measurements) for models.
- **`Characterization`**: Advanced demographic and clinical characteristic summarization.
- **`ClinicalCharacteristics`**: Table shell generation and clinical baseline summary calculations.
- **`DbDiagnostics`**: 24-point database feasibility evaluation against study requirements.
- **`DataQualityDashboard` (DQD)**: Systematic evaluation of OMOP CDM data quality (Conformance, Completeness, Plausibility).

### 4. Population-Level Estimation & Causal Inference (HADES Core)
- **`CohortMethod`**: New-user active comparator cohort designs using propensity score matching/stratification and Cox regression.
- **`SelfControlledCaseSeries` (SCCS)**: Within-person causal inference controlling for time-invariant confounders.
- **`Cyclops`**: High-performance cyclic coordinate descent optimization for large-scale L1/L2 regularized regression.
- **`EvidenceSynthesis`**: Fixed-effects and empirical Bayes meta-analysis across multi-site data networks.
- **`EmpiricalCalibration`**: Synthetic negative control calibration to correct for observational confounding and systematic error.
- **`MethodEvaluation`**: Empirical evaluation of epidemiological study designs.
- **`CaseControl`**: Retrospective matched case-control designs.
- **`CaseCrossover`**: Case-crossover designs for transient clinical exposures.

### 5. Patient-Level Prediction & Machine Learning (HADES Core)
- **`PatientLevelPrediction` (PLP)**: Standardized framework for training, validating, and evaluating clinical prediction models (LASSO, Random Forest, XGBoost).
- **`DeepPatientLevelPrediction`**: Deep learning models applied to longitudinal OMOP patient timelines.
- **`BigKnn`**: High-throughput K-nearest neighbors classifier for OMOP data.

### 6. Study Execution, Storage & Visualization
- **`Strategus`**: Modular orchestrator executing multi-analysis study pipelines across network sites.
- **`ResultModelManager`**: Schema manager and DDL deployer for the HADES Results Data Model.
- **`ROhdsiWebApi`**: REST client for interacting with OHDSI WebAPI (importing/exporting cohorts, concept sets, and initiating executions).
- **`OhdsiShinyModules`**: Reusable Shiny visualization components for interactive exploration of HADES results.
- **`ShinyAppBuilder`**: Framework for compiling HADES results into interactive Shiny dashboards.
- **`Eunomia`**: Embedded synthetic OMOP CDM dataset for testing and verification.
- **`Taxis`**: High-throughput Concept AB temporal association mining engine.

---

## 4. Operational Comparison

| Metric | Dedicated R Server (Current Standard) | Fragmented Microservices (Legacy Anti-Pattern) |
| :--- | :--- | :--- |
| **Active Containers** | **1 container** (`broadsea-hades`) | 50+ containers |
| **Idle RAM Consumption** | **~2 GB** | 25–35 GB |
| **Port Exposure** | Single port (`8787`) | 50+ ports (`8080`–`8120`) |
| **Startup Time** | **< 15 seconds** | 3–5 minutes (Docker compose storm) |
| **Data Frame Handling** | Native in-memory (`Andromeda`, `arrow`) | Multi-hop JSON HTTP serialization |
| **Database Connection** | Direct JDBC connection pooling | Re-connected on every HTTP request |
| **AI Agent Integration** | Official StudyAgent FastMCP direct tools | Brittle multi-hop HTTP REST calls |

---

## 5. R Server Interconnectivity: Connecting to CDM & WebAPI

The Dedicated R Server has direct connectivity to both the **OMOP CDM database** and the **WebAPI backend**:

### A. Environment Configuration
The container inherits the following environment variables:
```bash
# OMOP CDM Database Parameters
CDM_SERVER=ohdsi-postgres
CDM_PORT=5432
CDM_DATABASE=ohdsi
CDM_SCHEMA=cdm_synthea100k
VOCAB_SCHEMA=vocab_54
RESULTS_SCHEMA=results
CDM_USER=ohdsi_app_user
CDM_PASSWORD=ohdsi_app_secure_2026
DATABASECONNECTOR_JAR_FOLDER=/opt/drivers

# WebAPI & Atlas Parameters
WEBAPI_URL=http://webapi-classic:8080/WebAPI
ATLAS_URL=http://atlas-classic:80
```

### B. Analytical Connection Script (R Verification Pattern)
Researchers and automated pipelines connect directly using standard OHDSI R libraries:

```r
library(DatabaseConnector)
library(SqlRender)
library(ROhdsiWebApi)

# 1. Establish Direct JDBC Connection to OMOP CDM
connectionDetails <- createConnectionDetails(
  dbms = "postgresql",
  server = paste0(Sys.getenv("CDM_SERVER", "ohdsi-postgres"), ":", 
                  Sys.getenv("CDM_PORT", "5432"), "/", 
                  Sys.getenv("CDM_DATABASE", "ohdsi")),
  user = Sys.getenv("CDM_USER", "ohdsi_app_user"),
  password = Sys.getenv("CDM_PASSWORD"),
  pathToDriver = Sys.getenv("DATABASECONNECTOR_JAR_FOLDER", "/opt/drivers")
)

conn <- connect(connectionDetails)

# Query CDM person count
cdmSchema <- Sys.getenv("CDM_SCHEMA", "cdm_synthea100k")
personCount <- querySql(conn, paste0("SELECT COUNT(*) AS total_patients FROM ", cdmSchema, ".person;"))
print(paste("Total patients in CDM:", personCount$TOTAL_PATIENTS))

disconnect(conn)

# 2. Establish REST Connection to WebAPI
webApiUrl <- Sys.getenv("WEBAPI_URL", "http://webapi-classic:8080/WebAPI")
webApiInfo <- getWebApiVersion(baseUrl = webApiUrl)
print(paste("Connected to WebAPI version:", webApiInfo))

# Query available CDM data sources in WebAPI
sources <- getCdmSources(baseUrl = webApiUrl)
print(sources[, c("sourceId", "sourceName", "sourceKey")])
```

---

## 6. Execution Channels for the Dedicated R Server

### Channel 1: Interactive RStudio Server Web Interface
- **Local Access**: `http://localhost:8787`
- **Public Domain Access**: `https://research.yourdomain.org/rstudio/`
- **Default Credentials**: User `ohdsi` / Password `ohdsi2026` (configurable via `HADES_PASSWORD` in `.env`).
- **Use Case**: Interactive cohort design in Capr, exploratory data analysis, reviewing diagnostic plots in RStudio graphics viewer.

### Channel 2: Headless Docker Execution
DevOps teams execute analytical scripts in batch mode via standard container execution:
```bash
# Execute an R script headlessly inside the R Server container:
docker exec -it broadsea-hades Rscript -e "source('/path/to/study_pipeline.R')"

# Check installed package versions:
docker exec -it broadsea-hades Rscript -e "installed.packages()[, c('Package', 'Version')]"
```

### Channel 3: Agentic AI Tool Calling via StudyAgent FastMCP
External AI models (Claude, Cursor, Antigravity) connect to the official [`OHDSI/StudyAgent`](https://github.com/OHDSI/StudyAgent) FastMCP server on port 8790, which executes approved analytical queries against the Dedicated R Server and CDM database with enforced small-cell suppression (`MIN_CELL_COUNT >= 5`).

---

## 7. Verification & Acceptance Criteria

1. **R Server Status**: `docker inspect --format='{{.State.Status}}' broadsea-hades` returns `running`.
2. **CDM Reachability**: R Server resolves `ohdsi-postgres:5432` and completes a test query against `cdm.person`.
3. **WebAPI Reachability**: R Server resolves `http://webapi-classic:8080/WebAPI/info` and retrieves HTTP 200 with version payload.
4. **Package Suite Availability**: Executing `sapply(c('DatabaseConnector', 'Capr', 'CohortMethod', 'PatientLevelPrediction', 'Strategus', 'ROhdsiWebApi'), requireNamespace)` returns all `TRUE`.
5. **Memory Utilization**: Baseline idle memory of the R Server remains under **2.5 GB RAM**.
