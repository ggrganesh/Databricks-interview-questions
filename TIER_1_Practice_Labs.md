# Tier 1: Hands-On Practice Labs & Diagnostic Scenarios

**15 Real-World Problems to Solve**
*Test your problem-solving skills (answers provided at end)*

---

## LAB 1: Memory Explosion in ETL Job

### Scenario
A data engineer reports that their ETL job that normally completes in 30 minutes is now taking 4 hours. They say "nothing changed" but they did a git pull yesterday.

**Given Information:**
- Job: `daily_customer_etl`
- Cluster: r5.4xlarge, 4 workers (128GB total RAM)
- Runtime: 4 hours (was 30 minutes)
- Error: Job completes (no error) but very slow
- Data size: 50GB input, 100GB output

**Your Task:** Diagnose the problem in 5 minutes.

### What You Should Do:
1. What's your first question?
2. What metrics would you check?
3. What logs would you examine?
4. What's the most likely cause?
5. How would you fix it?

---

## LAB 2: Cluster Won't Start (Security)

### Scenario
After enabling PrivateLink yesterday, clusters in production workspace can no longer start. Error: "Failed to reach control plane". Other workspaces (without PrivateLink) work fine.

**Given Information:**
- Workspace: prod-primary
- PrivateLink: Just enabled 24 hours ago
- Error: "Failed to reach control plane" 
- Timeout: 5 minutes waiting for driver node
- EC2 instances: Launch fine, but can't connect to cluster UI

**Your Task:** Identify the root cause in 5 minutes.

### What You Should Do:
1. What are the two VPC endpoints needed for PrivateLink?
2. What's likely misconfigured?
3. Where would you check security group rules?
4. What DNS resolution should you verify?
5. Write the fix steps.

---

## LAB 3: Unauthorized S3 Access Detected

### Scenario
Security team alerts: CloudTrail shows cluster nodes accessing S3 bucket they shouldn't have access to. Bucket: `analytics-sensitive` contains PII data.

**Given Information:**
- Cluster: data-science-cluster-123 (r5.2xlarge)
- Accessed bucket: s3://analytics-sensitive (DENY, only finance should access)
- Time: 2024-03-05 14:23 UTC
- Action: GetObject (reading files)
- Number of objects: 1,247 files accessed
- Data scientist: john.doe@company.com

**Your Task:** Create incident response plan (60 seconds per minute of incident).

### What You Should Do:
- Minute 0-10: Contain the breach
- Minute 10-20: Investigate
- Minute 20-60: Remediate
- Post-incident: Prevent recurrence

---

## LAB 4: MERGE Taking 6 Hours (Should Be 45 Minutes)

### Scenario
Financial team's daily reconciliation MERGE is taking 6 hours. It used to take 45 minutes. They're about to miss deadline for EOD reporting.

**Given Information:**
```sql
MERGE INTO transactions_daily t
USING new_transactions s
ON t.transaction_id = s.transaction_id
WHEN MATCHED THEN UPDATE SET amount = s.amount, status = s.status
WHEN NOT MATCHED THEN INSERT *
```

- Target table: 500GB, 2 billion rows
- Source: 100MB, 5 million rows
- Ran yesterday in 45 minutes
- Today: 6 hours (8x slower)
- No data size increase

**Your Task:** Identify bottleneck in 10 minutes.

### What You Should Do:
1. What DESCRIBE HISTORY would tell you?
2. What metrics matter for MERGE?
3. What's the most common MERGE problem?
4. What's your first fix to try?
5. What should you check for data skew?

---

## LAB 5: Jobs Failing with "No Space Left on Device"

### Scenario
Overnight batch job fails: "No space left on device". This happened to 15 clusters simultaneously. Friday they all ran fine.

**Given Information:**
- 50 jobs scheduled to run 10PM-6AM
- 15 jobs failed with disk full error
- Time: Saturday 2AM
- All failures in same 1-hour window
- Error message: `/databricks/spark/logs: No space left on device`

**Your Task:** Determine root cause and immediate fix.

### What You Should Do:
1. Where did the disk space go?
2. Why did 15 clusters all fill up at once?
3. What's the immediate fix (< 5 min)?
4. What's the long-term prevention?
5. How would you calculate disk usage per workspace?

---

## LAB 6: Data Quality Test Failing (Silent Failure)

### Scenario
Delta Live Tables pipeline shows "COMPLETED" status, but downstream team reports aggregation numbers are wrong for 3 days straight.

**Given Information:**
```python
@dlt.expect('positive_amount', 'amount > 0')
@dlt.expect('valid_date', 'date BETWEEN '2024-01-01' AND current_date()')
def transactions():
    return spark.read.table('raw.transactions')
```

- Pipeline status: COMPLETED ✓
- No errors in logs
- Data quality metrics: Not checked
- Downstream: Dashboard showing wrong totals

**Your Task:** Identify the silent failure.

### What You Should Do:
1. What's different about expect() vs expect_or_fail()?
2. Where would you look for dropped/failed records?
3. How would you find when the issue started?
4. What SQL would validate the data?
5. How would you prevent this in future?

---

## LAB 7: SQL Warehouse Queue Growing (During Business Hours)

### Scenario
BI team complains: "Dashboards loading slow during business hours (9AM-5PM)". Same queries are fast at 6PM+.

**Given Information:**
- SQL Warehouse: 10 clusters (Classic)
- Peak utilization: 8 concurrent queries
- Queue time: 45 seconds during peak
- At 6PM: Queue time < 2 seconds
- Warehouse size: Doesn't change automatically

**Your Task:** Solve the slow dashboard problem without much cost increase.

### What You Should Do:
1. What metrics prove the warehouse is undersized?
2. What's the cheapest fix?
3. How would you size the warehouse correctly?
4. What's the alternative to scaling up?
5. What would you measure to validate the fix?

---

## LAB 8: Cross-Region Replication Lag Growing

### Scenario
DR testing: S3 cross-region replication lag is growing. Started at 10 minutes, now 3 hours. Need 15-minute RPO for RTO < 1 hour.

**Given Information:**
- Source: us-east-1 DBFS bucket (1GB/day writes)
- Target: us-west-2 (replicated)
- Replication lag: Started 10 min, now 3+ hours
- Time span: Last 3 days
- No recent changes noted

**Your Task:** Diagnose replication lag.

### What You Should Do:
1. What AWS metric shows replication lag?
2. What could cause lag to increase suddenly?
3. How would you check S3 request rate?
4. What's the issue most likely to be?
5. What's the fix?

---

## LAB 9: DBU Spend Tripled Overnight

### Scenario
CFO alerts: "DBU spend went from $20K/day to $60K/day overnight. No new workloads were added."

**Given Information:**
- Yesterday: $20K/day
- Today: $60K/day (3x increase)
- No PRs merged with new jobs
- No marketing campaign started
- No new team members added

**Your Task:** Find the cost spike in 30 minutes.

### What You Should Do:
1. What's your first diagnostic query?
2. Where would you look in system.billing.usage?
3. What could cause 3x cost increase?
4. How would you identify the culprit cluster?
5. How would you prevent this in future?

---

## LAB 10: New User Can't See Any Clusters

### Scenario
Onboarded data scientist: "I see the workspace but can't attach to any clusters. Everyone else can."

**Given Information:**
- User: alice@company.com
- Status: Just added to workspace
- SCIM: Synced from Okta
- SCIM completed: 30 minutes ago
- Workspace admins: All report fine
- Cluster count: 20 clusters available

**Your Task:** Get alice attached to a cluster in < 10 minutes.

### What You Should Do:
1. What's your troubleshooting order?
2. What permission is most commonly missing?
3. Where would you check ACLs?
4. What's the quickest fix vs right fix?
5. How would you prevent this recurring?

---

## LAB 11: ETL Pipeline Running 24/7 (Unexpected Cost)

### Scenario
Cost analysis shows Delta Live Tables pipeline consuming more DBU than all other workloads combined. It was supposed to run daily, but team says "it's supposed to be on 24/7 now."

**Given Information:**
- Pipeline: customer-360
- Mode: CONTINUOUS (running 24/7)
- Data arrival: 100 new records/day
- Cost: $30K/month (62% of total spend)
- Team decision: "We wanted real-time, so continuous"

**Your Task:** Optimize the cost.

### What You Should Do:
1. What's the difference between CONTINUOUS and TRIGGERED?
2. Is CONTINUOUS really needed for 100 records/day?
3. What's the cost-optimal solution?
4. What's the trade-off (latency vs cost)?
5. How would you measure the improvement?

---

## LAB 12: GDPR Erasure Request - Delete User from 50 Tables

### Scenario
Customer requests right to erasure (GDPR). User found in 50 Delta tables across 8 schemas. Must delete within 30 days.

**Given Information:**
```sql
-- User to delete: customer_id = 12345
-- Found in tables:
-- orders, order_items, payments, refunds,
-- customer_profile, customer_demographics, ...
-- (47 more tables)
```

- Deletion deadline: 30 days
- Tables: 50 across 8 schemas
- User ID: customer_id = 12345
- Urgency: GDPR compliance required

**Your Task:** Design safe deletion process.

### What You Should Do:
1. What's the danger of VACUUM here?
2. What's crypto-shredding?
3. How would you safely delete without VACUUM?
4. What about Delta Lake time travel?
5. How would you audit the deletion?

---

## LAB 13: Query Slow (What's the Plan?)

### Scenario
BI dashboard query taking 2 minutes. Should take 10 seconds. Same query ran fast yesterday.

```sql
SELECT 
  customer_region,
  SUM(order_amount) as total,
  COUNT(*) as order_count
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= date_sub(current_date(), 90)
GROUP BY customer_region
```

**Given Information:**
- `orders`: 2B rows, 500GB
- `customers`: 100M rows, 50GB
- Query time: 2 min (should be 10 sec)
- No errors, just slow
- Ran fine yesterday

**Your Task:** Optimize the query.

### What You Should Do:
1. How would you get the EXPLAIN plan?
2. What does EXPLAIN tell you about joins?
3. What's likely the problem?
4. What would you add to the query?
5. How would you verify the fix?

---

## LAB 14: Compliance Audit - Prove Encryption

### Scenario
SOC 2 auditor asks: "Prove all data is encrypted at rest with customer-managed keys."

**Given Information:**
- Multiple databases in us-east-1
- S3 DBFS bucket: s3://databricks-prod
- RDS instance: prod-metastore
- EBS volumes: All cluster volumes
- Backup storage: 3 years of backups

**Your Task:** Provide evidence package.

### What You Should Do:
1. What would you show for S3 encryption?
2. How would you prove KMS CMK is used?
3. What about EBS volumes?
4. How would you show key rotation?
5. What about backups?

---

## LAB 15: Regional Failover (Simulate 60-Minute Recovery)

### Scenario
You receive alert: "us-east-1 Databricks workspace unavailable." RPO: 15 min, RTO: 1 hour.

**Given Information:**
- Primary: us-east-1 prod-primary workspace
- DR: us-west-2 prod-dr workspace (standby)
- Data: Replicated via S3 CRR (lag: 10 min)
- RDS: Read replica in us-west-2
- Jobs: Stored in git
- Current time: 14:00 UTC (incident declared)

**Your Task:** Execute failover in 60 minutes.

### What You Should Do:
- Minute 0-10: What's your first action?
- Minute 10-30: What do you validate?
- Minute 30-50: How do you activate DR?
- Minute 50-60: How do you verify?
- What metrics prove success?

---

---

## SOLUTIONS & EXPLANATIONS

### LAB 1 Solution: Memory Explosion in ETL Job

**Diagnosis:**
The job is slow but completes = likely memory pressure (spilling to disk) not an OOM crash.

**Root Cause:**
"Nothing changed" + "git pull yesterday" = Code change. Most likely:
1. New nested JSON columns in schema (increases deserialization memory)
2. New library dependency with larger footprint
3. Changed JOIN logic adding extra dataframes in memory

**How to Find:**
1. Check `git log` for schema or code changes
2. Spark UI > Executor tab > Check "Spill (Memory)" and "Spill (Disk)"
   - If Spill (Disk) >> 0, you're spilling to disk = slow
3. Check GC time: If > 10% of task time = memory pressure
4. Review Spark UI > Storage tab > Peak Execution Memory

**Fix (Priority Order):**
1. **Immediate:** Increase executor memory overhead
   ```
   spark.executor.memoryOverhead = 4096 (from default 10%)
   spark.memory.fraction = 0.8
   ```

2. **Medium-term:** Check git diff for schema changes
   ```sql
   -- If new nested columns:
   SELECT * FROM table LIMIT 1  -- Check schema
   ```

3. **Long-term:** Add AQE to auto-coalesce partitions
   ```
   spark.sql.adaptive.enabled=true
   ```

---

### LAB 2 Solution: Cluster Won't Start (PrivateLink)

**Root Cause:**
PrivateLink requires TWO VPC endpoints:
1. **Workspace endpoint** - For UI and REST API
2. **Relay endpoint** - For cluster nodes (SCC tunnel)

Most likely: **Missing relay endpoint OR private DNS not enabled**

**Diagnosis Steps:**
1. **Check VPC Endpoints:**
   - AWS Console > VPC > Endpoints
   - Should see TWO endpoints for Databricks
   - If only one: That's the problem!

2. **Check Private DNS:**
   - Click each endpoint > Details
   - "Enable Private DNS" should be = YES
   - If NO: Clusters can't resolve endpoint

3. **Check Security Groups:**
   - Relay endpoint SG should allow inbound HTTPS (443)
   - From: Cluster security group

**Fix (Order):**
1. If missing relay endpoint: Create it (5 min)
2. If private DNS disabled: Enable it (2 min)
3. If security group wrong: Add inbound rule from cluster SG (5 min)
4. Restart cluster (5 min)

**Validation:**
```bash
# From cluster node, should resolve to private IP
nslookup workspace.cloud.databricks.com
# Should return: 10.x.x.x (not public IP)
```

---

### LAB 3 Solution: Unauthorized S3 Access

**Incident Response (60 Minutes):**

**Minute 0-10: Contain**
```bash
# 1. Terminate cluster immediately
POST /api/2.0/clusters/delete {"cluster_id": "data-science-cluster-123"}

# 2. Add DENY policy to S3 bucket (takes effect in seconds)
# AWS Console > S3 > analytics-sensitive > Edit bucket policy
{
  "Effect": "Deny",
  "Principal": {"AWS": "arn:aws:iam::ACCOUNT:role/DataScienceRole"},
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::analytics-sensitive/*"
}
```

**Minute 10-20: Investigate**
```bash
# 1. Check IAM role attached to cluster
# AWS Console > EC2 > Instance > IAM Instance Profile
# Should show what role was used

# 2. Check cluster's IAM policy
# AWS Console > IAM > Roles > [role-name]
# Does it have s3:* permissions? That's the problem!

# 3. Check CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=analytics-sensitive \
  --max-results 10
```

**Minute 20-60: Remediate**
```
1. Review IAM policy - Is it too permissive? (likely yes)
2. Update policy: Specific buckets only, not s3:*
3. Add permission boundary to all cluster roles
4. Cluster policy: Restrict which IAM roles can be used
5. Enable Databricks UC external locations (credential vending)
```

**Post-Incident Prevention:**
1. Implement least-privilege IAM policies
2. Cluster policy enforces allowed IAM roles
3. UC external locations for S3 access (not direct IAM)
4. CloudTrail alert on any s3:* permissions

---

### LAB 4 Solution: MERGE Taking 6 Hours

**Root Cause Analysis:**

```sql
-- Check table history
DESCRIBE HISTORY transactions_daily;
-- Look at: numFiles, numDeletedFiles, sizeInBytes

-- If numFiles > 100K = Small file problem!
-- MERGE re-scans entire table → with many small files = slow
```

**Most Likely:** Small files accumulated (numFiles grew from 1K to 500K files)

**Fixes (Priority Order):**

1. **Immediate (Try first):**
   ```sql
   OPTIMIZE transactions_daily ZORDER BY (transaction_id);
   -- Consolidates small files into ~128MB chunks
   -- Expected: 500K files → 4K files
   -- Then re-run MERGE (should be 45 min again)
   ```

2. **Second (If still slow):**
   ```sql
   -- Add partition pruning to MERGE
   MERGE INTO transactions_daily t
   USING new_transactions s
   ON t.transaction_id = s.transaction_id 
   AND t.date_partition >= DATE_SUB(CURRENT_DATE, 1)  -- NEW
   ```

3. **Third (If still slow):**
   ```sql
   -- Enable deletion vectors (DBR 12.2+)
   SET spark.databricks.delta.merge.enableLowShuffle.merge = true;
   ```

4. **Verify fix:**
   ```sql
   -- Should be < 1 hour now
   DESCRIBE HISTORY transactions_daily;
   -- numFiles should be ~4K (not 500K)
   ```

---

### LAB 5 Solution: Disk Full Errors

**Root Cause:**
Jobs ran 10PM-6AM. At 2AM (8 hours later), clusters filled up. Why?

```
Most likely: Long-running job didn't clean up shuffle files on driver
├─ Job 1 writes 100GB shuffle files → Fails
├─ Job 2 writes 100GB shuffle files → Accumulates
├─ ...
└─ By 2AM: 15 × 100GB = 1.5TB on driver disk (which is 500GB) = FULL!
```

**Immediate Fix (< 5 min):**
```bash
# 1. Restart all clusters (clears /local_disk0)
# 2. Disable event logging temporarily
spark.eventLog.enabled=false
```

**Long-term Prevention:**

```sql
-- 1. Use ephemeral job clusters (fresh per job)
-- Cluster terminates after job → Disk always clean

-- 2. Enable external shuffle service
spark.shuffle.service.enabled=true
-- Shuffle files live on worker nodes (not driver)

-- 3. Enable log rotation
spark.eventLog.rolling.enabled=true
spark.eventLog.rolling.maxFileSize=128m
-- Prevents event logs growing unbounded

-- 4. CloudWatch alarm
-- EBS utilization > 80% → Alert

-- 5. Use i3/i3en instances
-- 3.8TB+ local NVMe SSD (not EBS)
```

---

### LAB 6 Solution: Silent DLT Failure

**Root Cause:**
```python
@dlt.expect('positive_amount', 'amount > 0')
# This LOGS violations but CONTINUES (WARN mode)
# ≠ expect_or_fail() which FAILS the pipeline
```

Solution shows wrong numbers because:
- Some rows have amount ≤ 0 (violates expectation)
- Rows were written to table (not dropped)
- Downstream SUM includes negative amounts = wrong total!

**How to Find:**

```sql
-- Query DLT event log
SELECT * FROM databricks.default.event_log
WHERE event_type = 'flow_progress' 
  AND details:flow_progress:data_quality IS NOT NULL
ORDER BY timestamp DESC;

-- Look for: failed_records count > 0
```

**Fix (3 changes):**

1. **Change expectation to fail_or_fail:**
   ```python
   @dlt.expect_or_fail('positive_amount', 'amount > 0')
   # Now pipeline fails if any violations
   ```

2. **Run Full Refresh to reprocess:**
   ```
   DLT UI > Pipeline > Full Refresh
   # Reprocesses all historical data
   ```

3. **Add reconciliation job:**
   ```sql
   -- Daily: Check row counts match
   SELECT COUNT(*) FROM source.transactions;
   SELECT COUNT(*) FROM dlt.transactions;
   -- Alert if counts don't match
   ```

---

### LAB 7 Solution: SQL Warehouse Queue Growing

**Diagnosis:**
```
9AM-5PM: Queue time = 45 sec
6PM-9AM: Queue time = 2 sec
→ Peak hours, warehouse is undersized
```

**Root Cause:**
Warehouse has 10 clusters (static). At peak: 8 concurrent queries + 2 more arrive → Queue builds up.

**Solutions (by cost impact):**

1. **Cheapest (Change mode):**
   ```
   Switch from Classic to Serverless SQL warehouse
   - Auto-scales: 0 → 100 clusters on demand
   - Idle periods: Cost = $0
   - Save: ~30% (because of idle utilization)
   ```

2. **Simple (Add capacity):**
   ```
   Increase max clusters: 10 → 15
   Cost increase: 50% (5 extra idle clusters)
   But queue time solved
   ```

3. **Optimal (Auto-scaling on Classic):**
   ```
   Min clusters: 1 (off-hours)
   Max clusters: 15 (peak hours)
   Auto-scale trigger: Queue depth > 0 for 2 min
   Result: Dynamically scales to demand
   ```

4. **Best (Optimize queries):**
   ```sql
   -- Enable Query Results Cache
   SELECT * FROM table -- Fast if cached
   
   -- Enable Photon (SQL vectorization)
   -- Result: 2-5x faster queries = less need for clusters
   ```

**Validation Metrics:**
```
After fix, should see:
├─ Queue time: < 5 seconds
├─ P95 query latency: < 20 seconds
└─ Cost: -10% to -30% depending on solution
```

---

### LAB 8 Solution: CRR Lag Growing

**Diagnosis:**

```bash
# Check S3 replication metrics
# AWS S3 Console > Source bucket > Replication
# Metric: Maximum replication time

# If lag is growing:
# Minute 1: 10 min (normal)
# Minute 2: 30 min (growing)
# Minute 3: 3+ hours (very behind)

# Why? Something is writing to source faster than S3 can replicate
```

**Root Cause:** Something changed 3 days ago
```
Possible causes:
├─ New job writing 10x more data to DBFS
├─ New streaming job writing continuously
├─ Backup job moved to same bucket
└─ S3 throttling (rate limit on destination bucket)
```

**How to Find:**

```sql
-- Check DBU usage by workspace (past 3 days)
SELECT usage_date, SUM(dbus)
FROM system.billing.usage
WHERE usage_date >= DATE_SUB(CURRENT_DATE, 3)
GROUP BY usage_date
ORDER BY usage_date;

-- If 3 days ago: spike in DBU
-- → Some new workload added

-- Check S3 request rate
-- CloudWatch > Metric: PutObject on source bucket
-- If spiked 3 days ago: Confirms hypothesis
```

**Fix:**

```
Option 1: Increase replication capacity
├─ Move to higher-tier replication (not available)

Option 2: Change replication strategy
├─ From: Continuous CRR (real-time)
└─ To: Daily batch sync (acceptable for non-critical data)
    Savings: 80% transfer cost
    Trade-off: RPO = 24 hours (vs 15 min)

Option 3: Shard by prefix
├─ Instead of replicating all:
│  ├─ Replicate /prod → s3://dest-prod (real-time)
│  └─ Replicate /dev → Daily batch
└─ Result: Only critical data has 15-min RPO
```

---

### LAB 9 Solution: DBU Spend Tripled

**Diagnosis Query:**

```sql
-- Find top DBU consumers
SELECT 
  cluster_id,
  SUM(dbus) as total_dbus,
  COUNT(*) as run_count,
  AVG(dbus) as avg_dbu_per_run
FROM system.billing.usage
WHERE usage_date = CURRENT_DATE
GROUP BY cluster_id
ORDER BY total_dbus DESC
LIMIT 10;

-- OR by creator:
SELECT
  creator_user_name,
  COUNT(*) as num_clusters,
  SUM(dbus) as total_dbus
FROM system.billing.usage
WHERE usage_date = CURRENT_DATE
GROUP BY creator_user_name
ORDER BY total_dbus DESC;
```

**Most Common Causes (in order):**
1. Someone left a large cluster running (r5.24xlarge = 10x cost)
2. New batch job running (that wasn't running before)
3. GPU cluster spun up (10x DBU cost)
4. Job retry storm (failing job retrying 100x)

**Find It:**

```bash
# Check for long-running clusters (> 8 hours)
# Databricks UI > Clusters > Sort by "Up time"
# Any cluster > 480 minutes?

# Check EC2 (via AWS Console)
# EC2 > Instances > Filter by tag: Vendor=Databricks
# Are there new instances with expensive types (r5.24xlarge)?

# Check for failed jobs (CloudWatch logs)
# Any job retrying excessively?
```

**Prevention:**

```sql
-- Cluster policy
├─ Max instance: r5.2xlarge (not r5.24xlarge)
├─ Auto-terminate: 120 minutes (no overnight idling)
└─ GPU: Forbidden (unless approved)

-- CloudWatch alert
├─ Daily spend > 150% of average → Alert
└─ Anomaly detection on DBU trend
```

---

### LAB 10 Solution: New User Can't See Clusters

**Troubleshooting Order (Most to Least Common):**

```
1. CLUSTER ACLs (60% of cases)
   ├─ Admin Console > Clusters > Permissions
   ├─ User needs: "Can Attach To"
   └─ Fix: Grant permission

2. WORKSPACE ENTITLEMENT (30% of cases)
   ├─ Admin Console > Users > Find alice
   ├─ Entitlement: Should be "Can Use" (not "Can View only")
   └─ Fix: Grant "Can Use"

3. SCIM SYNC LAG (5% of cases)
   ├─ SCIM syncs every 40 minutes
   ├─ Just added: May not be synced yet
   └─ Fix: Wait 30 min OR manually add in Admin Console

4. CLUSTER POLICY (5% of cases)
   ├─ If policy forbids user from seeing clusters
   └─ Fix: Assign cluster policy with appropriate visibility
```

**Quick Fix (< 10 min):**

```
1. Admin Console > Clusters > Select any cluster
2. Permissions tab > Add "Can Attach To" for alice
3. Alice should now see cluster
```

**Right Fix (Prevents recurrence):**

```
1. Create shared cluster
2. Grant "Can Attach To" to all users (or group)
3. Cluster policy: Limits instance types
4. Result: All users can self-serve without manual permission

OR use Workspace Admin Groups:
1. Admin Console > Groups
2. Add alice to group "Data Scientists"
3. Grant group "Can Attach To" on shared cluster
4. Result: All future data scientists auto-get access
```

---

### LAB 11 Solution: ETL Running 24/7

**Diagnosis:**

```python
# Pipeline mode can be:
@dlt.compute()  # Default
├─ Triggered mode: Runs only when explicitly triggered
├─ Cost: Low (runs ~daily)

@dlt.compute()  # With CONTINUOUS
├─ Continuous mode: Runs 24/7
├─ Cost: High (100x cost vs triggered)
└─ Good for: True real-time (< 1 second latency)
```

**Root Cause:**
Team wanted "real-time" → Set to CONTINUOUS → Became 24/7 cluster

**Cost Analysis:**

```
CONTINUOUS (24/7):
├─ Cluster runtime: 24 × 365 = 8,760 hours/year
├─ Cost: 8,760 hrs × $1.5/hr = $13,140/year = $1,095/month

TRIGGERED (1x/day):
├─ Cluster runtime: 1 × 365 = 365 hours/year
├─ Cost: 365 hrs × $1.5/hr = $548/year = $46/month

SAVINGS: $1,049/month (96% reduction!)
```

**Question:**
For 100 new records/day, do you need continuous (< 1 sec latency) or triggered (daily) mode?

**Answer:** Triggered mode
- 100 records/day = 1 record every 864 seconds
- No business case for sub-second latency
- Triggered mode once/day is sufficient

**Implementation:**

```python
# Change from:
# DLT UI: CONTINUOUS mode

# To:
# DLT UI: TRIGGERED mode

# Schedule daily run:
# Databricks Job: Run pipeline at 7 AM daily
# Expected latency: 5 minutes old data at 7 AM

# If real-time needed in future:
# Add streaming source (Kafka, Event Hub)
# Keep pipeline TRIGGERED, source continuous
```

---

### LAB 12 Solution: GDPR Erasure (50 Tables)

**Challenge:**
Delete customer from 50 tables without exposing to data recovery via time travel.

**Problem:**
```sql
DELETE FROM orders WHERE customer_id = 12345;
-- This deletes row from current table
-- But VACUUM doesn't run automatically
-- AND old versions still contain the row
-- GDPR requirement: Data must be unrecoverable
```

**Solution Options:**

**Option A: VACUUM (Risky but Simple)**
```sql
DELETE FROM orders WHERE customer_id = 12345;
VACUUM orders RETAIN 0 HOURS;
-- WARNING: This deletes ALL history for ALL customers!
-- Use only if you want to destroy all time travel for all users
```

**Option B: Crypto-Shredding (Best for GDPR)**

```python
# Before: Encrypt sensitive columns
# When encrypting originally:
encrypted_customer_id = kms.encrypt(
  key_id="arn:aws:kms:...",
  plaintext=customer_id
)

# To erase:
# 1. Delete KMS key (or schedule deletion)
# 2. Historical data remains, but unreadable (no key)
# 3. Current data has key deleted
# 4. Perfect for GDPR: Data technically exists but not recoverable

# Implementation:
# - Store customer_id encrypted with per-customer KMS key
# - On erasure request: Schedule KMS key deletion (7 day minimum)
# - After 7 days: Key deleted, all historical data unreadable
```

**Option C: Complete Deletion (Safest for GDPR)**

```sql
-- For each table:
1. DELETE FROM table WHERE customer_id = 12345;
2. Wait for next OPTIMIZE cycle (or run manually)
3. Only then is data truly gone from files
4. Set TBLPROPERTIES("delta.deletedFileRetentionDuration"="0 days")
   to reduce retention window

-- Verify deletion:
SELECT COUNT(*) FROM table WHERE customer_id = 12345;
-- Should be 0

-- Audit:
SELECT * FROM system.access.audit 
WHERE user_id = '12345' 
  AND action_name IN ('READ', 'MODIFY')
-- Log the deletion request, user, timestamp
```

**Recommended Approach for GDPR:**
1. Use crypto-shredding (encrypt at ingestion time)
2. On erasure: Delete encryption key
3. Data unreadable without key (satisfies GDPR)
4. Meets 30-day deadline easily
5. Auditable via KMS CloudTrail logs

---

### LAB 13 Solution: Slow Query

**Get the Query Plan:**

```sql
EXPLAIN SELECT * FROM orders ...
-- Output will show:
-- ├─ Scan parquet default.orders [...]
-- ├─ Scan parquet default.customers [...]
-- └─ HashAggregate [sum(amount), count(1)]
--    └─ HashJoin [Inner] (broadcast)
```

**Likely Problem:**
Without hint, Spark chooses: "HashJoin (broadcast)"
- If customers table > broadcast limit (10GB default), falls back to shuffle join
- Shuffle join of 2B + 100M rows = SLOW

**How to Fix:**

```sql
-- Option 1: Add broadcast hint (if customers < 100GB)
SELECT /*+ BROADCAST(c) */ 
  c.customer_region,
  SUM(o.order_amount) as total,
  COUNT(*) as order_count
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= date_sub(current_date(), 90)
GROUP BY c.customer_region
-- Result: Broadcast join (no shuffle) = fast

-- Option 2: Check if table was cached
-- Add to cluster config:
spark.databricks.io.cache.enabled=true
-- Delta Cache caches the table on NVMe SSD
-- Result: Subsequent queries hit cache = fast

-- Option 3: Add statistics (if missing)
ANALYZE TABLE orders COMPUTE STATISTICS;
ANALYZE TABLE customers COMPUTE STATISTICS;
-- Helps Spark optimizer choose better plan
```

**Verify Fix:**

```sql
-- After adding broadcast hint:
EXPLAIN SELECT /*+ BROADCAST(c) */ ...
-- Should show: BroadcastHashJoin (not ShuffledHashJoin)

-- Check execution time:
-- Should go from 2 min → 10 sec (12x faster)
```

---

### LAB 14 Solution: Prove Encryption for SOC 2

**Evidence Package:**

**1. DBFS Root S3 Encryption:**
```
AWS S3 Console:
├─ Bucket: databricks-prod-dbfs
├─ Properties > Default encryption: SSE-KMS ✓
├─ KMS key ARN: arn:aws:kms:us-east-1:ACCOUNT:key/12345-xxx ✓
└─ Screenshot: Take of encryption settings
```

**2. KMS Key Management:**
```
AWS KMS Console:
├─ Key: databricks-root-key
├─ Key rotation: Enabled (365 days) ✓
├─ Key policy: Shows Databricks service role has decrypt permission ✓
├─ CloudTrail: Sample logs showing all KMS operations audited ✓
└─ Screenshot: Show key policy and rotation settings
```

**3. EBS Volume Encryption:**
```
AWS EC2 Console:
├─ Volumes: All Databricks cluster volumes
├─ Encrypted: YES ✓
├─ KMS key: arn:aws:kms:us-east-1:ACCOUNT:key/... ✓
└─ Screenshot: Show multiple volumes all encrypted
```

**4. RDS Encryption:**
```
AWS RDS Console:
├─ Databases: All Databricks metastore RDS instances
├─ Storage encrypted: YES ✓
├─ Encryption type: AWS KMS CMK ✓
└─ Screenshot: Show encryption settings
```

**5. Backup Encryption:**
```
AWS Backup Console:
├─ Backup vaults: All containing Databricks backups
├─ Encryption: Enabled ✓
├─ KMS key: CMK (not AWS managed) ✓
└─ Screenshot: Backup vault encryption settings
```

**6. Audit Trail:**
```
AWS CloudTrail:
├─ Logs: All KMS operations in CloudTrail
├─ Sample: Show KMS:Decrypt and KMS:GenerateDataKey entries
├─ Retention: Logs delivered to S3 bucket ✓
└─ Screenshot: CloudTrail showing KMS operations
```

**Documentation to Provide:**

```
Document 1: Encryption Policy
├─ States: All data encrypted with CMK
├─ Scope: DBFS, EBS, RDS, backups
├─ Rotation: Annual automatic
└─ Signed by: CISO

Document 2: Key Management Policy
├─ Process: How keys are created, rotated, deleted
├─ Access control: Who can use keys (only Databricks service role)
├─ Audit: All key operations logged
└─ Signed by: Security team

Document 3: AWS Config Compliance Report
├─ Rule: s3-bucket-server-side-encryption-enabled (COMPLIANT)
├─ Rule: encrypted-volumes (COMPLIANT)
└─ Date: Compliance check date
```

---

### LAB 15 Solution: Regional Failover (60 Minutes)

**Minute 0-10: Declare & Assess**

```
Action items:
├─ 1. Confirm outage: AWS Service Health Dashboard
│     "us-east-1 Databricks services down"
├─ 2. Page on-call SRE & manager
├─ 3. Open incident room (Slack thread #incident-response)
├─ 4. Start timer (Target: < 60 min to recovery)
└─ 5. Initial message: "Primary us-east-1 unavailable. Initiating failover to us-west-2 DR"
```

**Minute 10-30: Prepare DR**

```
Check replication status:
├─ S3 CRR lag: AWS S3 > Replication metrics
│   Should be: < 15 minutes
│   If > 1 hour: Assess RPO impact
├─ RDS sync: RDS Console > prod-replica
│   Replication lag should be: < 30 seconds
└─ Jobs in git: Confirm latest commit is deployable

Actions:
├─ 1. Start EC2 instances in us-west-2 (if stopped for cost)
├─ 2. Verify RDS read replica is in "Available" state
├─ 3. Confirm: UC metastore location is writable
└─ 4. Check: DNS switch is ready (Route 53 record prepared)
```

**Minute 30-50: Activate DR**

```bash
# Step 1: Deploy jobs from git
for job in jobs_config/*.json; do
  databricks jobs create \
    --json @$job \
    --url https://prod-dr.cloud.databricks.com
done
# Expected: ~5 minutes for 100 jobs

# Step 2: Promote RDS read replica to master
# AWS Console > RDS > Databases > prod-replica
# Action: Promote read replica
# Expected: 1-2 minutes

# Step 3: Update DNS (Route 53)
# AWS Console > Route 53 > Hosted Zones > company.com
# Record: workspace.company.com (CNAME)
# Change to: prod-dr.cloud.databricks.com
# TTL: 60 seconds (for quick DNS propagation)

# Step 4: Update Databricks UC metastore location
# ALTER METASTORE SET LOCATION = s3://dr-storage-bucket
```

**Minute 50-60: Validate**

```
Smoke tests:
├─ Test 1: User login
│   Open: https://workspace.company.com
│   Expected: Login to prod-dr (not prod-primary)
├─ Test 2: Query data
│   SELECT COUNT(*) FROM prod.finance.transactions
│   Expected: Row count matches (from latest backup)
├─ Test 3: Run critical job
│   Trigger: Daily reconciliation job
│   Expected: Completes successfully
└─ Test 4: BI dashboard
│   Open: https://dashboard.company.com
│   Expected: Connects to DR, data loads
```

**Metrics to Verify Success:**

```
✓ All users can login
✓ Cluster creation works
✓ Queries return correct row counts
✓ Jobs complete successfully
✓ BI dashboards display data
✓ No data newer than 15 min is missing (RPO met)
✓ RTO < 60 minutes achieved
```

**Communication Milestones:**

```
Minute 15: "Failover in progress. DR activation started."
Minute 45: "DR infrastructure activated. Running validation tests."
Minute 60: "Failover complete. Operating from us-west-2 DR region. RCA pending."
```

---

## Scoring Guide

**How to evaluate your answers:**

- **Full points:** You would actually solve the problem in real scenario
- **75%:** You'd identify root cause but might need minor hints on fix
- **50%:** You'd know what to check but missing some steps
- **25%:** You recognize the problem type but unclear on solution
- **0%:** Not familiar with this scenario

**Passing Score:** 11/15 (73%)

**Expert Score:** 14/15 (93%)

---

End of Tier 1 Labs

