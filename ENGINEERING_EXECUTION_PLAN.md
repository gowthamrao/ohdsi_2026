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
│ Gate 4  │ Public Ingress & FastMCP AI  │ Nginx TLS 1.3, Let's Encrypt, StudyAgent  │ MCP suite 100%    │
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

### Stage Gate 2: High-Capacity Data, Dedicated R Server & CDM Interconnect
1. Launch PostgreSQL 16 tuned for 64GB+ shared buffers, Redis, and Solr:
   ```bash
   docker compose up -d ohdsi-postgres ohdsi-redis broadsea-solr-vocab broadsea-hades
   ```
2. Ingest Athena vocabularies and build trigram GIN indexes.
3. Verify R Server interconnectivity with the shared CDM database and WebAPI:
   ```bash
   # Verify JDBC connection to CDM:
   docker exec -it broadsea-hades Rscript -e "
     library(DatabaseConnector)
     conn <- connect(createConnectionDetails(
       dbms = 'postgresql',
       server = paste0(Sys.getenv('CDM_SERVER', 'ohdsi-postgres'), '/', Sys.getenv('CDM_DATABASE', 'ohdsi')),
       user = Sys.getenv('CDM_USER', 'ohdsi_app_user'),
       password = Sys.getenv('CDM_PASSWORD'),
       pathToDriver = Sys.getenv('DATABASECONNECTOR_JAR_FOLDER', '/opt/drivers')
     ))
     res <- querySql(conn, paste0('SELECT COUNT(*) FROM ', Sys.getenv('CDM_SCHEMA', 'cdm'), '.person'))
     print(res)
     disconnect(conn)
   "

   # Verify REST connection to WebAPI:
   docker exec -it broadsea-hades Rscript -e "
     library(ROhdsiWebApi)
     print(getWebApiVersion(baseUrl = Sys.getenv('WEBAPI_URL', 'http://webapi-classic:8080/WebAPI')))
   "
   ```

### Stage Gate 3: Atlas 3.0 Next-Gen Frontend & WebAPI 3.0
1. Deploy modern Atlas 3.0 single-spa micro-frontend and WebAPI 3.0:
   ```bash
   docker compose up -d atlas3-webapi atlas3-frontend atlas3-db-init
   ```
2. Verify Atlas 3.0 loads at `http://localhost:3000` or via unified path `/`.

### Stage Gate 4: Public Domain TLS Ingress & Hosted FastMCP AI Gateway
1. Deploy Nginx reverse proxy with TLS 1.3 termination and Let's Encrypt certificate renewal.
2. Launch the official StudyAgent MCP container and local Ollama inference:
   ```bash
   docker compose up -d study-agent-mcp study-agent-acp ollama-service
   ```
3. Test MCP agent connectivity:
   ```bash
   curl -s -H "Authorization: Bearer <token>" https://<domain>/mcp/sse
   ```

---

## 3. Production Operations & Maintenance

- **Sign-Off Audit**: Verify against the checklists in [`STAGE_GATED_SPECIFICATIONS.md`](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md).
- **R Package Ecosystem**: Analytical packages run directly inside `broadsea-hades` without Plumber microservice fragmentation.
- **External Network Studies**: Clone study packages such as [`ohdsi-studies/Taxis`](https://github.com/ohdsi-studies/Taxis) directly into the Dedicated R Server.
- **Automated Certificate Renewal**: Certbot companion container checks and renews Let's Encrypt certificates every 12 hours.
