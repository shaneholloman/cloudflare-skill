# Cloudflare Pages Skill

Expert guidance for building and deploying full-stack applications with Cloudflare Pages.

## Overview

Cloudflare Pages is a JAMstack platform for deploying static sites and full-stack applications to Cloudflare's global network. Unlike Workers, Pages is specifically designed for frontend frameworks and static site generators, with built-in Git integration, preview deployments, and serverless functions via Pages Functions.

**Key Differentiators from Workers:**
- Git-based deployment workflow with automatic builds
- Preview deployments for every branch/PR
- Static asset serving with intelligent caching
- File-based routing for Functions (vs single Worker script)
- Framework-specific optimizations and guides
- Built-in redirects and headers configuration via files

## Deployment Methods

### 1. Git Integration (Recommended for Production)

Connect to GitHub/GitLab for automatic deployments:

```bash
# Setup via Cloudflare dashboard:
# 1. Workers & Pages > Create application > Pages > Connect to Git
# 2. Select repository and configure build settings
# 3. Every push triggers a deployment
```

**Build Configuration:**
- **Build command**: Framework-specific (e.g., `npm run build`)
- **Build output directory**: Framework-specific (e.g., `dist`, `out`, `build`)
- **Environment variables**: Set in dashboard under Settings > Environment variables

### 2. Direct Upload (wrangler pages deploy)

Deploy pre-built assets directly without Git:

```bash
# Deploy a directory
npx wrangler pages deploy ./dist --project-name=my-project

# First-time deploy (creates project)
npx wrangler pages deploy ./dist --project-name=my-project --branch=main

# Deploy to specific branch (preview)
npx wrangler pages deploy ./dist --project-name=my-project --branch=staging
```

### 3. C3 CLI (Quick Start)

Create and deploy a new project:

```bash
npm create cloudflare@latest my-pages-app
# Select Pages framework (React, Vue, Svelte, etc.)
# Automatically sets up project and deploys
```

## Pages Functions

Pages Functions bring serverless compute to Pages using the Workers runtime.

### File-Based Routing

Functions use filesystem-based routing in the `/functions` directory:

```
/functions
  /index.ts                    → example.com/
  /api
    /users.ts                  → example.com/api/users
    /users
      /[id].ts                 → example.com/api/users/:id
      /[[catchall]].ts         → example.com/api/users/* (multi-segment)
  /_middleware.ts              → Runs before all /api/* routes
```

**Routing Rules:**
- `index.ts` → maps to directory path
- `[param].ts` → single dynamic segment
- `[[param]].ts` → multi-segment wildcard (greedy match)
- More specific routes take precedence
- Falls back to static assets if no Function matches

### Request Handlers

Functions export HTTP method handlers:

```typescript
// functions/api/user.ts
import type { PagesFunction } from '@cloudflare/workers-types';

interface Env {
  DB: D1Database;
  KV: KVNamespace;
}

// Handle all methods
export const onRequest: PagesFunction<Env> = async (context) => {
  return new Response('All methods');
};

// Method-specific handlers
export const onRequestGet: PagesFunction<Env> = async (context) => {
  const { request, env, params, data } = context;
  
  const user = await env.DB.prepare(
    'SELECT * FROM users WHERE id = ?'
  ).bind(params.id).first();
  
  return Response.json(user);
};

export const onRequestPost: PagesFunction<Env> = async (context) => {
  const body = await context.request.json();
  // Handle POST
  return Response.json({ success: true });
};

// Also available: onRequestPut, onRequestPatch, onRequestDelete, 
// onRequestHead, onRequestOptions
```

### Context Object

Every handler receives a context object:

```typescript
interface EventContext<Env, Params, Data> {
  request: Request;              // Incoming HTTP request
  env: Env;                      // Bindings (KV, D1, R2, etc.)
  params: Params;                // Dynamic route parameters
  data: Data;                    // Shared data from middleware
  functionPath: string;          // Current function path
  waitUntil: (promise: Promise<any>) => void;  // Background tasks
  next: (input?: RequestInfo, init?: RequestInit) => Promise<Response>;  // Next handler
  passThroughOnException: () => void;  // Fallback on error (not in advanced mode)
}
```

### Dynamic Routes

**Single Segment (`[param]`):**

```typescript
// functions/users/[id].ts
export const onRequestGet: PagesFunction = async ({ params }) => {
  // /users/123 → params.id = "123"
  return Response.json({ userId: params.id });
};
```

**Multi-Segment (`[[catchall]]`):**

```typescript
// functions/files/[[path]].ts
export const onRequestGet: PagesFunction = async ({ params }) => {
  // /files/docs/api/v1.md → params.path = ["docs", "api", "v1.md"]
  const filePath = (params.path as string[]).join('/');
  return new Response(filePath);
};
```

### Middleware

Middleware runs before route handlers, defined in `_middleware.ts`:

```typescript
// functions/_middleware.ts
import type { PagesFunction } from '@cloudflare/workers-types';

// Single middleware
export const onRequest: PagesFunction = async (context) => {
  // Run before subsequent handlers
  const response = await context.next();
  
  // Modify response
  response.headers.set('X-Custom-Header', 'value');
  return response;
};

// Chained middleware (runs in order)
const errorHandler: PagesFunction = async (context) => {
  try {
    return await context.next();
  } catch (err) {
    return new Response(err.message, { status: 500 });
  }
};

const auth: PagesFunction = async (context) => {
  const token = context.request.headers.get('Authorization');
  if (!token) {
    return new Response('Unauthorized', { status: 401 });
  }
  
  // Attach user data for subsequent handlers
  context.data.userId = await verifyToken(token);
  return context.next();
};

const logging: PagesFunction = async (context) => {
  const start = Date.now();
  const response = await context.next();
  console.log(`${context.request.method} ${context.request.url} - ${Date.now() - start}ms`);
  return response;
};

export const onRequest = [errorHandler, logging, auth];
```

**Middleware Scope:**
- `functions/_middleware.ts` → Runs on ALL requests (including static assets)
- `functions/api/_middleware.ts` → Runs only on `/api/*` routes
- Middleware runs before any route handler in same directory or subdirectories

### TypeScript Support

Generate types from bindings:

```bash
npx wrangler types --path='./functions/types.d.ts'
```

Configure TypeScript:

```json
// functions/tsconfig.json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "lib": ["esnext"],
    "types": ["./types.d.ts"]
  }
}
```

Use generated types:

```typescript
// functions/api/data.ts
import type { PagesFunction } from '@cloudflare/workers-types';

// Env interface should be in types.d.ts (generated by wrangler types)
export const onRequestGet: PagesFunction<Env> = async ({ env }) => {
  const value = await env.KV.get('key');  // Fully typed!
  return new Response(value);
};
```

### Advanced Mode

For full Workers API access, use `_worker.js` (bypasses file-based routing):

```javascript
// functions/_worker.js
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // Custom routing logic
    if (url.pathname.startsWith('/api/')) {
      return new Response('API response');
    }
    
    // REQUIRED: Serve static assets via ASSETS binding
    return env.ASSETS.fetch(request);
  }
};
```

**When to use:**
- Need WebSocket support
- Complex routing logic beyond file-based
- Want full control over request/response flow
- Need scheduled handlers or email handlers

## Bindings

Bindings connect Functions to Cloudflare resources.

### Configuration

Define bindings in `wrangler.toml` (or `wrangler.json`):

```toml
name = "my-pages-project"
pages_build_output_dir = "./dist"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

# KV Namespace
[[kv_namespaces]]
binding = "KV"
id = "abcd1234..."

# D1 Database
[[d1_databases]]
binding = "DB"
database_id = "xxxx-xxxx-xxxx-xxxx"
database_name = "production-db"

# R2 Bucket
[[r2_buckets]]
binding = "BUCKET"
bucket_name = "my-bucket"

# Durable Object
[[durable_objects.bindings]]
name = "COUNTER"
class_name = "Counter"
script_name = "counter-worker"

# Service Binding
[[services]]
binding = "API"
service = "api-worker"

# Queue Producer
[[queues.producers]]
binding = "QUEUE"
queue = "my-queue"

# Vectorize Index
[[vectorize]]
binding = "VECTORIZE"
index_name = "my-index"

# Workers AI
[ai]
binding = "AI"

# Analytics Engine
[[analytics_engine_datasets]]
binding = "ANALYTICS"

# Environment Variables (non-sensitive)
[vars]
API_URL = "https://api.example.com"
ENVIRONMENT = "production"

# Environment-specific overrides
[env.preview]
[env.preview.vars]
API_URL = "https://staging-api.example.com"
ENVIRONMENT = "preview"

[[env.preview.kv_namespaces]]
binding = "KV"
id = "preview-namespace-id"

[env.production]
[env.production.vars]
API_URL = "https://api.example.com"
ENVIRONMENT = "production"

[[env.production.kv_namespaces]]
binding = "KV"
id = "production-namespace-id"
```

### Local Development

Use `.dev.vars` for local secrets (never commit):

```bash
# .dev.vars
SECRET_KEY="local-secret-key"
API_TOKEN="dev-token-123"
DATABASE_URL="http://localhost:5432"
```

Run dev server with bindings:

```bash
# Automatic binding from wrangler.toml
npx wrangler pages dev ./dist

# Override specific bindings
npx wrangler pages dev ./dist --kv KV --d1 DB=local-db-id

# Local D1 database
npx wrangler pages dev ./dist --d1 DB=xxxx-xxxx-xxxx-xxxx

# Local Durable Objects (requires running DO Worker separately)
# Terminal 1:
cd path/to/do-worker && npx wrangler dev
# Terminal 2:
npx wrangler pages dev ./dist --do COUNTER=Counter@do-worker

# Custom port
npx wrangler pages dev ./dist --port=3000

# Enable local persistence
npx wrangler pages dev ./dist --persist-to=./.wrangler/state/v3
```

### Usage in Functions

```typescript
// functions/api/example.ts
import type { PagesFunction } from '@cloudflare/workers-types';

interface Env {
  KV: KVNamespace;
  DB: D1Database;
  BUCKET: R2Bucket;
  QUEUE: Queue;
  AI: Ai;
}

export const onRequestGet: PagesFunction<Env> = async ({ env, request }) => {
  // KV
  const cached = await env.KV.get('key', 'json');
  await env.KV.put('key', JSON.stringify({ data: 'value' }), {
    expirationTtl: 3600
  });
  
  // D1
  const result = await env.DB.prepare(
    'SELECT * FROM users WHERE id = ?'
  ).bind(userId).first();
  
  // R2
  const object = await env.BUCKET.get('file.txt');
  const content = await object?.text();
  
  // Queue
  await env.QUEUE.send({ event: 'user.signup', userId: 123 });
  
  // Workers AI
  const aiResponse = await env.AI.run('@cf/meta/llama-2-7b-chat-int8', {
    prompt: 'Hello world'
  });
  
  return Response.json({ success: true });
};
```

### Production Secrets

Set secrets via Wrangler (encrypted at rest):

```bash
# Interactive secret entry
echo "super-secret-value" | npx wrangler pages secret put SECRET_KEY --project-name=my-project

# For specific environment
echo "prod-secret" | npx wrangler pages secret put SECRET_KEY --project-name=my-project --env=production

# List secrets
npx wrangler pages secret list --project-name=my-project

# Delete secret
npx wrangler pages secret delete SECRET_KEY --project-name=my-project
```

Access in Functions:

```typescript
export const onRequest: PagesFunction<Env> = async ({ env }) => {
  const secretKey = env.SECRET_KEY;  // Accessed like binding
  return new Response('OK');
};
```

## Static Configuration Files

### Redirects (`_redirects`)

Place in build output directory (e.g., `dist/_redirects`):

```txt
# Comments start with #

# Simple redirect (default 302)
/old-page /new-page

# Permanent redirect (301)
/old-page /new-page 301

# External redirect
/blog https://blog.example.com

# Preserve query strings (automatic)
/search /new-search  # /search?q=test → /new-search?q=test

# Splat wildcard (greedy match)
/blog/* /news/:splat 301
# /blog/2024/post → /news/2024/post

# Placeholders
/users/:id /members/:id 301
# /users/123 → /members/123

# Fragments
/page /page2#section 301

# Trailing slash normalization
/path /path/ 301

# Proxying (200 status) - serves content from different path
/api/* /api-v2/:splat 200

# Limit: 2,000 static + 100 dynamic redirects (2,100 total)
# Each line: 1,000 char max
```

**Important:**
- Redirects do NOT apply to Pages Functions routes
- Functions always take precedence over redirects
- Use `_routes.json` to exclude Function routes if redirects needed

### Headers (`_headers`)

Place in build output directory (e.g., `dist/_headers`):

```txt
# Comments start with #

# Apply headers to specific paths
/secure/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  Referrer-Policy: no-referrer
  Content-Security-Policy: default-src 'self'

# CORS for API
/api/*
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
  Access-Control-Allow-Headers: Content-Type, Authorization

# Cache static assets aggressively
/static/*
  Cache-Control: public, max-age=31536000, immutable

# Prevent preview URLs from being indexed
https://:project.pages.dev/*
  X-Robots-Tag: noindex

# Multiple environments
https://:branch.:project.pages.dev/*
  X-Robots-Tag: noindex

# Remove inherited header (prepend !)
/special-page
  ! X-Frame-Options

# Placeholders and splats
/movies/:title
  X-Movie: You are watching ":title"

# Limits: 100 rules max, 2,000 chars per line
```

**Important:**
- Headers do NOT apply to Pages Functions responses
- Functions must set headers in Response object
- Headers apply only to static assets

### Function Routes (`_routes.json`)

Control which requests invoke Functions (auto-generated for most frameworks):

```json
{
  "version": 1,
  "include": ["/*"],
  "exclude": [
    "/build/*",
    "/static/*",
    "/assets/*",
    "/*.ico",
    "/*.png",
    "/*.jpg",
    "/*.css",
    "/*.js"
  ]
}
```

**Purpose:**
- Functions invocations are metered (unlike static requests)
- Exclude static assets to keep them free and fast
- Wildcards match any depth: `/static/*` matches `/static/a/b/c/file.js`
- `exclude` takes precedence over `include`
- Max 100 rules combined, 100 chars per rule
- Must have at least 1 include rule

**When to create manually:**
- Framework doesn't auto-generate
- Want fine-grained control over Function invocations
- Mixing static and dynamic routes

## Wrangler CLI Commands

### Project Management

```bash
# List all Pages projects
npx wrangler pages project list

# Create new project
npx wrangler pages project create my-project --production-branch=main

# Delete project
npx wrangler pages project delete my-project
```

### Deployment

```bash
# Deploy directory
npx wrangler pages deploy ./dist --project-name=my-project

# Deploy with custom branch (for previews)
npx wrangler pages deploy ./dist --project-name=my-project --branch=feature-x

# Deploy with commit info
npx wrangler pages deploy ./dist \
  --project-name=my-project \
  --commit-hash=$(git rev-parse HEAD) \
  --commit-message="$(git log -1 --pretty=%B)"

# List deployments
npx wrangler pages deployment list --project-name=my-project

# Tail deployment logs
npx wrangler pages deployment tail --project-name=my-project
```

### Local Development

```bash
# Start dev server
npx wrangler pages dev ./dist

# With specific port
npx wrangler pages dev ./dist --port=8080

# With bindings
npx wrangler pages dev ./dist \
  --kv=KV \
  --d1=DB=local-db-id \
  --r2=BUCKET

# With persistence (keeps KV/D1 data between restarts)
npx wrangler pages dev ./dist --persist-to=./.wrangler/state/v3

# With compatibility flags
npx wrangler pages dev ./dist --compatibility-flags=nodejs_compat

# Live reload with build command
npx wrangler pages dev ./dist --live-reload

# Proxy to external server (useful for SSR frameworks)
npx wrangler pages dev -- npm run dev
```

### Configuration

```bash
# Generate types from bindings
npx wrangler types --path='./functions/types.d.ts'

# Validate wrangler.toml
npx wrangler pages project validate
```

## Common Patterns

### API Routes

```typescript
// functions/api/todos/[id].ts
import type { PagesFunction } from '@cloudflare/workers-types';

interface Env {
  DB: D1Database;
}

export const onRequestGet: PagesFunction<Env> = async ({ env, params }) => {
  const todo = await env.DB.prepare(
    'SELECT * FROM todos WHERE id = ?'
  ).bind(params.id).first();
  
  if (!todo) {
    return new Response('Not found', { status: 404 });
  }
  
  return Response.json(todo);
};

export const onRequestPut: PagesFunction<Env> = async ({ env, params, request }) => {
  const body = await request.json();
  
  await env.DB.prepare(
    'UPDATE todos SET title = ?, completed = ? WHERE id = ?'
  ).bind(body.title, body.completed, params.id).run();
  
  return Response.json({ success: true });
};

export const onRequestDelete: PagesFunction<Env> = async ({ env, params }) => {
  await env.DB.prepare('DELETE FROM todos WHERE id = ?').bind(params.id).run();
  return new Response(null, { status: 204 });
};
```

### Authentication Middleware

```typescript
// functions/_middleware.ts
import type { PagesFunction } from '@cloudflare/workers-types';

interface Env {
  JWT_SECRET: string;
}

const auth: PagesFunction<Env> = async (context) => {
  const { request, env } = context;
  
  // Skip auth for public routes
  if (request.url.includes('/public/')) {
    return context.next();
  }
  
  const authHeader = request.headers.get('Authorization');
  if (!authHeader?.startsWith('Bearer ')) {
    return new Response('Unauthorized', { status: 401 });
  }
  
  const token = authHeader.substring(7);
  
  try {
    // Verify JWT (use a proper library in production)
    const payload = await verifyJWT(token, env.JWT_SECRET);
    
    // Attach user to context.data for downstream handlers
    context.data.user = payload;
    
    return context.next();
  } catch (err) {
    return new Response('Invalid token', { status: 401 });
  }
};

export const onRequest = [auth];
```

### CORS Handling

```typescript
// functions/api/_middleware.ts
import type { PagesFunction } from '@cloudflare/workers-types';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization',
  'Access-Control-Max-Age': '86400',
};

const cors: PagesFunction = async (context) => {
  // Handle preflight
  if (context.request.method === 'OPTIONS') {
    return new Response(null, { headers: corsHeaders });
  }
  
  // Add CORS headers to response
  const response = await context.next();
  Object.entries(corsHeaders).forEach(([key, value]) => {
    response.headers.set(key, value);
  });
  
  return response;
};

export const onRequest = [cors];
```

### Server-Side Rendering (SSR)

For framework-specific SSR (Next.js, SvelteKit, etc.):

```typescript
// functions/_middleware.ts - Example for generic SSR
import type { PagesFunction } from '@cloudflare/workers-types';

export const onRequest: PagesFunction = async (context) => {
  const { request, env, next } = context;
  const url = new URL(request.url);
  
  // Serve static assets directly
  if (url.pathname.startsWith('/static/')) {
    return next();
  }
  
  // SSR for pages
  const rendered = await renderPage(url.pathname, {
    request,
    env,
  });
  
  return new Response(rendered, {
    headers: { 'Content-Type': 'text/html; charset=utf-8' }
  });
};
```

Most frameworks handle this automatically - refer to framework guides:
- Next.js: `@cloudflare/next-on-pages`
- SvelteKit: Built-in adapter
- Remix: `@remix-run/cloudflare-pages`

### Form Handling

```typescript
// functions/api/contact.ts
import type { PagesFunction } from '@cloudflare/workers-types';

interface Env {
  QUEUE: Queue;
}

export const onRequestPost: PagesFunction<Env> = async ({ request, env }) => {
  const formData = await request.formData();
  
  const submission = {
    name: formData.get('name'),
    email: formData.get('email'),
    message: formData.get('message'),
    timestamp: new Date().toISOString(),
  };
  
  // Queue for processing
  await env.QUEUE.send(submission);
  
  // Return HTML response
  return new Response(
    '<h1>Thanks for your submission!</h1>',
    { headers: { 'Content-Type': 'text/html' } }
  );
};
```

### Background Tasks

```typescript
// functions/api/process.ts
import type { PagesFunction } from '@cloudflare/workers-types';

export const onRequestPost: PagesFunction = async ({ request, waitUntil }) => {
  const data = await request.json();
  
  // Start background task (continues after response sent)
  waitUntil(
    (async () => {
      await fetch('https://api.example.com/webhook', {
        method: 'POST',
        body: JSON.stringify(data),
      });
      console.log('Webhook sent');
    })()
  );
  
  // Respond immediately
  return Response.json({ queued: true });
};
```

### Error Handling

```typescript
// functions/_middleware.ts
import type { PagesFunction } from '@cloudflare/workers-types';

const errorHandler: PagesFunction = async (context) => {
  try {
    return await context.next();
  } catch (error) {
    console.error('Function error:', error);
    
    // Return JSON error for API routes
    if (context.request.url.includes('/api/')) {
      return Response.json(
        { error: error.message },
        { status: 500 }
      );
    }
    
    // Return HTML error for pages
    return new Response(
      `<!DOCTYPE html>
      <html>
        <body>
          <h1>Error</h1>
          <p>${error.message}</p>
        </body>
      </html>`,
      { 
        status: 500,
        headers: { 'Content-Type': 'text/html' }
      }
    );
  }
};

export const onRequest = [errorHandler];
```

### Caching Strategies

```typescript
// functions/api/data.ts
import type { PagesFunction } from '@cloudflare/workers-types';

interface Env {
  KV: KVNamespace;
  DB: D1Database;
}

export const onRequestGet: PagesFunction<Env> = async ({ env, request }) => {
  const url = new URL(request.url);
  const cacheKey = `data:${url.pathname}`;
  
  // Check KV cache
  const cached = await env.KV.get(cacheKey, 'json');
  if (cached) {
    return Response.json(cached, {
      headers: { 'X-Cache': 'HIT' }
    });
  }
  
  // Fetch from database
  const data = await env.DB.prepare(
    'SELECT * FROM data WHERE path = ?'
  ).bind(url.pathname).first();
  
  // Cache for 1 hour
  await env.KV.put(cacheKey, JSON.stringify(data), {
    expirationTtl: 3600
  });
  
  return Response.json(data, {
    headers: { 
      'X-Cache': 'MISS',
      'Cache-Control': 'public, max-age=3600'
    }
  });
};
```

## Framework Integration

Popular frameworks have official guides and adapters:

### Next.js
```bash
npm create cloudflare@latest my-next-app -- --framework=next
# Uses @cloudflare/next-on-pages adapter
# Transforms Next.js build for Pages
```

### SvelteKit
```bash
npm create cloudflare@latest my-svelte-app -- --framework=svelte
# Uses @sveltejs/adapter-cloudflare
```

### Remix
```bash
npm create cloudflare@latest my-remix-app -- --framework=remix
# Uses @remix-run/cloudflare-pages
```

### Nuxt
```bash
npm create cloudflare@latest my-nuxt-app -- --framework=nuxt
```

### Astro
```bash
npm create cloudflare@latest my-astro-app -- --framework=astro
# Uses @astrojs/cloudflare adapter
```

### Qwik
```bash
npm create cloudflare@latest my-qwik-app -- --framework=qwik
```

Refer to [Framework Guides](https://developers.cloudflare.com/pages/framework-guides/) for specific configurations.

## Best Practices

### Performance

1. **Exclude static assets from Functions:**
   - Use `_routes.json` to exclude `/static/*`, `/assets/*`, etc.
   - Reduces Function invocations and improves cache hit rates

2. **Leverage KV for caching:**
   - Cache API responses, rendered content, etc.
   - Set appropriate TTLs based on data freshness needs

3. **Use Cache API for edge caching:**
   ```typescript
   const cache = caches.default;
   let response = await cache.match(request);
   if (!response) {
     response = await fetch(request);
     await cache.put(request, response.clone());
   }
   return response;
   ```

4. **Minimize Function code size:**
   - Tree-shake unused dependencies
   - Use dynamic imports for large libraries
   - Keep Functions under 1MB compressed

### Security

1. **Set security headers (for static sites):**
   ```txt
   # _headers
   /*
     X-Frame-Options: DENY
     X-Content-Type-Options: nosniff
     Referrer-Policy: strict-origin-when-cross-origin
     Permissions-Policy: geolocation=(), camera=()
   ```

2. **Use secrets for sensitive data:**
   ```bash
   # Never commit secrets to wrangler.toml
   echo "secret-value" | npx wrangler pages secret put SECRET_KEY
   ```

3. **Validate all inputs:**
   ```typescript
   const body = await request.json();
   if (!body.email || !body.email.includes('@')) {
     return Response.json({ error: 'Invalid email' }, { status: 400 });
   }
   ```

4. **Implement rate limiting:**
   ```typescript
   // Use KV or Durable Objects for rate limit state
   const identifier = request.headers.get('CF-Connecting-IP');
   const key = `ratelimit:${identifier}`;
   const count = await env.KV.get(key);
   if (count && parseInt(count) > 100) {
     return new Response('Rate limit exceeded', { status: 429 });
   }
   await env.KV.put(key, String(parseInt(count || '0') + 1), { expirationTtl: 60 });
   ```

### Development Workflow

1. **Use preview deployments:**
   - Every branch/PR gets unique preview URL
   - Test changes before merging to production
   - Share previews with stakeholders

2. **Test locally with wrangler pages dev:**
   - Mirrors production environment
   - Hot reload for rapid iteration
   - Test with actual bindings (KV, D1, etc.)

3. **Use environment-specific configs:**
   ```toml
   [env.preview.vars]
   API_URL = "https://staging-api.example.com"
   
   [env.production.vars]
   API_URL = "https://api.example.com"
   ```

4. **Monitor with logging:**
   ```typescript
   console.log('Request:', request.method, request.url);
   console.log('Env:', env);  // Structured logging
   ```
   
   View logs: `npx wrangler pages deployment tail`

### Deployment

1. **Use rollbacks for quick recovery:**
   - Cloudflare Dashboard → Pages Project → Deployments → Rollback
   - Instant rollback to any previous deployment

2. **Set up Deploy Hooks for external triggers:**
   ```bash
   curl -X POST https://api.cloudflare.com/client/v4/pages/webhooks/deploy/hooks/...
   ```

3. **Configure build caching:**
   - Automatically enabled for git deployments
   - Speeds up builds by caching node_modules, etc.

4. **Use monorepo support:**
   - Set "Root directory" in dashboard for monorepos
   - Only builds/deploys when files in that directory change

## Limitations

- **Functions:**
  - 100,000 requests/day on Free plan (unlimited on Paid)
  - 10ms CPU time per request
  - 128MB memory limit
  - 1MB script size (compressed)
  
- **Deployments:**
  - 500 deployments/month on Free (unlimited on Paid)
  - 20,000 files per deployment
  - 25MB per file
  
- **Configuration:**
  - 2,100 total redirects (2,000 static + 100 dynamic)
  - 100 header rules
  - 100 `_routes.json` rules
  
- **Build:**
  - 20 minute build timeout
  - Can't run Docker containers in build

See [Platform Limits](https://developers.cloudflare.com/pages/platform/limits/) for complete details.

## Troubleshooting

### Functions Not Running

1. Check `_routes.json` - might be excluding Function routes
2. Verify file naming - must be `.js` or `.ts`, not `.jsx` or `.tsx`
3. Check build output - Functions dir must be at root of output

### 404 on Static Assets

1. Verify build output directory setting
2. Check if Functions are catching requests (use `_routes.json` to exclude)
3. Ensure `env.ASSETS.fetch()` is called in advanced mode

### Bindings Not Working

1. Check wrangler.toml syntax
2. Verify binding IDs are correct
3. For local dev, check `.dev.vars` file
4. Regenerate types: `npx wrangler types`

### Build Failures

1. Check build logs in dashboard
2. Verify build command and output directory
3. Check node version compatibility
4. Review environment variables

### Deployment Fails

1. Check file count (max 20,000 files)
2. Verify file sizes (max 25MB per file)
3. Review build output for errors
4. Check wrangler.toml validation

## Resources

- [Pages Docs](https://developers.cloudflare.com/pages/)
- [Functions API Reference](https://developers.cloudflare.com/pages/functions/api-reference/)
- [Framework Guides](https://developers.cloudflare.com/pages/framework-guides/)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/)
- [Discord #functions channel](https://discord.com/channels/595317990191398933/910978223968518144)
- [Workers Examples](https://developers.cloudflare.com/workers/examples/)

## Quick Reference

```bash
# Create new project
npm create cloudflare@latest

# Local dev
npx wrangler pages dev ./dist

# Deploy
npx wrangler pages deploy ./dist --project-name=my-project

# Generate types
npx wrangler types --path='./functions/types.d.ts'

# Manage secrets
echo "value" | npx wrangler pages secret put KEY --project-name=my-project

# View logs
npx wrangler pages deployment tail --project-name=my-project
```

---

**When to Use Pages vs Workers:**
- **Use Pages**: Static sites, JAMstack apps, framework-based (Next.js, etc.), git workflow
- **Use Workers**: Pure APIs, complex routing, WebSockets, scheduled tasks, email handlers
- **Can combine**: Pages Functions use Workers runtime, can bind to Workers
