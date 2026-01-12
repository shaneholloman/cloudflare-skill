# Cloudflare Static Assets Skill

Expert guidance for deploying and configuring static assets with Cloudflare Workers. This skill covers configuration patterns, routing architectures, asset binding usage, and best practices for SPAs, SSG sites, and full-stack applications.

## Core Concepts

### What are Static Assets?

Static Assets allow you to upload HTML, CSS, images, and other files as part of your Worker. Cloudflare handles caching and serving them globally.

**Key characteristics:**
- Deployed alongside Worker code in a single operation
- Automatically cached at edge locations worldwide
- Tiered caching reduces latency and origin fetches
- Assets directory specified in `wrangler.toml` or `wrangler.jsonc`

### How Deployment Works

1. **Local development**: Wrangler serves files from configured `assets.directory`
2. **Deployment**: `wrangler deploy` uploads both Worker code + assets
3. **Runtime**: Requests routed to nearest location, with automatic caching

## Configuration

### Basic Setup

Minimal configuration requires only `assets.directory`:

```jsonc
{
  "name": "my-worker",
  "compatibility_date": "2025-01-01",
  "assets": {
    "directory": "./dist"
  }
}
```

```toml
name = "my-worker"
compatibility_date = "2025-01-01"

[assets]
directory = "./dist"
```

### Full Configuration Options

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-01",
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS",
    "not_found_handling": "single-page-application",
    "html_handling": "auto-trailing-slash",
    "run_worker_first": ["/api/*", "!/api/docs/*"]
  }
}
```

**Configuration keys:**

- `directory` (string, required): Path to assets folder (e.g. `./dist`, `./public`, `./build`)
- `binding` (string, optional): Name to access assets in Worker code (e.g. `env.ASSETS`)
- `not_found_handling` (string, optional): Behavior when asset not found
  - `"single-page-application"`: Serve `/index.html` with 200 OK
  - `"404-page"`: Serve nearest `404.html` with 404 status
  - Default: 404 response
- `html_handling` (string, optional): How to handle HTML extensions
  - `"auto-trailing-slash"`: `/foo/` → `/foo/index.html`
  - `"drop-trailing-slash"`: `/foo` → `/foo.html`
  - `"none"`: Exact path matching only
- `run_worker_first` (boolean | string[], optional): Run Worker before serving assets
  - `true`: All requests invoke Worker first
  - `false` (default): Serve matching assets directly
  - Array of patterns: Selective Worker-first routing (e.g. `["/api/*", "!/api/docs/*"]`)

### Ignoring Files

Create `.assetsignore` in assets directory (uses `.gitignore` syntax):

```txt
_worker.js
_redirects
_headers
*.map
node_modules/
```

## Routing Architectures

### Architecture 1: Single Page Application (SPA)

**Use case**: Client-rendered React, Vue, Svelte apps

**Configuration:**

```jsonc
{
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application"
  }
}
```

**Routing behavior:**
1. Request matches asset → Serve asset directly
2. Navigation request (has `Sec-Fetch-Mode: navigate` header) → Serve `/index.html` with 200 OK
3. Non-navigation request + no match → Invoke Worker (if present) or serve `/index.html`

**Code pattern - API routes with SPA fallback:**

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // Handle API routes
    if (url.pathname.startsWith("/api/")) {
      return new Response(JSON.stringify({ name: "Cloudflare" }), {
        headers: { "Content-Type": "application/json" }
      });
    }
    
    // All other routes fall back to index.html automatically
    return new Response(null, { status: 404 });
  }
} satisfies ExportedHandler<Env>;
```

**Advanced routing control:**

For explicit control without relying on `Sec-Fetch-Mode` headers:

```jsonc
{
  "main": "./src/index.ts",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application",
    "binding": "ASSETS",
    "run_worker_first": ["/api/*", "!/api/docs/*"]
  }
}
```

This runs Worker first for `/api/*` paths (except `/api/docs/*`), serving assets for everything else.

### Architecture 2: Static Site Generation (SSG)

**Use case**: Static sites with custom 404 pages

**Configuration:**

```jsonc
{
  "assets": {
    "directory": "./dist",
    "not_found_handling": "404-page"
  }
}
```

**Routing behavior:**
1. Request matches asset → Serve asset
2. No match → Search for nearest `404.html` in directory hierarchy
3. Serve `404.html` with 404 status

**Directory structure example:**

```
dist/
├── index.html
├── about.html
├── 404.html
└── blog/
    ├── index.html
    ├── post-1.html
    └── 404.html
```

Request to `/blog/nonexistent` serves `/blog/404.html`
Request to `/nonexistent` serves `/404.html`

### Architecture 3: Full-Stack Application

**Use case**: Server-side rendered apps (Astro, Next.js, Remix, etc.)

**Configuration:**

```jsonc
{
  "name": "my-app",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-01",
  "assets": {
    "directory": "./dist/client",
    "binding": "ASSETS"
  }
}
```

**Code pattern - SSR with static assets:**

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // Server-side routes
    if (url.pathname.startsWith("/api/")) {
      const data = await env.DB.prepare("SELECT * FROM users").all();
      return Response.json(data.results);
    }
    
    // Dynamic pages
    if (url.pathname.startsWith("/products/")) {
      const productId = url.pathname.split("/")[2];
      const product = await getProduct(productId, env);
      return new Response(renderProductPage(product), {
        headers: { "Content-Type": "text/html" }
      });
    }
    
    // Fallback to static assets
    return env.ASSETS.fetch(request);
  }
} satisfies ExportedHandler<Env>;
```

### Architecture 4: Worker-First Routing

**Use case**: Authentication, logging, request modification before serving assets

**Configuration:**

```jsonc
{
  "main": "src/index.ts",
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS",
    "run_worker_first": true
  }
}
```

**Code pattern - Auth gating:**

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // Public routes
    if (url.pathname === "/login" || url.pathname === "/") {
      return env.ASSETS.fetch(request);
    }
    
    // Check authentication
    const sessionId = getCookie(request, "sessionId");
    const isAuthenticated = await validateSession(sessionId, env.DB);
    
    if (!isAuthenticated) {
      return Response.redirect(new URL("/login", url), 302);
    }
    
    // Authenticated - serve asset
    return env.ASSETS.fetch(request);
  }
} satisfies ExportedHandler<Env>;
```

## Assets Binding API

The assets binding allows programmatic access to static assets from Worker code.

### Binding Declaration

```typescript
interface Env {
  ASSETS: Fetcher;
}
```

### `env.ASSETS.fetch()`

**Signature:**
```typescript
fetch(input: Request | URL | string): Promise<Response>
```

**Parameters:**
- `request`: Request object, URL object, or URL string
- Requests made through binding have `html_handling` and `not_found_handling` applied

**Returns:** `Promise<Response>` - Static asset response

### Common Patterns

**1. Forward request to assets:**

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    return env.ASSETS.fetch(request);
  }
};
```

**2. Fetch specific asset by path:**

```typescript
const response = await env.ASSETS.fetch("https://assets.local/logo.png");
```

**3. Modify request before fetching asset:**

```typescript
const url = new URL(request.url);
url.pathname = "/index.html";
return env.ASSETS.fetch(new Request(url, request));
```

**4. Transform asset response:**

```typescript
const response = await env.ASSETS.fetch(request);
const modifiedResponse = new Response(response.body, response);
modifiedResponse.headers.set("X-Custom-Header", "value");
modifiedResponse.headers.set("Cache-Control", "public, max-age=3600");
return modifiedResponse;
```

**5. Conditional asset serving:**

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // Try to serve asset
    const assetResponse = await env.ASSETS.fetch(request);
    
    // If asset not found, handle gracefully
    if (assetResponse.status === 404) {
      if (url.pathname.startsWith("/admin/")) {
        return new Response("Admin area not found", { status: 404 });
      }
      // Fallback to index.html for client-side routing
      return env.ASSETS.fetch(new URL("/index.html", url.origin));
    }
    
    return assetResponse;
  }
};
```

**6. Asset existence check:**

```typescript
const assetResponse = await env.ASSETS.fetch("/feature.html");
const hasFeature = assetResponse.status === 200;

if (hasFeature) {
  return assetResponse;
} else {
  return Response.redirect("/coming-soon");
}
```

## Wrangler Commands

### Local Development

```bash
# Start dev server
npx wrangler dev

# Dev with remote resources
npx wrangler dev --remote

# Dev on specific port
npx wrangler dev --port 8080
```

### Deployment

```bash
# Deploy to production
npx wrangler deploy

# Deploy to specific environment
npx wrangler deploy --env staging

# Deploy with custom compatibility date
npx wrangler deploy --compatibility-date 2025-01-01
```

### Scaffolding

```bash
# Create static site
npm create cloudflare@latest my-static-site

# Create React SPA with static assets
npm create cloudflare@latest my-react-app -- --framework=react

# Create SSR/full-stack app
npm create cloudflare@latest my-app
# Select "SSR / full-stack app" template
```

## Compatibility Flags

### `assets_navigation_prefers_asset_serving`

**Available from:** `compatibility_date = "2025-04-01"`

**Effect:** Navigation requests (with `Sec-Fetch-Mode: navigate` header) skip Worker invocation and serve assets directly.

**Use case:** Reduce billable Worker invocations for SPAs

**Example:**

```jsonc
{
  "compatibility_date": "2025-04-01",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application"
  }
}
```

With this flag:
- Browser navigation to `/app` → Serves `/index.html` (no Worker invocation)
- Fetch request to `/api/data` → Invokes Worker
- Fetch request to `/app` → Invokes Worker

### `assets_navigation_has_no_effect`

**Effect:** Disables navigation request optimization, always invokes Worker

**Use case:** When you need full control over all requests including navigation

## Common Use Cases

### Use Case 1: SPA with API Backend

```jsonc
{
  "name": "spa-with-api",
  "main": "src/index.ts",
  "compatibility_date": "2025-04-01",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application",
    "binding": "ASSETS",
    "run_worker_first": ["/api/*"]
  }
}
```

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    if (url.pathname.startsWith("/api/")) {
      // Handle API logic
      const data = await env.DB.prepare("SELECT * FROM items").all();
      return Response.json(data.results);
    }
    
    // Assets handled automatically by routing config
    return new Response(null, { status: 404 });
  }
};
```

### Use Case 2: Asset Transformation

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // Serve optimized images
    if (url.pathname.startsWith("/images/")) {
      const assetResponse = await env.ASSETS.fetch(request);
      
      if (assetResponse.ok && request.headers.get("Accept")?.includes("image/webp")) {
        // Transform to WebP (simplified example)
        return new Response(assetResponse.body, {
          ...assetResponse,
          headers: {
            ...Object.fromEntries(assetResponse.headers),
            "Content-Type": "image/webp"
          }
        });
      }
      
      return assetResponse;
    }
    
    return env.ASSETS.fetch(request);
  }
};
```

### Use Case 3: A/B Testing

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // A/B test on homepage
    if (url.pathname === "/" || url.pathname === "/index.html") {
      const variant = Math.random() < 0.5 ? "a" : "b";
      const cookie = `variant=${variant}; Path=/; Max-Age=86400`;
      
      const assetPath = variant === "a" ? "/index.html" : "/index-variant-b.html";
      const response = await env.ASSETS.fetch(new URL(assetPath, url.origin));
      
      const modifiedResponse = new Response(response.body, response);
      modifiedResponse.headers.set("Set-Cookie", cookie);
      return modifiedResponse;
    }
    
    return env.ASSETS.fetch(request);
  }
};
```

### Use Case 4: OAuth Callback with SPA

When using `assets_navigation_prefers_asset_serving`, navigation requests serve HTML. For OAuth callbacks, pass data to Worker with client-side JS:

**public/oauth/callback.html:**
```html
<!DOCTYPE html>
<html>
<head><title>OAuth Callback</title></head>
<body>
  <p>Loading...</p>
  <script>
    (async () => {
      const response = await fetch("/api/oauth/callback" + window.location.search);
      if (response.ok) {
        window.location.href = '/';
      } else {
        document.querySelector('p').textContent = 'Error: ' + (await response.json()).error;
      }
    })();
  </script>
</body>
</html>
```

**Worker:**
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    if (url.pathname === "/api/oauth/callback") {
      const code = url.searchParams.get("code");
      const sessionId = await exchangeCodeForSession(code, env);
      
      if (sessionId) {
        return new Response(null, {
          headers: {
            "Set-Cookie": `sessionId=${sessionId}; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=86400`
          }
        });
      } else {
        return Response.json({ error: "Invalid code" }, { status: 400 });
      }
    }
    
    return new Response(null, { status: 404 });
  }
};
```

### Use Case 5: Static Assets with Smart Placement

Smart Placement positions Worker code near backend infrastructure.

**With `run_worker_first = false` (default):**
- Assets served from nearest edge location to user
- Worker runs near backend when invoked
- Best for: Fast asset delivery, occasional backend calls

**With `run_worker_first = true`:**
- All requests go to Smart-Placed Worker first
- Assets fetched from Worker location
- Increased latency for assets
- Best for: Authentication, asset modification, backend integration

**Configuration:**

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "placement": { "mode": "smart" },
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS",
    "run_worker_first": false  // Prioritize fast asset delivery
  }
}
```

## Integration with Frameworks

### Vite Plugin

For Vite-powered frameworks, use `@cloudflare/vite-plugin`:

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import { cloudflare } from "@cloudflare/vite-plugin";

export default defineConfig({
  plugins: [
    cloudflare({
      configPath: "./wrangler.jsonc",
      environments: {
        dev: {
          mode: "local"
        }
      }
    })
  ]
});
```

**Benefits:**
- Vite-native dev experience
- Automatic asset directory configuration
- HMR support
- No manual `assets.directory` needed

**Deployment:**
```bash
npm run build  # Vite build
npx wrangler deploy
```

### Framework-Specific Patterns

**Astro:**
```typescript
// astro.config.mjs
import cloudflare from "@astrojs/cloudflare";

export default defineConfig({
  output: "server",
  adapter: cloudflare({
    mode: "directory",
    functionPerRoute: false
  })
});
```

**Next.js:**
```bash
# Use @cloudflare/next-on-pages
npm install -D @cloudflare/next-on-pages
npx @cloudflare/next-on-pages
wrangler deploy
```

**React Router (Remix):**
```typescript
// vite.config.ts
import { cloudflare } from "@cloudflare/vite-plugin";

export default defineConfig({
  plugins: [cloudflare()]
});
```

## Caching Behavior

### Automatic Caching

Static assets are automatically cached with Cloudflare's tiered caching:

1. **First request**: Asset fetched from storage, cached at nearest location
2. **Cache miss**: Retrieved from nearby cache tier (not origin)
3. **Cache hit**: Served directly from edge

### Cache Headers

Control caching with standard HTTP headers in transformed responses:

```typescript
const response = await env.ASSETS.fetch(request);
const modifiedResponse = new Response(response.body, response);

// Cache for 1 hour in browser, 1 day at edge
modifiedResponse.headers.set("Cache-Control", "public, max-age=3600, s-maxage=86400");

return modifiedResponse;
```

### Cache Purging

Purge assets via:
- Cloudflare Dashboard → Caching → Purge Cache
- API: `POST /zones/{zone_id}/purge_cache`
- Redeployment with `wrangler deploy` (uploads new versions)

## Best Practices

### 1. Use Selective Worker-First Routing

Instead of `run_worker_first = true`, use array patterns:

```jsonc
{
  "assets": {
    "run_worker_first": [
      "/api/*",           // API routes
      "/admin/*",         // Admin area
      "!/admin/assets/*"  // Except admin assets
    ]
  }
}
```

**Benefits:**
- Reduces Worker invocations
- Lowers costs
- Improves asset delivery performance

### 2. Leverage Navigation Request Optimization

For SPAs, use `compatibility_date = "2025-04-01"` or later:

```jsonc
{
  "compatibility_date": "2025-04-01",
  "assets": {
    "not_found_handling": "single-page-application"
  }
}
```

Navigation requests skip Worker invocation, reducing costs.

### 3. Type Safety with Bindings

Always type your environment:

```typescript
interface Env {
  ASSETS: Fetcher;
  DB: D1Database;
  KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // TypeScript ensures env.ASSETS exists
    return env.ASSETS.fetch(request);
  }
} satisfies ExportedHandler<Env>;
```

### 4. Handle Asset Fetch Failures

Always check asset response status:

```typescript
const assetResponse = await env.ASSETS.fetch(request);

if (assetResponse.status === 404) {
  // Handle missing asset
  return Response.redirect("/404");
}

if (!assetResponse.ok) {
  // Log error, return fallback
  console.error("Asset fetch failed:", assetResponse.status);
  return new Response("Service unavailable", { status: 503 });
}

return assetResponse;
```

### 5. Use `.assetsignore` for Sensitive Files

Never deploy secrets or build artifacts:

```txt
# .assetsignore in assets directory
.env
.env.*
*.map
_worker.js
_headers
_redirects
```

### 6. Optimize Asset Directory Structure

Organize by route for better SSG 404 handling:

```
dist/
├── index.html
├── 404.html          # Root-level fallback
├── app/
│   ├── index.html
│   ├── dashboard.html
│   └── 404.html      # App-specific 404
└── docs/
    ├── index.html
    └── 404.html      # Docs-specific 404
```

### 7. Test Locally with Remote Bindings

Test against production-like setup:

```bash
# Use remote resources (KV, D1, R2, etc.) but local assets
wrangler dev --remote
```

**Local mode** (default): Simulated bindings, local assets
**Remote mode**: Real Cloudflare resources

## Troubleshooting

### Issue: Assets not updating after deployment

**Cause:** Browser or edge cache

**Solution:**
```bash
# Clear Cloudflare cache via dashboard or API
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {token}" \
  -d '{"purge_everything": true}'

# Or deploy with cache-busting query params
const response = await env.ASSETS.fetch(request.url + "?v=" + Date.now());
```

### Issue: Worker invoked too frequently (high costs)

**Cause:** Missing `run_worker_first` configuration or navigation optimization

**Solution:**
```jsonc
{
  "compatibility_date": "2025-04-01",  // Enable navigation optimization
  "assets": {
    "not_found_handling": "single-page-application",
    "run_worker_first": ["/api/*"]  // Only invoke Worker for API routes
  }
}
```

### Issue: 404 errors in SPA when refreshing deep routes

**Cause:** Missing `not_found_handling` config

**Solution:**
```jsonc
{
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application"
  }
}
```

### Issue: Assets binding returns 404 for existing files

**Cause:** Incorrect URL format or path

**Debug:**
```typescript
const url = new URL(request.url);
console.log("Requesting asset:", url.pathname);

// Use origin URL, not relative path
const response = await env.ASSETS.fetch(new URL("/file.html", url.origin));
console.log("Asset response:", response.status);
```

### Issue: `wrangler dev` not serving assets

**Cause:** Incorrect `assets.directory` path

**Solution:**
```bash
# Check directory exists
ls -la ./dist

# Verify wrangler.jsonc path
cat wrangler.jsonc
# Ensure "directory" points to correct build output
```

## Billing and Limits

### Request Billing

- **Asset-only requests**: Not billed (served directly from cache)
- **Worker invocations**: Billed per request
- **Navigation requests** (with `compatibility_date >= 2025-04-01`): Not billed for SPAs

### Limits

- **Max asset size**: 25 MB per file
- **Total assets size**: 25 MB per Worker (combined)
- **Max files**: No hard limit, but total size constraint applies
- **Request size**: 100 MB max
- **Response size**: 100 MB max

### Optimization Tips

1. Minimize Worker invocations with selective routing
2. Use navigation optimization for SPAs
3. Enable tiered caching (automatic)
4. Compress assets (gzip/brotli) before deployment

## Reference Links

- [Static Assets Docs](https://developers.cloudflare.com/workers/static-assets/)
- [Routing Configuration](https://developers.cloudflare.com/workers/static-assets/routing/)
- [Assets Binding API](https://developers.cloudflare.com/workers/static-assets/binding/)
- [Wrangler Configuration](https://developers.cloudflare.com/workers/wrangler/configuration/)
- [Framework Guides](https://developers.cloudflare.com/workers/framework-guides/)
- [Vite Plugin](https://developers.cloudflare.com/workers/vite-plugin/)
