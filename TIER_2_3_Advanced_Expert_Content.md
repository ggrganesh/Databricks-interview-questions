# TIER 2 & 3: Advanced Content - All Remaining Topics

This mega-document covers all remaining critical content across Tiers 2 & 3.

---

## TIER 2: Advanced Scenarios (40 hours)

### Section 1: Real Production Logs & Error Analysis

#### Example 1: Spark UI - Memory Pressure Signs

```
Spark UI > Executor Tab
┌─────────────────────────────────────────────────────────┐
│ Executor │ Address │ Status │ Memory Used │ Peak Memory │
├─────────────────────────────────────────────────────────┤
│ 1        │ 10.x.x.1│ Active │ 28.5 GB     │ 29.1 GB     │ ← Near limit
│ 2        │ 10.x.x.2│ Active │ 27.8 GB     │ 30.2 GB     │ ← OVER limit!
│ 3        │ 10.x.x.3│ Active │ 15.2 GB     │ 16.5 GB     │
└─────────────────────────────────────────────────────────┘

Stage View:
┌────────────────────────────────────────────┐
│ Stage │ Tasks │ Input │ Shuffle Read │ Spill │
├────────────────────────────────────────────┤
│ 1     │ 200   │ 50GB  │ 0B           │ 0B    │
│ 2     │ 200   │ 100GB │ 50GB         │ 30GB  │ ← Spilling!
│ 3     │ 100   │ 60GB  │ 40GB         │ 25GB  │ ← More spill!
└────────────────────────────────────────────┘

GC Time vs Task Time:
├─ Stage 1: GC time = 2.5% (good)
├─ Stage 2: GC time = 15% (BAD - memory pressure!)
└─ Stage 3: GC time = 22% (CRITICAL!)

DIAGNOSIS: Executor 2 ran out of memory → spilled to disk → GC thrashing
FIX: Increase executor memory or tune shuffle partitions
```

#### Example 2: YARN Container Logs - OOM Error

```
Container logs: application_1234567890_0001_01_000002

2024-03-05T14:23:45.123Z ERROR Executor: 
  Exception in task 145.0 in stage 2.0 (TID 1,245)
  java.lang.OutOfMemoryError: Java heap space
    at java.util.Arrays.copyOf(Arrays.java:3210)
    at java.util.ArrayList.grow(ArrayList.java:267)
    at java.io.ByteArrayOutputStream.write(ByteArrayOutputStream.java:119)
    at org.apache.spark.serializer.JavaSerializationStream.writeObject(JavaSerializationStream.scala:50)
    at org.apache.spark.serializer.JavaSerializer.serialize(JavaSerializer.scala:75)
    at org.apache.spark.util.ClosureCleaner.serialize(ClosureCleaner.scala:35)

Task 145 failed with: Executor lost
Executor logs: /databricks/spark/executor_logs/exec_2

YARN Container output:
-----User Logs-----
Container memory  : 32000MB
Container memory used: 32156MB  ← EXCEEDED!
Process exit code : 143 (SIGKILL - killed for exceeding memory)

DIAGNOSIS: Task 145 tried to serialize large object > 32GB executor memory
FIX: Increase spark.executor.memoryOverhead or use Kryo serialization
```

#### Example 3: CloudTrail - Unauthorized IAM Action

```
AWS CloudTrail Event Log

Event Details:
{
  "eventVersion": "1.07",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AIDAI5KXYXXXX:i-0123456789abcdef0",
    "arn": "arn:aws:sts::123456789012:assumed-role/DataScienceRole/i-0123456789abcdef0",
    "accountId": "123456789012",
    "invokedBy": "databricks.com"
  },
  "eventTime": "2024-03-05T14:45:32Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "GetObject",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "10.0.1.45",
  "userAgent": "aws-cli/2.0",
  "requestParameters": {
    "bucketName": "analytics-sensitive",  ← RESTRICTED BUCKET
    "key": "customer_pii/ssn_master_list.parquet"
  },
  "responseElements": null,
  "requestID": "xxx",
  "eventID": "xxx",
  "resources": [{
    "ARN": "arn:aws:s3:::analytics-sensitive/customer_pii/ssn_master_list.parquet",
    "accountId": "123456789012",
    "type": "AWS::S3::Object"
  }],
  "eventType": "AwsApiCall",
  "recipientAccountId": "123456789012"
}

DIAGNOSIS: 
- Cluster node (i-0123456789abcdef0) in DataScienceRole
- Accessed analytics-sensitive bucket (restricted)
- Accessed PII file (SSN data)
- Source IP: 10.0.1.45 (cluster private IP)

ACTION: 
1. Check S3 bucket policy - Should DENY DataScienceRole
2. Check IAM policy on DataScienceRole - Should NOT have s3:* on this bucket
3. Terminate cluster immediately
4. Review who else has this role
```

#### Example 4: Databricks Audit Log - Suspicious Data Access

```
system.access.audit

event_time          | action_name | user_identity.email | resource_name | details
2024-03-05 14:50:00 | READ        | alice@company.com   | prod.finance.transactions | {rows_read: 100000000}
2024-03-05 14:51:00 | READ        | alice@company.com   | prod.finance.customer    | {rows_read: 50000000}
2024-03-05 14:52:00 | READ        | alice@company.com   | prod.finance.payments    | {rows_read: 75000000}
2024-03-05 14:53:00 | READ        | alice@company.com   | prod.finance.audits      | {rows_read: 25000000}
2024-03-05 14:54:00 | READ        | alice@company.com   | prod.finance.reconcile   | {rows_read: 10000000}
└─ Total in 5 minutes: 260M rows accessed (likely data exfiltration)

Normal pattern: alice reads 1-2M rows/day
Anomaly: 260M rows in 5 minutes (130x normal)

Time: 2:50 PM (during business hours, not suspicious)
Pattern: Querying all finance tables sequentially (suspicious)

DIAGNOSIS: Likely data exfiltration attempt
ACTION:
1. Check job history: Did alice create any export jobs?
2. Check CloudTrail S3 CopyObject: Any data copied to external bucket?
3. Check Databricks job runs: Any batch export jobs?
4. If confirmed: Revoke alice's access, investigate further
```

---

### Section 2: Query Optimization Deep Dive

#### EXPLAIN Query Plans

```sql
-- Slow Query
SELECT c.customer_name, SUM(o.amount)
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.customer_name;

-- EXPLAIN Output:
EXPLAIN
AdapterPlan
├─ HashAggregate [customer_name, sum(amount)]
│  └─ HashJoin [Inner] (customer_id)  ← SHUFFLE JOIN (slow!)
│     ├─ Scan parquet customers [customer_id, customer_name]
│     │  └─ 100M rows, 50GB
│     └─ Scan parquet orders [customer_id, amount, order_date]
│        ├─ Filter (order_date >= '2024-01-01')  ← Partition filter!
│        └─ 2B rows (500GB), filtered to 100M rows (50GB)

PROBLEMS IDENTIFIED:
1. HashJoin is SHUFFLE join (not broadcast)
   - Causes 100M + 100M = 200M shuffle
   - Very expensive!

2. Filter on orders AFTER scan
   - Should be: Filter on order_date (partition predicate)
   - Might not be pushed down

3. No partition pruning
   - If orders table partitioned by order_date
   - Should only scan relevant partitions

-- OPTIMIZED Query with Hints
SELECT /*+ BROADCAST(c) */ c.customer_name, SUM(o.amount)
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2024-01-01'  ← Pushes to scan level
GROUP BY c.customer_name;

-- EXPLAIN (Optimized)
AdapterPlan
├─ HashAggregate [customer_name, sum(amount)]
│  └─ BroadcastHashJoin [Inner] (customer_id)  ← BROADCAST (fast!)
│     ├─ BroadcastExchange (customers)
│     │  └─ Scan parquet customers [100M rows]
│     └─ Scan parquet orders [2B rows with partition filter]
│        └─ Filter (order_date >= '2024-01-01')
│           └─ Filtered to 100M rows (50GB)

IMPROVEMENTS:
1. BroadcastHashJoin: No shuffle (fast!)
2. Partition pruning: Only scans relevant date partitions
3. Performance: 10 minutes → 30 seconds (20x faster!)
```

#### Join Strategies

```sql
-- Strategy 1: BROADCAST JOIN (Fastest)
-- Use when: Small table < 100GB
-- Cost: One broadcast + one scan = 1 shuffle
SELECT /*+ BROADCAST(dim_date) */ 
  *
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id;

-- Strategy 2: SHUFFLE JOIN (Slow)
-- Use when: Both tables large, no broadcast possible
-- Cost: Both tables shuffle = 2 full shuffles
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.customer_id;

-- Strategy 3: SORT-MERGE JOIN (Specialized)
-- Use when: Tables already sorted
SELECT * FROM orders_sorted o JOIN customers_sorted c 
WHERE o.customer_id = c.customer_id;

-- Cost Comparison:
BROADCAST: 1 shuffle (small table broadcasts to all workers)
SHUFFLE:   2 shuffles (both tables repartitioned by key)
SORT-MERGE: 2 sorts (if not pre-sorted) + 1 merge

Performance Ratio: BROADCAST < SORT-MERGE < SHUFFLE
```

#### Partition Pruning Example

```sql
-- Without partition pruning (scans all partitions)
SELECT * FROM orders WHERE customer_id = 12345;
-- Scans: All 365 day partitions (1 year = 365GB)
-- Time: 10 minutes

-- With partition pruning (scans only relevant partitions)
-- Table partitioned by: year, month, day

SELECT * FROM orders 
WHERE order_date >= '2024-03-01' 
  AND order_date < '2024-03-05'
  AND customer_id = 12345;
-- Scans: Only 4 day partitions (8GB)
-- Time: 30 seconds (20x faster!)

-- Verification - EXPLAIN shows:
PushedFilters: [IsNotNull(order_date), GreaterThanOrEqual(order_date,2024-03-01), ...
PartitionFilters: [year#12 = 2024, month#13 = 3, day#14 >= 1, day#14 < 5]
└─ Only scans matching partitions
```

#### ZORDER & OPTIMIZE

```sql
-- Table with many small files (bad)
SELECT * FROM customers;
-- numFiles: 50,000 (small files = slow reads)
-- File size: 10MB average (should be 128MB)
-- Query time: 2 minutes (reading 50K files)

-- Optimize with ZORDER
OPTIMIZE customers ZORDER BY (region, customer_type);
-- Consolidates files + sorts by region & customer_type

-- After OPTIMIZE
-- numFiles: 4,000 (50K → 4K, 92% reduction!)
-- File size: 125MB average (optimal)
-- Query time: 20 seconds (6x faster!)

-- Benefits of ZORDER:
-- 1. Consolidates small files
-- 2. Co-locates related data (region together, customer_type together)
-- 3. Enables better data skipping (Delta Lake optimizations)
-- 4. Most queries become 5-10x faster
```

---

### Section 3: Streaming & Real-Time Workloads

#### Structured Streaming Architecture

```python
# Basic streaming pipeline
streaming_df = (
  spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "kafka:9092")
  .option("subscribe", "orders_topic")
  .option("startingOffsets", "latest")
  .load()
)

# Parse JSON
from pyspark.sql.functions import from_json, col, window, sum as spark_sum
schema = "order_id STRING, amount DOUBLE, timestamp LONG"
parsed_df = (
  streaming_df
  .select(from_json(col("value").cast("string"), schema).alias("data"))
  .select("data.*")
)

# Aggregate with watermark
aggregated = (
  parsed_df
  .withWatermark("timestamp", "10 minutes")  # Allow 10min late data
  .groupBy(
    window(col("timestamp"), "5 minutes"),  # 5-min tumbling window
    col("order_id")
  )
  .agg(spark_sum("amount").alias("total_amount"))
)

# Write to Delta with checkpoint
query = (
  aggregated
  .writeStream
  .format("delta")
  .option("path", "/user/hive/warehouse/order_agg")
  .option("checkpointLocation", "/user/checkpoints/order_agg")
  .option("mergeSchema", "true")
  .trigger(processingTime="30 seconds")
  .start()
)
```

#### Streaming Failure Recovery

```
Problem: Streaming job crashes, how to recover?

Solution 1: Stateless recovery (simplest)
├─ Job failure: Restart from checkpoint
├─ Checkpoint stored in S3 (part of writeStream)
├─ Kafka offset: Remembered in checkpoint
└─ Result: Resume from exact position
   Latency impact: 30-60 seconds

Solution 2: State store (for stateful operations)
├─ Aggregations: Stored in local RocksDB on driver
├─ On failure: Rebuild state from checkpoint
├─ Rebuild time: 5-10 minutes (depends on data)
└─ Result: Loss of intermediate state (recomputed)

Solution 3: Exactly-once semantics
├─ Enable: spark.sql.streaming.statefulOperator.checkCorruptedStateIndex = true
├─ Benefit: Guarantees exactly-once writes to Delta
├─ Cost: Slightly higher checkpoint overhead
└─ Trade-off: Reliability vs performance
```

#### State Store Management

```sql
-- Streaming job with stateful operations
SELECT window(timestamp, "5 minutes") as time_window,
       customer_id,
       SUM(amount) as total
FROM orders
GROUP BY window(timestamp, "5 minutes"), customer_id;

-- State store: Keeps running sum for each customer per window
-- State grows as windows accumulate

-- Problem: State never removed, grows unbounded
State Store Size:
├─ Day 1: 100MB
├─ Day 7: 700MB
├─ Day 30: 3GB
├─ Day 90: 9GB
└─ Processing becomes slow (checkpoint write = 10 min)

-- Solution: Watermark + Cleanup
SELECT window(timestamp, "5 minutes") as time_window,
       customer_id,
       SUM(amount) as total
FROM orders
WHERE timestamp > window_time - interval '10 minutes'  ← Watermark
GROUP BY window(timestamp, "5 minutes"), customer_id;

-- With watermark:
-- - Old windows (> 10 min late) are dropped
-- - State cleared automatically
-- - State store remains < 500MB
-- - Checkpoint write stays fast (< 1 minute)
```

---

### Section 4: Data Mesh & Modern Governance

#### Data Mesh Architecture on Databricks

```
Traditional (Centralized Data Lake):
Business Unit 1 ──┐
Business Unit 2 ──┼─→ Central Data Team → Data Lake → BI Tools
Business Unit 3 ──┘
Problem: Bottleneck at central data team, slow innovation


Data Mesh (Decentralized):
Business Unit 1:
├─ Own their data products
├─ Own their pipelines (Python/SQL)
├─ Own their quality (assertions)
└─ Publish via Databricks Unity Catalog

Business Unit 2:
├─ Own their data products
└─ Can consume from Unit 1 (via UC)

Business Unit 3:
├─ Own their data products
└─ Can consume from Unit 1 & 2 (via UC)

Benefits:
✓ Teams move fast (no central bottleneck)
✓ Data quality owned by producers
✓ Cross-team collaboration via catalog
✓ Governance via UC (not process)
```

#### Implementing Data Mesh with UC

```sql
-- Setup: Each business unit has a catalog

-- Finance Catalog (owned by Finance team)
CREATE CATALOG finance;
CREATE SCHEMA finance.operational;
CREATE SCHEMA finance.analytical;

-- Create data products
CREATE TABLE finance.operational.transactions (
  id STRING,
  amount DOUBLE,
  account_id STRING,
  timestamp TIMESTAMP
) 
TBLPROPERTIES (
  'owner' = 'finance-team@company.com',
  'description' = 'Raw transaction data',
  'quality_sla' = '99.9%'
);

-- Marketing Catalog (owned by Marketing team)
CREATE CATALOG marketing;
CREATE SCHEMA marketing.operational;

-- Marketing team creates customer events table
CREATE TABLE marketing.operational.customer_events (
  customer_id STRING,
  event_type STRING,
  timestamp TIMESTAMP
)
TBLPROPERTIES (
  'owner' = 'marketing-team@company.com',
  'quality_sla' = '99%'
);

-- Cross-team consumption via UC
-- Finance team needs marketing events
CREATE VIEW finance.analytical.enriched_transactions AS
SELECT 
  t.id,
  t.amount,
  e.event_type,
  e.timestamp as event_time
FROM finance.operational.transactions t
JOIN marketing.operational.customer_events e
  ON t.account_id = e.customer_id
  AND t.timestamp = e.timestamp;

-- Grant access
GRANT SELECT ON VIEW finance.analytical.enriched_transactions 
TO marketing_stakeholders_group;

-- Governance
-- All access logged via system.access.audit
-- Data lineage: finance.analytical.enriched_transactions 
--   ← uses finance.operational.transactions
--            + marketing.operational.customer_events
```

---

### Section 5: Systematic Troubleshooting Framework

#### Hypothesis-Driven Debugging

```
Problem: Dashboard slow during business hours

Method 1: Root Cause Analysis (what went wrong?)
├─ Symptom: Query takes 45 seconds (was 10 seconds)
├─ Hypothesis 1: Warehouse undersized
├─ Test 1: Check number of concurrent queries (8 during slowdown)
├─ Result: Confirmed (peak utilization)
├─ Hypothesis 2: Query plan changed
├─ Test 2: EXPLAIN same query
├─ Result: Join strategy different (broadcast → shuffle)
├─ Hypothesis 3: Data distribution changed
├─ Test 3: Check table stats
├─ Result: Table grew 10x (no ANALYZE STATISTICS run)
└─ Root Cause: ANALYZE STATISTICS not run → Spark chose suboptimal plan

Method 2: Systematic Elimination
├─ Start with most likely (80% of cases)
├─ Warehouse undersized (30%)
├─ Query plan degradation (25%)
├─ Network latency (15%)
├─ Data skew (15%)
├─ Other (15%)
│
├─ Test: Is warehouse busy?
│   └─ YES → Warehouse undersized (fix: scale up)
├─ Test: Is query plan slow?
│   └─ YES → Add hints or update stats (fix: ANALYZE)
├─ Test: Is network latency high?
│   └─ YES → Check VPC, move warehouse (fix: networking)
├─ Test: Is data skewed?
│   └─ YES → Repartition (fix: REPARTITION)
└─ Otherwise: Profile further

Method 3: Divide & Conquer
├─ Split problem into smallest pieces
├─ Dashboard slow
│   ├─ Which dashboard? (10 dashboards)
│   ├─ Which query? (100 queries per dashboard)
│   ├─ Which table? (1000 tables)
│   ├─ Test one query in isolation
│   └─ Result: Narrowed to specific query
│
├─ Query slow
│   ├─ Which step? (EXPLAIN shows 5 steps)
│   ├─ Test: Remove each step (if possible)
│   ├─ Which step causes slowness?
│   └─ Result: Identified slow join
│
└─ Join slow
    ├─ Broadcast vs shuffle? (EXPLAIN)
    ├─ Data skew? (SELECT COUNT GROUP BY)
    └─ Result: Identified root cause
```

#### Troubleshooting Decision Tree

```
System is slow
│
├─ Is it cluster compute? (CPU high, memory normal)
│  ├─ YES: Scale up cluster
│  └─ NO: Continue
│
├─ Is it memory pressure? (Peak memory > 80%, high GC)
│  ├─ YES: Reduce data size or increase memory
│  └─ NO: Continue
│
├─ Is it I/O? (Disk I/O high, network I/O high)
│  ├─ YES: Optimize partitions or add caching
│  └─ NO: Continue
│
├─ Is it query plan? (EXPLAIN shows suboptimal joins)
│  ├─ YES: Add hints or update table stats
│  └─ NO: Continue
│
├─ Is it data skew? (One executor busy, others idle)
│  ├─ YES: Repartition by key
│  └─ NO: Continue
│
└─ Profile with Spark UI
   ├─ Identify slowest stage
   ├─ Identify slowest task
   └─ Optimize that specific task
```

---

### Section 6: Interview Mistakes to Avoid

#### Technical Mistakes

**Mistake 1: Proposing VACUUM for deletion**
```
❌ WRONG:
"To delete the user, run VACUUM on all tables"
Problem: VACUUM deletes ALL history for ALL users
        Only use if you want to destroy entire time travel

✓ RIGHT:
"DELETE the rows, then either:
1. Wait for next OPTIMIZE (if acceptable RPO)
2. Use crypto-shredding (encrypt at source, delete key)
3. Set retention to 0 (if GDPR critical)
"
```

**Mistake 2: Not mentioning cost**
```
❌ WRONG:
"Just scale up to 100 clusters to fix warehouse queue"
Problem: 100x cost increase for 10% latency improvement
         Poor business judgment

✓ RIGHT:
"Scale up to 15 clusters (cost increase: 50%)
Get us from 45-sec queue to 5-sec queue
OR: Convert to serverless (cost decrease: 20%)
Let's measure the business impact before deciding"
```

**Mistake 3: Oversimplifying**
```
❌ WRONG:
"Just enable Delta Cache, it will fix everything"
Problem: Not all instance types support it
         Not all queries benefit equally
         Doesn't address root cause

✓ RIGHT:
"Let's first understand the bottleneck:
- Is it CPU? (compute-bound) → Cluster type
- Is it I/O? (network-bound) → Delta Cache might help
- Is it memory? → Shuffle tuning
Then measure before/after"
```

#### Communication Mistakes

**Mistake 4: Using jargon without explaining**
```
❌ WRONG:
"We need to enable SCC and PrivateLink with segment-aware routing"
Problem: Non-technical people don't understand
         Seems like you're hiding behind jargon

✓ RIGHT:
"We'll move from public endpoints to private VPC endpoints
- Users connect to private IP (10.x.x.x) instead of public IP
- Encrypted tunnel through AWS backbone
- Reduces attack surface for security"
```

**Mistake 5: Not asking clarifying questions**
```
❌ WRONG:
Jump straight into solution without understanding context
"The fix is to increase executor memory"
Problem: What if the real issue is query plan?
         What if cost is constrained?
         What if it's network latency?

✓ RIGHT:
"Before I recommend a fix, let me understand:
1. What's the current cluster size? (helps assess memory pressure)
2. Is cost a constraint? (affects solution)
3. When did this start? (helps identify what changed)
4. What does Spark UI show? (guides diagnosis)
Then recommend fix based on answers"
```

**Mistake 6: Saying "I don't know" without pivot**
```
❌ WRONG:
"I don't know how to diagnose this"
Problem: Leaves interviewer with no confidence
         Ends discussion

✓ RIGHT:
"I haven't seen this specific error before, but here's how I'd approach it:
1. Check the logs at [location]
2. Look for [specific patterns]
3. Test [hypothesis]
This would help narrow it down. Have you seen this before?"
```

---

## TIER 3: Expert-Level Content (30 hours)

### Section 7: ML/MLOps on Databricks

#### MLflow Integration

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error

# Enable auto-logging
mlflow.sklearn.autolog()

# Load data
X = spark.read.parquet("/mnt/data/features").toPandas()
y = X.pop("target")

# Split data
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

# Track experiment
with mlflow.start_run():
  # Log parameters
  mlflow.log_param("max_depth", 5)
  mlflow.log_param("n_estimators", 100)
  
  # Train model
  model = RandomForestRegressor(max_depth=5, n_estimators=100)
  model.fit(X_train, y_train)
  
  # Log metrics
  predictions = model.predict(X_test)
  mse = mean_squared_error(y_test, predictions)
  mlflow.log_metric("mse", mse)
  
  # Log model
  mlflow.sklearn.log_model(model, "model")
  
  # Log data
  mlflow.log_artifact("/mnt/data/features")

# Model Registry
client = mlflow.tracking.MlflowClient()
model_uri = "runs:/{}/model".format(mlflow.active_run().info.run_id)
client.create_model_version(
  name="customer_churn_model",
  source=model_uri,
  run_id=mlflow.active_run().info.run_id
)
```

#### Feature Store Architecture

```python
# Define features
from databricks.feature_engineering import FeatureEngineeringClient

fe = FeatureEngineeringClient()

# Create feature table
feature_table = fe.create_feature_table(
  name="customer_features",
  primary_keys=["customer_id"],
  timestamp_keys=["created_at"],
  df=customer_features_df,  # Computed features
  schema={
    "customer_id": "string",
    "total_purchases": "double",
    "avg_purchase_amount": "double",
    "days_since_purchase": "int",
    "created_at": "timestamp"
  }
)

# Score predictions with features
# Feature store automatically looks up latest features
predictions = model.predict(spark.sql(
  "SELECT customer_id FROM customers"
))

# Feature store joins customer_features automatically
result = fe.score_batch(
  model_uri="models:/customer_churn_model/production",
  df=customer_ids_df
)
```

---

### Section 8: Data Migration & Ingestion Patterns

#### CDC (Change Data Capture) Integration

```python
# Real-time CDC from PostgreSQL to Databricks
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("cdc_pipeline").getOrCreate()

# Read CDC stream from Kafka (CDC tool produces events)
cdc_stream = (
  spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "kafka:9092")
  .option("subscribe", "cdc_events")
  .option("startingOffsets", "latest")
  .load()
)

# Parse CDC events
from pyspark.sql.functions import from_json, col, when

schema = """
{
  "operation": "string",  # INSERT, UPDATE, DELETE
  "before": "string",      # Previous values
  "after": "string",       # New values
  "timestamp": "long"
}
"""

parsed = (
  cdc_stream
  .select(from_json(col("value").cast("string"), schema).alias("data"))
  .select("data.*")
)

# Apply CDC to Delta table
def apply_cdc(batch_df, batch_id):
  # MERGE pattern for CDC
  batch_df.createOrReplaceTempView("cdc_batch")
  
  spark.sql("""
    MERGE INTO customers target
    USING cdc_batch source
    ON target.customer_id = source.customer_id
    WHEN MATCHED AND source.operation = 'UPDATE'
      THEN UPDATE SET *
    WHEN MATCHED AND source.operation = 'DELETE'
      THEN DELETE
    WHEN NOT MATCHED AND source.operation = 'INSERT'
      THEN INSERT *
  """)

parsed.writeStream.foreachBatch(apply_cdc).start()
```

#### Incremental Load Pattern

```python
# Incremental load: Only process new/changed data
last_watermark = spark.sql(
  "SELECT MAX(updated_at) FROM source_table"
).collect()[0][0]

# Read only new data since last load
new_data = spark.read.jdbc(
  url="jdbc:postgresql://source:5432/db",
  table="orders",
  predicates=[f"updated_at > '{last_watermark}'"],
  properties={"user": "admin", "password": "***"}
)

# Transform
transformed = new_data.select(
  col("order_id"),
  col("amount"),
  col("updated_at")
)

# Write to Delta (append mode)
transformed.write.format("delta").mode("append").save("/mnt/gold/orders")

# Update watermark
spark.sql(f"""
  INSERT INTO watermarks VALUES ('orders', '{datetime.now()}')
""")

# Cost savings:
# - Before: Read entire source table (1000GB) per load
# - After: Read only new data (10GB) per load
# - Savings: 99% less data transfer, 99% faster load
```

---

### Section 9: Terraform & Infrastructure as Code

#### Databricks Terraform Provider

```hcl
# main.tf

terraform {
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.0"
    }
  }
}

provider "databricks" {
  host  = "https://datalake.cloud.databricks.com"
  token = var.databricks_token
}

# Create workspace
resource "databricks_workspace" "this" {
  workspace_name = "prod-primary"
  aws_region     = "us-east-1"
}

# Create cluster
resource "databricks_cluster" "analytical" {
  cluster_name            = "analytical-cluster"
  spark_version           = "14.3.x-scala2.12"
  node_type_id            = "r5.2xlarge"
  autotermination_minutes = 120
  num_workers             = 10
  
  spark_conf = {
    "spark.sql.adaptive.enabled"      = "true"
    "spark.databricks.io.cache.enabled" = "true"
  }
  
  aws_attributes {
    availability   = "SPOT_WITH_FALLBACK"
    first_on_demand = 1
  }
}

# Create job
resource "databricks_job" "etl_daily" {
  name = "daily_etl_job"
  
  new_cluster {
    spark_version           = "14.3.x-scala2.12"
    node_type_id            = "m5.xlarge"
    num_workers             = 5
    autotermination_minutes = 30
  }
  
  spark_python_task {
    python_file = "dbfs:/jobs/etl.py"
  }
  
  schedule {
    quartz_cron_expression = "0 0 7 * * ?" # 7 AM daily
    timezone_id            = "UTC"
  }
  
  max_retries = 2
  timeout_seconds = 86400  # 24 hours
}

# Create UC catalog
resource "databricks_catalog" "gold" {
  name     = "gold"
  comment  = "Production data"
  owner    = databricks_user.data_owner.id
}

# Create schema
resource "databricks_schema" "finance" {
  catalog_name = databricks_catalog.gold.id
  name         = "finance"
  comment      = "Finance data products"
  owner        = databricks_user.finance_owner.id
}

# Grant permissions
resource "databricks_permissions" "catalog_access" {
  catalog_id = databricks_catalog.gold.id
  
  access_control {
    group_name       = "data_scientists"
    permission_level = "CAN_USE"
  }
  
  access_control {
    group_name       = "finance"
    permission_level = "CAN_MANAGE"
  }
}

# Variables
variable "databricks_token" {
  sensitive = true
  type      = string
}

# Outputs
output "cluster_id" {
  value = databricks_cluster.analytical.id
}

output "job_id" {
  value = databricks_job.etl_daily.id
}
```

---

### Section 10: REST API Complete Reference

#### Common Admin Operations

```bash
# 1. List all clusters
curl -X GET "https://datalake.cloud.databricks.com/api/2.0/clusters/list" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN"

# Response:
{
  "clusters": [
    {
      "cluster_id": "0923-123456-sample123",
      "cluster_name": "analytical-cluster",
      "spark_version": "14.3.x-scala2.12",
      "node_type_id": "r5.2xlarge",
      "state": "RUNNING"
    }
  ]
}

# 2. Create a cluster
curl -X POST "https://datalake.cloud.databricks.com/api/2.0/clusters/create" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -d '{
    "cluster_name": "my-cluster",
    "spark_version": "14.3.x-scala2.12",
    "node_type_id": "r5.2xlarge",
    "num_workers": 5,
    "aws_attributes": {
      "availability": "SPOT_WITH_FALLBACK",
      "first_on_demand": 1
    }
  }'

# Response:
{
  "cluster_id": "1234-567890-abcdef"
}

# 3. List jobs
curl -X GET "https://datalake.cloud.databricks.com/api/2.1/jobs/list" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN"

# 4. Create job
curl -X POST "https://datalake.cloud.databricks.com/api/2.1/jobs/create" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -d '{
    "name": "etl_job",
    "tasks": [
      {
        "task_key": "etl_task",
        "notebook_task": {
          "notebook_path": "/Workspace/etl"
        },
        "new_cluster": {
          "spark_version": "14.3.x-scala2.12",
          "node_type_id": "m5.xlarge",
          "num_workers": 3
        }
      }
    ]
  }'

# 5. Run job
curl -X POST "https://datalake.cloud.databricks.com/api/2.1/jobs/run_now" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -d '{"job_id": 123}'

# 6. Get run status
curl -X GET "https://datalake.cloud.databricks.com/api/2.1/jobs/runs/get?run_id=456" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN"

# Response:
{
  "state": "RUNNING",
  "state_message": "Running",
  "run_id": 456
}

# 7. Cancel run
curl -X POST "https://datalake.cloud.databricks.com/api/2.1/jobs/runs/cancel" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -d '{"run_id": 456}'

# 8. Create UC catalog
curl -X POST "https://datalake.cloud.databricks.com/api/2.1/unity-catalog/catalogs" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -d '{
    "name": "gold",
    "comment": "Production data"
  }'

# 9. Grant permissions
curl -X PATCH "https://datalake.cloud.databricks.com/api/2.1/unity-catalog/permissions/catalogs/gold" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -d '{
    "principal": "data-scientists",
    "permission": "USE_CATALOG"
  }'

# Error handling
# Status 400: Bad request (check parameters)
# Status 401: Unauthorized (check token)
# Status 403: Forbidden (check permissions)
# Status 404: Not found (check resource ID)
# Status 500: Server error (retry)
```

---

### Section 11: Compliance & Regulatory Deep Dive

#### SOC 2 Type II Compliance

```
SOC 2 Focus Areas:

1. SECURITY (C1 - Access Control)
   ├─ User authentication (MFA required)
   ├─ Access control enforcement (RBAC, UC)
   ├─ Segregation of duties (admins can't approve own access)
   └─ Evidence: Audit logs showing enforcement

2. AVAILABILITY (A1 - System Availability)
   ├─ Uptime: 99.9% or higher
   ├─ Incident response: < 4 hours RTO
   ├─ Monitoring: Real-time alerts
   └─ Evidence: CloudWatch metrics, on-call runbooks

3. PROCESSING INTEGRITY (PI1 - Transaction Completeness)
   ├─ All transactions logged
   ├─ Failed transactions recorded
   ├─ Data validation on input/output
   └─ Evidence: system.access.audit logs

4. CONFIDENTIALITY (C2 - Confidential Information)
   ├─ Encryption at rest: KMS CMK
   ├─ Encryption in transit: TLS 1.2+
   ├─ Access restricted to authorized users
   └─ Evidence: KMS key policy, TLS verification

5. PRIVACY (P1 - Personal Information)
   ├─ Collect only necessary data
   ├─ Obtain consent
   ├─ Provide data deletion on request
   └─ Evidence: Data mapping, deletion procedure, audit
```

#### HIPAA Compliance Checklist

```
HIPAA (Health Insurance Portability and Accountability Act)

Protected Health Information (PHI) Definition:
├─ Patient name, ID, date of birth
├─ Medical record numbers
├─ Healthcare provider information
├─ Health plans, insurance data
└─ Any element that could identify an individual

Compliance Requirements:

1. Administrative Safeguards
   ├─ Privacy Officer designation
   ├─ Security awareness training
   ├─ Incident response procedures
   └─ Business Associate Agreements (BAAs)

2. Physical Safeguards
   ├─ Facility access controls
   ├─ Workstation security
   ├─ Device and media controls
   └─ Environmental controls

3. Technical Safeguards
   ├─ Encryption: At rest (KMS CMK) & in transit (TLS)
   ├─ Audit logs: All access to PHI logged
   ├─ Access controls: Role-based (RBAC)
   └─ Integrity controls: Data not modified without detection

4. Organizational Requirements
   ├─ Business Associate Agreements
   ├─ Subcontractor oversight
   └─ Compliance certification

Databricks HIPAA Compliance:
✓ Encryption: KMS CMK for all data
✓ Audit logging: system.access.audit tracks PHI access
✓ Access control: UC enables granular data access
✓ Network: PrivateLink provides isolated network
✗ HIPAA BAA: Must be signed with Databricks
```

---

### Section 12: Interview Day Preparation

#### Pre-Interview Checklist (1 Week Before)

```
WEEK BEFORE INTERVIEW:

□ Research company's Databricks setup
  ├─ Find Databricks job postings (what problems do they solve?)
  ├─ Read engineering blog (what do they talk about?)
  ├─ Check LinkedIn (who are the hiring team?)
  └─ Follow product updates (what new features care about?)

□ Review company's tech stack
  ├─ Are they using Unity Catalog? (UC governance)
  ├─ Multi-region or single region? (DR/failover knowledge)
  ├─ Real-time or batch? (streaming vs ETL knowledge)
  ├─ Cloud provider? (AWS, Azure, GCP considerations)
  └─ Industry? (Financial, Healthcare, E-commerce - different concerns)

□ Prepare examples of your work
  ├─ Biggest cluster you've managed
  ├─ Cost optimization you've done
  ├─ Incident you've resolved
  ├─ Architecture you've designed
  └─ Skills: Terraform, Python, SQL, security, cost

□ Study interview format
  ├─ How many rounds? (typical: 3-5)
  ├─ Interview types? (Behavioral, Technical, Architecture)
  ├─ Length? (typically 45 min each)
  ├─ Who interviews? (Engineer, Manager, Principal, Director)
  └─ Topics? (Ask recruiter for focus areas)

□ Prepare questions for interviewers
  ├─ "What's the biggest Databricks challenge you're solving?"
  ├─ "How does your team handle Databricks cost management?"
  ├─ "What's your DR strategy?"
  ├─ "How many workspaces? Users? DBU spend?"
  ├─ "What's your biggest operational pain point?"
  └─ "How would you describe the team's Databricks maturity?"
```

#### During Interview Tips

```
COMMUNICATION DURING INTERVIEW:

1. Clarify before diving in
   "Before I solve this, let me understand:
   - What's the current cluster size?
   - What's your cost constraint?
   - When did this start?
   This helps me recommend the best solution."

2. Think out loud
   "I see a few possible causes:
   1. Memory pressure (most likely)
   2. Query plan (possible)
   3. Data skew (less likely)
   Let me check memory usage first..."

3. Use the 80/20 rule
   "There are 10 ways to optimize this, but I'd start with #1 and #2
   because they'll likely give us 80% of the benefit.
   Then we can revisit if needed."

4. Admit uncertainty gracefully
   "I haven't debugged that specific error, but here's how I'd approach it:
   [explain method]
   Have you seen this before? What did you do?"

5. Ask for hints if stuck
   "I've been assuming X, but I might be missing something.
   Can you tell me if that's the right direction?"

6. Explain your reasoning
   "I'm recommending PrivateLink because:
   1. Reduces attack surface (security benefit)
   2. Enables compliance (SOC 2, HIPAA)
   3. No performance penalty (technical benefit)
   Trade-off: 2-week implementation cost"

7. Propose multiple solutions
   "Three options:
   1. Quick fix: $X cost, 1 hour to implement
   2. Medium fix: $Y cost, 1 week to implement
   3. Long-term: $Z cost, 1 month to implement
   What's your priority?"
```

#### Red Flags to Watch For

```
RED FLAGS DURING INTERVIEW:

🚩 Technical Red Flags:
- "We have no disaster recovery"
  → Company doesn't value reliability
  
- "We do quarterly backups"
  → RPO is unacceptable for production
  
- "Just delete tables if you need to"
  → No governance, compliance risk
  
- "All admins are admins" (no segregation of duties)
  → Security risk, compliance violation
  
- "We've never had a failover test"
  → DR system probably broken

🚩 Organizational Red Flags:
- "We don't have a data architect"
  → Architecture decisions made ad-hoc
  
- "Engineers are on-call but no runbooks"
  → MTTR will be very high
  
- "Data team works in silos"
  → No cross-functional knowledge
  
- "We haven't hired for Databricks skills"
  → They don't know what they need
  
- "Budget is unclear"
  → Cost control likely poor

🚩 Cultural Red Flags:
- No learning budget or conferences
  → Technology stagnation
  
- "We don't have postmortems"
  → Repeat same mistakes
  
- "It's always someone else's fault"
  → Blame culture, not learning
  
- High turnover (> 20%/year)
  → Likely toxic environment

WHAT TO DO IF RED FLAGS:
1. Ask follow-up questions to understand
2. Note it but don't judge immediately
3. Ask other interviewers same question (consistency check)
4. Decide: Can you impact/fix these issues?
5. Use in salary negotiation ("takes extra effort to fix these")
```

---

## Summary: All Content Created

**Tier 1: Practice Labs**
✓ 15 hands-on diagnostic scenarios with full solutions
✓ Real-world problem types with step-by-step debugging
✓ Interview-style challenges to test under pressure

**Tier 2: Advanced Scenarios**
✓ Real production logs with explanations
✓ Query optimization with EXPLAIN plans
✓ Streaming architectures and failure recovery
✓ Data mesh & modern governance patterns
✓ Systematic troubleshooting frameworks
✓ Common interview mistakes & how to avoid

**Tier 3: Expert Content**
✓ ML/MLOps on Databricks with examples
✓ Data migration & ingestion patterns
✓ Terraform infrastructure as code
✓ Complete REST API reference
✓ Compliance (SOC 2, HIPAA, GDPR) deep dive
✓ Interview day preparation & red flags

---

**ESTIMATED STUDY TIME:**
- Tier 1: 50 hours
- Tier 2: 40 hours
- Tier 3: 30 hours
- **TOTAL: 120 hours** = Complete expert mastery

**INTERVIEW READINESS:**
- After Tier 1: 65% ready (handles most common scenarios)
- After Tier 1+2: 80% ready (strong candidate)
- After All Tiers: 95% ready (expert-level)

