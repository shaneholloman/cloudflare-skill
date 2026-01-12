# Cloudflare Workers for Platforms

Expert guidance for building multi-tenant platforms with Cloudflare Workers for Platforms - run untrusted customer code securely at scale.

## When to Use This Skill

Use this skill when working with:
- Building multi-tenant SaaS platforms that run customer code
- Executing untrusted code or AI-generated code in secure sandboxes
- Creating programmable platforms with isolated compute environments
- Deploying unlimited applications at scale
- Managing customer Workers via dispatch namespaces
- Implementing hostname routing for vanity domains or subdomains
- Controlling egress and ingress for customer code
- Setting per-customer resource limits

**NOT for general Cloudflare Workers** - this skill is ONLY for Workers for Platforms architecture.

## Core Architecture

### Key Components

Workers for Platforms consists of four components:

1. **Dispatch Namespace**: Container holding all customer Workers
   - Unlimited Workers (no per-account script limits)
   - Automatic isolation between tenants
   - Workers run in untrusted mode
   - Best practice: One namespace per environment (e.g., `production`, `staging`)

2. **Dynamic Dispatch Worker**: Entry point routing requests to customer Workers
   - Routes based on hostname, path, headers, or custom logic
   - Runs platform-level logic (auth, rate limiting, validation)
   - Enforces custom limits per customer
   - Sanitizes requests/responses

3. **User Workers**: Customer-written code running in isolated sandboxes
   - Deployed via API by platform
   - Can have bindings to KV, D1, R2, Durable Objects
   - Complete isolation from other customers
   - No access to `request.cf` object

4. **Outbound Worker** (optional): Intercepts external fetch requests from user Workers
   - Controls egress (allow/block external APIs)
   - Logs subrequests
   - Injects authentication headers
   - Enforces security policies

### Request Lifecycle

```
1. Request arrives → Dynamic Dispatch Worker
2. Dispatch Worker determines which user Worker to call
3. Dispatch Worker calls env.DISPATCHER.get("customer-name")
4. User Worker executes (outbound requests → Outbound Worker if configured)
5. User Worker returns response
6. Dispatch Worker optionally modifies response
7. Response sent to client
```

## Wrangler Configuration

### Dispatch Namespace Binding

Configure dispatch namespace binding in your dispatch Worker:

```jsonc
// wrangler.jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "dispatch_namespaces": [
    {
      "binding": "DISPATCHER",
      "namespace": "production"
    }
  ]
}
```

```toml
# wrangler.toml
[[dispatch_namespaces]]
binding = "DISPATCHER"
namespace = "production"
```

### With Outbound Worker

```jsonc
// wrangler.jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "dispatch_namespaces": [
    {
      "binding": "DISPATCHER",
      "namespace": "production",
      "outbound": {
        "service": "outbound-worker",
        "parameters": ["customer_context"]
      }
    }
  ]
}
```

### Wrangler Commands for Dispatch Namespaces

```bash
# List all dispatch namespaces
wrangler dispatch-namespace list

# Get namespace information
wrangler dispatch-namespace get production

# Create a dispatch namespace
wrangler dispatch-namespace create production

# Delete a dispatch namespace
wrangler dispatch-namespace delete staging

# Rename a dispatch namespace
wrangler dispatch-namespace rename old-name new-name
```

## Dynamic Dispatch Worker Patterns

### Subdomain-Based Routing

Route `customer.platform.com` to user Worker named `customer`:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    try {
      const url = new URL(request.url);
      const userWorkerName = url.hostname.split(".")[0];
      
      const userWorker = env.DISPATCHER.get(userWorkerName);
      return await userWorker.fetch(request);
    } catch (e) {
      if (e.message.startsWith("Worker not found")) {
        return new Response("Worker not found", { status: 404 });
      }
      return new Response(e.message, { status: 500 });
    }
  },
};
```

### Path-Based Routing

Route `platform.com/customer-1` to user Worker named `customer-1`:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    try {
      const url = new URL(request.url);
      const pathParts = url.pathname.split("/").filter(Boolean);
      
      if (pathParts.length === 0) {
        return new Response("Invalid path", { status: 400 });
      }
      
      const userWorkerName = pathParts[0];
      const userWorker = env.DISPATCHER.get(userWorkerName);
      return await userWorker.fetch(request);
    } catch (e) {
      if (e.message.startsWith("Worker not found")) {
        return new Response("Worker not found", { status: 404 });
      }
      return new Response(e.message, { status: 500 });
    }
  },
};
```

### KV-Based Routing

Store hostname-to-Worker mappings in KV for flexible routing:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    try {
      const url = new URL(request.url);
      const routingKey = url.hostname;
      
      // Lookup user Worker name from KV
      const userWorkerName = await env.ROUTING_KV.get(routingKey);
      
      if (!userWorkerName) {
        return new Response("Route not configured", { status: 404 });
      }
      
      const userWorker = env.DISPATCHER.get(userWorkerName);
      return await userWorker.fetch(request);
    } catch (e) {
      if (e.message.startsWith("Worker not found")) {
        return new Response("Worker not found", { status: 404 });
      }
      return new Response(e.message, { status: 500 });
    }
  },
};
```

### Custom Limits by Plan Type

Enforce CPU and subrequest limits based on customer plan:

```typescript
interface Env {
  DISPATCHER: DispatchNamespace;
  CUSTOMERS_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    try {
      const url = new URL(request.url);
      const userWorkerName = url.hostname.split(".")[0];
      
      // Look up customer plan from KV
      const customerPlan = await env.CUSTOMERS_KV.get(userWorkerName);
      
      // Define limits per plan
      const plans = {
        enterprise: { cpuMs: 50, subRequests: 50 },
        pro: { cpuMs: 20, subRequests: 20 },
        free: { cpuMs: 10, subRequests: 5 },
      };
      const limits = plans[customerPlan as keyof typeof plans] || plans.free;
      
      const userWorker = env.DISPATCHER.get(userWorkerName, {}, { limits });
      return await userWorker.fetch(request);
    } catch (e) {
      if (e.message.includes("CPU time limit")) {
        return new Response("CPU limit exceeded", { status: 429 });
      }
      if (e.message.startsWith("Worker not found")) {
        return new Response("Worker not found", { status: 404 });
      }
      return new Response(e.message, { status: 500 });
    }
  },
};
```

## Custom Limits

Set limits on CPU time and subrequests per invocation:

```typescript
const userWorker = env.DISPATCHER.get(
  workerName,
  {},
  {
    limits: { 
      cpuMs: 10,        // Maximum CPU milliseconds
      subRequests: 5    // Maximum number of fetch() calls
    }
  }
);
```

When a user Worker exceeds limits, it immediately throws an exception. Handle these in your dispatch Worker:

```typescript
try {
  return await userWorker.fetch(request);
} catch (e) {
  if (e.message.includes("CPU time limit")) {
    // Track with Analytics Engine
    env.ANALYTICS.writeDataPoint({
      indexes: [workerName],
      blobs: ["cpu_limit_exceeded"],
    });
    return new Response("CPU limit exceeded", { status: 429 });
  }
  throw e;
}
```

## Outbound Workers

Control and monitor external fetch() requests from user Workers:

### Configure in Dispatch Worker

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const workerName = url.hostname.split(".")[0];
    
    const contextFromDispatcher = {
      customer_name: workerName,
      url: request.url,
    };
    
    const userWorker = env.DISPATCHER.get(
      workerName,
      {},
      {
        outbound: {
          customer_context: contextFromDispatcher,
        },
      }
    );
    
    return await userWorker.fetch(request);
  },
};
```

### Outbound Worker Implementation

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Access parameters from dispatch Worker
    const customerName = env.customer_name;
    const originalUrl = env.url;
    
    // Log all subrequests
    ctx.waitUntil(
      fetch("https://logs.example.com", {
        method: "POST",
        body: JSON.stringify({
          customer_name: customerName,
          original_url: originalUrl,
          subrequest_url: request.url,
        }),
      })
    );
    
    const url = new URL(request.url);
    
    // Block certain domains
    const blockedDomains = ["malicious-site.com", "spam-api.com"];
    if (blockedDomains.some(domain => url.hostname.includes(domain))) {
      return new Response("Domain blocked", { status: 403 });
    }
    
    // Inject authentication for internal APIs
    if (url.hostname === "api.example.com") {
      const jwt = generateJWT(customerName);
      const headers = new Headers(request.headers);
      headers.set("Authorization", `Bearer ${jwt}`);
      
      const newRequest = new Request(request, { headers });
      return fetch(newRequest);
    }
    
    // Allow all other requests
    return fetch(request);
  },
};
```

**Important**: Outbound Workers don't intercept fetch requests from Durable Objects or mTLS bindings.

## API Operations

### Deploy a User Worker

```bash
# Create worker script file
cat > worker.mjs << 'EOF'
export default {
  async fetch(request, env, ctx) {
    return new Response("Hello from user Worker!");
  },
};
EOF

# Deploy using multipart form (required for ES modules)
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME" \
  -H "Authorization: Bearer $API_TOKEN" \
  -F 'metadata={"main_module": "worker.mjs"};type=application/json' \
  -F 'worker.mjs=@worker.mjs;type=application/javascript+module'
```

### TypeScript SDK Deployment

```typescript
import Cloudflare from "cloudflare";

const client = new Cloudflare({ apiToken: process.env.API_TOKEN });

async function deployUserWorker(
  accountId: string,
  namespace: string,
  scriptName: string,
  scriptContent: string
) {
  const scriptFile = new File([scriptContent], `${scriptName}.mjs`, {
    type: "application/javascript+module",
  });

  return await client.workersForPlatforms.dispatch.namespaces.scripts.update(
    namespace,
    scriptName,
    {
      account_id: accountId,
      metadata: {
        main_module: `${scriptName}.mjs`,
      },
      files: [scriptFile],
    }
  );
}
```

### Deploy with Bindings and Tags

```bash
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME" \
  -H "Authorization: Bearer $API_TOKEN" \
  -F 'metadata={
    "main_module": "worker.mjs",
    "bindings": [
      {
        "type": "kv_namespace",
        "name": "MY_KV",
        "namespace_id": "'"$KV_NAMESPACE_ID"'"
      }
    ],
    "tags": ["customer-123", "production", "pro-plan"],
    "compatibility_date": "2024-01-01"
  };type=application/json' \
  -F 'worker.mjs=@worker.mjs;type=application/javascript+module'
```

### TypeScript: Deploy with Bindings and Tags

```typescript
async function deployWithBindingsAndTags(
  accountId: string,
  namespace: string,
  scriptName: string,
  scriptContent: string,
  kvNamespaceId: string,
  tags: string[]
) {
  const scriptFile = new File([scriptContent], `${scriptName}.mjs`, {
    type: "application/javascript+module",
  });

  return await client.workersForPlatforms.dispatch.namespaces.scripts.update(
    namespace,
    scriptName,
    {
      account_id: accountId,
      metadata: {
        main_module: `${scriptName}.mjs`,
        compatibility_date: "2024-01-01",
        bindings: [
          {
            type: "kv_namespace",
            name: "MY_KV",
            namespace_id: kvNamespaceId,
          },
        ],
        tags,
      },
      files: [scriptFile],
    }
  );
}

await deployWithBindingsAndTags(
  "account-id",
  "production",
  "customer-123-app",
  scriptContent,
  "kv-namespace-id",
  ["customer-123", "production", "pro-plan"]
);
```

### List Workers in Namespace

```bash
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts" \
  -H "Authorization: Bearer $API_TOKEN"
```

### Delete Worker by Name

```bash
curl -X DELETE \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME" \
  -H "Authorization: Bearer $API_TOKEN"
```

### Delete Workers by Tag

Bulk delete all Workers matching a tag (useful when customer leaves):

```bash
curl -X DELETE \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts?tags=customer-123%3Ayes" \
  -H "Authorization: Bearer $API_TOKEN"
```

## Bindings

Give user Workers access to Cloudflare resources via bindings.

### Supported Binding Types

- KV namespaces
- D1 databases
- R2 buckets
- Durable Objects
- Analytics Engine
- Service bindings
- Assets (for static files)

### Adding Bindings via API

Bindings must be specified in the `metadata` object of multipart upload:

```bash
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME" \
  -H "Authorization: Bearer $API_TOKEN" \
  -F 'metadata={
    "main_module": "worker.mjs",
    "bindings": [
      {
        "type": "kv_namespace",
        "name": "USER_KV",
        "namespace_id": "'"$KV_NAMESPACE_ID"'"
      },
      {
        "type": "r2_bucket",
        "name": "USER_STORAGE",
        "bucket_name": "customer-bucket"
      },
      {
        "type": "d1",
        "name": "DB",
        "id": "'"$D1_DATABASE_ID"'"
      }
    ]
  };type=application/json' \
  -F 'worker.mjs=@worker.mjs;type=application/javascript+module'
```

### Preserving Existing Bindings

Use `keep_bindings` to preserve existing bindings when adding new ones:

```bash
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME" \
  -H "Authorization: Bearer $API_TOKEN" \
  -F 'metadata={
    "bindings": [
      {
        "type": "r2_bucket",
        "name": "STORAGE",
        "bucket_name": "new-bucket"
      }
    ],
    "keep_bindings": ["kv_namespace", "d1"]
  }'
```

### Resource Isolation

For complete isolation, create unique resources per user Worker:
- Each customer gets their own KV namespace
- Each customer gets their own D1 database
- Each customer gets their own R2 bucket

## Tags

Use tags to organize, search, and filter user Workers at scale.

### Common Tag Patterns

- Customer ID: `customer-123`
- Plan type: `free`, `pro`, `enterprise`
- Environment: `production`, `staging`
- Project ID: `project-abc`
- Feature flags: `beta-features`

**Limits**: Max 8 tags per script. Avoid special characters like `,` and `&`.

### Tags API

```bash
# Get tags for a script
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME/tags" \
  -H "Authorization: Bearer $API_TOKEN"

# Set tags (replaces all existing tags)
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME/tags" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '["customer-123", "pro-plan", "production"]'

# Add a single tag
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME/tags/customer-123" \
  -H "Authorization: Bearer $API_TOKEN"

# Delete a single tag
curl -X DELETE \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME/tags/staging" \
  -H "Authorization: Bearer $API_TOKEN"

# Filter Workers by tag
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts?tags=production%3Ayes" \
  -H "Authorization: Bearer $API_TOKEN"

# Delete all Workers with tag
curl -X DELETE \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts?tags=customer-123%3Ayes" \
  -H "Authorization: Bearer $API_TOKEN"
```

## Hostname Routing

Route millions of hostnames to Workers without route limits.

### Wildcard Route (Recommended)

Configure `*/*` route on your SaaS domain pointing to dispatch Worker:

**Benefits**:
- Supports both subdomains and custom vanity domains
- Avoids route limits
- Programmatic routing control
- Works regardless of customer DNS proxy settings

**Setup**:
1. Configure custom hostnames via Cloudflare for SaaS
2. Set fallback origin (can be dummy: `A 192.0.2.0` if Worker is origin)
3. Configure DNS (CNAME to SaaS domain)
4. Create `*/*` route pointing to dispatch Worker
5. Implement routing logic in dispatch Worker

**Example**:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const hostname = new URL(request.url).hostname;
    
    // Get custom hostname metadata for routing
    const hostnameData = await env.ROUTING_KV.get(`hostname:${hostname}`, {
      type: "json",
    });
    
    if (!hostnameData?.workerName) {
      return new Response("Hostname not configured", { status: 404 });
    }
    
    // Route to appropriate user Worker
    const userWorker = env.DISPATCHER.get(hostnameData.workerName);
    return await userWorker.fetch(request);
  },
};
```

### Subdomain-Only Routing

For subdomain-only routing (e.g., `customer.saas.com`):

**Setup**:
1. Create wildcard DNS: `*.saas.com` → origin (or dummy `A 192.0.2.0`)
2. Set route: `*.saas.com/*` → dispatch Worker
3. Implement subdomain routing logic

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const subdomain = url.hostname.split(".")[0];
    
    if (subdomain && subdomain !== "saas") {
      const userWorker = env.DISPATCHER.get(subdomain);
      return await userWorker.fetch(request);
    }
    
    return new Response("Invalid subdomain", { status: 400 });
  },
};
```

### Orange-to-Orange Considerations

When customers use Cloudflare and CNAME to your domain:
- `*/*` wildcard route works regardless of DNS proxy settings (recommended)
- Specific hostname routes behave differently based on proxy mode
- Use wildcard to ensure consistent behavior

## Static Assets

Deploy front-end applications with static files (HTML, CSS, images) alongside Workers.

### Upload Process

1. **Create Upload Session** - Send manifest with file paths, hashes, sizes
2. **Upload File Contents** - Upload missing files in buckets
3. **Deploy Worker** - Attach assets using completion token

### Manifest Format

```json
{
  "/index.html": {
    "hash": "08f1dfda4574284ab3c21666d1ee8c7d4",
    "size": 1234
  },
  "/styles.css": {
    "hash": "36b8be012ee77df5f269b11b975611d3",
    "size": 5678
  }
}
```

**Hash**: First 16 bytes (32 hex chars) of SHA-256 digest

### API Example

```bash
# 1. Create upload session
curl -X POST \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME/assets-upload-session" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "manifest": {
      "/index.html": {
        "hash": "08f1dfda4574284ab3c21666d1ee8c7d4",
        "size": 1234
      }
    }
  }'

# Response includes jwt and buckets array

# 2. Upload file contents (if buckets present)
curl -X POST \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/assets/upload?base64=true" \
  -H "Authorization: Bearer $UPLOAD_JWT" \
  -F '08f1dfda4574284ab3c21666d1ee8c7d4=<BASE64_CONTENT>'

# Response includes completion jwt

# 3. Deploy Worker with assets
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/dispatch/namespaces/$NAMESPACE/scripts/$SCRIPT_NAME" \
  -H "Authorization: Bearer $API_TOKEN" \
  -F 'metadata={
    "main_module": "index.js",
    "assets": {
      "jwt": "<COMPLETION_TOKEN>",
      "config": {
        "html_handling": "auto-trailing-slash"
      }
    },
    "bindings": [{"type": "assets", "name": "ASSETS"}]
  };type=application/json' \
  -F 'index.js=export default { async fetch(request, env) { return env.ASSETS.fetch(request); } };type=application/javascript+module'
```

### Asset Isolation

Assets are associated with **namespace**, not individual Workers. Workers sharing same namespace can share assets with identical hashes.

**For strict isolation**: Salt hashes with unique identifier:
```typescript
const hash = sha256(accountId + fileContents).slice(0, 32);
```

### Wrangler Deployment

```jsonc
// wrangler.jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "customer-site",
  "main": "./src/index.js",
  "compatibility_date": "2025-01-29",
  "assets": {
    "directory": "./public",
    "binding": "ASSETS"
  }
}
```

```bash
# Deploy with Wrangler
npx wrangler deploy --name customer-site --dispatch-namespace production
```

## Observability

### Logpush

Capture logs for entire dispatch namespace or individual Workers:

1. Create Logpush job for Workers Trace Events
2. Enable logging on dispatch Worker (captures all user Workers)

For individual user Worker logs, enable logging on specific Worker.

Use Logpush filters on `Outcome` or `Script Name` to route logs.

### Tail Workers

For real-time logs with custom formatting:

1. Create Tail Worker
2. Add to dispatch Worker (captures all user Worker logs)

Tail Worker receives:
- HTTP statuses
- `console.log()` output
- Uncaught exceptions
- Diagnostics channel events

### Analytics Engine

Write custom events from user Workers or dispatch Worker:

```typescript
// Track limit violations
env.ANALYTICS.writeDataPoint({
  indexes: [customerName],
  blobs: ["cpu_limit_exceeded"],
});
```

Query by script tag to get per-customer aggregates.

### GraphQL Analytics API

Query metrics by dispatch namespace:

```graphql
query {
  viewer {
    accounts(filter: {accountTag: $accountId}) {
      workersInvocationsAdaptive(
        filter: {dispatchNamespaceName: "production"}
      ) {
        sum {
          requests
          errors
          cpuTime
        }
      }
    }
  }
}
```

## Best Practices

### Architecture

- **One namespace per environment** - Use `production`, `staging`, not per-customer namespaces
- **Unlimited Workers** - Dispatch namespaces have no script limits
- **Isolation by default** - User Workers never share cache, run in untrusted mode
- **Platform logic in dispatch Worker** - Auth, rate limiting, validation before routing

### Routing

- **Use wildcard routes** - `*/*` route avoids route limits and DNS issues
- **Store mappings in KV** - Decouple routing logic from dispatch Worker code
- **Handle Worker not found** - Always catch and handle missing Workers gracefully

### Limits & Security

- **Set custom limits** - Enforce CPU and subrequest limits based on plan
- **Track violations** - Use Analytics Engine to monitor limit breaches
- **Use outbound Workers** - Control and log external API calls
- **Sanitize responses** - Remove sensitive headers before returning to clients

### Bindings

- **Isolate resources** - Create unique KV/D1/R2 per customer for complete isolation
- **Use keep_bindings** - Preserve existing bindings when updating
- **Document bindings** - Track which resources are bound to which Workers

### Tags

- **Tag all Workers** - Include customer ID, plan, environment
- **Enable bulk operations** - Delete all customer Workers with single API call
- **Filter efficiently** - Use tags to query subsets of Workers

### Static Assets

- **Salt hashes for isolation** - Include customer ID in hash to prevent sharing
- **Never expose JWTs** - Keep upload tokens server-side only
- **Cache assets** - Cloudflare automatically caches at edge

### Observability

- **Enable Logpush on dispatch Worker** - Captures all user Worker logs
- **Use Tail Workers for real-time** - When you need immediate feedback
- **Track metrics with Analytics Engine** - Custom events for business logic
- **Query with GraphQL** - Aggregate metrics across namespace

## Common Use Cases

### Multi-Tenant SaaS Platform

Give each customer their own Worker with isolated resources:

```typescript
// Dispatch Worker
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const subdomain = new URL(request.url).hostname.split(".")[0];
    
    // Get customer plan from KV
    const customer = await env.CUSTOMERS_KV.get(subdomain, { type: "json" });
    
    if (!customer) {
      return new Response("Customer not found", { status: 404 });
    }
    
    // Set limits based on plan
    const limits = {
      free: { cpuMs: 10, subRequests: 5 },
      pro: { cpuMs: 30, subRequests: 20 },
      enterprise: { cpuMs: 100, subRequests: 100 },
    }[customer.plan];
    
    const userWorker = env.DISPATCHER.get(subdomain, {}, { limits });
    return await userWorker.fetch(request);
  },
};

// User Worker (customer code)
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Access customer's isolated KV
    const data = await env.USER_KV.get("key");
    return new Response(data);
  },
};
```

### AI Code Execution Platform

Execute AI-generated or user-submitted code safely:

```typescript
// Platform receives code from AI or user
async function deployGeneratedCode(
  customerName: string,
  generatedCode: string
) {
  const scriptFile = new File([generatedCode], `${customerName}.mjs`, {
    type: "application/javascript+module",
  });

  await client.workersForPlatforms.dispatch.namespaces.scripts.update(
    "production",
    customerName,
    {
      account_id: accountId,
      metadata: {
        main_module: `${customerName}.mjs`,
        tags: [customerName, "ai-generated"],
      },
      files: [scriptFile],
    }
  );
}

// Dispatch Worker routes to AI-generated Workers
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const sessionId = request.headers.get("X-Session-ID");
    
    // Short CPU limits for untrusted code
    const userWorker = env.DISPATCHER.get(
      sessionId,
      {},
      { limits: { cpuMs: 5, subRequests: 3 } }
    );
    
    return await userWorker.fetch(request);
  },
};
```

### Edge Functions Platform

Let customers deploy serverless functions at the edge:

```typescript
// Customer uploads function via your API
async function deployEdgeFunction(
  customerId: string,
  functionName: string,
  code: string,
  kvNamespaceId: string
) {
  const scriptFile = new File([code], `${functionName}.mjs`, {
    type: "application/javascript+module",
  });

  await client.workersForPlatforms.dispatch.namespaces.scripts.update(
    "production",
    `${customerId}-${functionName}`,
    {
      account_id: accountId,
      metadata: {
        main_module: `${functionName}.mjs`,
        bindings: [
          {
            type: "kv_namespace",
            name: "STORAGE",
            namespace_id: kvNamespaceId,
          },
        ],
        tags: [customerId, functionName],
      },
      files: [scriptFile],
    }
  );
}

// Route by path: /customer-id/function-name
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const [customerId, functionName] = url.pathname.split("/").filter(Boolean);
    
    const workerName = `${customerId}-${functionName}`;
    const userWorker = env.DISPATCHER.get(workerName);
    return await userWorker.fetch(request);
  },
};
```

### Website Builder Platform

Host customer websites with static assets and dynamic logic:

```typescript
// Deploy site with static assets and Worker
async function deploySite(
  customerId: string,
  files: Array<{ path: string; content: string; size: number }>,
  workerCode: string
) {
  // 1. Create manifest
  const manifest: Record<string, { hash: string; size: number }> = {};
  for (const file of files) {
    const hash = await sha256Hash(file.content);
    manifest[file.path] = { hash: hash.slice(0, 32), size: file.size };
  }

  // 2. Create upload session
  const sessionRes = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/workers/dispatch/namespaces/production/scripts/${customerId}/assets-upload-session`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ manifest }),
    }
  );
  const { jwt, buckets } = await sessionRes.json();

  // 3. Upload files if needed
  if (buckets) {
    for (const bucket of buckets) {
      const formData = new FormData();
      for (const hash of bucket) {
        const file = files.find(f => manifest[f.path].hash === hash);
        formData.append(hash, btoa(file.content));
      }
      
      const uploadRes = await fetch(
        `https://api.cloudflare.com/client/v4/accounts/${accountId}/workers/assets/upload?base64=true`,
        {
          method: "POST",
          headers: { Authorization: `Bearer ${jwt}` },
          body: formData,
        }
      );
      const { jwt: completionToken } = await uploadRes.json();
    }
  }

  // 4. Deploy Worker with assets
  const deployFormData = new FormData();
  deployFormData.append(
    "metadata",
    JSON.stringify({
      main_module: "index.js",
      assets: { jwt: completionToken },
      bindings: [{ type: "assets", name: "ASSETS" }],
      tags: [customerId, "website"],
    })
  );
  deployFormData.append("index.js", workerCode);

  await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/workers/dispatch/namespaces/production/scripts/${customerId}`,
    {
      method: "PUT",
      headers: { Authorization: `Bearer ${apiToken}` },
      body: deployFormData,
    }
  );
}
```

## TypeScript Types

```typescript
interface Env {
  DISPATCHER: DispatchNamespace;
  ROUTING_KV: KVNamespace;
  CUSTOMERS_KV: KVNamespace;
  ANALYTICS: AnalyticsEngineDataset;
}

interface DispatchNamespace {
  get(
    name: string,
    options?: Record<string, unknown>,
    config?: {
      limits?: {
        cpuMs?: number;
        subRequests?: number;
      };
      outbound?: Record<string, unknown>;
    }
  ): Fetcher;
}

interface Fetcher {
  fetch(request: Request): Promise<Response>;
}
```

## Reference Architecture

Workers for Platforms enables building:
- **Programmable Platforms** - Run customer code at scale
- **AI Vibe Coding Platforms** - Execute AI-generated code safely
- **Edge Functions Platforms** - Serverless functions at the edge
- **Website Builders** - Host static sites with dynamic logic
- **API Gateways** - Route and transform customer APIs

## Related Resources

- [Workers for Platforms Docs](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/)
- [Platform Starter Kit](https://github.com/cloudflare/templates/tree/main/worker-publisher-template)
- [VibeSDK](https://github.com/cloudflare/vibesdk) - AI coding platform
- [Cloudflare for SaaS](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/) - Custom hostnames

## Limits

- **Max 8 tags per Worker**
- **Upload session JWT valid 1 hour**
- **Completion token valid 1 hour**
- **No limits on Workers per namespace** (unlike regular Workers)
- **User Workers run in untrusted mode** (no `request.cf` access)
- **Outbound Workers don't intercept Durable Objects or mTLS fetch**
