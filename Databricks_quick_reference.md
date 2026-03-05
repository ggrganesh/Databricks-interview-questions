# AWS Databricks Administrator - Quick Reference Cheat Sheet

## 10 Critical Interview Domains at a Glance

### 1. CLUSTER FAILURES

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Out of Memory (OOM) | Memory overflow, data skew, new libraries | Increase executor memory, enable AQE, check for skew |
| Crashes after 2 hours | Auto-termination, Spot interruption, credentials expire | Check cluster settings, use on-demand for driver, verify IAM |
| Cannot attach to cluster | Missing cluster ACLs or workspace permissions | Grant 'Can Attach To' permission, check workspace access |
| Streaming lag growing | State store explosion, partition skew, broadcast timeout | Set watermark, cap offsets, increase broadcast timeout |
| Disk full errors | Log accumulation, unclean shuffle files | Restart cluster, enable log rotation, use external shuffle |

### 2. SECURITY & COMPLIANCE

| Scenario | Action | Timeline |
|----------|--------|----------|
| Unauthorized S3 access detected | Terminate cluster immediately, add DENY policy, revoke IAM role | 10 min |
| Find PII across 500+ tables | Query column names, use Macie, tag columns with UC | 2-3 hours |
| Disgruntled employee terminated | Revoke PATs, disable SSO, terminate clusters, pull audit logs | 60 min |
| SOC 2 audit preparation | Verify KMS encryption, TLS enforcement, CloudTrail logging | 1-2 weeks |

### 3. DATA PIPELINE & ETL

| Problem | Diagnostic | Solution |
|---------|----------|----------|
| DLT shows COMPLETED but data wrong | Query event_log for dropped_records | Run Full Refresh, set expect_or_fail |
| MERGE taking 6+ hours | Check for small files (numFiles), run DESCRIBE HISTORY | OPTIMIZE ZORDER, enable deletion vectors |
| Hadoop → Databricks 3x slower | S3 vs HDFS latency, partition mismatch | OPTIMIZE tables, enable Delta Cache, tune S3A |
| JDBC read hangs | Check network path, DNS resolution | Add connection timeout, set fetchsize, test SG rules |

### 4. COST EXPLOSION

| Root Cause | Detection | Prevention |
|------------|-----------|-----------|
| Forgotten cluster running | AWS Cost Explorer by ClusterId tag | Cluster policy with auto-termination |
| Wrong instance type | Check system.billing.usage by instance type | Cluster policy allowlist for instance types |
| Job retry storm | High cluster_id count in job history | Set max_retries=2, alert on >3 failures |
| DLT in CONTINUOUS mode | Check DLT pipeline settings | Use TRIGGERED mode for batch |

**Implement chargeback:** Tag everything (CostCenter, Project, Owner) → system.billing.usage → DBU pricing table → monthly dashboard

### 5. DISASTER RECOVERY

| Scenario | RTO | Key Component |
|----------|-----|----------------|
| Regional outage | < 1 hour | S3 CRR, DR workspace pre-built, DNS alias ready |
| Accidental DROP TABLE | < 5 min | RESTORE TABLE via time travel (or S3 recovery) |
| Cluster node failure | < 5 min | Kubernetes auto-recovery, spot fallback |

**Golden Rule:** Never run VACUUM without assessing data loss risk

### 6. MULTI-WORKSPACE ADMINISTRATION

| Task | Tool | Key Config |
|------|------|-----------|
| Onboard 500 users | SCIM + IdP + Databricks groups | Workspace assignment, UC permissions, cluster policy |
| Centralize config | Terraform + Git + CI/CD | Modules for security, UC, network |
| GDPR erasure | DELETE + crypto-shredding | Schedule KMS key deletion, don't VACUUM |

### 7. NETWORKING

| Requirement | Implementation | Gotcha |
|-------------|-----------------|--------|
| PrivateLink enabled | Two VPC endpoints + private DNS | Missing relay endpoint → clusters won't start |
| Firewall inspection | Transit Gateway + Network Firewall | S3 Gateway endpoint bypasses firewall (intentional) |
| No public IPs | SCC enabled | Outbound-only HTTPS tunnel to control plane |

### 8. UNITY CATALOG ADMIN

| Operation | Command | Risk |
|-----------|---------|------|
| Revoke catalog SELECT | REVOKE SELECT ON ALL TABLES IN CATALOG X FROM group | Doesn't break other grants (safe) |
| Row-level security | CREATE FUNCTION with row filter + ALTER TABLE SET ROW FILTER | Function must have SELECT on mapping table |
| Column masking | Requires UC credential vending | Not available for external locations |

### 9. MONITORING & OBSERVABILITY

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Cluster start latency | Cluster Events API | > 10 minutes |
| Job duration anomaly | system.lakeflow.job_run_timeline | > (7-day avg + 2σ) |
| SQL warehouse queue | system.query.history | Queue time > 10 sec |
| Cost anomaly | system.billing.usage | Daily spend > 120% of 30-day avg |

**Gold standard:** Proactive alerts BEFORE users complain

### 10. DBR & UPGRADES

| Phase | Days | Activities |
|-------|------|-----------|
| Assessment | 1-20 | Categorize jobs (high/med/low risk), identify breaking changes |
| Testing | 15-40 | Parallel execution, data validation, library compatibility |
| Staged migration | 40-80 | Week 1: Low-risk (100 jobs), Week 2: Medium (150), Week 3: High |
| Cutover | 80-90 | Remaining migration, rollback plan ready for 30 days |

---

## Memory Management Quick Guide

### OOM Errors - Diagnostic Path

```
1. Check Spark UI:
   - Peak Execution Memory near limit?
   - High Spill (Memory) + Spill (Disk)?
   - GC time > 10% of task time?

2. Examine YARN logs for:
   - "Java heap space OutOfMemoryError" (executor)
   - "GC overhead limit exceeded" (GC pressure)

3. Check for data skew:
   - One executor 100x data of others?

4. Fix in priority order:
   spark.executor.memoryOverhead = 4096
   spark.memory.fraction = 0.8
   spark.memory.storageFraction = 0.3
   Instance upgrade: r5.4xlarge vs m5.4xlarge
   Enable AQE: spark.sql.adaptive.enabled=true
```

---

## Access Control Decision Tree

```
User cannot attach to cluster?
├─ User has 'Can Use' workspace entitlement? NO → Grant entitlement
├─ Cluster has ACLs enabled? NO → Enable at Admin Console
├─ User has 'Can Attach To' on cluster? NO → Grant permission
├─ SCIM sync completed? NO → Wait 30 min or add manually
└─ UC user in account? NO → Add at account level

User cannot see certain tables?
├─ User in correct workspace? NO → Assign workspace
├─ GRANT SELECT on catalog? NO → GRANT USE CATALOG
├─ GRANT USE SCHEMA? NO → GRANT USE SCHEMA
├─ GRANT SELECT on TABLE? NO → GRANT SELECT on specific table
└─ Row filter blocking? NO → Verify row filter function
```

---

## Incident Response Playbooks (60 Seconds Each)

### Cluster Down
```
1. Check cluster event log for FAILED/UNAVAILABLE events
2. If Spot interruption: Check AWS Spot dashboard, increase On-Demand
3. If driver out of memory: Increase spark.driver.memory, restart
4. If node blacklisted: Instance failed, not your code
5. Auto-recovery will restart cluster within 2-5 min
```

### Data Loss Detected
```
1. DO NOT run VACUUM (prevents recovery)
2. Check if DROP vs TRUNCATE
3. If external table: CREATE TABLE on S3 location
4. If managed: Check Unity Catalog trash or use time travel
5. If neither: Recover from backup (weekly DEEP CLONE)
6. Time travel: RESTORE TABLE ... TO TIMESTAMP AS OF '...'
```

### Security Breach (Unauthorized Access)
```
1. Terminate cluster immediately (< 2 min)
2. Add DENY policy to S3 bucket (< 5 min)
3. Revoke IAM role (< 10 min)
4. Pull CloudTrail for data exfiltration (< 30 min)
5. Check Databricks audit logs for other anomalies
6. Post-mortem within 48 hours
```

### Regional Outage
```
1. Confirm outage on AWS Service Health Dashboard
2. Declare DR event to stakeholders
3. Validate S3 CRR lag (< 15 min acceptable)
4. Deploy jobs from git to DR workspace
5. Update Route 53 DNS alias to DR
6. Run smoke tests on critical pipelines
7. Notify stakeholders when operational
```

---

## Common Configs - Copy/Paste Ready

### Cluster Policy (Restrictive)
```json
{
  "aws_attributes": {
    "availability": "SPOT_WITH_FALLBACK",
    "first_on_demand": 1,
    "zone_id": "auto"
  },
  "spark_conf": {
    "spark.databricks.delta.merge.enableLowShuffle.merge": {"value": "true"},
    "spark.sql.adaptive.enabled": {"value": "true"}
  },
  "autotermination_minutes": 120,
  "num_workers": {"min_value": 1, "max_value": 20},
  "node_type_id": {"type": "allowlist", "values": ["m5.xlarge", "r5.xlarge"]},
  "instance_profile_arn": {"type": "allowlist", "values": ["arn:aws:iam::ACCOUNT:instance-profile/databricks-role"]}
}
```

### Spark Config for Large ETL
```
spark.executor.memoryOverhead=4096
spark.executor.memory=32g
spark.executor.cores=8
spark.sql.shuffle.partitions=200
spark.sql.adaptive.enabled=true
spark.sql.adaptive.coalescePartitions.enabled=true
spark.memory.fraction=0.8
spark.memory.storageFraction=0.3
spark.databricks.io.cache.enabled=true
spark.databricks.delta.merge.enableLowShuffle.merge=true
```

### UC Security Tagging
```sql
-- Tag PII columns
ALTER TABLE catalog.schema.table ALTER COLUMN ssn SET TAGS ('pii' = 'true', 'pii_type' = 'ssn');

-- Apply column mask
ALTER TABLE catalog.schema.table
ALTER COLUMN email SET MASK masked_email(email);

-- Query tagged columns
SELECT * FROM system.information_schema.column_tags WHERE tag_name = 'pii';
```

---

## Interview Red Flags to Avoid

| ❌ Bad | ✅ Good |
|--------|---------|
| "Just restart the cluster" (first solution) | Investigate root cause first |
| "Run VACUUM immediately" (data loss risk) | Assess retention, check for time travel |
| "Delete the user from everywhere" (overkill) | Revoke credentials, check audit logs, keep doc trail |
| "Max out cluster size" (costs explode) | Enable AQE, tune partitions, scale strategically |
| "That's a user problem" (not ownership) | Document issue, create runbook, enable self-service |

---

## Key System Tables (Unity Catalog)

```sql
-- Billing & usage
system.billing.usage                    -- DBU consumption by cluster, SKU, date
system.billing.list_compute_resources   -- Cluster cost attribution

-- Access control
system.access.audit                     -- All API calls and data access
system.access.audit_logs               -- Raw access logs
system.information_schema.tables        -- Table metadata (location, format)
system.information_schema.columns       -- Column names and types
system.information_schema.column_tags   -- UC tags on columns

-- Monitoring
system.lakeflow.job_run_timeline       -- Job run metrics
system.query.history                   -- SQL query history
system.compute.cluster_logs            -- Cluster event logs

-- Workspace config
system.workspace.catalogs              -- Catalog details
system.workspace.schemas               -- Schema details
system.workspace.tables                -- Table details
```

---

## Databricks API Key Endpoints

```bash
# Cluster management
GET /api/2.0/clusters/list              # List all clusters
POST /api/2.0/clusters/delete            # Delete cluster
GET /api/2.0/clusters/get                # Get cluster status

# Jobs
GET /api/2.1/jobs/list                  # List all jobs
POST /api/2.1/jobs/cancel-run            # Cancel job run
POST /api/2.1/jobs/runs/list             # Get run history

# Tokens
GET /api/2.0/token/list                 # List PATs
DELETE /api/2.0/token/delete             # Revoke PAT

# Unity Catalog
GET /api/2.1/unity-catalog/catalogs     # List catalogs
GET /api/2.1/unity-catalog/schemas      # List schemas
GET /api/2.1/unity-catalog/tables       # List tables
```

---

## Study Focus Areas by Experience Level

### For Intermediate → Advanced
- [ ] Master Spark memory internals (heap, overhead, storage fraction)
- [ ] Understand Delta Lake transactions and MVCC
- [ ] Learn Unity Catalog privilege model deeply
- [ ] Study AWS networking (VPC, TGW, PrivateLink)
- [ ] Know cluster policy constraint syntax by heart

### For Advanced → Expert
- [ ] Design multi-workspace governance models
- [ ] Implement chaos testing for DR scenarios
- [ ] Optimize MERGE for large tables (100TB+)
- [ ] Manage complex SCIM/SSO integrations
- [ ] Build observability stack from scratch

### For Expert → Architect
- [ ] Design Databricks deployments for regulated industries
- [ ] Migrate legacy data platforms at scale (1000+ workloads)
- [ ] Architect multi-region, multi-cloud setups
- [ ] Design cost optimization frameworks
- [ ] Build self-healing, self-scaling infrastructure

---

**Print this page and review 15 minutes before interview!**

Last updated: March 2026
