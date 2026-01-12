# Cloudflare Workers Analytics Engine

Expert guidance for implementing unlimited-cardinality analytics at scale using Cloudflare Workers Analytics Engine. This skill covers ONLY Analytics Engine - NOT the broader Cloudflare platform.

## Core Concepts

### What is Analytics Engine?

Workers Analytics Engine provides:
- **Unlimited-cardinality analytics**: Track metrics with any number of unique dimensions without explosion in cost/complexity
- **Built-in write API**: Non-blocking `writeDataPoint()` method in Workers runtime
- **SQL API**: Query data using SQL with 3 months retention
- **Automatic sampling**: Intelligent weighted sampling for high-volume data without accuracy loss
- **Time-series optimization**: Automatic timestamp field, optimized for time-based queries

### Common Use Cases

1. **Custom customer-facing analytics** - Expose analytics dashboards to your users
2. **Usage-based billing** - Track per-customer/per-feature usage for metered billing
3. **Service health monitoring** - Per-customer/per-user health metrics
4. **High-frequency instrumentation** - Add telemetry to hot code paths without performance impact
5. **Event tracking** - User actions, API calls, errors, performance metrics

## Configuration

### Wrangler Setup

**wrangler.toml:**
```toml
[[analytics_engine_datasets]]
binding = "WEATHER"
dataset = "weather_data"
```

**wrangler.jsonc:**
```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "analytics_engine_datasets": [
    {
      "binding": "WEATHER",
      "dataset": "weather_data"
    }
  ]
}
```

**Key Points:**
- Datasets are created automatically on first write - no manual dashboard setup needed
- `binding` = variable name accessible in Worker env
- `dataset` = logical table name (like SQL table - rows/columns should have consistent meaning)
- Multiple datasets can be defined per Worker

### TypeScript Types

```typescript
interface Env {
  WEATHER: AnalyticsEngineDataset;
}

// The binding exposes this interface
interface AnalyticsEngineDataset {
  writeDataPoint(data: AnalyticsEngineDataPoint): void;
}

interface AnalyticsEngineDataPoint {
  blobs?: string[];      // Up to 20 strings (dimensions for grouping/filtering)
  doubles?: number[];    // Up to 20 numbers (metrics to record)
  indexes?: [string];    // Single string for sampling key (max 96 bytes)
}
```

## Writing Data Points

### Basic Pattern

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Write data point - NON-BLOCKING, no await needed
    env.WEATHER.writeDataPoint({
      blobs: ["Seattle", "USA", "pro_sensor_9000"],
      doubles: [25.0, 0.5],
      indexes: ["customer_123"]
    });
    
    return new Response("OK!");
  }
}
```

### Critical Rules

**DO NOT await `writeDataPoint()`**
- Returns void immediately
- Workers runtime handles writing in background
- Like `console.log()` - fire and forget

**Field Ordering Matters**
- Values are ordered arrays
- Must provide fields in consistent order across all writes
- Document your schema clearly

**Index Field**
- Only provide ONE string despite array type
- Multiple indexes will cause data point to be rejected
- Used as sampling key - choose carefully (see Sampling section)

### Real-World Examples

#### API Usage Tracking

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const customerId = request.headers.get("X-Customer-Id");
    const startTime = Date.now();
    
    // Handle request...
    const response = await handleRequest(request);
    const duration = Date.now() - startTime;
    
    // Track API usage
    env.API_USAGE.writeDataPoint({
      blobs: [
        url.pathname,                      // blob1: endpoint
        request.method,                    // blob2: HTTP method  
        response.status.toString(),        // blob3: status code
        request.cf?.colo as string,        // blob4: datacenter
        request.cf?.country as string      // blob5: country
      ],
      doubles: [
        duration,                          // double1: latency ms
        1                                  // double2: request count (for summing)
      ],
      indexes: [customerId || "anonymous"] // Index by customer for sampling
    });
    
    return response;
  }
}
```

#### Feature Flag Usage

```typescript
env.FEATURE_USAGE.writeDataPoint({
  blobs: [
    userId,
    featureName,
    featureVariant,
    request.cf?.country as string
  ],
  doubles: [1], // Simple counter
  indexes: [userId]
});
```

#### Error Tracking

```typescript
try {
  await riskyOperation();
} catch (error) {
  env.ERRORS.writeDataPoint({
    blobs: [
      error.name,
      error.message.substring(0, 100), // Stay under blob size limit
      request.url,
      request.cf?.colo as string
    ],
    doubles: [1],
    indexes: [customerId]
  });
  throw error;
}
```

#### Performance Monitoring with Request Context

```typescript
env.PERFORMANCE.writeDataPoint({
  blobs: [
    request.url,
    request.cf?.colo as string,
    request.cf?.country as string,
    request.cf?.city as string,
    request.headers.get("user-agent")?.substring(0, 100)
  ],
  doubles: [
    ttfb,        // double1: time to first byte
    totalTime,   // double2: total request time
    dbTime,      // double3: database query time
    cacheHit ? 1 : 0  // double4: cache hit indicator
  ],
  indexes: [sensorId]
});
```

### Limits

- **Max 250 data points per Worker invocation** (per client HTTP request)
- **Max 20 blobs**, max 20 doubles, exactly 1 index per data point
- **Blob size limit**: Total size of all blobs ≤ 16 KB per data point
- **Index size**: Max 96 bytes
- **Data retention**: 3 months

## Querying Data

### SQL API

**Endpoint:**
```
POST https://api.cloudflare.com/client/v4/accounts/{account_id}/analytics_engine/sql
```

**Authentication:**
```bash
Authorization: Bearer <API_TOKEN>
```

Create token at https://dash.cloudflare.com/profile/api-tokens with:
- Permission: Account Analytics > Read
- Scope: Specific account

**Making Queries:**
```bash
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/analytics_engine/sql" \
  --header "Authorization: Bearer <API_TOKEN>" \
  --data "SELECT blob1 AS city, COUNT() AS total FROM weather_data WHERE timestamp > NOW() - INTERVAL '1' DAY GROUP BY city"
```

### Table Structure

Every dataset becomes a table with this schema:

| Column | Type | Description |
|--------|------|-------------|
| `dataset` | string | Dataset name (in every row) |
| `timestamp` | DateTime | Auto-populated write time |
| `_sample_interval` | integer | Sampling rate (1 = unsampled, 100 = 1% sample) |
| `index1` | string | Your index value |
| `blob1` ... `blob20` | string | Your blob values |
| `double1` ... `double20` | double | Your double values |

### Essential Query Patterns

#### Basic Query with Aliases

```sql
SELECT
  timestamp,
  blob1 AS location_id,
  blob2 AS sensor_type,
  double1 AS temperature,
  double2 AS humidity
FROM weather_data
WHERE timestamp > NOW() - INTERVAL '7' DAY
```

#### Sampling-Aware Aggregations

**CRITICAL**: Account for `_sample_interval` in aggregations

| Without Sampling | With Sampling (CORRECT) |
|-----------------|-------------------------|
| `COUNT()` | `SUM(_sample_interval)` |
| `SUM(double1)` | `SUM(_sample_interval * double1)` |
| `AVG(double1)` | `SUM(_sample_interval * double1) / SUM(_sample_interval)` |
| `quantile(0.95)(double1)` | `quantileExactWeighted(0.95)(double1, _sample_interval)` |

**Examples:**

```sql
-- Count events (sampling-aware)
SELECT
  blob1 AS customer_id,
  SUM(_sample_interval) AS total_requests
FROM api_usage
WHERE timestamp > NOW() - INTERVAL '1' DAY
GROUP BY customer_id;

-- Average latency (sampling-aware)
SELECT
  blob1 AS endpoint,
  SUM(_sample_interval * double1) / SUM(_sample_interval) AS avg_latency_ms
FROM api_usage
WHERE timestamp > NOW() - INTERVAL '1' HOUR
GROUP BY endpoint;

-- 95th percentile with sampling
SELECT
  blob1 AS endpoint,
  quantileExactWeighted(0.95)(double1, _sample_interval) AS p95_latency
FROM api_usage
WHERE timestamp > NOW() - INTERVAL '1' DAY
GROUP BY endpoint;
```

#### Time Series Queries

```sql
-- Round to 5-minute buckets
SELECT
  intDiv(toUInt32(timestamp), 300) * 300 AS t,
  blob1 AS customer_id,
  SUM(_sample_interval * double1) AS total_bytes
FROM bandwidth_usage
WHERE timestamp >= NOW() - INTERVAL '1' DAY
GROUP BY t, customer_id
ORDER BY t, total_bytes DESC;

-- Hourly aggregation
SELECT
  toStartOfHour(timestamp) AS hour,
  blob1 AS feature,
  SUM(_sample_interval) AS usage_count
FROM feature_usage
WHERE timestamp >= NOW() - INTERVAL '7' DAY
GROUP BY hour, feature
ORDER BY hour DESC;
```

#### Top N Queries

```sql
-- Top 10 customers by API usage
SELECT
  index1 AS customer_id,
  SUM(_sample_interval) AS request_count,
  SUM(_sample_interval * double1) / SUM(_sample_interval) AS avg_latency
FROM api_metrics
WHERE
  timestamp > NOW() - INTERVAL '1' DAY
  AND double1 > 0
GROUP BY customer_id
ORDER BY request_count DESC
LIMIT 10;
```

#### Filtering Patterns

```sql
-- Filter on blobs and doubles
SELECT
  blob1 AS endpoint,
  blob3 AS status_code,
  SUM(_sample_interval) AS count
FROM api_usage
WHERE
  timestamp > NOW() - INTERVAL '1' HOUR
  AND blob3 IN ('500', '502', '503')  -- Error status codes
  AND double1 > 1000  -- Latency > 1 second
GROUP BY endpoint, status_code;
```

### Querying from a Worker

```typescript
interface Env {
  ACCOUNT_ID: string;
  API_TOKEN: string;  // Store as secret: npx wrangler secret put API_TOKEN
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const query = `
      SELECT
        blob1 AS city,
        SUM(_sample_interval * double1) / SUM(_sample_interval) AS avg_temp,
        SUM(_sample_interval * double2) / SUM(_sample_interval) AS avg_humidity
      FROM weather_data
      WHERE timestamp > NOW() - INTERVAL '1' DAY
      GROUP BY city
      ORDER BY city
    `;
    
    const apiUrl = `https://api.cloudflare.com/client/v4/accounts/${env.ACCOUNT_ID}/analytics_engine/sql`;
    
    const response = await fetch(apiUrl, {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${env.API_TOKEN}`
      },
      body: query
    });
    
    if (response.status !== 200) {
      console.error("Query failed:", await response.text());
      return new Response("Query failed", { status: 500 });
    }
    
    const result = await response.json();
    
    // result.data is array of objects with column names as keys
    // result.meta contains column metadata
    // result.rows contains row count
    
    return Response.json(result.data);
  }
}
```

**Response Format:**
```json
{
  "data": [
    {"city": "Seattle", "avg_temp": 25.5, "avg_humidity": 0.65},
    {"city": "Portland", "avg_temp": 23.2, "avg_humidity": 0.72}
  ],
  "meta": [
    {"name": "city", "type": "String"},
    {"name": "avg_temp", "type": "Float64"},
    {"name": "avg_humidity", "type": "Float64"}
  ],
  "rows": 2
}
```

### Grafana Integration

Use [Altinity ClickHouse plugin](https://grafana.com/grafana/plugins/vertamedia-clickhouse-datasource/)

**Configuration:**
- URL: `https://api.cloudflare.com/client/v4/accounts/<account_id>/analytics_engine/sql`
- Auth: Custom header `Authorization: Bearer <token>`
- All other auth settings OFF

**Time series query with Grafana macros:**
```sql
SELECT
  $timeSeries AS t,
  blob1 AS label,
  SUM(_sample_interval * double1) / SUM(_sample_interval) AS avg_metric
FROM dataset_name
WHERE $timeFilter
GROUP BY label, t
ORDER BY t
```

Set `Column:DateTime` to `timestamp` in query builder for macros to work.

## Sampling

### How Sampling Works

Analytics Engine uses **equitable sampling** + **adaptive bit rate (ABR)** sampling:

**Write-time sampling:**
- Samples based on `index1` field
- High-volume indexes get sampled more aggressively
- Low-volume indexes may remain unsampled
- Goal: Equal data points stored per index regardless of volume

**Read-time sampling (ABR):**
- Queries over longer time ranges use higher sample intervals
- Ensures queries complete in fixed time budget
- Data stored at multiple resolutions (100%, 10%, 1%)

### Why Sampling is Good

1. **Cost efficiency** - Write millions of events, pay for ~10% storage
2. **Query performance** - Fixed-time queries regardless of dataset size
3. **Statistical accuracy** - Sufficient sample size for accurate results
4. **No capacity limits** - Scale to any volume without degradation

### Choosing an Index

**Index determines sampling granularity**. Choose based on your query patterns:

**Good index choices:**
- Customer ID - if you query per-customer metrics
- User ID - if you analyze per-user behavior
- Hostname - if you track per-site analytics
- Tenant ID - for multi-tenant SaaS
- Device ID - for IoT scenarios

**Bad index choices:**
- UUID per event - defeats sampling purpose, slows queries
- Timestamp - not useful for aggregation queries
- Constant value - all data sampled together

**Index Properties:**

What you CAN do:
✓ Accurate stats across all index values
✓ Accurate count of unique index values
✓ Accurate aggregations within one index value
✓ See top N values of non-index fields
✓ Filter on most fields
✓ Run quantiles and other aggregations

What you CANNOT do:
✗ Accurate unique counts of non-index fields (e.g., count unique URLs if indexed by customer)
✗ Observe very rare values of non-index fields
✗ Accurate queries across multiple specific index values simultaneously (can query one or all)
✗ Retrieve specific individual records
✗ Reconstruct exact event sequences

### Example: Multi-tenant SaaS

```typescript
// Good: Index by tenant
env.METRICS.writeDataPoint({
  blobs: [featureName, userId, region],
  doubles: [latency, requestSize],
  indexes: [tenantId]  // Each tenant gets fair sampling
});

// Query per-tenant metrics (accurate even with sampling)
// SELECT tenant_id, SUM(_sample_interval) FROM metrics GROUP BY tenant_id
```

## Advanced Patterns

### Non-blocking Batch Writes

```typescript
// Write multiple data points in one invocation
const metrics = await collectMetrics();

for (const metric of metrics) {
  env.METRICS.writeDataPoint(metric);
  // No await needed, all happen in background
}

// All 250 max data points can be written without blocking
```

### Wrapper Function for Consistency

```typescript
interface MetricData {
  endpoint: string;
  method: string;
  status: number;
  country: string;
  latency: number;
  customerId: string;
}

function trackAPICall(env: Env, data: MetricData): void {
  env.API_METRICS.writeDataPoint({
    blobs: [
      data.endpoint,
      data.method,
      data.status.toString(),
      data.country
    ],
    doubles: [
      data.latency,
      1  // Request count
    ],
    indexes: [data.customerId]
  });
}

// Usage
trackAPICall(env, {
  endpoint: "/api/users",
  method: "GET",
  status: 200,
  country: "US",
  latency: 45,
  customerId: "customer_123"
});
```

### Conditional Writes

```typescript
// Only track slow requests
if (latency > 1000) {
  env.SLOW_REQUESTS.writeDataPoint({
    blobs: [endpoint, customerId],
    doubles: [latency],
    indexes: [customerId]
  });
}

// Sample specific user actions
if (action === "purchase" || action === "signup") {
  env.CONVERSIONS.writeDataPoint({
    blobs: [action, userId, source],
    doubles: [amount],
    indexes: [userId]
  });
}
```

### Time Bucketing Best Practices

```sql
-- 5-minute buckets
intDiv(toUInt32(timestamp), 300) * 300 AS bucket_5min

-- 1-hour buckets
toStartOfHour(timestamp) AS bucket_hour

-- 1-day buckets
toStartOfDay(timestamp) AS bucket_day

-- Custom interval (e.g., 15 minutes = 900 seconds)
intDiv(toUInt32(timestamp), 900) * 900 AS bucket_15min
```

### Dealing with Blob Size Limits

```typescript
// Truncate long strings to fit in 16KB limit
const truncatedMessage = error.message.substring(0, 500);
const truncatedUrl = request.url.substring(0, 200);

env.EVENTS.writeDataPoint({
  blobs: [
    truncatedMessage,
    truncatedUrl,
    userId
  ],
  doubles: [1],
  indexes: [userId]
});

// Or hash long identifiers
import { createHash } from 'crypto';
const hashedSessionId = createHash('sha256')
  .update(longSessionId)
  .digest('hex')
  .substring(0, 32);
```

## Common Pitfalls

### ❌ Awaiting writeDataPoint()

```typescript
// WRONG
await env.METRICS.writeDataPoint(data);

// RIGHT
env.METRICS.writeDataPoint(data);
```

### ❌ Forgetting Sample Interval in Queries

```sql
-- WRONG: Undercounts when data is sampled
SELECT COUNT() FROM metrics;

-- RIGHT: Accounts for sampling
SELECT SUM(_sample_interval) FROM metrics;
```

### ❌ Using Multiple Indexes

```typescript
// WRONG: Data point will be rejected
env.METRICS.writeDataPoint({
  blobs: ["data"],
  doubles: [123],
  indexes: ["customer_1", "region_us"]  // Only one allowed!
});

// RIGHT: Use single index, put other dimension in blob
env.METRICS.writeDataPoint({
  blobs: ["data", "region_us"],
  doubles: [123],
  indexes: ["customer_1"]
});
```

### ❌ Inconsistent Field Order

```typescript
// Write 1
env.METRICS.writeDataPoint({
  blobs: ["endpoint", "method"],  // blob1=endpoint, blob2=method
  doubles: [latency]
});

// Write 2 - WRONG ORDER
env.METRICS.writeDataPoint({
  blobs: ["method", "endpoint"],  // blob1=method, blob2=endpoint - INCONSISTENT!
  doubles: [latency]
});

// Queries will mix up data - maintain consistent schema
```

### ❌ Exceeding Blob Size Limit

```typescript
// WRONG: Might exceed 16KB if url is very long
env.METRICS.writeDataPoint({
  blobs: [request.url],  // Could be thousands of chars with query params
  doubles: [1]
});

// RIGHT: Truncate or normalize
env.METRICS.writeDataPoint({
  blobs: [new URL(request.url).pathname.substring(0, 200)],
  doubles: [1]
});
```

### ❌ Using UUID as Index

```typescript
// WRONG: Defeats sampling, slows aggregation queries
env.METRICS.writeDataPoint({
  blobs: [customerId, endpoint],
  doubles: [latency],
  indexes: [crypto.randomUUID()]  // Every event sampled separately!
});

// RIGHT: Index by meaningful grouping dimension
env.METRICS.writeDataPoint({
  blobs: [endpoint, requestId],
  doubles: [latency],
  indexes: [customerId]  // Queries per customer will be accurate
});
```

## SQL Reference Quick Guide

**Supported SQL Features:**
- SELECT, WHERE, GROUP BY, ORDER BY, LIMIT
- JOIN operations
- Aggregate functions: SUM, COUNT, AVG, MIN, MAX, quantileExactWeighted
- Date/time functions: NOW(), INTERVAL, toStartOfHour, toStartOfDay, intDiv, toUInt32
- String functions: substring, concat, lower, upper
- Mathematical functions: round, floor, ceil, abs
- Conditional functions: if, multiIf

**Common Functions:**

```sql
-- Date/time
NOW()                              -- Current timestamp
NOW() - INTERVAL '1' DAY           -- 24 hours ago
NOW() - INTERVAL '7' DAY           -- 7 days ago
toStartOfHour(timestamp)           -- Round to hour
toStartOfDay(timestamp)            -- Round to day
intDiv(toUInt32(timestamp), 300) * 300  -- Round to 5 minutes

-- Aggregates
SUM(_sample_interval)                      -- Count with sampling
SUM(_sample_interval * double1)            -- Sum with sampling
SUM(_sample_interval * double1) / SUM(_sample_interval)  -- Average with sampling
quantileExactWeighted(0.95)(double1, _sample_interval)   -- 95th percentile with sampling

-- String manipulation
substring(blob1, 1, 100)           -- First 100 chars
concat(blob1, ' - ', blob2)        -- Concatenate strings
lower(blob1)                       -- Lowercase

-- Conditionals
if(double1 > 100, 'slow', 'fast')  -- Simple if
multiIf(double1 > 1000, 'very_slow', double1 > 100, 'slow', 'fast')  -- Multiple conditions
```

**FORMAT clause:**
```sql
SELECT * FROM dataset FORMAT JSON        -- Default, returns JSON
SELECT * FROM dataset FORMAT JSONCompact -- Compact JSON
SELECT * FROM dataset FORMAT CSV         -- CSV output
```

## Architecture Recommendations

### Data Model Design

1. **Plan your schema** - Document blob/double meanings upfront
2. **Keep blobs semantic** - Store categorical data (strings, enums)
3. **Keep doubles numeric** - Store metrics, counters, measurements
4. **Choose index carefully** - Based on query access patterns
5. **Version your schema** - Use blob field for schema version if evolving

Example schema documentation:

```typescript
/**
 * API_METRICS dataset schema
 * 
 * Blobs:
 * - blob1: endpoint path (e.g., "/api/users")
 * - blob2: HTTP method (GET, POST, etc.)
 * - blob3: status code (200, 404, etc.)
 * - blob4: datacenter (e.g., "SJC", "IAD")
 * - blob5: country code (e.g., "US", "GB")
 * 
 * Doubles:
 * - double1: latency in milliseconds
 * - double2: request body size in bytes
 * - double3: response body size in bytes
 * 
 * Index:
 * - index1: customer_id for equitable sampling
 */
```

### Multi-Dataset Strategy

Use separate datasets for different data types:

```toml
[[analytics_engine_datasets]]
binding = "API_METRICS"
dataset = "api_usage"

[[analytics_engine_datasets]]
binding = "ERROR_TRACKING"
dataset = "errors"

[[analytics_engine_datasets]]
binding = "FEATURE_FLAGS"
dataset = "feature_usage"
```

Benefits:
- Clearer data models
- Isolated retention/sampling per use case
- Simpler queries
- Independent schema evolution

### Query Optimization

1. **Always filter on timestamp** - Use recent time ranges
2. **Limit result sets** - Use LIMIT clause
3. **Pre-aggregate in query** - Don't fetch raw data if you need aggregates
4. **Use appropriate time buckets** - Match granularity to visualization needs
5. **Leverage indexes** - Filter/group by index1 when possible

## Testing and Development

### Local Development

```typescript
// Mock Analytics Engine for local testing
interface MockAnalyticsEngine {
  writeDataPoint(data: AnalyticsEngineDataPoint): void;
}

const mockAE: MockAnalyticsEngine = {
  writeDataPoint(data) {
    console.log("Analytics event:", JSON.stringify(data, null, 2));
  }
};

// Use in dev
const ae = env.ANALYTICS ?? mockAE;
ae.writeDataPoint({ blobs: ["test"], doubles: [1], indexes: ["dev"] });
```

### Verifying Data Writes

```bash
# Check dataset exists
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/analytics_engine/sql" \
  --header "Authorization: Bearer <TOKEN>" \
  --data "SHOW TABLES"

# Check recent data
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/analytics_engine/sql" \
  --header "Authorization: Bearer <TOKEN>" \
  --data "SELECT * FROM your_dataset WHERE timestamp > NOW() - INTERVAL '5' MINUTE LIMIT 10"

# Count total events
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/analytics_engine/sql" \
  --header "Authorization: Bearer <TOKEN>" \
  --data "SELECT SUM(_sample_interval) AS total FROM your_dataset WHERE timestamp > NOW() - INTERVAL '1' HOUR"
```

## Resources

- [Official Docs](https://developers.cloudflare.com/analytics/analytics-engine/)
- [SQL Reference](https://developers.cloudflare.com/analytics/analytics-engine/sql-reference/)
- [Sampling Deep Dive](https://developers.cloudflare.com/analytics/analytics-engine/sampling/)
- [Grafana Setup](https://developers.cloudflare.com/analytics/analytics-engine/grafana/)
- [Limits](https://developers.cloudflare.com/analytics/analytics-engine/limits/)

## Quick Reference Card

```typescript
// Write data point (non-blocking)
env.DATASET.writeDataPoint({
  blobs: ["str1", "str2"],    // Max 20, total ≤16KB
  doubles: [123, 456],         // Max 20 numbers
  indexes: ["key"]             // Exactly 1 string, ≤96 bytes
});

// Query with sampling awareness
SELECT
  blob1,
  SUM(_sample_interval) AS count,
  SUM(_sample_interval * double1) / SUM(_sample_interval) AS avg
FROM dataset
WHERE timestamp > NOW() - INTERVAL '1' DAY
GROUP BY blob1;

// Time series bucketing
intDiv(toUInt32(timestamp), 300) * 300 AS t  -- 5-min buckets
toStartOfHour(timestamp) AS t                 -- 1-hour buckets

// Limits
- 250 data points per Worker invocation
- 20 blobs, 20 doubles, 1 index per point
- 16KB total blob size per point
- 96 bytes per index
- 3 months retention
```
