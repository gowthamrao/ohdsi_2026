# OHDSI Sandbox Deployment: Engineering Execution Plan

> **Platform Mission**: High-Resilience Developer Sandbox for Innovating, Collaborating & Testing Latest Ideas in Clinical Informatics and Data Science  
> **Target Audience**: Cloud DevOps Engineers, Lead Systems Engineers, SREs, Systems Administrators  
> **Status**: Approved Deployment Runbook & Handover Execution Plan  
> **Cross-References**: [DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md) | [STAGE_GATED_SPECIFICATIONS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/STAGE_GATED_SPECIFICATIONS.md) | [OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/OHDSI_SERVER_ENVIRONMENT_REQUIREMENTS.md) | [PUBLIC_DOMAIN_HOSTING_GUIDE.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/PUBLIC_DOMAIN_HOSTING_GUIDE.md)

---

## 1. Overview & 4-Phase Deployment Progression

This runbook guides DevOps engineers step-by-step through provisioning, hardening, and delivering the OHDSI Sandbox. The deployment progresses through four linear phases:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   DEVOPS SANDBOX DEPLOYMENT PIPELINE                                   │
├─────────┬──────────────────────────────┬───────────────────────────────────────────┬───────────────────┤
│ Phase   │ Focus Area                   │ Primary Components                        │ Exit Verification │
├─────────┼──────────────────────────────┼───────────────────────────────────────────┼───────────────────┤
│ Phase 1 │ Host, CoW Storage & Swap     │ Ubuntu 24.04, ZFS/Btrfs CoW, 64GB Swap    │ CoW mount & sysctl│
├─────────┼──────────────────────────────┼───────────────────────────────────────────┼───────────────────┤
│ Phase 2 │ Core Data Platform           │ Postgres 16 (64GB shared_buffers), Vocab  │ Vocab query <50ms │
├─────────┼──────────────────────────────┼───────────────────────────────────────────┼───────────────────┤
│ Phase 3 │ Compute Tier & Edge Ingress  │ Dedicated R Server, Shiny, Atlas 3.0, Nginx│ TLS 1.3 all paths │
├─────────┼──────────────────────────────┼───────────────────────────────────────────┼───────────────────┤
│ Phase 4 │ Agentic Gateways & Handover  │ FastMCP, WebApiMcp, Arachne, SQL, MinIO   │ Handover test 100%│
└─────────┴──────────────────────────────┴───────────────────────────────────────────┴───────────────────┘
```

---

## 2. Step-by-Step Deployment Runbook

### Phase 1: Host Hardening, CoW Storage & Anti-Panic Swap

1. **Verify Operating System**: Ensure host is running Ubuntu Server 24.04 LTS (x86_64).
2. **Configure Copy-on-Write Storage Datasets (ZFS or Btrfs)**:
   Mount NVMe partitions with CoW capability to enable `< 30-second` developer rollbacks:
   ```bash
   # Create ZFS datasets for PostgreSQL data and R workspace:
   zfs create -o mountpoint=/var/lib/postgresql/data -o compression=lz4 -o atime=off rpool/pgdata
   zfs create -o mountpoint=/home/ohdsi -o compression=lz4 rpool/rstudio-workspace
   ```
3. **Provision 64GB NVMe Swap Partition**:
   Absorb sudden multi-core causal inference memory spikes without kernel panics:
   ```bash
   fallocate -l 64G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
   echo '/swapfile none swap sw 0 0' >> /etc/fstab
   ```
4. **Apply Kernel Tuning (`/etc/sysctl.d/99-ohdsi.conf`)**:
   ```ini
   vm.swappiness = 10
   vm.dirty_ratio = 15
   vm.dirty_background_ratio = 5
   net.core.somaxconn = 65535
   net.ipv4.tcp_max_syn_backlog = 8192
   ```
   Apply with: `sudo sysctl --system`
5. **Configure Host Perimeter Firewall (UFW)**:
   ```bash
   ufw default deny incoming
   ufw default allow outgoing
   ufw allow 22/tcp comment 'SSH'
   ufw allow 80/tcp comment 'HTTP ACME'
   ufw allow 443/tcp comment 'HTTPS TLS 1.3'
   ufw enable
   ```
6. **Exit Verification**:
   ```bash
   sysctl vm.swappiness net.core.somaxconn
   swapon --show
   zfs list
   ufw status verbose
   ```

---

### Phase 2: Core Data Platform Deployment

1. **Launch PostgreSQL 16 & Redis**:
   ```bash
   docker compose up -d ohdsi-postgres ohdsi-redis broadsea-solr-vocab
   ```
2. **Ingest Standardized Athena Vocabularies & Build GIN Trigram Indexes**:
   ```bash
   # Load concept, concept_ancestor, concept_relationship into schema vocab_54
   # Build trigram index for instant autocomplete (< 50ms):
   docker exec -it ohdsi-postgres psql -U ohdsi_admin -d ohdsi -c "
     CREATE EXTENSION IF NOT EXISTS pg_trgm;
     CREATE INDEX IF NOT EXISTS idx_concept_name_trgm ON vocab_54.concept USING gin (concept_name gin_trgm_ops);
   "
   ```
3. **Seed Synthetic CDM Datasets (Synthea 100k / CMS SynPUF 2.3M)**:
   ```bash
   # Restore pre-seeded synthetic benchmark data:
   docker exec -i ohdsi-postgres psql -U ohdsi_admin -d ohdsi < /opt/ohdsi/seeds/synthea100k.sql
   ```
4. **Boot WebAPI Classic & Atlas Classic**:
   ```bash
   docker compose up -d webapi-classic atlas-classic
   ```
5. **Exit Verification**:
   ```bash
   # Verify WebAPI and Vocabulary search speed:
   curl -s http://localhost:8080/WebAPI/info | jq .
   docker exec -it ohdsi-postgres psql -U ohdsi_admin -d ohdsi -c \
     "EXPLAIN ANALYZE SELECT * FROM vocab_54.concept WHERE concept_name ILIKE '%aspirin%' LIMIT 20;"
   ```

---

### Phase 3: Compute Engine & Edge Ingress Routing

1. **Deploy Dedicated R Server (`broadsea-hades`)**:
   ```bash
   docker compose up -d broadsea-hades
   ```
   *Verify R Server connects to both OMOP CDM and Vocabulary schemas via JDBC:*
   ```bash
   docker exec -it broadsea-hades Rscript -e "
     library(DatabaseConnector)
     conn <- connect(createConnectionDetails(
       dbms = 'postgresql',
       server = paste0(Sys.getenv('CDM_SERVER'), '/', Sys.getenv('CDM_DATABASE')),
       user = Sys.getenv('CDM_USER'),
       password = Sys.getenv('CDM_PASSWORD'),
       pathToDriver = Sys.getenv('DATABASECONNECTOR_JAR_FOLDER')
     ))
     res <- querySql(conn, 'SELECT COUNT(*) FROM cdm_synthea100k.person;')
     print(paste('Connected! Synthetic person count:', res[1,1]))
     disconnect(conn)
   "
   ```
2. **Deploy OHDSI Study Shiny Server**:
   ```bash
   docker compose up -d ohdsi-shiny
   ```
   *Mount directories: `/srv/shiny-server` (interactive apps) and `/srv/reports` (static HTML reports).*
3. **Deploy Atlas 3.0 Next-Gen Frontend & WebAPI 3.0**:
   ```bash
   docker compose up -d atlas3-webapi atlas3-frontend atlas3-db-init
   ```
4. **Acquire TLS 1.3 Certificate & Start Nginx Ingress**:
   ```bash
   # Acquire Let's Encrypt certificate:
   certbot certonly --webroot -w /var/www/certbot -d research.yourdomain.org --agree-tos --email devops@yourdomain.org
   # Boot edge proxy:
   docker compose up -d reverse-proxy certbot
   ```
5. **Exit Verification**:
   ```bash
   curl -I -k https://localhost/
   curl -I -k https://localhost/atlas/
   curl -I -k https://localhost/rstudio/
   curl -I -k https://localhost/shiny/
   ```

---

### Phase 4: Agentic Gateways, Developer Tooling & Handover Certification

1. **Deploy Agentic Gateways & Federated Node**:
   ```bash
   docker compose up -d study-agent-mcp webapi-mcp arachne-data-node arachne-exec-engine ollama-service
   ```
2. **Deploy Developer Productivity Tools (Web SQL Studio & MinIO S3 Mock)**:
   ```bash
   docker compose up -d cloudbeaver-sql minio-s3
   ```
3. **Verify MCP Agent Tool Discovery**:
   ```bash
   # Test WebApiMcp tool registry:
   curl -s -X POST https://research.yourdomain.org/webapi-mcp/mcp \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {}}' | jq .tools[].name

   # Test StudyAgent FastMCP SSE stream:
   curl -N -s https://research.yourdomain.org/mcp/sse -H "Authorization: Bearer <valid-token>" | head -n 5
   ```
4. **Test Sub-Minute Rollback Workflow**:
   ```bash
   # Create snapshot before testing:
   sudo ohdsi-snapshot create test-handover-snap
   # Test instant rollback:
   sudo ohdsi-snapshot rollback test-handover-snap
   # Remove test snapshot:
   sudo ohdsi-snapshot delete test-handover-snap
   ```
5. **Execute Handover Certification**:
   Run the 10-Point Handover Readiness Test from [DEVOPS_QUICKSTART.md](file:///c:/files/git/github/ohdsi/ohdsi_2026/DEVOPS_QUICKSTART.md#5-the-10-point-handover-readiness-test).
   When all 10 checks return `[ PASS ]`, email the handover card to the Data Science & Informatics leads.

---

## 3. Post-Handover Operations & Routine Maintenance

- **Automated Certificate Renewal**: The `certbot` container checks and renews Let's Encrypt certificates every 12 hours automatically.
- **Developer Package Compilations**: All R packages compile into the persistent `/home/ohdsi` volume; host containers do not need to be rebuilt when users install GitHub packages.
- **Resource Monitoring**: Monitor container memory and PostgreSQL active queries via CloudBeaver (`/sql/`) or `docker stats`. If any container reaches its limit, cgroup memory clamping ensures it fails safely without host disruption.
