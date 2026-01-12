# Cloudflare Cron Triggers Skill

Expert guidance for implementing, configuring, and managing Cloudflare Workers Cron Triggers.

## Overview

Cloudflare Cron Triggers enable Workers to execute on a schedule using cron expressions. They run on Cloudflare's global network during underutilized periods, ideal for periodic jobs like maintenance, API polling, data synchronization, and batch processing.

**Key characteristics:**
- Execute on UTC time only
- Run on underutilized machines for optimal capacity usage
- Support 5-field cron expressions with Quartz scheduler extensions
- Can trigger Workflows for multi-step, long-running tasks
- Changes take up to 15 minutes to propagate globally

## Handler Implementation

### Basic Scheduled Handler

The `scheduled()` handler is invoked when a cron trigger fires:

```typescript
interface Env {
  // Define bindings here (KV, R2, D1, etc.)
}

export default {
  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ): Promise<void> {
    // Your scheduled logic here
    console.log("Cron executed at:", new Date(controller.scheduledTime));
  },
};
```

```javascript
export default {
  async scheduled(controller, env, ctx) {
    console.log("Cron executed at:", new Date(controller.scheduledTime));
  },
};
```

```python
from workers import WorkerEntrypoint

class Default(WorkerEntrypoint):
    async def scheduled(self, controller, env, ctx):
        print(f"Cron executed at: {controller.scheduledTime}")
```

### Handler Parameters

**`controller: ScheduledController`**
- `controller.scheduledTime: number` - Unix timestamp (ms) when scheduled to run. Parse with `new Date(controller.scheduledTime)`
- `controller.cron: string` - Cron expression that triggered the event (e.g., `"*/5 * * * *"`)
- `controller.type: string` - Always returns `"scheduled"`

**`env: Env`**
- Contains all bindings (KV namespaces, R2 buckets, D1 databases, secrets, service bindings)

**`ctx: ExecutionContext`**
- `ctx.waitUntil(promise)` - Extends execution beyond handler return for async tasks (logging, cleanup, external API calls). First failure is recorded in Cron Events table.

### Multiple Cron Triggers Pattern

Handle different schedules with switch statement on `controller.cron`:

```typescript
import { Hono } from "hono";

interface Env {
  MY_KV: KVNamespace;
}

const app = new Hono<{ Bindings: Env }>();

app.get("/", (c) => c.text("API Running"));

export default {
  fetch: app.fetch,

  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ) {
    switch (controller.cron) {
      case "*/3 * * * *":
        // Every 3 minutes - fast updates
        ctx.waitUntil(updateRecentData(env));
        break;
      
      case "0 * * * *":
        // Hourly - moderate updates
        ctx.waitUntil(processHourlyAggregation(env));
        break;
      
      case "0 2 * * *":
        // Daily at 2am UTC - heavy jobs
        ctx.waitUntil(performDailyMaintenance(env));
        break;
      
      default:
        console.warn(`Unhandled cron: ${controller.cron}`);
    }
  },
};

async function updateRecentData(env: Env) {
  // Fast, frequent task
}

async function processHourlyAggregation(env: Env) {
  // Hourly processing
}

async function performDailyMaintenance(env: Env) {
  // Heavy daily operations
}
```

### Long-Running Tasks with ctx.waitUntil

Use `ctx.waitUntil()` for async operations that shouldn't block handler return:

```typescript
export default {
  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ) {
    // Critical path - runs immediately
    const data = await fetchCriticalData();
    
    // Non-blocking async tasks
    ctx.waitUntil(
      Promise.all([
        logToAnalytics(data),
        cleanupOldRecords(env.DB),
        notifyWebhook(env.WEBHOOK_URL, data),
      ])
    );
    
    // Handler can return while waitUntil tasks complete
    console.log("Handler complete, background tasks running");
  },
};
```

## Wrangler Configuration

### JSONC Format (wrangler.jsonc)

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-cron-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-01-01",
  
  "triggers": {
    "crons": [
      "*/5 * * * *",     // Every 5 minutes
      "0 */2 * * *",     // Every 2 hours
      "0 9 * * MON-FRI", // Weekdays at 9am UTC
      "0 2 1 * *"        // Monthly on 1st at 2am UTC
    ]
  }
}
```

### TOML Format (wrangler.toml)

```toml
name = "my-cron-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[triggers]
crons = [
  "*/5 * * * *",     # Every 5 minutes
  "0 */2 * * *",     # Every 2 hours
  "0 9 * * MON-FRI", # Weekdays at 9am UTC
  "0 2 1 * *"        # Monthly on 1st at 2am UTC
]
```

### Environment-Specific Triggers

Configure different schedules per environment:

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-cron-worker",
  
  "triggers": {
    "crons": ["0 */6 * * *"]  // Production: every 6 hours
  },
  
  "env": {
    "staging": {
      "triggers": {
        "crons": ["*/15 * * * *"]  // Staging: every 15 min
      }
    },
    "dev": {
      "triggers": {
        "crons": ["*/5 * * * *"]  // Dev: every 5 min
      }
    }
  }
}
```

```toml
name = "my-cron-worker"

[triggers]
crons = ["0 */6 * * *"]  # Production: every 6 hours

[env.staging.triggers]
crons = ["*/15 * * * *"]  # Staging: every 15 min

[env.dev.triggers]
crons = ["*/5 * * * *"]  # Dev: every 5 min
```

### Removing Triggers

**Remove all triggers:**
```jsonc
{
  "triggers": {
    "crons": []  // Empty array removes all
  }
}
```

**Leave triggers unchanged:**
```jsonc
{
  // Omit "triggers" entirely to preserve existing crons
}
```

## Cron Expression Syntax

### Field Structure

5-field format (minute, hour, day-of-month, month, day-of-week):

```
 ┌─────────── minute (0-59)
 │ ┌───────── hour (0-23)
 │ │ ┌─────── day of month (1-31)
 │ │ │ ┌───── month (1-12, JAN-DEC)
 │ │ │ │ ┌─── day of week (1-7, SUN-SAT, 1=Sunday)
 │ │ │ │ │
 * * * * *
```

### Special Characters

| Char | Meaning | Example |
|------|---------|---------|
| `*` | Any value | `* * * * *` = every minute |
| `,` | List | `1,15,30 * * * *` = min 1, 15, 30 of every hour |
| `-` | Range | `0 9-17 * * MON-FRI` = 9am-5pm on weekdays |
| `/` | Step | `*/10 * * * *` = every 10 minutes |
| `L` | Last | `0 0 L * *` = last day of month<br>`0 18 * * FRI-L` = last Friday |
| `W` | Weekday | `0 9 15W * *` = 9am on weekday nearest 15th |
| `#` | Nth weekday | `0 10 * * MON#1` = 10am on 1st Monday |

### Common Patterns

```bash
# Every N minutes
*/5 * * * *      # Every 5 minutes
*/15 * * * *     # Every 15 minutes
*/30 * * * *     # Every 30 minutes

# Hourly
0 * * * *        # Top of every hour
30 * * * *       # 30 minutes past every hour

# Specific times
0 9 * * *        # 9am UTC daily
0 2 * * *        # 2am UTC daily (off-peak)
0 12 * * MON-FRI # Noon UTC on weekdays
0 6,18 * * *     # 6am and 6pm UTC daily

# Weekly
0 8 * * MON      # 8am UTC every Monday
0 23 * * SUN     # 11pm UTC every Sunday

# Monthly
0 0 1 * *        # Midnight UTC on 1st of month
0 12 15 * *      # Noon UTC on 15th of month
0 9 L * *        # 9am UTC on last day of month

# Complex patterns
0 */4 * * *      # Every 4 hours
*/10 9-17 * * MON-FRI  # Every 10 min, 9am-5pm, weekdays
0 2 * * 6        # 2am UTC every Saturday
59 23 LW * *     # 11:59pm UTC last weekday of month
0 10 * * MON#2   # 10am UTC 2nd Monday of month
```

### Cron Validation Tips

- **Always UTC** - No timezone support, plan for UTC
- **Day-of-month + day-of-week** - If both specified, either match triggers (OR logic)
- **Minimum interval** - Respect rate limits, avoid sub-minute for expensive ops
- **Propagation delay** - Changes take up to 15 minutes to apply globally

## Testing & Development

### Local Testing with Wrangler

Test scheduled handlers locally without waiting for real cron:

```bash
# Start dev server with scheduled handler support
npx wrangler dev

# In another terminal, trigger scheduled event
curl "http://localhost:8787/__scheduled"

# Test specific cron pattern
curl "http://localhost:8787/__scheduled?cron=*/5+*+*+*+*"

# Override scheduled time (Unix timestamp in seconds)
curl "http://localhost:8787/__scheduled?cron=0+2+*+*+*&time=1745856238"
```

**Python Workers:** Use `/cdn-cgi/handler/scheduled` endpoint instead:
```bash
curl "http://localhost:8787/cdn-cgi/handler/scheduled?cron=*/5+*+*+*+*"
```

### Testing with Vite Plugin

If using `@cloudflare/vite-plugin-cloudflare`:

```bash
npm run dev

# Trigger in another terminal
curl "http://localhost:5173/cdn-cgi/handler/scheduled?cron=*/10+*+*+*+*"
```

### Test Response Format

```json
{
  "outcome": "ok",
  "noRetry": false
}
```

### Integration Testing Pattern

```typescript
// test/scheduled.test.ts
import { describe, it, expect } from "vitest";
import worker from "../src/index";

describe("Scheduled Handler", () => {
  it("processes scheduled event", async () => {
    const env = getMiniflareBindings();
    const ctx = {
      waitUntil: (promise: Promise<any>) => promise,
      passThroughOnException: () => {},
    };
    
    const controller = {
      scheduledTime: Date.now(),
      cron: "*/5 * * * *",
      type: "scheduled" as const,
    };
    
    await worker.scheduled(controller, env, ctx);
    
    // Assert side effects
    const result = await env.MY_KV.get("last_run");
    expect(result).toBeDefined();
  });
});
```

## Observability & Monitoring

### Viewing Past Events

**Via Dashboard:**
1. Navigate to **Workers & Pages** → Select Worker
2. **Settings** → **Triggers** → **Cron Events** → **View events**
3. Shows last 100 invocations with outcome status

**Note:** New Workers may take up to 30 minutes to appear in Past Cron Events.

### Workers Logs

Use Workers Logs for longer retention and filtering:

```bash
# Tail logs in real-time
npx wrangler tail

# Filter for scheduled events only
npx wrangler tail --format json | jq 'select(.event.cron != null)'
```

### GraphQL Analytics API

Query historical cron execution data:

```graphql
query CronMetrics($accountTag: string!, $workerName: string!) {
  viewer {
    accounts(filter: { accountTag: $accountTag }) {
      workersInvocationsAdaptive(
        filter: {
          scriptName: $workerName
          eventType: "scheduled"
        }
        limit: 100
      ) {
        dimensions {
          datetime
          eventType
          status
        }
        sum {
          requests
          errors
        }
      }
    }
  }
}
```

### Logging Best Practices

```typescript
export default {
  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ) {
    const startTime = Date.now();
    const cronTime = new Date(controller.scheduledTime).toISOString();
    
    console.log(`[START] Cron ${controller.cron} triggered at ${cronTime}`);
    
    try {
      const result = await performTask(env);
      
      const duration = Date.now() - startTime;
      console.log(`[SUCCESS] Task completed in ${duration}ms`, {
        cron: controller.cron,
        recordsProcessed: result.count,
      });
      
      // Log to external service
      ctx.waitUntil(
        logToExternal(env.ANALYTICS_ENDPOINT, {
          event: "cron_success",
          duration,
          cron: controller.cron,
        })
      );
      
    } catch (error) {
      const duration = Date.now() - startTime;
      console.error(`[ERROR] Task failed after ${duration}ms`, {
        cron: controller.cron,
        error: error.message,
        stack: error.stack,
      });
      
      // Alert on failures
      ctx.waitUntil(
        notifyFailure(env.ALERT_WEBHOOK, {
          cron: controller.cron,
          error: error.message,
        })
      );
      
      throw error; // Mark as failed in Cron Events
    }
  },
};
```

## Common Use Cases & Patterns

### 1. API Data Synchronization

Poll external APIs and cache results:

```typescript
interface Env {
  MY_KV: KVNamespace;
  API_KEY: string;
}

export default {
  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ) {
    const response = await fetch("https://api.example.com/data", {
      headers: { "Authorization": `Bearer ${env.API_KEY}` },
    });
    
    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }
    
    const data = await response.json();
    
    // Store in KV for fetch handler access
    ctx.waitUntil(
      env.MY_KV.put("cached_data", JSON.stringify(data), {
        expirationTtl: 3600, // 1 hour
      })
    );
    
    console.log(`Cached ${data.length} items`);
  },
};
```

### 2. Database Cleanup

Periodic maintenance tasks:

```typescript
interface Env {
  DB: D1Database;
}

export default {
  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ) {
    // Daily at 2am UTC - delete expired records
    const result = await env.DB.prepare(`
      DELETE FROM sessions 
      WHERE expires_at < datetime('now')
    `).run();
    
    console.log(`Deleted ${result.meta.changes} expired sessions`);
    
    // Vacuum to reclaim space
    ctx.waitUntil(
      env.DB.prepare("VACUUM").run()
    );
  },
};
```

### 3. Report Generation

Generate and distribute periodic reports:

```typescript
interface Env {
  DB: D1Database;
  REPORTS_BUCKET: R2Bucket;
  SEND_EMAIL: Fetcher; // Email Worker binding
}

export default {
  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ) {
    // Weekly on Monday at 9am UTC
    const startOfWeek = new Date();
    startOfWeek.setDate(startOfWeek.getDate() - 7);
    
    const { results } = await env.DB.prepare(`
      SELECT date, revenue, orders
      FROM daily_stats
      WHERE date >= ?
      ORDER BY date
    `).bind(startOfWeek.toISOString()).all();
    
    // Generate report
    const report = generateReport(results);
    const reportKey = `reports/weekly-${Date.now()}.json`;
    
    // Store in R2
    await env.REPORTS_BUCKET.put(reportKey, JSON.stringify(report));
    
    // Send email notification
    ctx.waitUntil(
      env.SEND_EMAIL.fetch("https://example.com/send", {
        method: "POST",
        body: JSON.stringify({
          to: "team@example.com",
          subject: "Weekly Report Available",
          reportUrl: `https://reports.example.com/${reportKey}`,
        }),
      })
    );
  },
};

function generateReport(data: any[]) {
  // Report generation logic
  return {
    period: "weekly",
    totalRevenue: data.reduce((sum, d) => sum + d.revenue, 0),
    totalOrders: data.reduce((sum, d) => sum + d.orders, 0),
    dailyBreakdown: data,
  };
}
```

### 4. Health Checks

Monitor external services:

```typescript
interface Env {
  STATUS_KV: KVNamespace;
  ALERT_WEBHOOK: string;
}

export default {
  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ) {
    const services = [
      { name: "API", url: "https://api.example.com/health" },
      { name: "CDN", url: "https://cdn.example.com/health" },
      { name: "DB", url: "https://db.example.com/health" },
    ];
    
    const checks = await Promise.all(
      services.map(async (service) => {
        const start = Date.now();
        try {
          const response = await fetch(service.url, { 
            signal: AbortSignal.timeout(5000) 
          });
          return {
            name: service.name,
            status: response.ok ? "up" : "down",
            responseTime: Date.now() - start,
            statusCode: response.status,
          };
        } catch (error) {
          return {
            name: service.name,
            status: "down",
            responseTime: Date.now() - start,
            error: error.message,
          };
        }
      })
    );
    
    // Store status
    ctx.waitUntil(
      env.STATUS_KV.put("health_status", JSON.stringify(checks))
    );
    
    // Alert on failures
    const failures = checks.filter(c => c.status === "down");
    if (failures.length > 0) {
      ctx.waitUntil(
        fetch(env.ALERT_WEBHOOK, {
          method: "POST",
          body: JSON.stringify({
            text: `${failures.length} service(s) down: ${failures.map(f => f.name).join(", ")}`,
            checks: failures,
          }),
        })
      );
    }
  },
};
```

### 5. Rate-Limited API Processing

Process queued items respecting rate limits:

```typescript
interface Env {
  QUEUE_KV: KVNamespace;
  API_KEY: string;
}

export default {
  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ) {
    // Every 5 minutes, process up to 100 items
    const queueData = await env.QUEUE_KV.get("pending_items", "json");
    if (!queueData || queueData.length === 0) {
      console.log("Queue empty");
      return;
    }
    
    const batchSize = 100;
    const batch = queueData.slice(0, batchSize);
    const remaining = queueData.slice(batchSize);
    
    const results = await Promise.allSettled(
      batch.map(item => processItem(item, env.API_KEY))
    );
    
    const succeeded = results.filter(r => r.status === "fulfilled").length;
    const failed = results.filter(r => r.status === "rejected").length;
    
    console.log(`Processed ${succeeded} items, ${failed} failed`);
    
    // Update queue
    ctx.waitUntil(
      env.QUEUE_KV.put("pending_items", JSON.stringify(remaining))
    );
  },
};

async function processItem(item: any, apiKey: string) {
  const response = await fetch("https://api.example.com/process", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(item),
  });
  
  if (!response.ok) {
    throw new Error(`Failed to process item ${item.id}`);
  }
  
  return response.json();
}
```

### 6. Workflow Integration

Trigger long-running Workflows on schedule:

```typescript
import { WorkflowEntrypoint } from "cloudflare:workers";

interface Env {
  MY_WORKFLOW: Workflow;
}

export class DataProcessingWorkflow extends WorkflowEntrypoint {
  async run(event: any, step: any) {
    // Multi-step workflow with automatic retries
    const data = await step.do("fetch-data", async () => {
      return await fetchLargeDataset();
    });
    
    const processed = await step.do("process-data", async () => {
      return await processDataset(data);
    });
    
    await step.do("store-results", async () => {
      return await storeResults(processed);
    });
  }
}

export default {
  async scheduled(
    controller: ScheduledController,
    env: Env,
    ctx: ExecutionContext,
  ) {
    // Trigger workflow instead of running directly
    const instance = await env.MY_WORKFLOW.create({
      params: {
        scheduledTime: controller.scheduledTime,
        cron: controller.cron,
      },
    });
    
    console.log(`Started workflow instance: ${instance.id}`);
  },
};
```

## Limits & Constraints

### Platform Limits

- **Max cron triggers per Worker:** 3 on Free plan, unlimited on Paid plans
- **CPU time:** 10ms on Free, 50ms on Paid (per invocation)
- **Minimum interval:** No hard limit, but avoid sub-minute for expensive operations
- **Propagation delay:** Changes take up to 15 minutes to apply globally
- **Timezone:** UTC only, no timezone configuration
- **Execution guarantee:** At-least-once delivery (may execute multiple times rarely)

### Best Practices for Limits

```typescript
// 1. Use ctx.waitUntil for non-critical work to maximize CPU time
export default {
  async scheduled(controller, env, ctx) {
    // Critical work within CPU limit
    const data = await fetchQuickData();
    
    // Heavy work in waitUntil
    ctx.waitUntil(processHeavyTask(data, env));
  },
};

// 2. Batch operations to reduce invocation count
export default {
  async scheduled(controller, env, ctx) {
    // Process in batches rather than per-item
    const items = await env.KV.list({ prefix: "pending_" });
    
    await Promise.all(
      items.keys.map(async (key) => {
        const item = await env.KV.get(key.name, "json");
        return processItem(item);
      })
    );
  },
};

// 3. Use Workflows for long-running tasks exceeding CPU limits
export default {
  async scheduled(controller, env, ctx) {
    // Delegate to Workflow for extended processing
    await env.WORKFLOW.create({ /* params */ });
  },
};
```

## Green Compute

Enable renewable energy-powered execution:

**Account-level configuration (Dashboard):**
1. **Workers & Pages** → **Account details** → **Compute Setting**
2. Select **Change** → **Green Compute** → **Confirm**

When enabled, cron triggers execute only in data centers powered by renewable energy sources (on-site generation, PPAs, or RECs).

**Considerations:**
- May reduce available execution locations
- Slightly increased latency possible (fewer POPs)
- Ideal for non-time-critical periodic jobs

## API Management

### Cloudflare API Endpoints

**Get Cron Triggers for Worker:**
```bash
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/workers/scripts/{script_name}/schedules" \
  -H "Authorization: Bearer {api_token}"
```

**Update Cron Triggers:**
```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/{account_id}/workers/scripts/{script_name}/schedules" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "crons": ["*/5 * * * *", "0 2 * * *"]
  }'
```

**Delete All Cron Triggers:**
```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/{account_id}/workers/scripts/{script_name}/schedules" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{"crons": []}'
```

### Wrangler CLI Commands

```bash
# Deploy with cron triggers from config
npx wrangler deploy

# Deploy to specific environment
npx wrangler deploy --env production

# View deployed worker configuration
npx wrangler deployments list

# Tail logs for scheduled events
npx wrangler tail --format pretty

# Local development with scheduled handler
npx wrangler dev
```

## Troubleshooting

### Cron Not Executing

**Check:**
1. Handler exists: `scheduled()` function exported
2. Configuration deployed: Recent `wrangler deploy` after config changes
3. Propagation time: Wait up to 15 minutes after deployment
4. Syntax validity: Use cron validator (crontab.guru)
5. Account limits: Verify plan allows triggers
6. Worker status: Check dashboard for errors

**Debug:**
```typescript
export default {
  async scheduled(controller, env, ctx) {
    // Log execution proof
    console.log("EXECUTED", {
      time: new Date().toISOString(),
      scheduledTime: new Date(controller.scheduledTime).toISOString(),
      cron: controller.cron,
    });
    
    // Store execution timestamp for verification
    ctx.waitUntil(
      env.KV.put("last_execution", Date.now().toString())
    );
  },
};
```

### Execution Failures

**Common causes:**
- Exceeding CPU time limit
- Unhandled exceptions
- Network timeouts to external services
- Binding misconfiguration

**Mitigation:**
```typescript
export default {
  async scheduled(controller, env, ctx) {
    try {
      // Set timeouts for external calls
      const controller = new AbortController();
      const timeout = setTimeout(() => controller.abort(), 5000);
      
      const response = await fetch("https://api.example.com/data", {
        signal: controller.signal,
      });
      clearTimeout(timeout);
      
      // Validate response
      if (!response.ok) {
        throw new Error(`API returned ${response.status}`);
      }
      
      const data = await response.json();
      await processData(data, env);
      
    } catch (error) {
      // Log detailed error info
      console.error("Scheduled task failed", {
        error: error.message,
        stack: error.stack,
        cron: controller.cron,
        scheduledTime: controller.scheduledTime,
      });
      
      // Optional: Don't fail entire execution for non-critical errors
      // If you want to mark as success despite errors, don't re-throw
      // throw error; // Re-throw to mark as failed in Cron Events
    }
  },
};
```

### Testing Locally Issues

**wrangler dev not exposing `/__scheduled`:**
- Ensure `wrangler dev` runs without errors
- Check for `scheduled()` handler in code
- Verify Wrangler version: `npx wrangler --version` (use latest)

**Cron query parameter format:**
```bash
# Correct - URL encode spaces as +
curl "http://localhost:8787/__scheduled?cron=*/5+*+*+*+*"

# Or use %20
curl "http://localhost:8787/__scheduled?cron=*%2F5%20*%20*%20*%20*"

# Test without parameters
curl "http://localhost:8787/__scheduled"
```

## Security Considerations

### Preventing Unauthorized Execution

Local dev endpoints are public by default. In production, scheduled events only trigger from Cloudflare infrastructure.

**For production-grade local testing:**
```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext) {
    const url = new URL(request.url);
    
    // Protect /__scheduled endpoint in production
    if (url.pathname === "/__scheduled") {
      // Only allow in development
      if (env.ENVIRONMENT === "production") {
        return new Response("Not found", { status: 404 });
      }
      
      // Simulate scheduled event
      await this.scheduled(
        {
          scheduledTime: Date.now(),
          cron: url.searchParams.get("cron") || "* * * * *",
          type: "scheduled",
        },
        env,
        ctx
      );
      
      return new Response("OK");
    }
    
    // Normal request handling
    return new Response("Hello");
  },
  
  async scheduled(controller, env, ctx) {
    // Your scheduled logic
  },
};
```

### Secrets Management

Never hardcode secrets in cron handlers:

```typescript
// ❌ Bad - hardcoded secrets
export default {
  async scheduled(controller, env, ctx) {
    const response = await fetch("https://api.example.com/data", {
      headers: { "Authorization": "Bearer sk_live_abc123..." },
    });
  },
};

// ✅ Good - use environment bindings
interface Env {
  API_KEY: string;
}

export default {
  async scheduled(controller: ScheduledController, env: Env, ctx: ExecutionContext) {
    const response = await fetch("https://api.example.com/data", {
      headers: { "Authorization": `Bearer ${env.API_KEY}` },
    });
  },
};
```

Add secrets via Wrangler:
```bash
npx wrangler secret put API_KEY
# Enter secret value when prompted
```

## Additional Resources

- [Cron Triggers Official Docs](https://developers.cloudflare.com/workers/configuration/cron-triggers/)
- [Scheduled Handler API Reference](https://developers.cloudflare.com/workers/runtime-apis/handlers/scheduled/)
- [Cloudflare Workflows](https://developers.cloudflare.com/workflows/) - For long-running tasks
- [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Crontab Guru](https://crontab.guru/) - Cron expression validator
- [Wrangler Configuration](https://developers.cloudflare.com/workers/wrangler/configuration/)
