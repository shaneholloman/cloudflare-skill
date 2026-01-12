# Cloudflare Workers Skill

Expert guidance for building, deploying, and optimizing Cloudflare Workers applications.

## When to Use This Skill

Use when working with:
- Cloudflare Workers serverless functions
- Wrangler CLI configuration and deployment
- Workers runtime APIs and bindings
- Edge computing patterns and optimization
- Workers-specific architecture decisions

## Core Concepts

### Workers Runtime

Cloudflare Workers run on V8 isolates (NOT containers/VMs):
- Extremely fast cold starts (< 1ms)
- Global deployment across 300+ locations
- Web standards compliant (fetch, URL, Headers, Request, Response)
- Support JS/TS, Python, Rust, and WebAssembly

**Key principle**: Workers use web platform APIs wherever possible for portability.

### Module Worker Pattern (Recommended)

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    return new Response('Hello World!');
  },
};
```

**Handler parameters**:
- `request`: Incoming HTTP request (standard Request object)
- `env`: Environment bindings (KV, D1, R2, secrets, vars)
- `ctx`: Execution context (`waitUntil`, `passThroughOnException`)

### Execution Context (`ctx`)

Critical for async operations that outlive response:

```typescript
export default {
  async fetch(request, env, ctx) {
    // Fire-and-forget without blocking response
    ctx.waitUntil(logAnalytics(request));
    
    return new Response('OK');
  },
};
```

**Never** use `await` for operations you want to background - use `ctx.waitUntil()`.

## Wrangler Configuration

### Modern Config (wrangler.jsonc - RECOMMENDED)

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-01-01",
  "compatibility_flags": [],
  
  // Bindings (non-inheritable across environments)
  "vars": {
    "ENVIRONMENT": "production"
  },
  "kv_namespaces": [
    {
      "binding": "MY_KV",
      "id": "abc123"
    }
  ],
  "r2_buckets": [
    {
      "binding": "MY_BUCKET",
      "bucket_name": "my-bucket"
    }
  ],
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-db",
      "database_id": "xyz789"
    }
  ],
  
  // Environments
  "env": {
    "staging": {
      "vars": {
        "ENVIRONMENT": "staging"
      },
      "kv_namespaces": [
        {
          "binding": "MY_KV",
          "id": "staging-id"
        }
      ]
    }
  }
}
```

### Key Configuration Rules

1. **Compatibility Date**: ALWAYS set to current date for new projects
2. **Inheritable keys**: `name`, `main`, `compatibility_date`, `routes`, `workers_dev`
3. **Non-inheritable keys**: All bindings (`vars`, `kv_namespaces`, `r2_buckets`, etc.)
4. **Top-level only**: `migrations`, `keep_vars`, `send_metrics`

### Automatic Provisioning (Beta)

Bindings without IDs are auto-created on deploy:

```jsonc
{
  "kv_namespaces": [
    { "binding": "MY_KV" }  // ID added automatically on wrangler deploy
  ]
}
```

## Wrangler CLI Patterns

### Development

```bash
# Local dev (uses local storage/bindings)
npx wrangler dev

# Remote dev (uses actual Cloudflare resources)
npx wrangler dev --remote

# Specific environment
npx wrangler dev --env staging

# Custom port
npx wrangler dev --port 8787
```

### Deployment

```bash
# Deploy to production
npx wrangler deploy

# Deploy to specific environment
npx wrangler deploy --env staging

# Dry run (validate without deploying)
npx wrangler deploy --dry-run

# Deploy with version metadata
npx wrangler versions upload
npx wrangler versions deploy
```

### Tail Logs (Live)

```bash
# Stream production logs
npx wrangler tail

# Filter by status
npx wrangler tail --status error

# Specific environment
npx wrangler tail --env staging
```

### Secret Management

```bash
# Set secret (interactive prompt)
npx wrangler secret put API_KEY

# Set secret from file
cat secret.txt | npx wrangler secret put API_KEY

# List secrets (names only, not values)
npx wrangler secret list

# Delete secret
npx wrangler secret delete API_KEY
```

## Runtime APIs & Patterns

### Fetch API (Core)

Standard Web API - primary entry point for HTTP requests:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // Method handling
    if (request.method === 'POST') {
      const body = await request.json();
      // Process POST data
    }
    
    // Path routing
    if (url.pathname === '/api/users') {
      return new Response(JSON.stringify({ users: [] }), {
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    // Subrequest to origin
    return fetch(request);
  },
};
```

### Bindings Access via `env`

All bindings available through the `env` parameter:

```typescript
interface Env {
  MY_KV: KVNamespace;
  MY_BUCKET: R2Bucket;
  DB: D1Database;
  API_KEY: string;  // Secret
  ENVIRONMENT: string;  // Var
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // KV
    const value = await env.MY_KV.get('key');
    await env.MY_KV.put('key', 'value', { expirationTtl: 3600 });
    
    // R2
    const object = await env.MY_BUCKET.get('file.txt');
    await env.MY_BUCKET.put('file.txt', 'content');
    
    // D1
    const result = await env.DB.prepare('SELECT * FROM users WHERE id = ?')
      .bind(1)
      .first();
    
    // Secrets/vars
    const apiKey = env.API_KEY;
    
    return new Response('OK');
  },
};
```

### Cache API

Programmatic cache control:

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const cache = caches.default;
    const cacheKey = new Request(request.url, request);
    
    // Try cache first
    let response = await cache.match(cacheKey);
    
    if (!response) {
      // Cache miss - fetch from origin
      response = await fetch(request);
      
      // Clone and cache (response body can only be read once)
      response = new Response(response.body, response);
      response.headers.set('Cache-Control', 'max-age=3600');
      
      // Don't await - cache in background
      ctx.waitUntil(cache.put(cacheKey, response.clone()));
    }
    
    return response;
  },
};
```

**Critical**: Always use `response.clone()` when caching - response bodies are streams.

### HTMLRewriter (Streaming HTML Parser)

Transform HTML without buffering entire response:

```typescript
class MetaTagRewriter {
  element(element: Element) {
    element.setAttribute('content', 'new value');
  }
}

export default {
  async fetch(request: Request): Promise<Response> {
    const response = await fetch(request);
    
    return new HTMLRewriter()
      .on('meta[property="og:title"]', new MetaTagRewriter())
      .on('a[href]', {
        element(el) {
          const href = el.getAttribute('href');
          if (href?.startsWith('http://')) {
            el.setAttribute('href', href.replace('http://', 'https://'));
          }
        }
      })
      .transform(response);
  },
};
```

**Use cases**: 
- A/B testing injection
- Analytics/tracking insertion  
- Content transformation
- Link rewriting

### WebSockets

Bidirectional communication:

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const upgradeHeader = request.headers.get('Upgrade');
    if (upgradeHeader !== 'websocket') {
      return new Response('Expected websocket', { status: 426 });
    }
    
    const [client, server] = Object.values(new WebSocketPair());
    
    server.accept();
    server.addEventListener('message', event => {
      server.send(`Echo: ${event.data}`);
    });
    
    return new Response(null, {
      status: 101,
      webSocket: client,
    });
  },
};
```

### Durable Objects

Strongly consistent, stateful coordination:

```typescript
// Define Durable Object class
export class Counter {
  private state: DurableObjectState;
  private value: number = 0;
  
  constructor(state: DurableObjectState) {
    this.state = state;
    this.state.blockConcurrencyWhile(async () => {
      this.value = (await this.state.storage.get('value')) || 0;
    });
  }
  
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    
    if (url.pathname === '/increment') {
      this.value++;
      await this.state.storage.put('value', this.value);
      return new Response(String(this.value));
    }
    
    return new Response(String(this.value));
  }
}

// Worker that uses Durable Object
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const id = env.COUNTER.idFromName('global');
    const stub = env.COUNTER.get(id);
    return stub.fetch(request);
  },
};
```

**When to use**:
- Real-time collaboration
- Connection state management
- Rate limiting per-entity
- Strongly consistent counters

### Scheduled Events (Cron Triggers)

```typescript
export default {
  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext) {
    // Runs on cron schedule
    ctx.waitUntil(doCleanup(env));
  },
};
```

Configuration in `wrangler.jsonc`:

```jsonc
{
  "triggers": {
    "crons": ["0 */6 * * *"]  // Every 6 hours
  }
}
```

### Queue Handlers

Producer:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    await env.MY_QUEUE.send({ timestamp: Date.now() });
    return new Response('Queued');
  },
};
```

Consumer:

```typescript
export default {
  async queue(batch: MessageBatch<any>, env: Env): Promise<void> {
    for (const message of batch.messages) {
      await processMessage(message.body);
      message.ack();
    }
  },
};
```

## Common Patterns

### Routing

```typescript
const router = {
  'GET /api/users': handleGetUsers,
  'POST /api/users': handleCreateUser,
  'GET /api/users/:id': handleGetUser,
};

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const route = `${request.method} ${url.pathname}`;
    
    const handler = router[route];
    if (handler) {
      return handler(request, env);
    }
    
    return new Response('Not Found', { status: 404 });
  },
};
```

**Better**: Use framework like Hono, itty-router, or Worktop for production.

### Error Handling

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    try {
      // Your logic
      return new Response('OK');
    } catch (error) {
      console.error('Error:', error);
      
      return new Response('Internal Server Error', { 
        status: 500,
        headers: { 'Content-Type': 'text/plain' }
      });
    }
  },
};
```

**Production pattern**: Structured error responses:

```typescript
class HTTPError extends Error {
  constructor(public status: number, message: string) {
    super(message);
  }
}

function errorResponse(error: unknown): Response {
  if (error instanceof HTTPError) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: error.status,
      headers: { 'Content-Type': 'application/json' }
    });
  }
  
  console.error('Unexpected error:', error);
  return new Response('Internal Server Error', { status: 500 });
}
```

### CORS Handling

```typescript
const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type',
};

export default {
  async fetch(request: Request): Promise<Response> {
    // Handle preflight
    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }
    
    const response = new Response('OK');
    
    // Add CORS headers to response
    Object.entries(corsHeaders).forEach(([key, value]) => {
      response.headers.set(key, value);
    });
    
    return response;
  },
};
```

### Authentication

```typescript
function verifyJWT(token: string, secret: string): boolean {
  // Use Web Crypto API or imported library
  // Workers supports jose, jsonwebtoken, etc via npm
  return true; // Simplified
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const authHeader = request.headers.get('Authorization');
    
    if (!authHeader?.startsWith('Bearer ')) {
      return new Response('Unauthorized', { status: 401 });
    }
    
    const token = authHeader.substring(7);
    if (!verifyJWT(token, env.JWT_SECRET)) {
      return new Response('Invalid token', { status: 403 });
    }
    
    // Authorized request
    return new Response('Protected data');
  },
};
```

### Rate Limiting (with Durable Objects)

```typescript
export class RateLimiter {
  private state: DurableObjectState;
  
  constructor(state: DurableObjectState) {
    this.state = state;
  }
  
  async fetch(request: Request): Promise<Response> {
    const key = new URL(request.url).searchParams.get('key')!;
    const now = Date.now();
    const minute = Math.floor(now / 60000);
    const rateKey = `${key}:${minute}`;
    
    const count = ((await this.state.storage.get(rateKey)) || 0) as number;
    
    if (count >= 100) {
      return new Response('Rate limit exceeded', { status: 429 });
    }
    
    await this.state.storage.put(rateKey, count + 1, {
      expirationTtl: 60
    });
    
    return new Response('OK');
  },
}
```

## Performance Optimization

### Minimize Subrequests

```typescript
// ❌ BAD - Sequential
const user = await fetch(`/api/user/${id}`);
const posts = await fetch(`/api/posts?user=${id}`);

// ✅ GOOD - Parallel
const [user, posts] = await Promise.all([
  fetch(`/api/user/${id}`),
  fetch(`/api/posts?user=${id}`)
]);
```

### Stream Large Responses

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const stream = new ReadableStream({
      async start(controller) {
        for (let i = 0; i < 1000; i++) {
          controller.enqueue(new TextEncoder().encode(`Item ${i}\n`));
          
          // Avoid blocking - yield periodically
          if (i % 100 === 0) await new Promise(r => setTimeout(r, 0));
        }
        controller.close();
      }
    });
    
    return new Response(stream);
  },
};
```

### Cache Aggressively

```typescript
// Cache static assets at edge
if (url.pathname.match(/\.(jpg|png|css|js)$/)) {
  const cache = caches.default;
  let response = await cache.match(request);
  
  if (!response) {
    response = await fetch(request);
    response = new Response(response.body, response);
    response.headers.set('Cache-Control', 'public, max-age=31536000');
    ctx.waitUntil(cache.put(request, response.clone()));
  }
  
  return response;
}
```

### Use TransformStreams

```typescript
// Process response without buffering
const response = await fetch(request);

const transformed = response.body
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(new TransformStream({
    transform(chunk, controller) {
      controller.enqueue(chunk.toUpperCase());
    }
  }))
  .pipeThrough(new TextEncoderStream());

return new Response(transformed);
```

## TypeScript Configuration

Cloudflare provides official types:

```bash
npm install -D @cloudflare/workers-types
```

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "lib": ["ES2022"],
    "types": ["@cloudflare/workers-types"]
  }
}
```

Define environment interface:

```typescript
interface Env {
  MY_KV: KVNamespace;
  MY_VAR: string;
  MY_SECRET: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Full type safety
    const value = await env.MY_KV.get('key');
    return new Response(value);
  },
};
```

## Testing Strategies

### Unit Tests

```typescript
// test/worker.test.ts
import { describe, it, expect } from 'vitest';
import worker from '../src/index';

describe('Worker', () => {
  it('returns 200 for GET /', async () => {
    const request = new Request('http://localhost/');
    const env = {
      MY_VAR: 'test-value'
    };
    const ctx = {
      waitUntil: () => {},
      passThroughOnException: () => {}
    };
    
    const response = await worker.fetch(request, env, ctx);
    expect(response.status).toBe(200);
  });
});
```

### Integration Tests with Wrangler

```bash
# Start dev server
npx wrangler dev --port 8787 &

# Run integration tests
npm test

# Stop dev server
```

### Use Miniflare for Local Testing

```typescript
import { Miniflare } from 'miniflare';

const mf = new Miniflare({
  script: `
    export default {
      async fetch(request, env) {
        return new Response('Hello ' + env.NAME);
      }
    }
  `,
  bindings: { NAME: 'World' }
});

const response = await mf.dispatchFetch('http://localhost');
console.log(await response.text()); // "Hello World"
```

## Deployment Best Practices

### Environments

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-01-01",
  
  "vars": {
    "ENVIRONMENT": "production"
  },
  
  "env": {
    "staging": {
      "name": "my-worker-staging",
      "vars": {
        "ENVIRONMENT": "staging"
      }
    },
    "dev": {
      "name": "my-worker-dev",
      "vars": {
        "ENVIRONMENT": "development"
      }
    }
  }
}
```

Deploy commands:
```bash
npx wrangler deploy              # production
npx wrangler deploy --env staging
npx wrangler deploy --env dev
```

### Versioned Deployments

```bash
# Upload version (doesn't activate)
npx wrangler versions upload --message "Add new feature"

# List versions
npx wrangler versions list

# Deploy specific version
npx wrangler versions deploy <version-id>

# Rollback
npx wrangler rollback
```

### CI/CD Pattern

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      
      - run: npm ci
      - run: npm test
      
      - name: Deploy
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

## Debugging & Observability

### Console Logging

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    console.log('Request received:', request.method, request.url);
    console.error('Error condition:', { details: 'info' });
    console.warn('Warning:', 'something unexpected');
    
    return new Response('OK');
  },
};
```

View logs:
```bash
npx wrangler tail
npx wrangler tail --status error
```

### Workers Logs (Built-in Observability)

Automatic log persistence when enabled:

```jsonc
{
  "observability": {
    "enabled": true,
    "head_sampling_rate": 0.1  // 10% sampling
  }
}
```

Access via dashboard or GraphQL API.

### Custom Analytics

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const start = Date.now();
    
    try {
      const response = await handleRequest(request, env);
      
      // Log metrics (don't block response)
      ctx.waitUntil(
        env.ANALYTICS.writeDataPoint({
          doubles: [Date.now() - start],
          blobs: [request.url, response.status.toString()]
        })
      );
      
      return response;
    } catch (error) {
      ctx.waitUntil(logError(error, env));
      throw error;
    }
  },
};
```

### Performance Monitoring

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const timing = {
      start: Date.now(),
      dbQuery: 0,
      apiCall: 0
    };
    
    const dbStart = Date.now();
    await env.DB.prepare('SELECT 1').first();
    timing.dbQuery = Date.now() - dbStart;
    
    const apiStart = Date.now();
    await fetch('https://api.example.com/data');
    timing.apiCall = Date.now() - apiStart;
    
    return new Response(JSON.stringify({
      data: 'result',
      _timing: timing
    }));
  },
};
```

## Common Gotchas

### 1. CPU Time Limits

Standard plan: 10ms CPU time (30ms on Unbound)
- Use `waitUntil()` for background work
- Offload heavy compute to Durable Objects
- Consider Workers AI for ML workloads

### 2. No Persistent State in Worker

Workers are stateless between requests:

```typescript
// ❌ BAD - resets between requests
let counter = 0;

export default {
  async fetch() {
    counter++; // Unreliable!
    return new Response(String(counter));
  },
};

// ✅ GOOD - use KV, D1, or Durable Objects
export default {
  async fetch(request, env) {
    const current = await env.KV.get('counter') || '0';
    const next = parseInt(current) + 1;
    await env.KV.put('counter', String(next));
    return new Response(String(next));
  },
};
```

### 3. Response Bodies Are Streams

```typescript
// ❌ BAD
const response = await fetch(url);
await logBody(response.text());  // First read
return response;  // Body already consumed!

// ✅ GOOD
const response = await fetch(url);
const text = await response.text();
await logBody(text);
return new Response(text, response);  // New response
```

### 4. No Node.js Built-ins (by default)

Workers use web standards, not Node APIs:

```typescript
// ❌ BAD
import fs from 'fs';  // Not available

// ✅ GOOD - use Workers APIs
const data = await env.MY_BUCKET.get('file.txt');

// OR enable Node.js compat
{
  "compatibility_flags": ["nodejs_compat_v2"]
}
```

### 5. Fetch in Global Scope Forbidden

```typescript
// ❌ BAD
const config = await fetch('/config.json');  // Error!

export default {
  async fetch() {
    return new Response('OK');
  },
};

// ✅ GOOD
export default {
  async fetch() {
    const config = await fetch('/config.json');  // OK in handler
    return new Response('OK');
  },
};
```

## Advanced Patterns

### Service Bindings (Worker-to-Worker RPC)

```typescript
// service-a/src/index.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Call service-b without HTTP
    return env.SERVICE_B.fetch(request);
  },
};
```

Config:

```jsonc
{
  "services": [
    {
      "binding": "SERVICE_B",
      "service": "service-b"
    }
  ]
}
```

**Benefits**: Zero latency, no internet round-trip, type-safe with RPC.

### Smart Placement

Auto-locate compute near data sources:

```jsonc
{
  "placement": {
    "mode": "smart"
  }
}
```

Workers automatically run near D1/Hyperdrive databases.

### Tail Workers

Consume logs from other Workers:

```typescript
export default {
  async tail(events: TraceItem[], env: Env): Promise<void> {
    for (const event of events) {
      if (event.outcome === 'exception') {
        await env.ERRORS_QUEUE.send({
          error: event.exceptions[0],
          timestamp: event.eventTimestamp
        });
      }
    }
  },
};
```

### Gradual Rollouts

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const rolloutPercent = 10;
    const userId = request.headers.get('X-User-ID');
    
    // Consistent hashing for stable rollout
    const hash = await crypto.subtle.digest(
      'SHA-256',
      new TextEncoder().encode(userId)
    );
    const bucket = new Uint8Array(hash)[0] % 100;
    
    if (bucket < rolloutPercent) {
      return newFeatureHandler(request, env);
    }
    
    return oldFeatureHandler(request, env);
  },
};
```

## Framework Integration

### Hono (Recommended)

```typescript
import { Hono } from 'hono';

const app = new Hono<{ Bindings: Env }>();

app.get('/', (c) => c.text('Hello!'));

app.get('/api/users/:id', async (c) => {
  const id = c.req.param('id');
  const user = await c.env.DB.prepare('SELECT * FROM users WHERE id = ?')
    .bind(id)
    .first();
  return c.json(user);
});

export default app;
```

### Next.js (with @cloudflare/next-on-pages)

```bash
npm install -D @cloudflare/next-on-pages
npx @cloudflare/next-on-pages
```

Deploy static & dynamic Next.js apps to Workers.

### Remix (with Wrangler)

```typescript
// server.ts
import { createRequestHandler } from '@remix-run/cloudflare';

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext) {
    const handler = createRequestHandler({
      build: await import('./build/index.js'),
    });
    return handler(request, { env, ctx });
  },
};
```

## Security Best Practices

### 1. Use Secrets for Sensitive Data

```bash
npx wrangler secret put DATABASE_URL
npx wrangler secret put API_KEY
```

Never commit secrets to version control.

### 2. Validate Input

```typescript
function validateEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

export default {
  async fetch(request: Request): Promise<Response> {
    const body = await request.json();
    
    if (!validateEmail(body.email)) {
      return new Response('Invalid email', { status: 400 });
    }
    
    // Process valid input
    return new Response('OK');
  },
};
```

### 3. Set Security Headers

```typescript
const securityHeaders = {
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': 'geolocation=(), microphone=(), camera=()',
  'Content-Security-Policy': "default-src 'self'",
};

export default {
  async fetch(request: Request): Promise<Response> {
    const response = new Response('OK');
    
    Object.entries(securityHeaders).forEach(([key, value]) => {
      response.headers.set(key, value);
    });
    
    return response;
  },
};
```

### 4. Rate Limiting

Implement per-IP or per-user rate limits with Durable Objects or KV.

### 5. HMAC Request Signing

```typescript
async function verifySignature(
  payload: string,
  signature: string,
  secret: string
): Promise<boolean> {
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['verify']
  );
  
  const signatureBytes = Uint8Array.from(
    atob(signature),
    c => c.charCodeAt(0)
  );
  
  return crypto.subtle.verify(
    'HMAC',
    key,
    signatureBytes,
    encoder.encode(payload)
  );
}
```

## Resources & References

**Official Docs**: https://developers.cloudflare.com/workers/  
**Runtime APIs**: https://developers.cloudflare.com/workers/runtime-apis/  
**Examples**: https://developers.cloudflare.com/workers/examples/  
**Discord**: https://discord.cloudflare.com  
**GitHub**: https://github.com/cloudflare/workers-sdk

## Quick Reference

### Handler Signatures

```typescript
// HTTP requests
async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response>

// Cron triggers
async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext): Promise<void>

// Queue consumer
async queue(batch: MessageBatch, env: Env, ctx: ExecutionContext): Promise<void>

// Tail consumer
async tail(events: TraceItem[], env: Env, ctx: ExecutionContext): Promise<void>
```

### Common Wrangler Commands

```bash
wrangler dev                    # Local development
wrangler deploy                 # Deploy to production
wrangler tail                   # Stream logs
wrangler kv:key put KEY VALUE   # Write to KV
wrangler d1 execute DB --command "SELECT 1"
wrangler secret put SECRET_NAME
wrangler versions upload        # Create version
wrangler rollback               # Rollback deployment
```

### Binding Types

| Binding | Config Key | Type | Use Case |
|---------|-----------|------|----------|
| KV | `kv_namespaces` | `KVNamespace` | Key-value cache |
| R2 | `r2_buckets` | `R2Bucket` | Object storage |
| D1 | `d1_databases` | `D1Database` | SQL database |
| Durable Objects | `durable_objects.bindings` | Custom | Stateful coordination |
| Queue | `queues.producers` | `Queue` | Message queue |
| Service | `services` | `Fetcher` | Worker-to-worker |
| Analytics | `analytics_engine_datasets` | `AnalyticsEngineDataset` | Custom analytics |

### Response Helpers

```typescript
// JSON
new Response(JSON.stringify(data), {
  headers: { 'Content-Type': 'application/json' }
});

// Redirect
Response.redirect('https://example.com', 302);

// Error
new Response('Not Found', { status: 404 });

// Stream
new Response(stream, {
  headers: { 'Content-Type': 'text/plain' }
});
```

---

**Remember**: Workers are serverless edge compute - design for stateless, fast, global execution.
