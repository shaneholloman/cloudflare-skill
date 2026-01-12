# Cloudflare R2 Data Catalog Skill

Expert guidance for Cloudflare R2 Data Catalog - Apache Iceberg catalog built into R2 buckets.

## Overview

R2 Data Catalog is a managed Apache Iceberg REST catalog built directly into R2 buckets. It enables analytical workloads like log analytics, BI, and data pipelines with zero egress fees. Currently in public beta with no additional billing beyond standard R2 costs.

## Core Concepts

### What is R2 Data Catalog?
- **Managed Apache Iceberg catalog**: Standard REST catalog interface
- **Built into R2 buckets**: One catalog per bucket
- **Zero egress fees**: Access data from any cloud/region without transfer costs
- **Public beta**: Free during beta (standard R2 storage/operations apply)

### Apache Iceberg Table Format
- **ACID transactions**: Reliable concurrent reads/writes with data integrity
- **Optimized metadata**: Indexed metadata avoids full table scans
- **Schema evolution**: Add/rename/delete columns without rewriting data
- **Wide engine support**: Spark, Trino, Snowflake, DuckDB, ClickHouse, etc.

### Why Data Catalogs Matter
Catalogs track table lists and current metadata pointers (like a library index). Without catalogs, query engines would:
- Waste time searching for data
- Access outdated versions
- Risk conflicts and data corruption

## Architecture

### File Structure
```
r2-bucket/
├── <namespace>/
│   └── <table>/
│       ├── metadata/           # Iceberg metadata files
│       │   ├── v1.metadata.json
│       │   ├── v2.metadata.json
│       │   └── snap-*.avro
│       └── data/               # Parquet data files
│           ├── 00000-0-*.parquet
│           ├── 00001-1-*.parquet
│           └── compacted-*.parquet  # Auto-compacted files
```

### Key Components
1. **Catalog REST API**: Standard Iceberg REST interface
2. **Warehouse**: Logical container for tables in a bucket
3. **Namespaces**: Organizational grouping for tables (like schemas)
4. **Tables**: Iceberg tables with ACID guarantees

## Configuration

### Prerequisites
- Cloudflare account with R2 subscription
- R2 bucket created
- API token with both R2 Storage and R2 Data Catalog permissions

### Enable Catalog on Bucket

**Via Wrangler:**
```bash
npx wrangler r2 bucket catalog enable <BUCKET_NAME>
```

**Via Dashboard:**
1. Go to R2 → Select bucket → Settings tab
2. Scroll to "R2 Data Catalog" → Click "Enable"
3. Note the **Catalog URI** and **Warehouse name**

**Output:**
- Catalog URI: `https://<account-id>.r2.cloudflarestorage.com/iceberg/<bucket-name>`
- Warehouse: `<bucket-name>`

### API Token Creation

**Dashboard Method (Recommended):**
1. R2 → Manage API tokens → Create API token
2. Select **Admin Read & Write** or **Admin Read only**
   - Includes both R2 Data Catalog + R2 Storage permissions
3. Copy token value

**API Method (Programmatic):**
```json
{
  "policies": [{
    "effect": "allow",
    "resources": {
      "com.cloudflare.edge.r2.bucket.<account-id>_<jurisdiction>_<bucket-name>": "*"
    },
    "permission_groups": [
      {
        "id": "d229766a2f7f4d299f20eaa8c9b1fde9",
        "name": "Workers R2 Data Catalog Write"
      },
      {
        "id": "2efd5506f9c8494dacb1fa10a3e7d5b6",
        "name": "Workers R2 Storage Bucket Item Write"
      }
    ]
  }]
}
```

## Wrangler Commands

### Catalog Management
```bash
# Enable catalog
npx wrangler r2 bucket catalog enable <BUCKET_NAME>

# Disable catalog
npx wrangler r2 bucket catalog disable <BUCKET_NAME>
```

### Compaction (Table Maintenance)

**Catalog-level (all tables):**
```bash
# Enable with custom target size
npx wrangler r2 bucket catalog compaction enable my-bucket \
  --target-size 128 \
  --token $R2_CATALOG_TOKEN

# Disable
npx wrangler r2 bucket catalog compaction disable my-bucket
```

**Table-level (specific table):**
```bash
# Enable
npx wrangler r2 bucket catalog compaction enable my-bucket my-namespace my-table \
  --target-size 256

# Disable
npx wrangler r2 bucket catalog compaction disable my-bucket my-namespace my-table
```

**Target file sizes:**
- Latency-sensitive: 64-128 MB
- Streaming ingest: 128-256 MB
- OLAP queries: 256-512 MB
- Range: 64-512 MB (current limits)

### Snapshot Expiration

**Catalog-level:**
```bash
# Enable
npx wrangler r2 bucket catalog snapshot-expiration enable my-bucket \
  --token $R2_CATALOG_TOKEN \
  --older-than-days 7 \
  --retain-last 10

# Disable
npx wrangler r2 bucket catalog snapshot-expiration disable my-bucket
```

**Table-level:**
```bash
# Enable
npx wrangler r2 bucket catalog snapshot-expiration enable my-bucket my-namespace my-table \
  --older-than-days 2 \
  --retain-last 5

# Disable
npx wrangler r2 bucket catalog snapshot-expiration disable my-bucket my-namespace my-table
```

**Retention policies:**
- Dev/test: 2-7 days, 5 snapshots
- Production: 7-30 days, 10-20 snapshots
- Compliance: 30-90 days, 50+ snapshots
- High-frequency: Higher snapshot counts

## Code Patterns

### PyIceberg (Python)

**Basic Setup:**
```python
import pyarrow as pa
from pyiceberg.catalog.rest import RestCatalog

# Connection details
WAREHOUSE = "<warehouse-name>"
TOKEN = "<api-token>"
CATALOG_URI = "https://<account-id>.r2.cloudflarestorage.com/iceberg/<bucket>"

# Connect to catalog
catalog = RestCatalog(
    name="my_catalog",
    warehouse=WAREHOUSE,
    uri=CATALOG_URI,
    token=TOKEN,
)
```

**Namespace Operations:**
```python
# Create namespace
catalog.create_namespace("default")
catalog.create_namespace_if_not_exists("analytics")

# List namespaces
namespaces = catalog.list_namespaces()

# Check if namespace exists
exists = catalog.namespace_exists("default")
```

**Table Creation:**
```python
# Create PyArrow table
df = pa.table({
    "id": [1, 2, 3],
    "name": ["Alice", "Bob", "Charlie"],
    "score": [80.0, 92.5, 88.0],
})

# Create Iceberg table
table = catalog.create_table(
    ("default", "people"),
    schema=df.schema,
)

# Create or load pattern
test_table = ("default", "my_table")
if not catalog.table_exists(test_table):
    table = catalog.create_table(test_table, schema=df.schema)
else:
    table = catalog.load_table(test_table)
```

**Data Operations:**
```python
# Append data
table.append(df)

# Overwrite data
table.overwrite(df)

# Scan/query table
scanned = table.scan().to_arrow()
df_pandas = scanned.to_pandas()

# Filtered scan
filtered = table.scan(
    row_filter="score > 85.0"
).to_arrow()
```

**Table Management:**
```python
# List tables in namespace
tables = catalog.list_tables("default")

# Drop table
catalog.drop_table(("default", "people"))

# Rename table
catalog.rename_table(
    ("default", "old_name"),
    ("default", "new_name")
)
```

### Apache Spark (Scala)

**SparkSession Configuration:**
```scala
import org.apache.spark.sql.SparkSession

val uri = sys.env("CATALOG_URI")
val warehouse = sys.env("WAREHOUSE")
val token = sys.env("TOKEN")

val spark = SparkSession.builder()
    .appName("R2 Data Catalog Demo")
    .master("local[*]")
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
    .config("spark.sql.catalog.mydemo", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.mydemo.type", "rest")
    .config("spark.sql.catalog.mydemo.uri", uri)
    .config("spark.sql.catalog.mydemo.warehouse", warehouse)
    .config("spark.sql.catalog.mydemo.token", token)
    .getOrCreate()
```

**build.sbt Configuration:**
```scala
name := "R2DataCatalogDemo"
version := "1.0"

val sparkVersion = "3.5.3"
val icebergVersion = "1.8.1"
scalaVersion := "2.12.18"

libraryDependencies ++= Seq(
    "org.apache.spark" %% "spark-core" % sparkVersion,
    "org.apache.spark" %% "spark-sql" % sparkVersion,
    "org.apache.iceberg" % "iceberg-core" % icebergVersion,
    "org.apache.iceberg" % "iceberg-spark-runtime-3.5_2.12" % icebergVersion,
    "org.apache.iceberg" % "iceberg-aws-bundle" % icebergVersion,
)

assembly / assemblyMergeStrategy := {
    case PathList("META-INF", "services", xs @ _*) => MergeStrategy.concat
    case PathList("META-INF", xs @ _*) => MergeStrategy.discard
    case "reference.conf" => MergeStrategy.concat
    case "application.conf" => MergeStrategy.concat
    case x if x.endsWith(".properties") => MergeStrategy.first
    case x => MergeStrategy.first
}
```

**Spark SQL Operations:**
```scala
import spark.implicits._

// Create data
val data = Seq(
    (1, "Alice", 25),
    (2, "Bob", 30),
    (3, "Charlie", 35)
).toDF("id", "name", "age")

// Use catalog
spark.sql("USE mydemo")

// Create namespace
spark.sql("CREATE NAMESPACE IF NOT EXISTS demoNamespace")

// Create/replace table
data.writeTo("demoNamespace.demotable").createOrReplace()

// Query table
val result = spark.sql("SELECT * FROM demoNamespace.demotable WHERE age > 30")
result.show()

// Append data
data.writeTo("demoNamespace.demotable").append()
```

### DuckDB

**CLI Setup:**
```sql
-- Install extensions
INSTALL iceberg;
LOAD iceberg;
INSTALL httpfs;
LOAD httpfs;

-- Create secret
CREATE SECRET r2_secret (
    TYPE ICEBERG,
    TOKEN '<token>'
);

-- Attach catalog
ATTACH '<warehouse_name>' AS my_r2_catalog (
    TYPE ICEBERG,
    ENDPOINT '<catalog_uri>'
);
```

**SQL Operations:**
```sql
-- Create schema
CREATE SCHEMA my_r2_catalog.default;
USE my_r2_catalog.default;

-- Create table
CREATE TABLE my_iceberg_table AS SELECT a FROM range(4) t(a);

-- Show tables
SHOW ALL TABLES;

-- Query
SELECT * FROM my_r2_catalog.default.my_iceberg_table;
```

### PySpark (Python)

**SparkSession Configuration:**
```python
from pyspark.sql import SparkSession
import os

spark = SparkSession.builder \
    .appName("R2 Data Catalog") \
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.r2_catalog", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.r2_catalog.type", "rest") \
    .config("spark.sql.catalog.r2_catalog.uri", os.environ["CATALOG_URI"]) \
    .config("spark.sql.catalog.r2_catalog.warehouse", os.environ["WAREHOUSE"]) \
    .config("spark.sql.catalog.r2_catalog.token", os.environ["TOKEN"]) \
    .getOrCreate()

# Use catalog
spark.sql("USE r2_catalog")

# Create namespace
spark.sql("CREATE NAMESPACE IF NOT EXISTS analytics")

# Create DataFrame
df = spark.createDataFrame([
    (1, "event_a", 100),
    (2, "event_b", 200)
], ["id", "event_type", "value"])

# Write to Iceberg
df.writeTo("analytics.events").createOrReplace()

# Query
result = spark.sql("SELECT * FROM analytics.events WHERE value > 150")
result.show()
```

## Common Use Cases

### 1. Log Analytics Pipeline

**Scenario:** Ingest and query application logs

```python
from pyiceberg.catalog.rest import RestCatalog
import pyarrow as pa
import pandas as pd

# Setup catalog
catalog = RestCatalog(
    name="logs_catalog",
    warehouse=WAREHOUSE,
    uri=CATALOG_URI,
    token=TOKEN,
)

catalog.create_namespace_if_not_exists("logs")

# Create schema for logs
log_schema = pa.schema([
    ("timestamp", pa.timestamp("ms")),
    ("level", pa.string()),
    ("service", pa.string()),
    ("message", pa.string()),
    ("user_id", pa.int64()),
])

# Create table
logs_table = catalog.create_table(
    ("logs", "application_logs"),
    schema=log_schema,
)

# Append logs incrementally
log_data = pa.table({
    "timestamp": [pd.Timestamp.now(), pd.Timestamp.now()],
    "level": ["ERROR", "INFO"],
    "service": ["auth-service", "api-gateway"],
    "message": ["Failed login attempt", "Request processed"],
    "user_id": [12345, 67890],
})

logs_table.append(log_data)

# Query error logs
errors = logs_table.scan(
    row_filter="level = 'ERROR'"
).to_arrow()
```

### 2. Data Warehouse Migration

**Scenario:** Migrate from traditional warehouse to R2 lakehouse

```python
# Export from existing warehouse (e.g., Snowflake)
# Load into R2 Data Catalog

catalog.create_namespace_if_not_exists("warehouse")

# Create dimension table
customers_table = catalog.create_table(
    ("warehouse", "customers"),
    schema=customer_schema,
)

# Batch load initial data
customers_table.overwrite(initial_customer_data)

# Create fact table
orders_table = catalog.create_table(
    ("warehouse", "orders"),
    schema=orders_schema,
)

# Enable compaction for optimal query performance
# (via wrangler - see Wrangler Commands section)
```

### 3. Streaming Data Pipeline

**Scenario:** Ingest streaming events with compaction

```python
# Setup with compaction enabled
catalog.create_namespace_if_not_exists("streaming")

events_table = catalog.create_table(
    ("streaming", "user_events"),
    schema=event_schema,
)

# Continuous ingestion loop
def ingest_batch(events_batch):
    """Ingest small batches frequently"""
    events_arrow = pa.Table.from_pandas(events_batch)
    events_table.append(events_arrow)

# Enable automatic compaction via wrangler:
# npx wrangler r2 bucket catalog compaction enable my-bucket \
#   --target-size 128 --token $TOKEN

# Compaction runs automatically, combining small files
```

### 4. Multi-Engine Analytics

**Scenario:** Use different engines for different workloads

```python
# PyIceberg for data ingestion
from pyiceberg.catalog.rest import RestCatalog

catalog = RestCatalog(
    name="multi_engine",
    warehouse=WAREHOUSE,
    uri=CATALOG_URI,
    token=TOKEN,
)

catalog.create_namespace_if_not_exists("analytics")
table = catalog.create_table(("analytics", "sales"), schema=sales_schema)
table.append(sales_data)
```

```scala
// Spark for batch transformations
val spark = SparkSession.builder()
    .config("spark.sql.catalog.r2", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.r2.type", "rest")
    .config("spark.sql.catalog.r2.uri", catalogUri)
    .config("spark.sql.catalog.r2.warehouse", warehouse)
    .config("spark.sql.catalog.r2.token", token)
    .getOrCreate()

spark.sql("""
    INSERT INTO r2.analytics.sales_aggregates
    SELECT date, region, SUM(amount) 
    FROM r2.analytics.sales 
    GROUP BY date, region
""")
```

```sql
-- DuckDB for ad-hoc analysis
ATTACH 'my-warehouse' AS r2_catalog (
    TYPE ICEBERG,
    ENDPOINT 'https://...'
);

SELECT region, AVG(amount) 
FROM r2_catalog.analytics.sales 
WHERE date >= '2024-01-01'
GROUP BY region;
```

### 5. Time Travel Queries

```python
# Access historical data via snapshots
table = catalog.load_table(("analytics", "sales"))

# Get snapshot history
snapshots = table.metadata.snapshots

# Query specific snapshot
historical_data = table.scan(
    snapshot_id=snapshots[0].snapshot_id
).to_arrow()

# Time travel via timestamp (if supported by engine)
# Configure snapshot expiration to retain history
```

## Table Maintenance

### Compaction

**Why Compaction Matters:**
- Every write creates new files → unbounded file growth
- More files = slower queries, more I/O, higher costs
- Larger metadata files slow query planning
- Smaller files compress less efficiently

**How It Works:**
- Combines small files into larger ones
- Runs automatically once per hour per table
- Compacts up to 2 GB of files per run (beta limit)
- Compacted files prefixed with `compacted-` in `/data/` directory
- Only supports Parquet format currently

**Configuration:**
- **Catalog-level**: All tables, requires API token
- **Table-level**: Specific table only
- **Target file size**: 64-512 MB
- **Retroactive**: Applies to existing tables

### Snapshot Expiration

**Why Snapshot Expiration Matters:**
- Every write creates a new snapshot
- Old snapshots cause:
  - Metadata bloat → slower query planning
  - Increased storage costs
  - Slower table operations

**How It Works:**
- Automatically removes old snapshots
- Two parameters control expiration:
  - `--older-than-days`: Age threshold (default: 30)
  - `--retain-last`: Minimum recent snapshots (default: 5)
- Both conditions must be met for expiration
- Runs automatically

**Configuration:**
- **Catalog-level**: All tables, requires API token
- **Table-level**: Specific table only

## Best Practices

### Authentication & Security
1. **Use Admin Read & Write tokens** for read/write operations
2. **Use Admin Read only tokens** for read-only query engines
3. **Store tokens securely** in environment variables, not code
4. **Rotate tokens regularly** via dashboard
5. **Use catalog-level maintenance** with service tokens for automation

### Performance Optimization
1. **Enable compaction** for all production tables
2. **Choose appropriate target file sizes**:
   - Start with 128 MB for most workloads
   - Tune based on query patterns
3. **Configure snapshot expiration** to match retention requirements
4. **Use partitioning** in table schema for large datasets
5. **Monitor query performance** and adjust compaction settings

### Data Modeling
1. **Use namespaces** to organize tables logically (e.g., by team, project)
2. **Define schemas explicitly** with appropriate data types
3. **Consider partition strategies** for time-series data
4. **Use schema evolution** instead of recreating tables
5. **Test with small datasets** before loading production data

### Operational
1. **Start with table-level maintenance** for critical tables
2. **Upgrade to catalog-level** once patterns are established
3. **Monitor storage costs** and snapshot counts
4. **Document catalog URIs and warehouse names** in team docs
5. **Version control** Wrangler commands in scripts

### Engine-Specific
1. **PyIceberg**: Best for data ingestion and Python notebooks
2. **Spark**: Best for large-scale batch transformations
3. **DuckDB**: Best for ad-hoc analysis and BI tools
4. **Snowflake**: Best for enterprise data warehousing
5. **Multiple engines**: Use R2 Data Catalog as single source of truth

## Limitations (Public Beta)

### Current Constraints
- **Compaction**: Up to 2 GB per hour per table
- **File format**: Only Parquet supported for compaction
- **Orphan file cleanup**: Not yet supported
- **Target file size**: 64-512 MB range
- **Billing**: Free during beta (standard R2 charges apply)

### Not Supported
- Cross-bucket catalogs (one catalog per bucket)
- Custom compaction schedules
- Manual compaction triggers
- Catalog cloning/duplication
- Catalog export/import

## Troubleshooting

### Common Issues

**1. "Unauthorized" errors:**
```
Solution: Verify API token has both:
- R2 Data Catalog permissions
- R2 Storage permissions
Use "Admin Read & Write" from dashboard
```

**2. Catalog not found:**
```
Solution: Ensure catalog is enabled on bucket:
npx wrangler r2 bucket catalog enable <BUCKET_NAME>
```

**3. Connection timeouts:**
```
Solution: Check catalog URI format:
https://<account-id>.r2.cloudflarestorage.com/iceberg/<bucket-name>
Ensure no typos in account ID or bucket name
```

**4. Compaction not running:**
```
Solution for catalog-level:
- Verify API token is provided
- Token must have storage + catalog write permissions

Solution for table-level:
- Check table exists: catalog.table_exists(("namespace", "table"))
- Verify namespace and table names are correct
```

**5. Schema evolution errors:**
```
Solution: Iceberg supports adding/renaming columns, not all operations:
- Can add new columns
- Can rename columns
- Cannot change column types (in most cases)
- Cannot delete columns (they become optional)
```

### Debugging Tips

**1. Verify catalog connectivity:**
```python
from pyiceberg.catalog.rest import RestCatalog

catalog = RestCatalog(
    name="test",
    warehouse=WAREHOUSE,
    uri=CATALOG_URI,
    token=TOKEN,
)

# List namespaces to test connection
print(catalog.list_namespaces())
```

**2. Check table metadata:**
```python
table = catalog.load_table(("namespace", "table"))
print(f"Schema: {table.schema()}")
print(f"Location: {table.location()}")
print(f"Snapshots: {table.metadata.snapshots}")
```

**3. Inspect Wrangler output:**
```bash
# Add --json for structured output
npx wrangler r2 bucket catalog enable my-bucket --json
```

**4. Verify environment variables:**
```python
import os
print(f"Catalog URI: {os.environ.get('CATALOG_URI')}")
print(f"Warehouse: {os.environ.get('WAREHOUSE')}")
print(f"Token: {'***' if os.environ.get('TOKEN') else 'NOT SET'}")
```

## Integration Patterns

### Marimo Notebooks
```python
import marimo

app = marimo.App(width="medium")

@app.cell
def _():
    from pyiceberg.catalog.rest import RestCatalog
    
    WAREHOUSE = "<warehouse>"
    TOKEN = "<token>"
    CATALOG_URI = "<uri>"
    
    catalog = RestCatalog(
        name="my_catalog",
        warehouse=WAREHOUSE,
        uri=CATALOG_URI,
        token=TOKEN,
    )
    return catalog,

@app.cell
def _(catalog):
    catalog.create_namespace_if_not_exists("default")
    return

if __name__ == "__main__":
    app.run()
```

### CI/CD Pipeline
```bash
#!/bin/bash
# deploy-catalog.sh

set -e

# Enable catalog
npx wrangler r2 bucket catalog enable $BUCKET_NAME

# Enable compaction
npx wrangler r2 bucket catalog compaction enable $BUCKET_NAME \
  --target-size 256 \
  --token $R2_CATALOG_TOKEN

# Enable snapshot expiration
npx wrangler r2 bucket catalog snapshot-expiration enable $BUCKET_NAME \
  --token $R2_CATALOG_TOKEN \
  --older-than-days 14 \
  --retain-last 20

echo "Catalog configured for $BUCKET_NAME"
```

### Environment Configuration
```bash
# .env file
CATALOG_URI=https://<account-id>.r2.cloudflarestorage.com/iceberg/<bucket>
WAREHOUSE=<bucket-name>
R2_CATALOG_TOKEN=<token>
```

```python
# Python: Load from .env
from dotenv import load_dotenv
import os

load_dotenv()

catalog = RestCatalog(
    name="prod_catalog",
    warehouse=os.environ["WAREHOUSE"],
    uri=os.environ["CATALOG_URI"],
    token=os.environ["R2_CATALOG_TOKEN"],
)
```

## Reference

### Official Documentation
- R2 Data Catalog: https://developers.cloudflare.com/r2/data-catalog/
- Getting Started: https://developers.cloudflare.com/r2/data-catalog/get-started/
- Managing Catalogs: https://developers.cloudflare.com/r2/data-catalog/manage-catalogs/
- Table Maintenance: https://developers.cloudflare.com/r2/data-catalog/table-maintenance/
- Apache Iceberg: https://iceberg.apache.org/

### Engine Documentation
- PyIceberg: https://py.iceberg.apache.org/
- Apache Spark: https://spark.apache.org/docs/latest/sql-data-sources-iceberg.html
- DuckDB: https://duckdb.org/docs/stable/core_extensions/iceberg/
- Trino: https://trino.io/docs/current/connector/iceberg.html

### Wrangler Commands Reference
```bash
# Catalog
npx wrangler r2 bucket catalog enable <BUCKET>
npx wrangler r2 bucket catalog disable <BUCKET>

# Compaction
npx wrangler r2 bucket catalog compaction enable <BUCKET> [NAMESPACE] [TABLE] \
  --target-size <SIZE> [--token <TOKEN>]
npx wrangler r2 bucket catalog compaction disable <BUCKET> [NAMESPACE] [TABLE]

# Snapshot Expiration
npx wrangler r2 bucket catalog snapshot-expiration enable <BUCKET> [NAMESPACE] [TABLE] \
  --token <TOKEN> --older-than-days <DAYS> --retain-last <COUNT>
npx wrangler r2 bucket catalog snapshot-expiration disable <BUCKET> [NAMESPACE] [TABLE]
```

### API Token Permission Groups
- **Workers R2 Data Catalog Write**: `d229766a2f7f4d299f20eaa8c9b1fde9`
- **Workers R2 Data Catalog Read**: `(ID not documented)`
- **Workers R2 Storage Bucket Item Write**: `2efd5506f9c8494dacb1fa10a3e7d5b6`
- **Workers R2 Storage Bucket Item Read**: `(ID not documented)`

---

## Quick Start Checklist

- [ ] Create R2 bucket
- [ ] Enable catalog on bucket
- [ ] Note Catalog URI and Warehouse name
- [ ] Create API token with Admin Read & Write
- [ ] Test connection with PyIceberg or preferred engine
- [ ] Create namespace
- [ ] Create first table
- [ ] Enable compaction (start with 128 MB target)
- [ ] Configure snapshot expiration (7-30 days)
- [ ] Load sample data
- [ ] Run test queries
- [ ] Document connection details for team
