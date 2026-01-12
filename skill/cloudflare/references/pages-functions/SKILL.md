# Cloudflare Pages Functions Skill

Expert guidance for building serverless functions on Cloudflare Pages.

## Overview

Pages Functions enable full-stack development on Cloudflare Pages by executing server-side code with Cloudflare Workers runtime. This skill covers file-based routing, middleware, bindings, configuration, and best practices.

## Core Concepts

### File-Based Routing

Functions use file-based routing via the `/functions` directory:

```
/functions
  ├── index.js              → example.com/
  ├── api.js                → example.com/api
  ├── users/
  │   ├── index.js          → example.com/users/
  │   ├── [user].js         → example.com/users/:user
  │   └── [[catchall]].js   → example.com/users/*
  └── _middleware.js        → runs on all routes
```

**Key Routing Rules:**
- `index.js` maps to directory root
- Trailing slash optional: `/foo` and `/foo/` both route to `/functions/foo.js` or `/functions/foo/index.js`
- More specific routes take precedence
- Falls back to static assets if no function matches

### Dynamic Routes

**Single path segment** (one set of brackets):
```javascript
// /functions/users/[user].js
export function onRequest(context) {
  return new Response(`Hello ${context.params.user}`);
}
// Matches: /users/nevi, /users/daniel
// context.params.user is a string
```

**Multi-path segments** (double brackets):
```javascript
// /functions/users/[[catchall]].js
export function onRequest(context) {
  return new Response(JSON.stringify(context.params.catchall));
}
// Matches: /users/nevi/foobar, /users/daniel/xyz/123
// context.params.catchall is an array: ["daniel", "xyz", "123"]
```

## Request Handlers

### HTTP Method Handlers

Export named functions for specific HTTP methods:

```typescript
interface EventContext<Env = any> {
  request: Request;
  functionPath: string;
  waitUntil(promise: Promise<any>): void;
  passThroughOnException(): void;
  next(input?: Request | string, init?: RequestInit): Promise<Response>;
  env: Env;
  params: Record<string, string | string[]>;
  data: any;
}

// Generic handler (runs if no specific method handler exists)
export async function onRequest(context: EventContext): Promise<Response> {
  return new Response('Any method');
}

// Method-specific handlers (take precedence)
export async function onRequestGet(context: EventContext): Promise<Response> {
  return new Response('GET request');
}

export async function onRequestPost(context: EventContext): Promise<Response> {
  const body = await context.request.json();
  return Response.json({ received: body });
}

export async function onRequestPut(context: EventContext): Promise<Response> { }
export async function onRequestPatch(context: EventContext): Promise<Response> { }
export async function onRequestDelete(context: EventContext): Promise<Response> { }
export async function onRequestHead(context: EventContext): Promise<Response> { }
export async function onRequestOptions(context: EventContext): Promise<Response> { }
```

### Context Object Properties

- **`request`**: Incoming `Request` object
- **`functionPath`**: Path of the request
- **`waitUntil(promise)`**: Schedule background work
- **`passThroughOnException()`**: Pass to next handler on error (not in advanced mode)
- **`next()`**: Pass request to next function or asset server
- **`env`**: Environment variables, secrets, and bindings
- **`params`**: Dynamic route parameters
- **`data`**: Shared data between functions (middleware)

## Middleware

Middleware runs before request handlers. Define in `_middleware.js`:

```javascript
// functions/_middleware.js - runs on all routes
// functions/users/_middleware.js - runs on /users/* routes

// Single middleware
export async function onRequest(context) {
  try {
    return await context.next();
  } catch (err) {
    return new Response(`${err.message}\n${err.stack}`, { status: 500 });
  }
}

// Chained middleware (runs in array order)
async function errorHandling(context) {
  try {
    return await context.next();
  } catch (err) {
    return new Response(`Error: ${err.message}`, { status: 500 });
  }
}

function authentication(context) {
  const token = context.request.headers.get('authorization');
  if (!token) {
    return new Response('Unauthorized', { status: 403 });
  }
  return context.next();
}

function logging(context) {
  console.log(`[${new Date().toISOString()}] ${context.request.method} ${context.request.url}`);
  return context.next();
}

export const onRequest = [errorHandling, authentication, logging];
```

**Middleware Best Practices:**
- First middleware should be error handling (wraps others)
- Use `context.next()` to pass control
- Can modify `context.data` to share state
- Use method-specific exports if needed

## Bindings

Bindings enable interaction with Cloudflare resources. Access via `context.env`.

### KV (Key-Value Storage)

```typescript
interface Env {
  TODO_LIST: KVNamespace;
}

export const onRequest: PagesFunction<Env> = async (context) => {
  // Get value
  const task = await context.env.TODO_LIST.get('Task:123');
  
  // Get with metadata
  const value = await context.env.TODO_LIST.get('key', { type: 'json' });
  
  // Put value
  await context.env.TODO_LIST.put('Task:123', 'Buy milk');
  
  // Put with expiration
  await context.env.TODO_LIST.put('session', data, { expirationTtl: 3600 });
  
  // Delete
  await context.env.TODO_LIST.delete('Task:123');
  
  // List keys
  const keys = await context.env.TODO_LIST.list({ prefix: 'Task:' });
  
  return new Response(task);
};
```

### D1 (SQL Database)

```typescript
interface Env {
  DB: D1Database;
}

export const onRequest: PagesFunction<Env> = async (context) => {
  // Prepared statement
  const ps = context.env.DB.prepare('SELECT * FROM users WHERE id = ?').bind(123);
  const result = await ps.first();
  
  // Multiple queries
  const data = await context.env.DB.batch([
    context.env.DB.prepare('SELECT * FROM users'),
    context.env.DB.prepare('SELECT * FROM posts')
  ]);
  
  // Raw query (avoid - use prepared statements)
  const raw = await context.env.DB.prepare('SELECT * FROM users').all();
  
  return Response.json(result);
};
```

### R2 (Object Storage)

```typescript
interface Env {
  BUCKET: R2Bucket;
}

export const onRequest: PagesFunction<Env> = async (context) => {
  const url = new URL(context.request.url);
  const key = url.pathname.slice(1); // remove leading /
  
  if (context.request.method === 'GET') {
    const obj = await context.env.BUCKET.get(key);
    if (!obj) return new Response('Not found', { status: 404 });
    
    return new Response(obj.body, {
      headers: {
        'Content-Type': obj.httpMetadata?.contentType || 'application/octet-stream',
        'ETag': obj.httpEtag,
      }
    });
  }
  
  if (context.request.method === 'PUT') {
    await context.env.BUCKET.put(key, context.request.body, {
      httpMetadata: {
        contentType: context.request.headers.get('content-type') || 'application/octet-stream'
      }
    });
    return new Response('Uploaded', { status: 201 });
  }
  
  if (context.request.method === 'DELETE') {
    await context.env.BUCKET.delete(key);
    return new Response('Deleted', { status: 204 });
  }
  
  return new Response('Method not allowed', { status: 405 });
};
```

### Durable Objects

```typescript
interface Env {
  COUNTER: DurableObjectNamespace;
}

export const onRequest: PagesFunction<Env> = async (context) => {
  // Get Durable Object by name
  const id = context.env.COUNTER.idFromName('global-counter');
  
  // Get Durable Object by unique ID
  const uniqueId = context.env.COUNTER.newUniqueId();
  
  // Get stub
  const stub = context.env.COUNTER.get(id);
  
  // Forward request to DO
  return stub.fetch(context.request);
};
```

### Workers AI

```typescript
interface Env {
  AI: Ai;
}

export const onRequest: PagesFunction<Env> = async (context) => {
  const input = { 
    prompt: 'What is the origin of the phrase Hello, World?' 
  };
  
  const answer = await context.env.AI.run(
    '@cf/meta/llama-3.1-8b-instruct',
    input
  );
  
  return Response.json(answer);
};
```

### Service Bindings

```typescript
interface Env {
  AUTH_SERVICE: Fetcher;
}

export const onRequest: PagesFunction<Env> = async (context) => {
  // Call another Worker
  const response = await context.env.AUTH_SERVICE.fetch(context.request);
  return response;
  
  // Or create custom request
  const customRequest = new Request('https://internal/verify', {
    method: 'POST',
    body: JSON.stringify({ token: 'xyz' })
  });
  return context.env.AUTH_SERVICE.fetch(customRequest);
};
```

### Environment Variables & Secrets

```typescript
interface Env {
  API_KEY: string;
  ENVIRONMENT: string;
  DATABASE_URL: string;
}

export const onRequest: PagesFunction<Env> = async (context) => {
  const apiKey = context.env.API_KEY;
  const isProduction = context.env.ENVIRONMENT === 'production';
  
  if (isProduction) {
    // Use production logic
  }
  
  return new Response('OK');
};
```

## Configuration (wrangler.json / wrangler.toml)

### Basic Configuration

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-pages-app",
  "pages_build_output_dir": "./dist",
  "compatibility_date": "2024-01-15",
  "compatibility_flags": ["nodejs_compat"]
}
```

```toml
name = "my-pages-app"
pages_build_output_dir = "./dist"
compatibility_date = "2024-01-15"
compatibility_flags = ["nodejs_compat"]
```

### Bindings Configuration

```jsonc
{
  "name": "my-app",
  "pages_build_output_dir": "./dist",
  
  "vars": {
    "API_URL": "https://api.example.com",
    "ENVIRONMENT": "production"
  },
  
  "kv_namespaces": [
    { "binding": "KV", "id": "abc123" }
  ],
  
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "production-db",
      "database_id": "xyz789"
    }
  ],
  
  "r2_buckets": [
    { "binding": "BUCKET", "bucket_name": "my-bucket" }
  ],
  
  "durable_objects": {
    "bindings": [
      {
        "name": "COUNTER",
        "class_name": "Counter",
        "script_name": "counter-worker"
      }
    ]
  },
  
  "services": [
    { "binding": "AUTH", "service": "auth-worker" }
  ],
  
  "ai": {
    "binding": "AI"
  },
  
  "vectorize": [
    {
      "binding": "VECTORIZE",
      "index_name": "my-index"
    }
  ],
  
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "hyperdrive-id"
    }
  ],
  
  "analytics_engine_datasets": [
    { "binding": "ANALYTICS" }
  ]
}
```

### Environment-Specific Configuration

```jsonc
{
  "name": "my-app",
  "pages_build_output_dir": "./dist",
  
  "kv_namespaces": [
    { "binding": "KV", "id": "local-kv-id" }
  ],
  
  "vars": {
    "API_URL": "http://localhost:8787"
  },
  
  "env": {
    "preview": {
      "kv_namespaces": [
        { "binding": "KV", "id": "preview-kv-id" }
      ],
      "vars": {
        "API_URL": "https://preview.example.com"
      }
    },
    "production": {
      "kv_namespaces": [
        { "binding": "KV", "id": "production-kv-id" }
      ],
      "vars": {
        "API_URL": "https://api.example.com"
      }
    }
  }
}
```

**Environment Inheritance Rules:**
- Top-level config applies to local dev
- `env.preview` overrides for preview deployments
- `env.production` overrides for production
- **Non-inheritable keys**: If ANY non-inheritable key is overridden in env, ALL must be defined
- **Non-inheritable keys**: `vars`, `kv_namespaces`, `d1_databases`, `r2_buckets`, `durable_objects`, `services`, `ai`, etc.

## Local Development

### Using Wrangler

```bash
# Start local dev server
npx wrangler pages dev ./dist

# With bindings via CLI
npx wrangler pages dev ./dist \
  --kv=KV \
  --d1=DB=database-id \
  --r2=BUCKET \
  --binding=API_KEY=secret123

# With Durable Objects
# Terminal 1: Run DO worker
cd do-worker && npx wrangler dev

# Terminal 2: Run Pages with DO binding
cd pages-project && npx wrangler pages dev ./dist \
  --do COUNTER=CounterClass@do-worker

# With Service bindings
# Terminal 1: Run service worker
cd auth-worker && npx wrangler dev

# Terminal 2: Run Pages with service binding
cd pages-project && npx wrangler pages dev ./dist \
  --service AUTH=auth-worker
```

### Local Secrets (.dev.vars)

```bash
# .dev.vars (DO NOT COMMIT)
SECRET_KEY="my-secret-value"
API_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9"
DATABASE_URL="postgresql://user:pass@localhost:5432/db"
```

**Important:**
- Use `.dev.vars` for secrets (encrypted in production)
- Use `vars` in wrangler.json for non-sensitive config
- Add `.dev.vars*` to `.gitignore`
- Environment-specific: `.dev.vars.preview`, `.dev.vars.production`

## Advanced Mode

For complex Workers, use `_worker.js` instead of `/functions`:

```typescript
// _worker.js (in build output directory)
interface Env {
  ASSETS: Fetcher;
  KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // Custom API logic
    if (url.pathname.startsWith('/api/')) {
      return new Response('API response');
    }
    
    // Serve static assets
    return env.ASSETS.fetch(request);
  }
} satisfies ExportedHandler<Env>;
```

**When to use Advanced Mode:**
- Existing Worker code too complex for file-based routing
- Need full control over routing logic
- Framework-generated Workers (Next.js, SvelteKit, etc.)

**Important:**
- Must use Module Worker syntax
- `/functions` directory completely ignored
- Must manually call `env.ASSETS.fetch()` for static assets
- `passThroughOnException()` not available

## Common Patterns

### Authentication Middleware

```typescript
// functions/_middleware.ts
async function authMiddleware(context: EventContext<Env>) {
  const token = context.request.headers.get('authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return new Response('Unauthorized', { status: 401 });
  }
  
  try {
    const session = await context.env.KV.get(`session:${token}`);
    if (!session) {
      return new Response('Invalid token', { status: 401 });
    }
    
    // Store user in context.data for downstream handlers
    context.data.user = JSON.parse(session);
    return context.next();
  } catch (err) {
    return new Response('Auth error', { status: 500 });
  }
}

export const onRequest = [authMiddleware];
```

### CORS Handling

```typescript
const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization',
  'Access-Control-Max-Age': '86400',
};

export async function onRequestOptions(context: EventContext) {
  return new Response(null, { headers: corsHeaders });
}

export async function onRequest(context: EventContext) {
  const response = await context.next();
  
  // Add CORS headers to response
  Object.entries(corsHeaders).forEach(([key, value]) => {
    response.headers.set(key, value);
  });
  
  return response;
}
```

### Error Handling

```typescript
// functions/_middleware.ts
async function errorHandler(context: EventContext) {
  try {
    return await context.next();
  } catch (err) {
    console.error('Error:', err);
    
    const isDev = context.env.ENVIRONMENT === 'development';
    
    return new Response(
      JSON.stringify({
        error: isDev ? err.message : 'Internal server error',
        stack: isDev ? err.stack : undefined,
      }),
      {
        status: 500,
        headers: { 'Content-Type': 'application/json' }
      }
    );
  }
}

export const onRequest = [errorHandler];
```

### Rate Limiting

```typescript
// functions/api/_middleware.ts
interface Env {
  RATE_LIMIT: KVNamespace;
}

async function rateLimitMiddleware(context: EventContext<Env>) {
  const clientIP = context.request.headers.get('CF-Connecting-IP') || 'unknown';
  const key = `ratelimit:${clientIP}`;
  
  const current = await context.env.RATE_LIMIT.get(key);
  const count = current ? parseInt(current) : 0;
  
  if (count >= 100) {
    return new Response('Rate limit exceeded', { status: 429 });
  }
  
  // Increment counter with 1 hour expiration
  await context.env.RATE_LIMIT.put(key, (count + 1).toString(), { expirationTtl: 3600 });
  
  return context.next();
}

export const onRequest = [rateLimitMiddleware];
```

### Form Handling

```typescript
export async function onRequestPost(context: EventContext) {
  const contentType = context.request.headers.get('content-type') || '';
  
  if (contentType.includes('application/json')) {
    const data = await context.request.json();
    return Response.json({ received: data });
  }
  
  if (contentType.includes('application/x-www-form-urlencoded')) {
    const formData = await context.request.formData();
    const data = Object.fromEntries(formData);
    return Response.json({ received: data });
  }
  
  if (contentType.includes('multipart/form-data')) {
    const formData = await context.request.formData();
    const file = formData.get('file') as File;
    
    if (file) {
      // Store in R2
      await context.env.BUCKET.put(file.name, file.stream());
      return Response.json({ uploaded: file.name });
    }
  }
  
  return new Response('Unsupported content type', { status: 400 });
}
```

### Caching

```typescript
export async function onRequest(context: EventContext) {
  const cache = caches.default;
  const cacheKey = new Request(context.request.url, context.request);
  
  // Try cache first
  let response = await cache.match(cacheKey);
  
  if (!response) {
    // Generate response
    response = new Response('Hello World');
    
    // Cache for 1 hour
    response.headers.set('Cache-Control', 'public, max-age=3600');
    context.waitUntil(cache.put(cacheKey, response.clone()));
  }
  
  return response;
}
```

### Redirects

```typescript
export async function onRequest(context: EventContext) {
  const url = new URL(context.request.url);
  
  // Redirect old paths
  if (url.pathname === '/old-page') {
    return Response.redirect(`${url.origin}/new-page`, 301);
  }
  
  // Force HTTPS
  if (url.protocol === 'http:') {
    url.protocol = 'https:';
    return Response.redirect(url.toString(), 301);
  }
  
  return context.next();
}
```

### Serving Assets

```typescript
export async function onRequest(context: EventContext) {
  // Custom logic for certain paths
  if (context.request.url.includes('/api/')) {
    return new Response('API response');
  }
  
  // Serve static assets
  const response = await context.env.ASSETS.fetch(context.request);
  
  // Add custom headers to asset responses
  const newResponse = new Response(response.body, response);
  newResponse.headers.set('X-Custom-Header', 'value');
  
  return newResponse;
}
```

## TypeScript Support

### Type Definitions

```typescript
import type { PagesFunction, EventContext } from '@cloudflare/workers-types';

// Define environment interface
interface Env {
  KV: KVNamespace;
  DB: D1Database;
  BUCKET: R2Bucket;
  API_KEY: string;
}

// Typed handler
export const onRequest: PagesFunction<Env> = async (context) => {
  const value = await context.env.KV.get('key');
  return new Response(value);
};

// Or with explicit context type
export async function onRequest(context: EventContext<Env>): Promise<Response> {
  return new Response('OK');
}
```

### Installing Types

```bash
npm install -D @cloudflare/workers-types
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "ES2022",
    "lib": ["ES2021"],
    "types": ["@cloudflare/workers-types"]
  }
}
```

## Deployment

### Via Git Integration

```bash
# Commit wrangler.json with your code
git add wrangler.json functions/
git commit -m "Add Pages Functions"
git push origin main
# Pages automatically deploys on push
```

### Via Wrangler CLI

```bash
# Deploy to production
npx wrangler pages deploy ./dist

# Deploy to preview
npx wrangler pages deploy ./dist --branch preview

# Deploy specific branch
npx wrangler pages deploy ./dist --branch feature-xyz
```

### Download Existing Config

```bash
# Download dashboard config to wrangler.json
npx wrangler pages download config my-project-name
```

## Performance Best Practices

1. **Minimize Cold Starts:**
   - Keep dependencies small
   - Use dynamic imports for heavy code
   - Avoid large initialization in global scope

2. **Use Appropriate Bindings:**
   - KV for infrequent reads, global data
   - D1 for relational data
   - R2 for large files
   - Durable Objects for coordination/state

3. **Leverage Edge Caching:**
   - Set appropriate `Cache-Control` headers
   - Use `caches.default` API
   - Cache static API responses

4. **Optimize Database Queries:**
   - Use prepared statements
   - Batch operations when possible
   - Index frequently queried columns

5. **Handle Errors Gracefully:**
   - Implement error boundaries
   - Use `passThroughOnException()` for fallback
   - Log errors to external service

## Security Best Practices

1. **Environment Variables:**
   - Never commit secrets to git
   - Use secrets (encrypted) not vars for sensitive data
   - Validate environment on startup

2. **Input Validation:**
   - Validate all user input
   - Sanitize data before database operations
   - Use prepared statements (no SQL injection)

3. **Authentication:**
   - Implement auth middleware
   - Use JWT or session tokens
   - Store sessions in KV with expiration

4. **CORS:**
   - Set appropriate CORS headers
   - Validate origins in production
   - Use credentials cautiously

5. **Rate Limiting:**
   - Implement per-IP rate limits
   - Use KV for rate limit counters
   - Return 429 status code

## Debugging

### Console Logging

```typescript
export async function onRequest(context: EventContext) {
  console.log('Request:', context.request.method, context.request.url);
  console.log('Headers:', Object.fromEntries(context.request.headers));
  
  const response = await context.next();
  
  console.log('Response status:', response.status);
  return response;
}
```

### Wrangler Tail

```bash
# Stream real-time logs
npx wrangler pages deployment tail

# Filter logs
npx wrangler pages deployment tail --status error
```

### Source Maps

```jsonc
// wrangler.json
{
  "upload_source_maps": true
}
```

## Pricing & Limits

- **Requests:** 100,000 free/day, then $0.50/million
- **CPU Time:** 10ms per request (Free), 50ms (Paid)
- **Memory:** 128 MB
- **Script Size:** 10 MB after compression
- **Environment Variables:** 5 KB per variable, 64 variables max
- **KV:** 100,000 reads/day free
- **D1:** 100,000 rows read/day free
- **R2:** No egress fees

## Common Issues

### Functions Not Invoking

**Problem:** All requests serve static assets, functions never run.

**Solution:**
- Verify `/functions` directory in correct location (project root)
- Check `pages_build_output_dir` in wrangler.json
- Ensure function files have `.js` or `.ts` extension
- Check `_routes.json` isn't excluding your function paths

### Binding Not Available

**Problem:** `context.env.MY_BINDING is undefined`

**Solution:**
- Verify binding configured in wrangler.json or dashboard
- Check binding name matches exactly (case-sensitive)
- For local dev, pass `--kv`, `--d1`, etc. flags OR configure in wrangler.json
- Redeploy after changing bindings

### TypeScript Errors

**Problem:** Type errors for `context.env`

**Solution:**
```typescript
interface Env {
  MY_BINDING: KVNamespace;
}

export const onRequest: PagesFunction<Env> = async (context) => {
  // Now context.env.MY_BINDING is typed
};
```

### Middleware Not Running

**Problem:** `_middleware.js` not executing

**Solution:**
- Ensure file named exactly `_middleware.js`
- Check middleware in correct directory for desired routes
- Verify `onRequest` or method-specific handler exported
- Use `context.next()` to pass to next handler

### Environment Variables Not Available

**Problem:** `context.env.VAR_NAME is undefined`

**Solution:**
- Verify `vars` defined in wrangler.json
- For secrets, ensure `.dev.vars` file exists locally
- For production, set in dashboard or wrangler.json
- Redeploy after changes

## Migration Guide

### From Workers to Pages Functions

1. **Move Worker code:**
   - Copy Worker code to `/functions` directory OR
   - Use Advanced Mode with `_worker.js`

2. **Update exports:**
   ```typescript
   // Worker
   export default {
     fetch(request, env) { }
   }
   
   // Pages Function
   export function onRequest(context) {
     const { request, env } = context;
   }
   ```

3. **Handle static assets:**
   ```typescript
   // In _worker.js
   return env.ASSETS.fetch(request);
   ```

### From Other Platforms

1. **Understand routing:**
   - `/functions/api/users.js` → `/api/users`
   - Dynamic routes use `[param]` not `:param`

2. **Adapt request handlers:**
   ```typescript
   // Express
   app.get('/api/users', (req, res) => {
     res.json({ users: [] });
   });
   
   // Pages Functions
   export function onRequestGet(context) {
     return Response.json({ users: [] });
   }
   ```

3. **Replace dependencies:**
   - Use Workers APIs (fetch, crypto, etc.)
   - Some Node.js APIs available with `nodejs_compat` flag
   - Consider polyfills for missing APIs

## Additional Resources

- [Official Docs](https://developers.cloudflare.com/pages/functions/)
- [Workers Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)
- [Example Projects](https://github.com/cloudflare/pages-example-projects)
- [Discord Community](https://discord.gg/cloudflaredev)

## Quick Reference

### File Structure
```
project/
├── functions/
│   ├── _middleware.js        # Global middleware
│   ├── index.js               # GET /
│   ├── api/
│   │   ├── _middleware.js     # API middleware
│   │   ├── users.js           # GET /api/users
│   │   └── users/
│   │       └── [id].js        # GET /api/users/:id
├── public/                    # Static assets
├── wrangler.json              # Configuration
├── .dev.vars                  # Local secrets (gitignored)
└── package.json
```

### Handler Template
```typescript
interface Env {
  // Define your bindings
}

export const onRequest: PagesFunction<Env> = async (context) => {
  const { request, env, params } = context;
  
  // Your logic here
  
  return new Response('OK');
};
```

### Common Commands
```bash
# Local dev
npx wrangler pages dev ./dist

# Deploy
npx wrangler pages deploy ./dist

# Logs
npx wrangler pages deployment tail

# Download config
npx wrangler pages download config my-project
```
