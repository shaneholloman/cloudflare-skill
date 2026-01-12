# Cloudflare Smart Placement Skill

Expert knowledge for Cloudflare Workers Smart Placement - automatic workload placement optimization to minimize latency by running Workers closer to backend infrastructure rather than end users.

## Core Concept

Smart Placement automatically analyzes Worker request duration across Cloudflare's global network and intelligently routes requests to optimal data center locations. Instead of defaulting to the location closest to the end user, Smart Placement can forward requests to locations closer to backend infrastructure when this reduces overall request duration.

### When to Use Smart Placement

**Enable Smart Placement when:**
- Worker makes multiple round trips to backend services/databases
- Backend infrastructure is geographically concentrated
- Request duration dominated by backend latency rather than network latency from user
- Running backend logic in Workers (APIs, data aggregation, server-side rendering with DB calls)

**Do NOT enable for:**
- Workers serving only static content or cached responses
- Workers without significant backend communication
- Pure edge logic (auth checks, redirects, simple transformations)
- Workers without fetch event handlers

### Key Architecture Pattern

**Recommended:** Split full-stack applications into separate Workers:
```
User → Frontend Worker (at edge, close to user)
         ↓ Service Binding
       Backend Worker (Smart Placement enabled, close to DB/API)
         ↓
       Database/Backend Service
```

This maintains fast, reactive frontends while optimizing backend latency.

## Configuration

### Via wrangler.toml

```toml
# Basic Smart Placement
[placement]
mode = "smart"

# With placement hint (specify preferred region)
[placement]
mode = "smart"
hint = "wnam"  # West North America
```

### Via wrangler.json/wrangler.jsonc

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "placement": {
    "mode": "smart",
    "hint": "wnam"  // Optional: preferred region hint
  }
}
```

### Placement Mode Values

- `"smart"` - Enable Smart Placement optimization
- Not specified - Default behavior (run at edge closest to user)

### Placement Hints

Optional region hints guide Smart Placement decisions:
- `"wnam"` - West North America
- `"enam"` - East North America  
- `"weur"` - Western Europe
- `"eeur"` - Eastern Europe
- `"apac"` - Asia Pacific
- etc.

**Note:** Hints are suggestions, not guarantees. Smart Placement makes final decision based on performance data.

## Requirements & Limitations

### Requirements
- **Wrangler version:** 2.20.0 or later
- **Analysis time:** Up to 15 minutes after enabling
- **Traffic requirements:** Consistent traffic from multiple global locations
- **Workers plan:** Available on all Workers plans (Free, Paid, Enterprise)

### What Smart Placement Affects
- ✅ **Affects:** `fetch` event handlers only
- ❌ **Does NOT affect:** 
  - RPC method invocations
  - Named entrypoints
  - Static asset serving (assets served from Smart Placement location if Worker retrieves them)
  - Workers without fetch handlers

### Baseline Traffic
Smart Placement automatically routes 1% of requests WITHOUT Smart Placement optimization as a baseline for performance comparison.

## Observability

### Placement Status API

Query Worker placement status via Cloudflare API:

```bash
curl -X GET "https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/workers/services/{WORKER_NAME}" \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json"
```

Response includes `placement_status` field:

```typescript
type PlacementStatus = 
  | undefined  // Not yet analyzed
  | 'SUCCESS'  // Successfully optimized
  | 'INSUFFICIENT_INVOCATIONS'  // Not enough traffic
  | 'UNSUPPORTED_APPLICATION';  // Made Worker slower (reverted)
```

### Status Meanings

**`undefined` (not present)**
- Worker not yet analyzed
- Always runs at default edge location closest to user

**`SUCCESS`**
- Analysis complete, Smart Placement active
- Worker runs in optimal location (may be edge or remote)

**`INSUFFICIENT_INVOCATIONS`**
- Not enough requests to make placement decision
- Requires consistent multi-region traffic
- Always runs at default edge location

**`UNSUPPORTED_APPLICATION`** (rare, <1% of Workers)
- Smart Placement made Worker slower
- Placement decision reverted
- Always runs at edge location
- Won't be re-analyzed until redeployed

### cf-placement Header (Beta)

Smart Placement adds response header indicating routing decision:

```typescript
// Remote placement (Smart Placement routed request)
"cf-placement: remote-LHR"  // Routed to London

// Local placement (default edge routing)  
"cf-placement: local-EWR"   // Stayed at Newark edge
```

Format: `{placement-type}-{IATA-code}`
- `remote-*` = Smart Placement routed to remote location
- `local-*` = Stayed at default edge location
- IATA code = nearest airport to data center

**Warning:** Beta feature, may be removed before GA.

### Request Duration Metrics

Available in Cloudflare dashboard when Smart Placement enabled:

**Workers & Pages → [Your Worker] → Metrics → Request Duration**

Shows histogram comparing:
- Request duration WITH Smart Placement (99% of traffic)
- Request duration WITHOUT Smart Placement (1% baseline)

**Request Duration vs Execution Duration:**
- **Request duration:** Total time from request arrival to response delivery (includes network latency)
- **Execution duration:** Time Worker code actively executing (excludes network waits)

Use request duration to measure Smart Placement impact.

## Code Patterns

### Backend Worker with Database Access

```typescript
// backend-worker.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Smart Placement will run this close to database
    const db = env.DATABASE;
    
    // Multiple round trips benefit from Smart Placement
    const user = await db.prepare('SELECT * FROM users WHERE id = ?')
      .bind(userId)
      .first();
    
    const orders = await db.prepare('SELECT * FROM orders WHERE user_id = ?')
      .bind(userId)
      .all();
    
    const preferences = await db.prepare('SELECT * FROM preferences WHERE user_id = ?')
      .bind(userId)
      .first();
    
    return Response.json({ user, orders, preferences });
  }
};
```

```toml
# wrangler.toml
name = "backend-api"
main = "backend-worker.ts"

[placement]
mode = "smart"

[[d1_databases]]
binding = "DATABASE"
database_id = "xxx"
```

### Full-Stack Split: Frontend + Backend Workers

**Frontend Worker (no Smart Placement):**
```typescript
// frontend-worker.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Runs at edge close to user for fast response
    const url = new URL(request.url);
    
    if (url.pathname.startsWith('/api/')) {
      // Forward API requests to backend worker via Service Binding
      return env.BACKEND.fetch(request);
    }
    
    // Serve static assets at edge
    return env.ASSETS.fetch(request);
  }
};
```

```toml
# frontend-worker wrangler.toml
name = "frontend"
main = "frontend-worker.ts"

# No [placement] - runs at edge

[[services]]
binding = "BACKEND"
service = "backend-api"
```

**Backend Worker (Smart Placement enabled):**
```typescript
// backend-worker.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Smart Placement runs this close to database
    const url = new URL(request.url);
    
    if (url.pathname === '/api/users') {
      const users = await env.DATABASE
        .prepare('SELECT * FROM users')
        .all();
      return Response.json(users);
    }
    
    // More API endpoints with DB access...
    return new Response('Not found', { status: 404 });
  }
};
```

```toml
# backend-api wrangler.toml
name = "backend-api"
main = "backend-worker.ts"

[placement]
mode = "smart"

[[d1_databases]]
binding = "DATABASE"
database_id = "xxx"
```

### Worker with External API Integration

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Smart Placement can run this closer to external API
    const apiUrl = 'https://api.partner.com';
    
    // Multiple API calls benefit from reduced latency
    const [profile, transactions, settings] = await Promise.all([
      fetch(`${apiUrl}/profile`, {
        headers: { 'Authorization': `Bearer ${env.API_KEY}` }
      }),
      fetch(`${apiUrl}/transactions`, {
        headers: { 'Authorization': `Bearer ${env.API_KEY}` }
      }),
      fetch(`${apiUrl}/settings`, {
        headers: { 'Authorization': `Bearer ${env.API_KEY}` }
      })
    ]);
    
    return Response.json({
      profile: await profile.json(),
      transactions: await transactions.json(),
      settings: await settings.json()
    });
  }
};
```

```toml
[placement]
mode = "smart"
hint = "enam"  # Hint if you know API is in East North America
```

### Detecting Smart Placement in Code

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Access cf-placement header (beta)
    const placementHeader = request.headers.get('cf-placement');
    
    if (placementHeader?.startsWith('remote-')) {
      // Request was routed via Smart Placement
      const location = placementHeader.split('-')[1];
      console.log(`Smart Placement routed to ${location}`);
    } else if (placementHeader?.startsWith('local-')) {
      // Request stayed at edge
      const location = placementHeader.split('-')[1];
      console.log(`Running at edge location ${location}`);
    }
    
    // Your Worker logic...
    return new Response('OK');
  }
};
```

## Wrangler Commands

### Deploy with Smart Placement

```bash
# Deploy Worker with Smart Placement enabled
wrangler deploy

# Deploy and tail logs
wrangler deploy && wrangler tail

# Publish to specific environment
wrangler deploy --env production
```

### Check Placement Status

```bash
# Get Worker metadata including placement status
curl -X GET "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/workers/services/${WORKER_NAME}" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" | jq '.result.placement_status'
```

### Monitor Request Duration

```bash
# Tail Worker logs to see real-time behavior
wrangler tail your-worker-name

# Tail with filters
wrangler tail your-worker-name --status error
wrangler tail your-worker-name --header cf-placement
```

### Local Development

```bash
# Smart Placement does NOT affect local development
wrangler dev

# Local dev always runs in single location
# Test Smart Placement by deploying to staging
wrangler deploy --env staging
```

## Dashboard Configuration

**Enable via Dashboard:**
1. Navigate to **Workers & Pages** in Cloudflare dashboard
2. Select your Worker
3. Go to **Settings** → **General**
4. Under **Placement**, select **Smart**
5. Wait 15 minutes for analysis
6. Check **Metrics** tab for request duration charts

## Common Use Cases

### 1. Database-Heavy Workers

**Scenario:** Worker queries D1, Postgres, or MySQL multiple times per request.

**Configuration:**
```toml
[placement]
mode = "smart"
# Smart Placement runs Worker near database region
```

**Benefits:**
- Reduced round-trip latency per query
- Lower total request duration
- Improved database connection efficiency

### 2. Multi-Service Backend Aggregation

**Scenario:** Worker calls multiple backend microservices in same region.

```typescript
// Aggregates data from multiple services
export default {
  async fetch(request: Request, env: Env) {
    const [orders, inventory, shipping] = await Promise.all([
      fetch('https://orders.internal.api'),
      fetch('https://inventory.internal.api'),
      fetch('https://shipping.internal.api')
    ]);
    
    return Response.json({
      orders: await orders.json(),
      inventory: await inventory.json(),
      shipping: await shipping.json()
    });
  }
};
```

**Configuration:**
```toml
[placement]
mode = "smart"
hint = "enam"  # If services in East North America
```

### 3. Server-Side Rendering with Backend Data

**Scenario:** SSR application fetching data from backend before rendering.

**Best Practice:** Split frontend and backend

```typescript
// Frontend SSR Worker (no Smart Placement)
export default {
  async fetch(request: Request, env: Env) {
    // Fetch data from backend worker
    const data = await env.BACKEND.fetch('/api/page-data');
    
    // Render at edge close to user
    const html = renderPage(await data.json());
    
    return new Response(html, {
      headers: { 'Content-Type': 'text/html' }
    });
  }
};
```

```typescript
// Backend data Worker (Smart Placement enabled)
export default {
  async fetch(request: Request, env: Env) {
    // Runs close to database via Smart Placement
    const data = await env.DATABASE
      .prepare('SELECT * FROM pages WHERE id = ?')
      .bind(pageId)
      .first();
    
    return Response.json(data);
  }
};
```

### 4. API Gateway with Backend Authentication

```typescript
// API Gateway at edge (no Smart Placement)
export default {
  async fetch(request: Request, env: Env) {
    // Quick auth check at edge
    const authToken = request.headers.get('Authorization');
    if (!authToken) {
      return new Response('Unauthorized', { status: 401 });
    }
    
    // Forward to backend service via binding
    return env.BACKEND_API.fetch(request);
  }
};
```

```typescript
// Backend API (Smart Placement enabled)
export default {
  async fetch(request: Request, env: Env) {
    // Heavy DB operations run near database
    const result = await performDatabaseOperations(env.DATABASE);
    return Response.json(result);
  }
};
```

## Troubleshooting

### "INSUFFICIENT_INVOCATIONS" Status

**Problem:** Not enough traffic for Smart Placement to analyze.

**Solutions:**
- Ensure Worker receives consistent global traffic
- Wait longer (analysis takes up to 15 minutes)
- Send test traffic from multiple global locations
- Check Worker has fetch event handler

### Smart Placement Making Things Slower

**Problem:** `placement_status: "UNSUPPORTED_APPLICATION"`

**Likely Causes:**
- Worker doesn't make backend calls (runs faster at edge)
- Backend calls are cached (network latency to user more important)
- Backend service has poor global distribution

**Solutions:**
- Disable Smart Placement for this Worker
- Review whether Worker actually benefits from Smart Placement
- Consider caching strategy to reduce backend calls

### No Request Duration Metrics

**Problem:** Request duration chart not showing in dashboard.

**Solutions:**
- Ensure Smart Placement enabled in config
- Wait 15+ minutes after deployment
- Verify Worker has sufficient traffic
- Check `placement_status` is `SUCCESS`

### cf-placement Header Missing

**Problem:** Header not present in responses.

**Possible Causes:**
- Smart Placement not enabled
- Beta feature removed (check latest docs)
- Worker hasn't been analyzed yet

## TypeScript Types

### Environment Bindings with Smart Placement

```typescript
interface Env {
  // Backend Worker binding (Smart Placement enabled)
  BACKEND: Fetcher;
  
  // Database binding
  DATABASE: D1Database;
  
  // KV for caching
  CACHE: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // TypeScript-safe Smart Placement Worker
    const data = await env.DATABASE
      .prepare('SELECT * FROM table')
      .all();
    
    return Response.json(data);
  }
} satisfies ExportedHandler<Env>;
```

### Placement Status Types

```typescript
type PlacementStatus = 
  | 'SUCCESS'
  | 'INSUFFICIENT_INVOCATIONS'
  | 'UNSUPPORTED_APPLICATION'
  | undefined;

interface WorkerMetadata {
  placement_mode?: 'smart';
  placement_status?: PlacementStatus;
  // ... other fields
}
```

### Service Binding with Smart Placement

```typescript
interface Env {
  BACKEND_SERVICE: Fetcher;
}

// Frontend Worker - runs at edge
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Service binding to backend with Smart Placement
    const response = await env.BACKEND_SERVICE.fetch(request);
    return response;
  }
} satisfies ExportedHandler<Env>;
```

## Best Practices

1. **Split Full-Stack Apps:** Frontend at edge, backend with Smart Placement
2. **Use Service Bindings:** Connect frontend and backend Workers efficiently
3. **Monitor Request Duration:** Compare before/after metrics to validate improvement
4. **Enable for Backend Logic:** APIs, data aggregation, server-side processing
5. **Don't Enable for Pure Edge Logic:** Auth, redirects, static content
6. **Test Before Production:** Deploy to staging, verify metrics improve
7. **Consider Placement Hints:** Guide Smart Placement if you know backend location
8. **Wait for Analysis:** Allow 15+ minutes after enabling for meaningful metrics
9. **Check Placement Status:** Verify `SUCCESS` status via API before relying on optimization
10. **Combine with Caching:** Cache frequently accessed data to reduce backend calls

## Anti-Patterns

❌ **Enabling on static content Workers**
```toml
# DON'T: Static Worker doesn't benefit
[placement]
mode = "smart"
```

❌ **Monolithic full-stack Worker with Smart Placement**
```typescript
// DON'T: Frontend and backend in one Worker
export default {
  async fetch(request: Request, env: Env) {
    // Frontend render at edge + backend DB calls
    // Smart Placement hurts frontend performance
  }
};
```

✅ **Split architecture:**
```typescript
// Frontend Worker (no Smart Placement)
// Backend Worker (Smart Placement enabled)
```

❌ **Not monitoring placement status**
```bash
# DON'T: Deploy and assume it works
wrangler deploy
```

✅ **Verify placement status:**
```bash
wrangler deploy
# Check API for placement_status
# Review request duration metrics
```

## Additional Resources

- **Official Docs:** https://developers.cloudflare.com/workers/configuration/smart-placement/
- **Request Duration Metrics:** https://developers.cloudflare.com/workers/observability/metrics-and-analytics/#request-duration
- **Service Bindings:** https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/
- **Wrangler Configuration:** https://developers.cloudflare.com/workers/wrangler/configuration/

## Quick Reference

### Minimal Smart Placement Setup

**wrangler.toml:**
```toml
name = "my-backend-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[placement]
mode = "smart"
```

**src/index.ts:**
```typescript
export default {
  async fetch(request: Request, env: Env) {
    // Backend logic with DB/API calls
    const data = await env.DATABASE.prepare('SELECT * FROM table').all();
    return Response.json(data);
  }
};
```

**Deploy:**
```bash
wrangler deploy
```

**Verify:**
```bash
# Check placement status
curl -H "Authorization: Bearer $TOKEN" \
  https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/services/my-backend-worker \
  | jq .result.placement_status
```

That's it! Smart Placement is enabled and optimizing your Worker.
