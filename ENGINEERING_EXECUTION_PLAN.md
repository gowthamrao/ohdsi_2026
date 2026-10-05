# OHDSI Deployment: Engineering Execution Plan

> **Target Audience**: Lead Systems Engineers, DevOps Teams, Cloud Infrastructure Administrators  
> **Repository**: Pure Requirements & Acceptance Criteria Specification  
> **Status**: Approved Production Runbook

---

## 1. Overview & 5-Stage Milestone Progression

This runbook guides DevOps engineers step-by-step from bare-metal host provisioning to a production-ready OHDSI platform with Atlas 3.0, Dedicated R Server, and hosted FastMCP endpoints for AI agents.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           STAGE-GATED ROADMAP                                          │
├─────────┬──────────────────────────────┬───────────────────────────────────────────┬───────────────────┤
│ Stage   │ Milestone Focus              │ Core Technologies                         │ Exit Verification │
├─────────┼──────────────────────────────┼───────────────────────────────────────────┼───────────────────┤
│ Gate 0  │ Host & Kernel Hardening      │ Ubuntu 24.04 LTS, NVMe noatime, UFW       │ Ports closed      │
│ Gate 1  │ Turnkey Broadsea Core        │ broadsea-atlasdb, webapi, atlas, hades    │ WebAPI /info UP   │
│ Gate 2  │ High-Capacity Data & Vocab   │ Postgres 16 (64GB shared_buffers), Redis  │ Vocab query <150ms│
│ Gate 3  │ Atlas 3.0 & WebAPI 3.0       │ Atlas 3.0 (Vue 3), WebAPI 3.0, R Server   │ Atlas 3.0 live    │
│ Gate 4  │ MCP AI & Federated Network   │ StudyAgent, WebApiMcp, Arachne, TLS 1.3   │ Gate 4 suite 100% │
└─────────┴──────────────────────────────┴───────────────────────────────────────────┴───────────────────┘
```

---

## 2. Step-by-Step Execution Runbook

### Stage Gate 0: Host OS Hardening & NVMe Mount
1. Provision Ubuntu 24.04 LTS on bare metal or cloud VM.
2. Mount NVMe data partition with `noatime,nodiratime` at `/var/lib/postgresql/data`:
   ```bash
   mount -o noatime,nodiratime /dev/nvme0n1p3 /var/lib/postgresql/data
   ```
3. Apply kernel sysctl parameters (`/etc/sysctl.d/99-ohdsi.conf`):
   ```ini
   vm.swappiness = 10
   vm.dirty_ratio = 15
   net.core.somaxconn = 65535
   ```
4. Verify Gate 0:
   ```bash
   sysctl vm.swappiness net.core.somaxconn
   mount | grep postgresql
   ```

### Stage Gate 1: Turnkey Broadsea Core
1. Configure environment file (`.env.broadsea`).
2. Start the Broadsea core stack:
   ```bash
   docker compose -f docker-compose.broadsea.yml up -d
   ```
3. Verify Gate 1:
   ```bash
   curl -s http://localhost:8080/WebAPI/info | jq .
   curl -s http://localhost:8082
   curl -s http://localhost:8983/solr/admin/info/system
   ```

### Stage Gate 2: High-Capacity Data, Dedicated R Server, Atlas & Shiny Deployment
1. Launch PostgreSQL 16 tuned for 64GB+ shared buffers, Redis, Solr, R Server, and Shiny Server:
   ```bash
   docker compose up -d ohdsi-postgres ohdsi-redis broadsea-solr-vocab broadsea-hades ohdsi-shiny
   ```
2. Ingest Athena vocabularies and build trigram GIN indexes.
3. Verify R Server interconnectivity with OMOP CDM and Vocabulary tables:
   ```bash
   # Verify JDBC connection to OMOP CDM and Athena Vocabularies:
   docker exec -it broadsea-hades Rscript -e "
     library(DatabaseConnector)
     conn <- connect(createConnectionDetails(
       dbms = 'postgresql',
       server = paste0(Sys.getenv('CDM_SERVER', 'ohdsi-postgres'), '/', Sys.getenv('CDM_DATABASE', 'ohdsi')),
       user = Sys.getenv('CDM_USER', 'ohdsi_app_user'),
       password = Sys.getenv('CDM_PASSWORD'),
       pathToDriver = Sys.getenv('DATABASECONNECTOR_JAR_FOLDER', '/opt/drivers')
     ))
     patients <- querySql(conn, paste0('SELECT COUNT(*) FROM ', Sys.getenv('CDM_SCHEMA', 'cdm'), '.person'))
     concepts <- querySql(conn, paste0('SELECT COUNT(*) FROM ', Sys.getenv('VOCAB_SCHEMA', 'vocab_54'), '.concept'))
     print(paste('Patients:', patients[1,1], '| Concepts:', concepts[1,1]))
     disconnect(conn)
   "
   ```
4. Verify PostgreSQL connection to Atlas Instance via WebAPI:
   ```bash
   curl -s http://localhost:8080/WebAPI/source/sources | jq .
   curl -s http://localhost:8080/WebAPI/vocabulary/vocab_54/search/aspirin | jq .
   ```
5. Deploy and verify OHDSI Study Shiny Apps and Reports:
   ```bash
   # Verify Shiny Server is operational:
   curl -s http://localhost:3838/
   # Publish study apps (e.g. CohortDiagnostics, Taxis) to /srv/shiny-server/<study_name>/
   # Publish study reports to /srv/reports/<study_name>/
   ```

### Stage Gate 3: Atlas 3.0 Next-Gen Frontend, WebAPI 3.0 & Public URL Ingress
1. Deploy modern Atlas 3.0 single-spa micro-frontend and WebAPI 3.0:
   ```bash
   docker compose up -d atlas3-webapi atlas3-frontend atlas3-db-init reverse-proxy certbot
   ```
2. Verify all platform components over the single **Public Domain URL**:
   ```bash
   # 1. Root: Atlas 3.0 Frontend
   curl -I -k https://<domain>/

   # 2. Atlas Classic Frontend
   curl -I -k https://<domain>/atlas/

   # 3. WebAPI Backend REST API
   curl -s -k https://<domain>/WebAPI/info | jq .

   # 4. Dedicated R Server (RStudio Server Web IDE)
   curl -I -k https://<domain>/rstudio/

   # 5. OHDSI Study Shiny Apps
   curl -I -k https://<domain>/shiny/

   # 6. OHDSI Study Analytical HTML Reports
   curl -I -k https://<domain>/reports/
   ```

### Stage Gate 4: Sovereign Agentic Tier (Hosted FastMCP & BYO-Agent) & Federated Network
1. Launch the official StudyAgent MCP container, WebApiMcp bridge server, Arachne node, and local Ollama inference:
   ```bash
   docker compose up -d study-agent-mcp webapi-mcp arachne-data-node arachne-exec-engine ollama-service
   ```
2. Test MCP agent connectivity and tool discovery:
   ```bash
   # Test StudyAgent MCP endpoint
   curl -s -k https://<domain>/mcp/sse -H "Authorization: Bearer <token>"

   # Test WebApiMcp bridge server health and tool discovery
   curl -s -k https://<domain>/webapi-mcp/health | jq .
   ```
3. Test Arachne Federated Data Node status:
   ```bash
   curl -s -k https://<domain>/arachne/api/v1/build-number | jq .
   ```

---

## 3. Production Operations & Maintenance

- **Sign-Off Audit**: Verify against the checklists in [`STAGE_GATED_SPECIFICATIONS.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md).
- **R Package Ecosystem**: Analytical packages run directly inside `broadsea-hades` without Plumber microservice fragmentation.
- **External Network Studies**: Clone study packages such as [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis) directly into the Dedicated R Server.
- **Automated Certificate Renewal**: Certbot companion container checks and renews Let's Encrypt certificates every 12 hours.
