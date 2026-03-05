# AWS Databricks Administrator Interview Preparation Guide

**28 Real-World Scenarios | 10 Critical Domains | Production-Grade Answers**

---

## Quick Reference: The 10 Domains

| Domain | Questions | Key Focus |
|--------|-----------|-----------|
| **Cluster Failures** | Q1-Q5 | OOM errors, crashes, access, streaming lag, disk space |
| **Security & Compliance** | Q6-Q9 | Data breach response, PII audits, employee termination, SOC 2 |
| **Data Pipeline & ETL** | Q10-Q13 | Silent DLT failures, MERGE performance, migration, JDBC |
| **Cost Explosion** | Q14-Q16 | Investigation, SQL warehouse tuning, chargeback system |
| **Disaster Recovery** | Q17-Q18 | Regional failover, table recovery, time travel |
| **Multi-Workspace** | Q19-Q21 | User onboarding, centralized management, GDPR erasure |
| **Networking** | Q22-Q23 | PrivateLink debugging, firewall integration |
| **Unity Catalog Admin** | Q24-Q25 | Precision revocation, row-level security |
| **Monitoring & Observability** | Q26-Q27 | Proactive alerts, compliance auditing |
| **DBR & Upgrades** | Q28 | Major version migration |

---

## SECTION 1: Real-World Cluster Failure Scenarios

### Q1: Memory Errors on Production ETL (Container killed by YARN)

**Situation:** After 3 months of stable operation, a production ETL suddenly fails with 'Container killed by YARN for exceeding memory limits'. Data volume unchanged.

**Root Causes to Investigate (In Order):**
1. **DBR upgrade** - Changed default memory configs or executor overhead ratios
2. **New libraries** - Package version changes increase memory footprint
3. **Schema changes** - New nested JSON/struct columns increase deserialization load
4. **Spark UI evidence** - Check Peak Execution Memory, Spill metrics, GC time > 10%

**Investigation Steps:**
- Check recent changes: `DESCRIBE EXTENDED table_name` 
- Analyze Spark UI: Storage + Executor tabs, GC time percentage
- Review YARN container logs: Look for "Java heap space OutOfMemoryError" vs "GC overhead limit"

**Immediate Fixes:**
```
spark.executor.memoryOverhead = 4096  (increase from default 10% or 384MB)
spark.memory.fraction = 0.8            (more execution memory)
spark.memory.storageFraction = 0.3     (less storage memory)
```

**Long-term Solutions:**
- Switch to memory-optimized instances (r5.4xlarge vs m5.4xlarge)
- Enable AQE: `spark.sql.adaptive.enabled=true`
- Check for data skew - one executor receiving 100x data

**Interview Tip:** Walk through the Spark UI, reference specific metrics (Peak Execution Memory, GC %), and show you understand the difference between executor memory, overhead, and driver memory.

---

### Q2: Cluster Crashes After Exactly 2 Hours (SparkContext Shutdown)

**Most Likely Causes:**

1. **Auto-termination misconfigured**
   - Cluster auto-terminate set to 120 minutes
   - Long idle loop or checkpoint at 2-hour mark
   - Fix: Set to 0 for job clusters, increase for interactive

2. **Spot instance interruption**
   - AWS reclaims Spot after 2-hour blocks (legacy) or with 2-min warning
   - Driver killed → SparkContext shutdown
   - Fix: `aws_attributes: { first_on_demand: 1 }` makes driver always on-demand

3. **IAM credential expiration**
   - STS tokens expire after 1-6 hours by default
   - If using hardcoded access keys (not instance profile), keys may be rotated
   - Fix: Always use IAM instance profiles, never hardcoded credentials

4. **Notebook timeout policy**
   - Workspace-level setting: Admin Console > Workspace Settings
   - Job-level timeout set to 7200 seconds

**Verification Steps:**
- Check cluster event log at exact 2-hour mark
- Look for: TERMINATED (inactivity), NODE_BLACKLISTED (Spot), DRIVER_UNAVAILABLE

---

### Q3: New Data Scientist Cannot Attach to Any Cluster

**Systematic Access Troubleshooting Checklist:**

1. **Workspace-level permissions**
   - Admin Console > Users > Verify 'Can Use' entitlement (not just 'Can View')
   - Check if user is in 'No cluster create' group without assigned shared cluster

2. **Cluster-level ACLs** (Most common culprit)
   - Admin Console > Access Control > Must be enabled
   - Each cluster has: Can Attach To, Can Restart, Can Manage permissions
   - Fix: Grant user 'Can Attach To' on shared cluster

3. **Cluster policies**
   - If policy restricts who can create clusters, user needs policy assignment
   - Check: Admin Console > Cluster Policies > Permissions

4. **SCIM/SSO sync delays**
   - SCIM sync can lag 10-30 minutes
   - Fix: Admin Console > Groups > Add user manually if urgent

5. **Unity Catalog user mapping**
   - If UC enabled, user must exist at account level
   - Email must match SSO identity (case-sensitive)
   - Account Console > Users > Verify exact email match

**Resolution Checklist:**
- [ ] Add user to correct Databricks group (e.g., data-scientists)
- [ ] Grant group 'Can Attach To' on shared cluster
- [ ] Assign cluster policy if user creates their own clusters
- [ ] Verify SCIM sync completed

---

### Q4: Structured Streaming Job - Latency Spikes from 5 to 45 Minutes

**Streaming Lag Diagnosis Framework:**

**Step 1: Identify the bottleneck**
- Spark UI > Structured Streaming tab > Check 'Processing Time' vs 'Trigger Interval'
- If Processing Time > Trigger Interval, you're falling behind
- Check 'Batch Duration' trend for sudden spike
- Monitor 'Input Rows' per batch for upstream bursts

**Common Root Causes:**

1. **State store explosion** (stateful operations)
   - Watermark not configured or too wide
   - State grows unbounded → checkpoint writes take minutes
   - Fix: `df.withWatermark('eventTime', '10 minutes')`
   - Use RocksDB backend: `spark.sql.streaming.stateStore.providerClass = org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider`

2. **Kafka partition skew**
   - One partition receiving 100x data of others
   - Fix: Increase partitions, set `maxOffsetsPerTrigger = 100000`

3. **S3 checkpoint slowdown**
   - S3 throttling on checkpoint writes
   - Fix: Move checkpoint to instance-local disk or use prefix sharding

4. **Broadcast join timeout**
   - Large table broadcast failing
   - Fix: Increase `spark.sql.broadcastTimeout=600`

**Immediate Mitigation:**
- Scale up cluster (add workers)
- Cap batch size with `maxOffsetsPerTrigger`

---

### Q5: Overnight Job Run Fails - 40 Jobs with "No space left on device"

**Root Cause:** Log and shuffle accumulation on driver disk

**Where It Accumulates:**
- `/databricks/spark/logs/` - event logs
- `/local_disk0/spark-*` - shuffle temp files from failed jobs
- `/tmp/` - Python package artifacts
- `/databricks/python/lib/` - installed packages

**Weekend-Specific Factor:**
- Long-running job failed mid-way without cleaning up shuffle files
- Multiple jobs on shared long-lived cluster filled disk cumulatively

**Immediate Fix:**
- Restart cluster (clears all temp files)
- Temporarily disable: `spark.eventLog.enabled=false`

**Long-term Prevention:**
- Use ephemeral job clusters (fresh per job, always clean disk)
- Enable log rotation: `spark.eventLog.rolling.enabled=true`, `spark.eventLog.rolling.maxFileSize=128m`
- CloudWatch alarm: EBS utilization > 80%
- Enable external shuffle service: `spark.shuffle.service.enabled=true`
- Use i3/i3en instances (3.8TB+ NVMe local disks)
- Init script: Clean `/tmp` at cluster start

---

## SECTION 2: Security & Compliance Breach Scenarios

### Q6: S3 Access Logs Show Unauthorized Bucket Access from Cluster Node

**Incident Response (First 60 Minutes):**

**Minutes 0-10: Identify and contain**
1. Extract IAM principal from S3 logs: `arn:aws:sts::<account>:assumed-role/<role>/i-<instance-id>`
2. Map instance-id to Databricks cluster (EC2 tags)
3. Identify who launched the cluster
4. **Immediately terminate cluster** via API:
   ```
   POST /api/2.0/clusters/delete { "cluster_id": "xxx" }
   ```
5. Add explicit DENY to bucket policy:
   ```json
   { "Effect": "Deny", "Principal": { "AWS": "arn:..." }, "Action": "s3:*", "Resource": "arn:aws:s3:::sensitive-bucket/*" }
   ```

**Minutes 10-20: Root cause analysis**
- Check IAM instance profile attached to cluster
- Verify if role has overly broad permissions (e.g., `s3:*` on `*`)
- Look for managed policies like `AmazonS3FullAccess`

**Remediation:**
- Implement least-privilege: Specific bucket ARNs, not wildcards
- Use IAM Permission Boundaries to cap maximum permissions
- Enforce via cluster policy: Restrict which instance profiles allowed
- Enable Unity Catalog external locations (UC credential vending)

**Post-Incident:**
- Enable CloudTrail data events on sensitive buckets
- Set AWS Config rule: `s3-bucket-policy-not-more-permissive`
- Audit all instance profiles quarterly with IAM Access Analyzer

---

### Q7: Audit: Prove No PII (SSN, Credit Card) in DBFS Root Across 500+ Tables

**Systematic PII Audit Approach:**

**Phase 1: Table enumeration**
```sql
SELECT table_catalog, table_schema, table_name, storage_location
FROM system.information_schema.tables
WHERE storage_location LIKE '%dbfs%'
```

**Phase 2: Schema scanning**
- Scan column names for PII keywords: `ssn`, `social_security`, `credit_card`, `cvv`, `dob`, `passport`
- Automated Databricks notebook using regex matching

**Phase 3: Content scanning (for flagged tables)**
- Use Amazon Macie: Point at DBFS root S3 bucket
- Macie auto-detects SSNs, credit cards using ML models
- Results in Macie console + S3 findings bucket

**Phase 4: Unity Catalog tagging**
- After audit, tag PII columns: `ALTER TABLE t ALTER COLUMN ssn SET TAGS ('pii' = 'true')`
- Query tagged columns: `SELECT * FROM system.information_schema.column_tags WHERE tag_name = 'pii'`
- Apply column masks

**Phase 5: Remediation & reporting**
- Move tables with PII from DBFS root to external locations
- Enable KMS encryption on DBFS root bucket
- Produce audit report: Table name, schema, PII columns, storage location, encryption status

---

### Q8: Disgruntled Employee Terminated - 60-Minute Security Runbook

**Minutes 0-10: Identity revocation**
- Disable SSO account in IdP (Okta/Azure AD) - blocks all new logins
- Revoke all PATs: `GET /api/2.0/token/list` → `DELETE /api/2.0/token/delete`
- Remove user from all workspaces: Account Console > Workspaces
- Disable SCIM if enabled

**Minutes 10-20: Credential audit**
- Check for service principals created by user
- Rotate all secrets in scopes user had access to
- Rotate IAM access keys for instance profiles they could use

**Minutes 20-40: Terminate active sessions**
- Identify and terminate clusters owned by user
- Cancel running jobs triggered by user
- Terminate active notebook sessions

**Minutes 40-60: Forensics & audit**
```sql
SELECT * FROM system.access.audit 
WHERE user_identity.email = 'terminated@co.com'
ORDER BY event_time DESC
```
- What data accessed? What tables queried? Were files exported?
- Check CloudTrail for S3 data exfiltration
- Search for backdoors in notebooks (hidden PAT tokens, webhooks)

**Post-Incident Hardening:**
- Implement break-glass account model (no individual admin direct access)
- Mandatory MFA for all admin accounts
- Real-time alert on new PAT creation by admins

---

### Q9: SOC 2 Type II Audit - Complete Encryption Evidence Package

**Data at Rest Evidence:**

1. **DBFS Root S3**
   - AWS Console > S3 > Default Encryption: SSE-KMS with Customer Managed Key
   - AWS Config rule: `s3-bucket-server-side-encryption-enabled` (COMPLIANT)

2. **EBS Volume Encryption (cluster nodes)**
   - Admin Console > Storage > EBS Encryption enabled with KMS key
   - All Databricks volumes encrypted with CMK
   - AWS Config rule: `encrypted-volumes` (COMPLIANT)

3. **Unity Catalog Managed Storage**
   - UC storage credential with KMS key configured
   - S3 bucket policy enforces SSE-KMS

4. **Notebook Storage (Control Plane)**
   - Account Console > Encryption > Customer Managed Key
   - Requires DBX Enterprise + KMS key with cross-account trust

5. **External Hive Metastore (RDS)**
   - RDS console > Storage Encrypted: YES, with KMS Key ARN

**Data in Transit Evidence:**

1. **TLS Enforcement**
   - Databricks enforces TLS 1.2+ for all API calls
   - VPC flow logs showing all traffic on port 443

2. **Secure Cluster Connectivity (SCC)**
   - SCC enabled: No public IPs on cluster nodes
   - All communication via outbound-only HTTPS tunnel

3. **PrivateLink**
   - VPC Endpoints configured for Databricks
   - DNS resolution for `*.cloud.databricks.com` resolves to private IPs

**Key Management Evidence:**

- AWS KMS console: Key rotation enabled (annual)
- KMS key policy showing only authorized roles
- CloudTrail logs of all KMS Decrypt/GenerateDataKey calls
- Quarterly key access reviews

---

## SECTION 3: Data Pipeline & ETL Failure Scenarios

### Q10: Delta Live Tables Pipeline Shows COMPLETED but Downstream Data Wrong

**Silent DLT Failure Investigation:**

**Step 1: Understand why it showed COMPLETED**
- DLT can complete successfully even if datasets violate expectations
- Violations logged but don't fail unless set to `expect_or_fail`

**Key Distinction:**
```python
@dlt.expect('valid', 'amount > 0')           # Logs violation, continues (WARN)
@dlt.expect_or_fail('valid', 'amount > 0')  # Fails pipeline on violation
@dlt.expect_or_drop('valid', 'amount > 0')  # Drops bad rows, continues
```

**Step 2: Query DLT event log**
```sql
SELECT * FROM <pipeline_storage>.event_log
WHERE event_type = 'flow_progress' AND details:flow_progress:data_quality IS NOT NULL
ORDER BY timestamp DESC
```
- Look for `dropped_records` or `failed_records` that increased 3 days ago

**Step 3: Identify root cause**
- Source schema changed? New column with NULLs?
- Upstream MERGE had a bug?
- Check git commit history for code changes 3 days ago

**Step 4: Data remediation**
- Identify exact window of bad data: `SELECT MIN(event_time), MAX(event_time) FROM source WHERE issue`
- Trigger 'Full Refresh' mode to reprocess all data
- Or selective table refresh (DLT 2023.xx+)

**Step 5: Prevention**
- Change critical expectations to `expect_or_fail`
- Add reconciliation job: Daily check comparing source rows to DLT output
- Set up alert on high dropped_record count

---

### Q11: MERGE on 50TB Delta Table - 6+ Hours (Was 45 Minutes)

**MERGE Performance Degradation Analysis:**

**What Changed:**
- Check history: `DESCRIBE HISTORY target_table` - numFiles exploding?
- Did OPTIMIZE get skipped recently?
- Source table size grow? (full historical load vs incremental)
- DBR upgrade change join strategy?

**Immediate Optimizations:**

1. **Small file problem** (Most common)
   ```sql
   OPTIMIZE target_table ZORDER BY (merge_key_column);
   -- Then re-run MERGE — typically 5-10x faster
   ```

2. **Partition pruning**
   ```sql
   MERGE INTO target t
   USING source s
   ON t.merge_key = s.merge_key AND t.date_partition >= '2024-01-01'
   -- Delta scans only overlapping partitions
   ```

3. **Deletion vectors** (DBR 12.2+ with Delta 2.3+)
   ```sql
   ALTER TABLE target SET TBLPROPERTIES ('delta.enableDeletionVectors' = 'true');
   -- Lightweight deletion instead of rewriting files
   ```

4. **Low-shuffle MERGE** (DBR 10.4+)
   ```sql
   SET spark.databricks.delta.merge.enableLowShuffle.merge = true;
   -- Only rewrites files containing matching rows
   ```

5. **Broadcast MERGE** (small source tables)
   ```sql
   SET spark.databricks.delta.merge.enableBroadcast = true;
   -- If source < 100MB, broadcasts instead of shuffle
   ```

6. **Liquid Clustering** (DBR 13.3+)
   ```sql
   ALTER TABLE target CLUSTER BY (merge_key_column);
   -- Incremental clustering, not full rewrites
   ```

---

### Q12: Hadoop-to-Databricks Migration - 3x Slower Performance

**Key Architectural Difference:**
- Hadoop: HDFS local disks, data locality, 1-3 GB/s sequential reads
- Databricks: S3 remote object store, no data locality, high latency (~100ms per request)

**Root Causes & Fixes:**

1. **S3 vs HDFS I/O**
   - Small file problem: Thousands of HTTP requests
   - Fix: Run OPTIMIZE on all Delta tables, use Delta instead of raw Parquet/ORC
   - `spark.hadoop.fs.s3a.fast.upload=true`
   - Increase S3 connection pool: `spark.hadoop.fs.s3a.connection.maximum=100`

2. **Partition count mismatch**
   - Hadoop tuned for 128MB HDFS blocks → 1000 partitions
   - S3 optimal: 1GB partitions
   - Fix: Set `spark.sql.shuffle.partitions` with AQE or manually

3. **Missing Delta Cache**
   - Requires i3, i3en, r5d instances (NVMe SSDs)
   - Enable: `spark.databricks.io.cache.enabled=true`
   - General-purpose instances (m5, r5) lack Delta Cache

4. **Hadoop-specific configs invalid**
   - `mapreduce.job.maps=500` → irrelevant on Spark
   - `dfs.replication=3` → S3 handles this
   - Remove all Hadoop-specific settings

5. **Serialization**
   - Enable Kryo: `spark.serializer=org.apache.spark.serializer.KryoSerializer`

**Tuning Plan:**
1. OPTIMIZE tables, enable Delta Cache, tune S3A pool
2. Enable AQE, tune shuffle partitions, add broadcast hints
3. Profile with Spark UI, fix skew and GC issues

---

### Q13: JDBC Read from PostgreSQL Hangs - DB Shows No Active Queries

**Diagnosis:**

**Network vs Database Issue:**
- Is JDBC stage running tasks or stuck at 0 tasks?
- 0 tasks = driver can't connect
- Running but stalled = executor connected but query blocked

**Common JDBC Hang Causes:**

1. **Connection pool exhaustion**
   - Default: 1 connection per partition (numPartitions tasks = numPartitions connections)
   - PostgreSQL max_connections=100, opening 200 → connection blocks
   - Fix: Set `numPartitions=10`, or use pgBouncer

2. **Network security group blocking**
   - Check: Cluster SG outbound rules allow port 5432 to RDS
   - RDS SG inbound allows port 5432 from cluster SG
   - VPC peering/Transit Gateway if different VPCs

3. **Query timeout not set**
   - Connection established but query hangs server-side

4. **DNS resolution failure**
   - Test: `nc -zv rds-endpoint 5432` from cluster

5. **SSL handshake hang**
   - Certificate validation failing
   - Test: Add `?ssl=true&sslmode=require` vs `sslmode=disable`

**Fix:**
```python
spark.read.jdbc(url, table, properties={
    'driver': 'org.postgresql.Driver',
    'connectTimeout': '30',
    'socketTimeout': '300',
    'loginTimeout': '30',
    'fetchsize': '10000'
})
```

---

## SECTION 4: Cost Explosion & FinOps Scenarios

### Q14: AWS Bill Tripled - EC2 Spend Exploded

**Investigation Playbook:**

**Step 1: Pinpoint the spend**
- AWS Cost Explorer: Filter Service=EC2, Tag:Vendor=Databricks
- Group by Usage Type: BoxUsage vs SpotUsage
- Break down by instance type
- Filter by ClusterId tag

**Step 2: Databricks system tables**
```sql
SELECT workspace_id, cluster_id, creator_user_name, sku_name,
       SUM(dbus) as total_dbus
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 30)
GROUP BY 1,2,3,4
ORDER BY total_dbus DESC LIMIT 20
```

**Common Hidden Causes:**

1. **Long-running interactive cluster**
   - Engineer forgot to terminate r5.4xlarge → ran 15 days
   - Fix: Cluster policy with max 120-min auto-termination

2. **Wrong instance type selected**
   - GPU selected for non-ML job → 10x DBU rate
   - Fix: Cluster policy restricts instance types

3. **Job retry storm**
   - Job failing and retrying 100x → multiplies cluster cost
   - Fix: Set `max_retries=2`, alert on >3 consecutive failures

4. **DLT in CONTINUOUS mode**
   - Pipeline runs 24/7 even with no data
   - Fix: Switch to Triggered mode for batch pipelines

5. **ML experiment left running**
   - Hyperparameter tuning cluster left overnight
   - Fix: Auto-termination enforcement

**Controls to Implement:**
- Cluster policies: max instance type, auto-termination, max workers
- AWS Budgets alerts at 80% and 100% of budget
- AWS Config rule: All Databricks EC2 must have CostCenter, Project, Owner tags
- Weekly cost report from system tables
- Spot instance enforcement: SPOT_WITH_FALLBACK for workers

---

### Q15: SQL Warehouse Slow During 9AM-5PM Business Hours

**Diagnosis:**
- Warehouse queue building up during peak hours
- Check Warehouse monitoring: Queue time, Peak concurrency, Autoscaling events
- If queue time > 10 seconds during 9AM-5PM, warehouse undersized

**Solution 1: Warehouse autoscaling** (Most cost-effective)
- Min clusters: 1 (saves cost off-hours)
- Max clusters: 10 (for peak concurrency)
- Scaling policy: 'Economy' (slower, cheaper) vs 'Reliability' (faster, higher cost)
- Scale-up trigger: Query queue depth > 0 for > 2 minutes

**Solution 2: Serverless SQL Warehouse**
- Eliminates cold start (1-3 seconds vs 2-5 minutes)
- Auto-scales instantly
- Slightly higher DBU rate but no idle cluster cost
- Best for unpredictable, bursty BI loads

**Solution 3: Query optimization**
- Enable Query Results Cache (24-hour cache for identical queries)
- Enable Photon: 2-5x faster for SQL queries
- Identify slow queries, add ZORDER/partitioning

**Solution 4: Workload isolation**
- Separate warehouses by team/priority:
  - Executive: 4 clusters, guaranteed capacity
  - Analyst: 2 clusters, standard queue
  - Batch: 1 cluster, off-hours only

**Solution 5: Pre-warming**
- Run 'warm-up' query 15 min before 9AM via Jobs
- Or disable auto-suspend during business hours

---

### Q16: Implement Chargeback - Bill Each Business Unit

**Architecture:**

**Layer 1: Tagging strategy** (Mandatory via cluster policy)
```json
{
  "CostCenter": "FINANCE-001",
  "Project": "revenue-forecast",
  "Environment": "prod",
  "Owner": "engineer@company.com"
}
```

**Layer 2: Data collection**
```sql
SELECT u.workspace_id, c.custom_tags['CostCenter'] as cost_center,
       u.sku_name, SUM(u.dbus) as total_dbus,
       SUM(u.dbus * p.dbu_price) as estimated_cost
FROM system.billing.usage u
JOIN system.compute.clusters c ON u.cluster_id = c.cluster_id
JOIN dbu_pricing p ON u.sku_name = p.sku_name
GROUP BY 1,2,3
```

**Layer 3: DBU price mapping**
- All-Purpose Compute: $0.55/DBU
- Jobs Compute: $0.22/DBU
- SQL Compute: $0.22/DBU
- DLT Core: $0.20/DBU
- Photon: additional multiplier

**Layer 4: Reporting**
- Databricks SQL dashboard: Cost by cost center, trends, top spenders
- Scheduled monthly PDF export to finance
- AWS Cost Explorer custom cost categories for unified billing

**Layer 5: Governance**
- Budget limits per cost center
- Require approval for clusters > 10 nodes or GPU
- Quarterly review: Cost per DBU vs industry benchmark

---

## SECTION 5: Disaster Recovery & High Availability

### Q17: Primary Workspace Regional Outage - 1-Hour Failover

**Pre-requisites (Must be in place before incident):**
- DR workspace in us-west-2 with same VPC structure
- S3 Cross-Region Replication (CRR) enabled from primary DBFS and Delta storage
- Job definitions in git (not just workspace)
- Unity Catalog metastore with S3 replication
- DNS alias pointing to primary (updatable)

**Minute 0-10: Assess and declare**
- Confirm outage via AWS Service Health Dashboard
- Declare DR event - notify stakeholders
- Do NOT attempt restart in us-east-1

**Minute 10-25: Data validation in DR region**
- Check S3 CRR replication lag: AWS Console > S3 > Replication metrics
- Verify latest Delta commit in DR matches primary
- If replication lag > 15 min, assess RPO impact

**Minute 25-45: Activate DR workspace**
- Deploy jobs from git using Databricks REST API
- Update DR workspace external locations to point to replicated S3 buckets
- Restart streaming jobs from last checkpoint
- Update Route 53 alias: databricks-workspace.company.com → dr-workspace.cloud.databricks.com

**Minute 45-60: Validation**
- Run smoke test notebooks for critical pipelines
- Verify row counts match expected values
- Confirm BI dashboards connect to DR
- Notify stakeholders: Failover complete

**Post-recovery:**
- Sync any DR-written data back to primary
- Validate both regions in sync before failing back
- Post-mortem within 48 hours

---

### Q18: Production Delta Table Accidentally Dropped (2 Years of Data)

**Recovery Procedure:**

**DO NOT run VACUUM** (Critical first step)
- If VACUUM not run with retention < 7 days, underlying Parquet files still exist
- Immediately message all engineers: "DO NOT run VACUUM on any tables right now"

**Assess the situation:**
- DROP TABLE vs TRUNCATE TABLE?
  - DROP: Metadata removed, may delete S3 data (depends on managed vs external)
  - TRUNCATE: Data marked for deletion but recoverable via time travel

**Recovery for external Delta table** (Easiest)
- DROP TABLE only removes metadata - _delta_log still on S3
- Re-register: `CREATE TABLE financial_transactions LOCATION 's3://bucket/path/'`

**Recovery for managed table**
- If Hive Metastore, files may still exist on S3
- `CREATE TABLE financial_transactions_recovered USING DELTA LOCATION 's3://path/'`
- If Unity Catalog, check UC trash within retention window

**Time travel restore** (If modified, not dropped)
```sql
RESTORE TABLE financial_transactions TO VERSION AS OF 42;
-- Or by timestamp:
RESTORE TABLE financial_transactions TO TIMESTAMP AS OF '2024-01-15 08:00:00';
```

**Post-recovery governance:**
- Enable table access control - require explicit DROP TABLE permission
- Revoke DROP TABLE from non-admins on production catalogs
- Add pre-drop hook (via Terraform with PR approval)
- Weekly backup: Schedule job to clone critical tables:
  ```sql
  CREATE TABLE backup.financial.transactions_backup_YYYYMMDD DEEP CLONE prod.financial.transactions
  ```

---

## SECTION 6: Multi-Workspace & Enterprise Administration

### Q19: Onboard 500 New Analysts - No Production Data Access

**Identity Provider Setup:**
- Create group in Okta/Azure AD: 'acquired-company-analysts'
- SCIM provisioner syncs users and group to Databricks
- Sync frequency: 40 minutes by default

**Databricks Account Console:**
- Verify users in Account Console > Users after SCIM
- Assign group to correct workspace (non-production)
- User role: User (not Admin)

**Unity Catalog access model:**
- Create catalog for sandbox: `CREATE CATALOG analyst_sandbox`
- Grant access:
  ```sql
  GRANT USE CATALOG ON CATALOG analyst_sandbox TO acquired-company-analysts;
  GRANT USE SCHEMA ON CATALOG analyst_sandbox TO acquired-company-analysts;
  GRANT SELECT ON ALL TABLES IN CATALOG analyst_sandbox TO acquired-company-analysts;
  ```

**Shared data (non-PII views of production):**
- Create 'shared_views' catalog with anonymized views
- Explicitly DENY production access (default deny if never granted)

**Cluster access:**
- Create shared SQL warehouse (Serverless for instant start)
- Cluster policy: max m5.xlarge, auto-terminate 30 min

**Self-service onboarding portal:**
- Databricks notebook new users run on first login
- Shows available catalogs, runs test query
- Send welcome email with workspace URL, warehouse name, documentation

**Validation:**
- Test with 5 pilot users first
- Can query analyst_sandbox ✓, Cannot query prod ✓

---

### Q20: Centralized Config Management for 15 Workspaces

**Architecture: Infrastructure-as-Code (Terraform) + CI/CD Pipeline**

**Layer 1: Terraform Databricks Provider**
- Centralized state per workspace in S3 backend with DynamoDB locking
- Workspace-specific variables (token) in AWS Secrets Manager

**Layer 2: Shared module library**
- `modules/cluster_policies/` - standard policies per team
- `modules/security_config/` - SCC, access control
- `modules/unity_catalog/` - catalog, schema, grant templates
- `modules/network/` - VPC, subnets, security groups

**Layer 3: GitOps pipeline**
- Central git repo: `databricks-platform-config/`
- PR process: terraform plan for ALL affected workspaces
- Reviewer sees exact diff of changes
- Merge triggers: terraform apply for affected workspaces

**Layer 4: Drift detection**
- Schedule daily: `terraform plan -detailed-exitcode` for each workspace
- If exit code = 2 (changes detected) → alert to Slack

**Layer 5: Sensitive config management**
- Service principal tokens in AWS Secrets Manager (not git)
- Rotate tokens every 90 days via Lambda
- Token has minimum required permissions

**Layer 6: Compliance check**
- Python script calls Databricks API across all workspaces
- Verify: SCC enabled, IP access list, SCIM configured, audit logging
- Run daily as Databricks job, alert on non-compliance

**Benefits:**
- Security policy change deploys to all 15 workspaces in one PR
- Full audit trail (git history)
- New workspace provisioning in < 30 minutes

---

### Q21: GDPR Right to Erasure - Delete User from 47 Tables

**Challenge:** Delta Lake's immutable log + time travel retention conflicts with erasure

**Step 1: Discovery**
```sql
SELECT table_catalog, table_schema, table_name
FROM system.information_schema.columns
WHERE column_name IN ('user_id', 'email', 'customer_id')
```

**Step 2: Deletion**
```sql
DELETE FROM catalog.schema.table WHERE user_id = 'TARGET_USER_ID';
```
- Delta logs this as new version - data removed from active table
- Verify: `SELECT COUNT(*) ... WHERE user_id = 'TARGET_USER_ID'` → must return 0

**Step 3: Time travel data** (The hard part)
- Old versions still contain user's data (GDPR risk)

**Solution A: VACUUM with short retention** (Easiest but destructive)
```sql
SET spark.databricks.delta.retentionDurationCheck.enabled = false;
VACUUM table RETAIN 0 HOURS;
```
- WARNING: Destroys time travel for ALL users, not just target user

**Solution B: Crypto-shredding** (Best practice for GDPR + Delta)
- Store PII encrypted with per-user key in AWS KMS
- To erase: Delete KMS key or schedule key deletion (7-day minimum)
- Historical versions contain only ciphertext - without key, data unrecoverable

**Step 4: Downstream systems**
- ML models trained on user's data → may need retraining
- Delta Sharing recipients → notify them
- Backups/archives → apply same deletion

**Step 5: Documentation**
- Log erasure: timestamp, hashed user_id, tables affected, method used
- Respond to user within 30 days per GDPR Article 17

---

## SECTION 7: Advanced Networking & PrivateLink

### Q22: After Enabling PrivateLink - Clusters Won't Start

**Root Cause:** Missing relay endpoint or misconfiguration

**PrivateLink requires TWO VPC endpoints:**
1. Workspace access endpoint (REST API)
2. Relay endpoint (SCC - secure cluster tunnel)

**Most Common Issues:**

1. **Missing relay endpoint**
   - Many admins configure only workspace endpoint
   - Without relay, cluster nodes can't establish SCC tunnel
   - Fix: Add second VPC endpoint for relay service

2. **Private DNS not enabled**
   - Must enable 'Private DNS' on both endpoints
   - Without it, DNS resolves to PUBLIC IPs
   - If security group blocks internet, connections fail
   - Fix: Enable private DNS, verify resolution: `nslookup <workspace-url>` → returns 10.x.x.x

3. **Security group on endpoint doesn't allow cluster**
   - Endpoint has own security group
   - Missing inbound rule: Port 443 from cluster SG
   - Fix: Add inbound rule - HTTPS (443) from cluster_security_group

4. **Route table misconfiguration**
   - Interface endpoints route via DNS (no route table change needed)
   - But custom DNS resolver may override Databricks domain
   - Fix: Check Route 53 Resolver rules

5. **Workspace not configured to use PrivateLink**
   - Even after creating endpoints, workspace must be configured
   - Fix: Account Console > Workspaces > Network > Enable Private Link

**Validation:**
- From cluster: `curl -v https://<workspace-url>/api/2.0/clusters/list`
- Check if connection through private IP (10.x.x.x) or public IP
- Test relay: `nc -zv scc-relay.cloud.databricks.com 443`

---

### Q23: All Databricks Traffic Through Centralized Firewall

**Architecture: Hub-and-Spoke with Inspection VPC**

```
Databricks VPC (Spoke) → Transit Gateway → Inspection VPC → AWS Network Firewall → Internet
```

**Step 1: Inspection VPC setup**
- Create Inspection VPC: 10.1.0.0/16
- Deploy AWS Network Firewall in Inspection VPC
- Firewall subnets in each AZ
- NAT Gateway in Inspection VPC

**Step 2: Transit Gateway**
- Create TGW with DNS Support enabled
- Attach: Databricks VPC + Inspection VPC
- Databricks route table: 0.0.0.0/0 → Transit Gateway
- TGW route table: 0.0.0.0/0 → Inspection VPC
- Inspection VPC (return): 10.0.0.0/8 → TGW

**Step 3: Network Firewall rules** (Stateful, domain-based)
```
PASS: *.cloud.databricks.com (control plane + relay)
PASS: s3.<region>.amazonaws.com (DBFS, Delta tables)
PASS: sts.<region>.amazonaws.com (IAM credential vending)
PASS: kinesis.<region>.amazonaws.com (log streaming)
PASS: *.pypi.org, *.anaconda.com (packages - or block + use Nexus mirror)
DROP: all other outbound (default deny)
```
- Alert on any blocked traffic from Databricks CIDR

**Step 4: Databricks VPC configuration**
- Remove NAT Gateway from Databricks VPC
- Private subnet route: 0.0.0.0/0 → TGW
- S3 Gateway endpoint: Still add (traffic goes directly to S3, not TGW - free + faster)

**Step 5: Validation**
- Launch test cluster - should start successfully
- Confirm traffic in Firewall flow logs
- Test blocked traffic: `curl http://example.com` from notebook → should block
- Verify S3 access works (bypasses firewall via Gateway endpoint as expected)

**Trade-off:** All non-S3 egress has extra latency through TGW + Firewall (~1-2ms)

---

## SECTION 8: Unity Catalog Advanced Administration

### Q24: Revoke Catalog-Level SELECT from Group - Don't Break Other Users

**Scenario:** Data steward accidentally granted 'SELECT on ALL TABLES in CATALOG prod' to overprivileged_group. 200 other users had legitimate access before the grant.

**Precision Revocation:**

**Step 1: Document current state**
```sql
SHOW GRANTS ON CATALOG prod;
SHOW GRANTS ON SCHEMA prod.finance;
-- Save output for reference
```

**Step 2: Revoke overly-broad grant**
```sql
REVOKE SELECT ON ALL TABLES IN CATALOG prod FROM overprivileged_group;
```
- Only removes the group's grant
- 200 legitimate users unaffected (grants via other groups or individual grants remain)

**Step 3: Identify legitimate access pattern**
```sql
SELECT DISTINCT resource_name
FROM system.access.audit
WHERE user_identity.groups ARRAY_CONTAINS('overprivileged_group')
AND action_name = 'READ'
AND event_time < '<accident_timestamp>'
ORDER BY resource_name
```

**Step 4: Re-grant only legitimate schemas**
```sql
GRANT USE SCHEMA ON SCHEMA prod.finance TO overprivileged_group;
GRANT SELECT ON ALL TABLES IN SCHEMA prod.finance TO overprivileged_group;
```

**Step 5: Verify 200 legitimate users unaffected**
- Only users with SOLE access through overprivileged_group lose access
- Users in other groups with proper grants retain access
- Test with 3-5 representative users: `SELECT ... FROM prod.finance.table`

**Prevention:**
- Alert on catalog-level SELECT grants
- 4-eye principle: Require second admin approval
- All UC grants via Terraform (PR review before applying)

---

### Q25: Row-Level Security - Filter Based on Logged-In User

**Implementation:**

**Step 1: Create user-to-region mapping table**
```sql
CREATE TABLE catalog.security.user_region_mapping (
  user_email STRING,
  allowed_region STRING
);
INSERT INTO catalog.security.user_region_mapping VALUES
('alice@company.com', 'NORTH'),
('bob@company.com', 'SOUTH'),
('carol@company.com', 'EAST'),
('david@company.com', 'ALL');
```

**Step 2: Create row filter function**
```sql
CREATE OR REPLACE FUNCTION catalog.security.region_filter(region STRING)
RETURNS BOOLEAN
RETURN (
  SELECT COUNT(*) > 0
  FROM catalog.security.user_region_mapping
  WHERE user_email = CURRENT_USER()
  AND (allowed_region = region OR allowed_region = 'ALL')
);
```

**Step 3: Apply filter to table**
```sql
ALTER TABLE catalog.sales.transactions
SET ROW FILTER catalog.security.region_filter ON (region);
```
- Every SELECT automatically filters by logged-in user

**Step 4: Verify it works**
```sql
-- As alice@company.com:
SELECT DISTINCT region FROM catalog.sales.transactions;
-- Returns: NORTH only

-- As david@company.com (admin):
SELECT DISTINCT region FROM catalog.sales.transactions;
-- Returns: NORTH, SOUTH, EAST, WEST (all)
```

**Step 5: Grant access correctly**
```sql
-- Row filter function needs SELECT on mapping table:
GRANT SELECT ON TABLE catalog.security.user_region_mapping TO FUNCTION catalog.security.region_filter;

-- Users get SELECT on sales table (not on mapping):
GRANT SELECT ON TABLE catalog.sales.transactions TO sales-managers;
REVOKE SELECT ON TABLE catalog.security.user_region_mapping FROM sales-managers;
```

**Step 6: Performance optimization**
- Row filter is subquery per row - slow on 10B row tables
- Cache user's allowed_region at session start
- Or partition table by region and add partition pruning hint

---

## SECTION 9: Advanced Monitoring & Observability

### Q26: Proactive Monitoring System - Alert Before Users Complain

**Layer 1: Cluster Health Metrics** (Real-time)
- Poll Cluster Events API every 5 min for: FAILED_TO_START, DRIVER_UNAVAILABLE, NODE_BLACKLISTED
- Metric: Cluster start latency > 8 min indicates AZ capacity issues
- CloudWatch Custom Metrics via Lambda
- Alert: Start time > 10 min → PagerDuty P2

**Layer 2: Job Performance** (Trend-based)
```sql
SELECT job_id, AVG(duration_seconds) as avg_duration,
       STDDEV(duration_seconds) as duration_stddev
FROM system.lakeflow.job_run_timeline
WHERE period_start >= date_sub(current_date(), 7)
GROUP BY job_id
```
- Alert if today's duration > (7-day avg + 2 × stddev)

**Layer 3: Warehouse Performance** (SQL SLA)
```sql
SELECT warehouse_id,
       PERCENTILE(total_duration_ms, 0.95)/1000 as p95_query_seconds,
       SUM(CASE WHEN queue_duration_ms > 30000 THEN 1 ELSE 0 END) as long_queue_count
FROM system.query.history
WHERE start_time >= current_timestamp() - INTERVAL 1 HOUR
GROUP BY warehouse_id
```
- Alert: p95 > 60s or long_queue_count > 10 → scale up warehouse proactively

**Layer 4: Cost Anomaly Detection**
- Compare daily DBU consumption to 30-day rolling average
- Alert: Daily spend > 120% of average

**Layer 5: AWS Infrastructure Metrics**
- CloudWatch: Spot interruption notices → pre-warm fallback On-Demand
- S3 request rate spike → small file problem
- EBS volume utilization → alert at 80%

**Layer 6: Alerting & dashboards**
- Central Grafana dashboard aggregating metrics across all workspaces
- PagerDuty integration: P1 (cluster down), P2 (degradation), P3 (cost)
- Weekly digest email: Top 5 slow jobs, top 5 DBU spenders, security events

**Layer 7: Chaos testing**
- Monthly: Simulate cluster failure → verify auto-recovery
- Quarterly: Simulate regional failover → measure actual RTO

---

### Q27: Compliance Audit Reporting - Data Access Trails

**Data Sources:**
- `system.access.audit` - all data access events
- `system.access.column_lineage` - column-level tracking
- `system.access.table_lineage` - table-level tracking
- AWS CloudTrail - S3 API, IAM events, KMS usage

**Step 1: Centralized audit log collection**
- All workspaces deliver to: `s3://central-audit-logs/databricks/`
- Path: `/databricks/<workspace_id>/date=YYYY-MM-DD/`
- Create Delta table on top of logs:
  ```sql
  CREATE TABLE audit_db.databricks_audit
  USING DELTA LOCATION 's3://central-audit-logs/databricks/'
  PARTITIONED BY (workspace_id, date)
  ```

**Step 2: Standard compliance queries**

Who accessed PII?
```sql
SELECT user_identity.email, resource_name, action_name, event_time, source_ip_address
FROM system.access.audit a
JOIN pii_tables_registry p ON a.resource_name = p.table_full_name
WHERE event_time >= date_trunc('month', current_date())
AND action_name IN ('READ', 'MODIFY')
ORDER BY event_time DESC
```

Failed access attempts:
```sql
SELECT user_identity.email, resource_name, action_name, error_message, event_time
FROM system.access.audit
WHERE status_code != 200 AND action_name IN ('READ', 'WRITE')
AND event_time >= date_sub(current_date(), 30)
```

Admin privilege usage:
```sql
SELECT user_identity.email, action_name, request_params, event_time
FROM system.access.audit
WHERE action_name IN ('createUser', 'deleteUser', 'updatePermissions', 'grantPrivilege')
AND event_time >= date_sub(current_date(), 30)
```

**Step 3: Automated report generation**
- Databricks Workflow: Runs 1st of month, 6AM
- Steps:
  1. Run compliance SQL → save to Delta tables
  2. Python notebook: Generate Excel with openpyxl
     - Sheet 1: Executive summary
     - Sheet 2: PII access log
     - Sheet 3: Failed access attempts
     - Sheet 4: Admin actions
  3. Upload to S3: `s3://compliance-reports/databricks-audit-YYYY-MM.xlsx`
  4. Email via SES to compliance@company.com, cc CISO

**Step 4: Real-time alerting**
- Databricks SQL Alert (check every 15 min):
  ```sql
  SELECT COUNT(*) FROM system.access.audit
  WHERE action_name = 'deleteCatalog'
  AND event_time >= current_timestamp() - INTERVAL 15 MINUTES
  ```
- Alert if > 0 → PagerDuty page to admin

- Other real-time alerts:
  - Mass data export (> 10M rows read in 1 hour)
  - Cross-region data access
  - PII access outside business hours

---

## SECTION 10: Databricks Runtime & Upgrade Scenarios

### Q28: Migrate 300+ Jobs from DBR 10.4 LTS to DBR 14.3 LTS

**Phase 1: Assessment (Days 1-20)**
```sql
SELECT cluster_id, runtime_engine, creator_user_name, job_name
FROM system.compute.clusters
WHERE spark_version LIKE '%10.4%'
```

- Categorize by risk:
  - **High risk:** Deprecated APIs, custom Hadoop configs, Scala 2.12 custom JARs
  - **Medium risk:** Complex Spark SQL, custom UDFs
  - **Low risk:** Simple PySpark ETL, SQL-only

**Key Breaking Changes (10.4 → 14.3):**
- Python: 3.8 → 3.10 (check library compatibility)
- Spark: 3.2 → 3.5 (check deprecation warnings in 3.3, 3.4, 3.5 release notes)
- Delta Lake protocol changes
- Default configs may differ

**Phase 2: Testing Environment (Days 15-40)**
- Migrate LOW RISK jobs first
- Run parallel execution: Same job on 10.4 AND 14.3 for 1 week
- Compare outputs:
  ```python
  df_old = spark.read.table('output_dbr104')
  df_new = spark.read.table('output_dbr143')
  assert df_old.count() == df_new.count(), 'Row count mismatch!'
  assert df_old.subtract(df_new).count() == 0, 'Data mismatch!'
  ```

**Phase 3: Library Compatibility (Days 20-50)**
- Verify each Python package supports Python 3.10
- Update packages if needed (e.g., XGBoost, TensorFlow)
- Create standard init script with pinned, tested versions

**Phase 4: Staged Migration (Days 40-80)**
- **Week 1:** Low-risk jobs (100 jobs)
- **Week 2:** Medium-risk jobs (150 jobs) with parallel validation
- **Week 3:** High-risk jobs with dedicated testing, JAR recompilation
- **Week 4:** Buffer for fixes

**Phase 5: Cutover (Days 80-90)**
- Remaining jobs migrated
- Old DBR 10.4 cluster policy deprecated
- DBR 14.3 becomes default
- Monitor for 2 weeks: Alert if failure rate increases

**Rollback Plan:** Keep 10.4 as fallback for 30 days post-migration. Any failing job can immediately revert by changing one line in job config.

---

## Interview Preparation Pro Tips

### The STAR Format
**Situation** → **Technical Analysis** → **Action Taken** → **Result**

Interviewers at senior level assess your systematic thinking, not just knowledge. Use this structure for every answer.

### Key Mindsets to Demonstrate

1. **Root Cause First** - Don't jump to fixes without understanding why
2. **Production Awareness** - Think about impact on users, SLAs, data integrity
3. **Prevention** - Always include governance, automation, and monitoring
4. **Documentation** - Show you leave an audit trail and knowledge base
5. **Escalation Paths** - Know when to involve security, finance, legal teams

### Common Interview Pitfalls to Avoid

- ❌ Suggesting dangerous operations (like VACUUM without thinking through implications)
- ❌ Proposing cost solutions that break compliance
- ❌ Overcomplicating simple scenarios
- ❌ Not considering blast radius of changes
- ❌ Forgetting about data validation after fixes

### Questions to Ask During Interview

1. "What's the current state of your Databricks infrastructure?" (Scale, workspaces, UC adoption)
2. "How are you currently handling security and compliance?" (Shows you think about governance)
3. "What's your biggest operational pain point?" (Shows you understand priorities)
4. "What's your disaster recovery posture?" (Critical for any platform)

### Study Tips

1. **Understand the "why" behind every recommendation** - Don't memorize answers
2. **Review Databricks documentation** for your specific DBR version
3. **Practice with the AWS console** - Get comfortable with EC2, S3, IAM, KMS
4. **Study Spark internals** - Memory management, shuffle, serialization
5. **Read recent Databricks blog posts** on product updates and best practices

---

## Quick Reference: Critical Commands & Configs

### Debugging Commands

```sql
-- Check cluster history
SELECT cluster_id, event_type, timestamp FROM system.compute.cluster_logs

-- View DBU consumption
SELECT workspace_id, cluster_id, SUM(dbus) FROM system.billing.usage

-- Find PII tables
SELECT table_schema, table_name FROM system.information_schema.columns 
WHERE column_name LIKE '%ssn%' OR column_name LIKE '%card%'

-- Check access audit
SELECT * FROM system.access.audit WHERE event_time >= date_sub(current_date(), 30)

-- DLT event log
SELECT * FROM <pipeline_storage>.event_log WHERE event_type = 'flow_progress'

-- Table history
DESCRIBE HISTORY table_name
```

### Critical Spark Configs

```
spark.executor.memoryOverhead = 4096          # For memory issues
spark.memory.fraction = 0.8                   # Execution vs storage
spark.sql.adaptive.enabled = true             # Enable AQE
spark.sql.shuffle.partitions = 200            # Tune based on data size
spark.databricks.io.cache.enabled = true      # Enable Delta Cache
spark.databricks.delta.merge.enableLowShuffle.merge = true  # MERGE optimization
```

### Cluster Policy Examples

```json
{
  "instance_profile_arn": {
    "type": "allowlist",
    "values": ["arn:aws:iam::...]"
  },
  "aws_attributes": {
    "availability": "SPOT_WITH_FALLBACK",
    "first_on_demand": 1
  },
  "spark_conf": {
    "spark.databricks.delta.merge.enableLowShuffle.merge": {
      "value": "true"
    }
  },
  "autotermination_minutes": 120,
  "num_workers": {
    "max_value": 20
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["m5.xlarge", "r5.xlarge"]
  }
}
```

---

**Last Updated:** March 2026  
**Total Scenarios:** 28 | **Estimated Study Time:** 15-20 hours  
**Expected Interview Duration:** 45-60 minutes per scenario

Good luck! Remember: Understanding the "why" is more important than memorizing the "what."
