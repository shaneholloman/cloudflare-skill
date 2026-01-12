# Cloudflare Observability Skill

**Purpose**: Comprehensive guidance for implementing observability in Cloudflare Workers, covering traces, logs, metrics, and analytics.

**Scope**: Cloudflare Observability features ONLY - Workers Logs, Traces, Analytics Engine, Logpush, Metrics & Analytics, and OpenTelemetry exports.

---

## Core Components

### 1. Workers Logs
Automatically collect, store, filter, and analyze logging data from Workers.

**Key Features**:
- Invocation logs (request/response metadata)
- Custom logs via `console.log()`
- Errors and uncaught exceptions
- 3-7 day retention (Free/Paid)

**When to Use**:
- Debug issues in production
- Track custom application events
- Monitor request/response patterns
- Analyze errors and exceptions

### 2. Workers Traces
End-to-end visibility into request lifecycle with automatic instrumentation.

**Key Features**:
- Automatic instrumentation (no code changes)
- Fetch calls, binding interactions, handler lifecycle
- OpenTelemetry-compatible
- Performance bottleneck identification

**When to Use**:
- Identify slow requests
- Debug complex request flows
- Measure subrequest performance
- Analyze KV/R2/Durable Object latency

### 3. Workers Analytics Engine
Unlimited-cardinality time-series analytics with SQL queries.

**Key Features**:
- Custom business metrics
- High-cardinality dimensions (customer IDs, API keys)
- SQL API for querying
- Non-blocking writes (no latency impact)

**When to Use**:
- Usage-based billing tracking
- Per-customer analytics
- Custom performance metrics
- Application-specific events

### 4. Workers Logpush
Send Workers Trace Event Logs to third-party destinations.

**Key Features**:
- Push to storage/SIEM/logging providers
- Includes metadata, console.log(), uncaught exceptions
- Configurable filters and sampling
- Enterprise feature (Workers Paid for trace events)

**When to Use**:
- Send logs to external monitoring systems
- Long-term log retention
- Compliance requirements
- Integration with existing observability stack

### 5. Workers Metrics & Analytics
Built-in metrics for Worker performance and traffic analysis.

**Key Features**:
- Request counts (success/error/subrequests)
- CPU and wall time quantiles
- Invocation statuses
- GraphQL API access

**When to Use**:
- Monitor Worker health
- Track success/error rates
- Analyze performance trends
- Understand resource usage

---

## Configuration Patterns

### Enable Workers Logs

```jsonc
// wrangler.jsonc
{
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1  // 100% sampling (default)
  }
}
```

```toml
# wrangler.toml
[observability]
enabled = true
head_sampling_rate = 1  # 100% sampling
```

**Best Practice**: Use structured JSON logging for better indexing

```typescript
// Good - structured logging
console.log({ 
  user_id: 123, 
  action: "login", 
  status: "success",
  duration_ms: 45
});

// Avoid - unstructured string
console.log("user_id: 123 logged in successfully in 45ms");
```

### Enable Workers Traces

```jsonc
// wrangler.jsonc
{
  "observability": {
    "traces": {
      "enabled": true,
      "head_sampling_rate": 0.05  // 5% sampling
    }
  }
}
```

```toml
# wrangler.toml
[observability.traces]
enabled = true
head_sampling_rate = 0.05  # 5% sampling
```

**Note**: Default sampling is 100%. For high-traffic Workers, use lower sampling (0.01-0.1).

### Configure Analytics Engine

```jsonc
// wrangler.jsonc
{
  "analytics_engine_datasets": [
    {
      "binding": "ANALYTICS",
      "dataset": "my_analytics"
    }
  ]
}
```

```toml
# wrangler.toml
[[analytics_engine_datasets]]
binding = "ANALYTICS"
dataset = "my_analytics"
```

### Export to OpenTelemetry Destinations

```jsonc
// wrangler.jsonc
{
  "observability": {
    "traces": {
      "enabled": true,
      "destinations": ["honeycomb-traces"],
      "head_sampling_rate": 0.05,
      "persist": false  // Disable Cloudflare dashboard storage
    },
    "logs": {
      "enabled": true,
      "destinations": ["grafana-logs"],
      "head_sampling_rate": 0.6,
      "persist": false
    }
  }
}
```

```toml
# wrangler.toml
[observability.traces]
enabled = true
destinations = ["honeycomb-traces"]
head_sampling_rate = 0.05
persist = false

[observability.logs]
enabled = true
destinations = ["grafana-logs"]
head_sampling_rate = 0.6
persist = false
```

**OTLP Endpoints**:
- Honeycomb: `https://api.honeycomb.io/v1/{traces|logs}`
- Grafana Cloud: `https://otlp-gateway-{region}.grafana.net/otlp/v1/{traces|logs}`
- Axiom: `https://api.axiom.co/v1/{traces|logs}`
- Sentry: `https://{HOST}/api/{PROJECT_ID}/integration/otlp/v1/{traces|logs}`

### Enable Workers Logpush

```jsonc
// wrangler.jsonc
{
  "logpush": true
}
```

```toml
# wrangler.toml
logpush = true
```

**Note**: Must create Logpush job in dashboard or via API first.

---

## Code Patterns

### Analytics Engine: Write Data Points

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const start = Date.now();
    
    // Non-blocking write - no await needed
    env.ANALYTICS.writeDataPoint({
      'blobs': [
        request.cf?.city || 'unknown',
        request.cf?.country || 'unknown',
        request.method
      ],
      'doubles': [
        Date.now() - start,  // response_time_ms
        1                     // request_count
      ],
      'indexes': [request.cf?.colo || 'unknown']
    });
    
    return new Response('OK');
  }
}
```

**Data Point Structure**:
- **Blobs** (strings): Dimensions for filtering/grouping (max 20)
- **Doubles** (numbers): Numeric metrics (max 20)
- **Indexes** (strings): Sampling key (provide exactly 1)

### Analytics Engine: Query with SQL API

```bash
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/analytics_engine/sql" \
  --header "Authorization: Bearer <API_TOKEN>" \
  --data "SELECT 
    blob1 AS city,
    blob2 AS country,
    SUM(_sample_interval * double1) / SUM(_sample_interval) AS avg_response_ms,
    SUM(_sample_interval * double2) AS total_requests
  FROM my_analytics
  WHERE timestamp > NOW() - INTERVAL '1' DAY
  GROUP BY city, country
  ORDER BY total_requests DESC
  LIMIT 10"
```

**Sampling Considerations**:
- Always multiply by `_sample_interval` for accurate aggregations
- Use `SUM(_sample_interval)` instead of `COUNT()`
- For averages: `SUM(_sample_interval * value) / SUM(_sample_interval)`

### Structured Logging Pattern

```typescript
interface LogContext {
  request_id: string;
  user_id?: string;
  endpoint: string;
  method: string;
  status?: number;
  duration_ms?: number;
  error?: string;
}

function log(level: 'info' | 'warn' | 'error', ctx: LogContext) {
  console.log({
    level,
    timestamp: new Date().toISOString(),
    ...ctx
  });
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const requestId = crypto.randomUUID();
    const start = Date.now();
    const url = new URL(request.url);
    
    try {
      log('info', {
        request_id: requestId,
        endpoint: url.pathname,
        method: request.method
      });
      
      const response = await handleRequest(request);
      
      log('info', {
        request_id: requestId,
        endpoint: url.pathname,
        method: request.method,
        status: response.status,
        duration_ms: Date.now() - start
      });
      
      return response;
    } catch (error) {
      log('error', {
        request_id: requestId,
        endpoint: url.pathname,
        method: request.method,
        duration_ms: Date.now() - start,
        error: error instanceof Error ? error.message : String(error)
      });
      throw error;
    }
  }
}
```

### Tracing: Automatic Instrumentation

No code changes required! Simply enable traces in wrangler config:

```typescript
// This code automatically generates trace spans:
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Automatic span: fetch handler
    
    // Automatic span: fetch call
    const apiResponse = await fetch('https://api.example.com/data');
    
    // Automatic span: KV read
    const cached = await env.KV.get('key');
    
    // Automatic span: R2 get
    const object = await env.BUCKET.get('file.txt');
    
    // Automatic span: Durable Object call
    const stub = env.COUNTER.get(env.COUNTER.idFromName('counter'));
    await stub.increment();
    
    return new Response('OK');
  }
}
```

**Instrumented Operations**:
- Fetch calls (all outbound HTTP)
- KV operations (get, put, delete, list)
- R2 operations (get, put, delete, head, list)
- Durable Object invocations
- Queue operations
- Handler lifecycle (fetch, scheduled, queue, etc.)

---

## Common Use Cases

### 1. Usage-Based Billing

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const customerId = request.headers.get('X-Customer-ID');
    const apiKey = request.headers.get('X-API-Key');
    
    if (!customerId || !apiKey) {
      return new Response('Unauthorized', { status: 401 });
    }
    
    // Track API usage per customer
    env.ANALYTICS.writeDataPoint({
      'blobs': [customerId, request.url, request.method],
      'doubles': [1], // request_count
      'indexes': [customerId]
    });
    
    return processRequest(request);
  }
}
```

**Query for billing**:
```sql
SELECT
  blob1 AS customer_id,
  SUM(_sample_interval * double1) AS total_api_calls
FROM api_usage
WHERE timestamp >= DATE_TRUNC('month', NOW())
GROUP BY customer_id
ORDER BY total_api_calls DESC
```

### 2. Performance Monitoring

```typescript
async function monitoredFetch(url: string, env: Env): Promise<Response> {
  const start = Date.now();
  
  try {
    const response = await fetch(url);
    const duration = Date.now() - start;
    
    env.ANALYTICS.writeDataPoint({
      'blobs': [url, response.status.toString(), 'success'],
      'doubles': [duration, 1],
      'indexes': [new URL(url).hostname]
    });
    
    return response;
  } catch (error) {
    const duration = Date.now() - start;
    
    env.ANALYTICS.writeDataPoint({
      'blobs': [url, '0', 'error'],
      'doubles': [duration, 1],
      'indexes': [new URL(url).hostname]
    });
    
    throw error;
  }
}
```

### 3. Error Tracking with Context

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    try {
      return await processRequest(request, env);
    } catch (error) {
      const errorLog = {
        error_type: error instanceof Error ? error.constructor.name : 'Unknown',
        error_message: error instanceof Error ? error.message : String(error),
        stack_trace: error instanceof Error ? error.stack : undefined,
        request_url: request.url,
        request_method: request.method,
        user_agent: request.headers.get('user-agent'),
        cf_ray: request.headers.get('cf-ray'),
        timestamp: new Date().toISOString()
      };
      
      console.error(errorLog);
      
      return new Response('Internal Server Error', { status: 500 });
    }
  }
}
```

### 4. Real-Time Log Streaming

```bash
# Stream logs in real-time during development
wrangler tail <WORKER_NAME>

# Filter logs
wrangler tail <WORKER_NAME> --status error
wrangler tail <WORKER_NAME> --method POST
wrangler tail <WORKER_NAME> --search "user_id"

# Sample logs
wrangler tail <WORKER_NAME> --sampling-rate 0.1
```

### 5. Multi-Destination Observability

```jsonc
// wrangler.jsonc - Send to multiple destinations
{
  "observability": {
    "traces": {
      "enabled": true,
      "destinations": ["honeycomb-prod", "grafana-backup"],
      "head_sampling_rate": 0.1
    },
    "logs": {
      "enabled": true,
      "destinations": ["datadog-logs", "s3-archive"],
      "head_sampling_rate": 0.5
    }
  }
}
```

---

## API Reference

### GraphQL Analytics API

**Endpoint**: `https://api.cloudflare.com/client/v4/graphql`

**Query Workers Metrics**:
```graphql
query {
  viewer {
    accounts(filter: { accountTag: $accountId }) {
      workersInvocationsAdaptive(
        limit: 100
        filter: {
          datetime_geq: "2025-01-01T00:00:00Z"
          datetime_leq: "2025-01-31T23:59:59Z"
          scriptName: "my-worker"
        }
      ) {
        sum {
          requests
          errors
          subrequests
        }
        quantiles {
          cpuTimeP50
          cpuTimeP99
          wallTimeP50
          wallTimeP99
        }
      }
    }
  }
}
```

### Analytics Engine SQL API

**Endpoint**: `https://api.cloudflare.com/client/v4/accounts/{account_id}/analytics_engine/sql`

**Authentication**: `Authorization: Bearer <API_TOKEN>` (Account Analytics Read permission)

**Common Queries**:

```sql
-- List all datasets
SHOW TABLES;

-- Time-series aggregation (5-minute buckets)
SELECT
  intDiv(toUInt32(timestamp), 300) * 300 AS t,
  blob1 AS dimension,
  SUM(_sample_interval * double1) AS metric_sum
FROM my_dataset
WHERE timestamp > NOW() - INTERVAL '1' DAY
GROUP BY t, dimension
ORDER BY t ASC;

-- Quantile calculation (weighted by sample interval)
SELECT
  blob1 AS category,
  QUANTILEEXACTWEIGHTED(0.5)(double1, _sample_interval) AS p50,
  QUANTILEEXACTWEIGHTED(0.95)(double1, _sample_interval) AS p95,
  QUANTILEEXACTWEIGHTED(0.99)(double1, _sample_interval) AS p99
FROM my_dataset
WHERE timestamp > NOW() - INTERVAL '1' HOUR
GROUP BY category;

-- JSON output format
SELECT * FROM my_dataset LIMIT 10 FORMAT JSON;
```

### Logpush API

**Create Logpush Job**:
```bash
curl "https://api.cloudflare.com/client/v4/accounts/<ACCOUNT_ID>/logpush/jobs" \
  --header 'X-Auth-Key: <API_KEY>' \
  --header 'X-Auth-Email: <EMAIL>' \
  --header 'Content-Type: application/json' \
  --data '{
    "name": "workers-logpush",
    "output_options": {
      "field_names": ["Event", "EventTimestampMs", "Outcome", "Exceptions", "Logs", "ScriptName"]
    },
    "destination_conf": "r2://<BUCKET_PATH>/{DATE}?account-id=<ACCOUNT_ID>&access-key-id=<R2_ACCESS_KEY_ID>&secret-access-key=<R2_SECRET_ACCESS_KEY>",
    "dataset": "workers_trace_events",
    "enabled": true,
    "filter": "{\"where\": {\"key\":\"Outcome\",\"operator\":\"!eq\",\"value\":\"exception\"}}"
  }'
```

---

## Wrangler Commands

```bash
# Real-time log tailing
wrangler tail <WORKER_NAME>
wrangler tail <WORKER_NAME> --status error --format pretty
wrangler tail <WORKER_NAME> --sampling-rate 0.1

# Deploy with observability
wrangler deploy

# View worker info
wrangler deployments list <WORKER_NAME>

# Validate configuration
wrangler deploy --dry-run
```

---

## Troubleshooting

### Logs not appearing

**Check**:
1. `observability.enabled = true` in wrangler config
2. Worker has been redeployed after config change
3. Worker is receiving traffic
4. Sampling rate is not too low (check `head_sampling_rate`)
5. Log size under 256 KB limit

**Solution**:
```bash
# Verify config
cat wrangler.toml | grep -A 5 observability

# Check deployment
wrangler deployments list <WORKER_NAME>

# Test with curl
curl https://your-worker.workers.dev
```

### Traces not being captured

**Check**:
1. `observability.traces.enabled = true`
2. `head_sampling_rate` is appropriate (1.0 for testing)
3. Worker deployed after traces enabled
4. Check destination status in dashboard

**Solution**:
```jsonc
// Temporarily set to 100% sampling for debugging
{
  "observability": {
    "traces": {
      "enabled": true,
      "head_sampling_rate": 1.0
    }
  }
}
```

### Analytics Engine queries returning incorrect counts

**Problem**: Using `COUNT()` instead of `SUM(_sample_interval)`

**Solution**:
```sql
-- Wrong
SELECT COUNT() FROM dataset;

-- Correct (accounts for sampling)
SELECT SUM(_sample_interval) FROM dataset;
```

### Logpush events being truncated

**Issue**: Combined `logs` + `exceptions` exceed 16,384 character limit

**Solution**:
1. Reduce verbosity of console.log messages
2. Use structured logging with shorter field names
3. Consider sampling or filtering less important logs
4. Send to multiple destinations with different filters

### High observability costs

**Strategies**:
1. Reduce `head_sampling_rate` for high-traffic Workers
2. Disable `invocation_logs` if not needed
3. Set `persist: false` when exporting to external destinations
4. Use Analytics Engine for high-cardinality data instead of logs

```jsonc
{
  "observability": {
    "logs": {
      "enabled": true,
      "head_sampling_rate": 0.01,  // 1% sampling
      "invocation_logs": false,     // Disable auto-logs
      "persist": false              // Don't store in CF dashboard
    }
  }
}
```

---

## Limits & Pricing

### Workers Logs (as of April 21, 2025)

| Plan | Events Written | Retention |
|------|----------------|-----------|
| Free | 200K/day | 3 days |
| Paid | 20M/month included, +$0.60/M | 7 days |

### Workers Traces (as of January 15, 2026)

| Plan | Spans | Retention |
|------|-------|-----------|
| Free | 200K/day | 3 days |
| Paid | 10M/month included, +$0.60/M | 7 days |

**Note**: Each span or log event counts as 1 observability event.

### Analytics Engine

| Plan | Writes | Queries | Retention |
|------|--------|---------|-----------|
| Free | 10M/month | 1M rows scanned/day | 31 days |
| Paid | 10M/month included, +$0.25/M | Unlimited | 93 days |

### OpenTelemetry Exports (as of January 15, 2026)

| Plan | Events | Pricing |
|------|--------|---------|
| Free | Not available | - |
| Paid | 10M/month included | $0.05/M additional |

### Logpush

| Plan | Availability |
|------|--------------|
| Workers Paid | Available for Workers Trace Events |
| Enterprise | Available for all datasets |

---

## Best Practices

1. **Use structured JSON logging** for better querying and filtering
2. **Sample high-traffic Workers** to control costs (0.01-0.1 for production)
3. **Don't await `writeDataPoint()`** - it's non-blocking and handled by runtime
4. **Always account for `_sample_interval`** in Analytics Engine queries
5. **Use indexes wisely** - they determine sampling keys (customer ID, user ID, etc.)
6. **Set `persist: false`** when exporting to external destinations to avoid double-storage
7. **Enable traces in development** with 100% sampling, reduce in production
8. **Create destinations in dashboard** before configuring wrangler for OTLP exports
9. **Use consistent field order** in `writeDataPoint()` for Analytics Engine
10. **Monitor destination status** in dashboard to catch delivery failures

---

## Integration Examples

### Honeycomb

```jsonc
{
  "observability": {
    "traces": {
      "enabled": true,
      "destinations": ["honeycomb"],
      "head_sampling_rate": 0.1
    }
  }
}
```

**Dashboard Destination Config**:
- Endpoint: `https://api.honeycomb.io/v1/traces`
- Header: `x-honeycomb-team: <API_KEY>`
- Header: `x-honeycomb-dataset: cloudflare-workers`

### Grafana Cloud

```jsonc
{
  "observability": {
    "traces": {
      "enabled": true,
      "destinations": ["grafana-traces"]
    },
    "logs": {
      "enabled": true,
      "destinations": ["grafana-logs"]
    }
  }
}
```

**Dashboard Destination Config** (Traces):
- Endpoint: `https://otlp-gateway-{region}.grafana.net/otlp/v1/traces`
- Header: `Authorization: Basic <base64(username:token)>`

### Custom Analytics with Grafana

```typescript
// Worker with Analytics Engine
env.ANALYTICS.writeDataPoint({
  'blobs': ['endpoint', 'status', 'method'],
  'doubles': [response_time_ms, 1],
  'indexes': [customer_id]
});
```

**Grafana Query**:
```sql
SELECT
  $__timeGroup(timestamp, 1m) as time,
  blob1 as endpoint,
  SUM(_sample_interval * double1) / SUM(_sample_interval) as avg_response_ms
FROM $dataset
WHERE $__timeFilter(timestamp)
GROUP BY time, endpoint
ORDER BY time
```

---

## Resources

- [Workers Observability Docs](https://developers.cloudflare.com/workers/observability/)
- [Analytics Engine Docs](https://developers.cloudflare.com/analytics/analytics-engine/)
- [Logpush Docs](https://developers.cloudflare.com/logs/logpush/)
- [GraphQL Analytics API](https://developers.cloudflare.com/analytics/graphql-api/)
- [OpenTelemetry Exports](https://developers.cloudflare.com/workers/observability/exporting-opentelemetry-data/)
- [Wrangler Configuration](https://developers.cloudflare.com/workers/wrangler/configuration/)

---

## Checklist for Implementing Observability

- [ ] Enable Workers Logs in wrangler config
- [ ] Configure appropriate sampling rate for traffic volume
- [ ] Use structured JSON logging throughout code
- [ ] Enable traces for performance monitoring
- [ ] Set up Analytics Engine binding for custom metrics
- [ ] Create OTLP destinations for external exports (if needed)
- [ ] Configure Logpush for long-term storage (if needed)
- [ ] Test log output in dashboard and via `wrangler tail`
- [ ] Set up alerts/monitors in external platform
- [ ] Document custom metric schema (blobs/doubles/indexes)
- [ ] Review costs and adjust sampling if needed
- [ ] Add error tracking with context
- [ ] Implement request correlation IDs
- [ ] Configure separate sampling rates per environment (staging/prod)
- [ ] Set up Grafana/visualization dashboards (if using Analytics Engine)
