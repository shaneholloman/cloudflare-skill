# Cloudflare Secrets Store Skill

Expert guidance for Cloudflare Secrets Store - account-level secret management for Workers and AI Gateway.

## Overview

Cloudflare Secrets Store is a secure, centralized location for account-level secrets, encrypted and stored across all Cloudflare data centers. Secrets are reusable across Workers and AI Gateway.

**Key distinctions:**
- **Secrets Store**: Account-level secrets, managed centrally, reusable across Workers
- **Worker Secrets**: Per-Worker secrets, managed via `wrangler secret put` or dashboard Variables & Secrets

## Core Concepts

### Secrets Store Architecture

- **Store**: Container for secrets (1 store per account during beta)
- **Secret**: String value ≤1024 bytes (API tokens, keys, passwords, etc.)
- **Scopes**: Permission boundaries (e.g., `workers`, `ai-gateway`)
- **Bindings**: Connect secrets to Workers via `env` object

### Access Control

**Roles:**
- **Super Administrator**: Full access + bindings + integrations
- **Secrets Store Admin**: Create/edit/delete secrets, view metadata
- **Secrets Store Deployer**: View metadata + add bindings (no secret management)
- **Secrets Store Reporter**: View metadata only

**API Token Permissions:**
- `Account Secrets Store Edit`: Create/edit/delete secrets
- `Account Secrets Store Read`: View metadata

### Limits (Open Beta)

- 100 secrets per account
- 1 store per account
- 1024 bytes max per secret value
- Production secrets count toward limit (local dev secrets don't)

## Configuration Patterns

### Wrangler Configuration

**wrangler.toml:**
```toml
name = "my-worker"
main = "./src/index.js"

secrets_store_secrets = [
  { binding = "API_KEY", store_id = "abc123", secret_name = "stripe_api_key" },
  { binding = "DB_PASS", store_id = "abc123", secret_name = "postgres_password" }
]
```

**wrangler.jsonc:**
```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "./src/index.js",
  "secrets_store_secrets": [
    {
      "binding": "API_KEY",
      "store_id": "abc123",
      "secret_name": "stripe_api_key"
    }
  ]
}
```

**Configuration fields:**
- `binding`: Variable name for `env` object access
- `store_id`: Target store ID (get via `wrangler secrets-store store list`)
- `secret_name`: Secret identifier (no spaces allowed)

### Environment-Specific Configuration

Use multiple bindings for different environments:

```toml
[env.production]
secrets_store_secrets = [
  { binding = "API_KEY", store_id = "prod-store", secret_name = "prod_api_key" }
]

[env.staging]
secrets_store_secrets = [
  { binding = "API_KEY", store_id = "staging-store", secret_name = "staging_api_key" }
]
```

## Code Patterns

### Basic Secret Access

**CRITICAL**: Secrets Store bindings require asynchronous `.get()` call:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Async call required - secrets are NOT directly available
    const apiKey = await env.API_KEY.get();
    
    const response = await fetch("https://api.example.com/data", {
      headers: { "Authorization": `Bearer ${apiKey}` }
    });
    
    return response;
  }
}
```

**Type safety:**
```typescript
interface Env {
  // Secrets Store binding type
  API_KEY: {
    get(): Promise<string>;
  };
}
```

### Common Use Cases

#### Database Connection Strings

```typescript
import postgres from "postgres";

interface Env {
  DB_CONNECTION: { get(): Promise<string> };
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const connectionString = await env.DB_CONNECTION.get();
    const sql = postgres(connectionString);
    
    const result = await sql`SELECT * FROM users LIMIT 10`;
    
    return Response.json(result);
  }
}
```

#### Third-Party API Integration

```typescript
interface Env {
  STRIPE_KEY: { get(): Promise<string> };
  SENDGRID_KEY: { get(): Promise<string> };
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const [stripeKey, sendgridKey] = await Promise.all([
      env.STRIPE_KEY.get(),
      env.SENDGRID_KEY.get()
    ]);
    
    // Use secrets in parallel requests
    const [payment, email] = await Promise.all([
      fetch("https://api.stripe.com/v1/charges", {
        headers: { "Authorization": `Bearer ${stripeKey}` }
      }),
      fetch("https://api.sendgrid.com/v3/mail/send", {
        headers: { "Authorization": `Bearer ${sendgridKey}` }
      })
    ]);
    
    return Response.json({ payment: payment.ok, email: email.ok });
  }
}
```

#### Signed Request Authentication

```typescript
interface Env {
  HMAC_SECRET: { get(): Promise<string> };
}

async function signRequest(
  data: string,
  secret: string
): Promise<string> {
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    "raw",
    encoder.encode(secret),
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["sign"]
  );
  
  const signature = await crypto.subtle.sign(
    "HMAC",
    key,
    encoder.encode(data)
  );
  
  return btoa(String.fromCharCode(...new Uint8Array(signature)));
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const secret = await env.HMAC_SECRET.get();
    const payload = await request.text();
    const signature = await signRequest(payload, secret);
    
    return Response.json({ signature });
  }
}
```

### Anti-Patterns

**❌ Don't access secrets synchronously:**
```typescript
// WRONG - will error
const key = env.API_KEY; // Missing .get()
```

**❌ Don't cache secrets at module level:**
```typescript
// WRONG - secrets not available during module initialization
import { env } from "cloudflare:workers";
const CACHED_KEY = await env.API_KEY.get(); // Will fail
```

**✅ Cache within request scope if needed:**
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // OK - cache for request duration
    const apiKey = await env.API_KEY.get();
    
    // Use apiKey multiple times within this request
    const result1 = await fetchWithAuth(apiKey, "/endpoint1");
    const result2 = await fetchWithAuth(apiKey, "/endpoint2");
    
    return Response.json({ result1, result2 });
  }
}
```

## Wrangler Commands

### Store Management

```bash
# List stores
wrangler secrets-store store list

# Create store (first store auto-created on first use)
wrangler secrets-store store create my-store --remote

# Delete store
wrangler secrets-store store delete <store-id> --remote
```

### Secret Management (Production)

**Create secret:**
```bash
# Interactive prompt for value
wrangler secrets-store secret create <store-id> \
  --name MY_SECRET \
  --scopes workers \
  --remote

# Pipe value from file
cat secret.txt | wrangler secrets-store secret create <store-id> \
  --name MY_SECRET \
  --scopes workers \
  --remote
```

**List secrets:**
```bash
wrangler secrets-store secret list <store-id> --remote
```

**Get secret metadata:**
```bash
wrangler secrets-store secret get <store-id> --name MY_SECRET --remote
```

**Update secret:**
```bash
wrangler secrets-store secret update <store-id> \
  --name MY_SECRET \
  --new-value "new value" \
  --remote
```

**Delete secret:**
```bash
wrangler secrets-store secret delete <store-id> --name MY_SECRET --remote
```

**Duplicate secret:**
```bash
wrangler secrets-store secret duplicate <store-id> \
  --name ORIGINAL_SECRET \
  --new-name COPIED_SECRET \
  --remote
```

### Local Development

**CRITICAL**: Production secrets (created with `--remote`) are NOT accessible in local dev.

**Local development workflow:**
```bash
# Create local-only secret (no --remote flag)
wrangler secrets-store secret create <store-id> \
  --name DEV_API_KEY \
  --scopes workers

# Run dev server (uses local secrets)
wrangler dev

# Deploy (uses production secrets)
wrangler deploy
```

**Best practice**: Separate secret names for local vs production:
```toml
[env.development]
secrets_store_secrets = [
  { binding = "API_KEY", store_id = "store", secret_name = "dev_api_key" }
]

[env.production]
secrets_store_secrets = [
  { binding = "API_KEY", store_id = "store", secret_name = "prod_api_key" }
]
```

## REST API Usage

Base URL: `https://api.cloudflare.com/client/v4`

### Authentication

```bash
# Using API token (recommended)
curl https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/secrets_store/stores \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"

# Using email + API key
curl https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/secrets_store/stores \
  -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
  -H "X-Auth-Key: $CLOUDFLARE_API_KEY"
```

### Store Operations

**List stores:**
```bash
GET /accounts/{account_id}/secrets_store/stores
```

**Create store:**
```bash
POST /accounts/{account_id}/secrets_store/stores
Content-Type: application/json

{
  "name": "my-store"
}
```

**Delete store:**
```bash
DELETE /accounts/{account_id}/secrets_store/stores/{store_id}
```

### Secret Operations

**List secrets:**
```bash
GET /accounts/{account_id}/secrets_store/stores/{store_id}/secrets
```

**Create secret:**
```bash
POST /accounts/{account_id}/secrets_store/stores/{store_id}/secrets
Content-Type: application/json

{
  "name": "my_secret",
  "value": "secret_value",
  "scopes": ["workers"],
  "comment": "Optional description"
}
```

**Create multiple secrets:**
```bash
POST /accounts/{account_id}/secrets_store/stores/{store_id}/secrets
Content-Type: application/json

[
  {
    "name": "secret_one",
    "value": "value1",
    "scopes": ["workers"]
  },
  {
    "name": "secret_two",
    "value": "value2",
    "scopes": ["workers", "ai-gateway"]
  }
]
```

**Get secret metadata:**
```bash
GET /accounts/{account_id}/secrets_store/stores/{store_id}/secrets/{secret_id}
```

**Update secret:**
```bash
PATCH /accounts/{account_id}/secrets_store/stores/{store_id}/secrets/{secret_id}
Content-Type: application/json

{
  "value": "new_value",
  "comment": "Updated value"
}
```

**Delete secret:**
```bash
DELETE /accounts/{account_id}/secrets_store/stores/{store_id}/secrets/{secret_id}
```

**Delete multiple secrets:**
```bash
DELETE /accounts/{account_id}/secrets_store/stores/{store_id}/secrets
Content-Type: application/json

{
  "secret_ids": ["secret-id-1", "secret-id-2"]
}
```

**Duplicate secret:**
```bash
POST /accounts/{account_id}/secrets_store/stores/{store_id}/secrets/{secret_id}/duplicate
Content-Type: application/json

{
  "name": "new_secret_name"
}
```

**View quota:**
```bash
GET /accounts/{account_id}/secrets_store/quota
```

### API Response Patterns

**Success response:**
```json
{
  "success": true,
  "result": {
    "id": "secret-id-123",
    "name": "my_secret",
    "created": "2025-01-11T12:00:00Z",
    "modified": "2025-01-11T12:00:00Z",
    "scopes": ["workers"],
    "comment": "Description"
  },
  "errors": [],
  "messages": []
}
```

**Error response:**
```json
{
  "success": false,
  "result": null,
  "errors": [
    {
      "code": 10000,
      "message": "Secret name already exists"
    }
  ],
  "messages": []
}
```

## Dashboard Workflows

### Creating Secrets

1. Navigate to **Secrets Store** in dashboard
2. Click **Create secret**
3. Fill fields:
   - **Name**: Identifier (no spaces)
   - **Value**: Secret data (hidden after save)
   - **Permission scope**: Select `Workers`
   - **Comment**: Optional description
4. Click **Save** (value no longer visible)

### Adding Bindings to Workers

**Method 1: From Worker settings**
1. **Workers & Pages** → Select Worker → **Settings** → **Bindings**
2. Click **Add** → **Secrets Store**
3. Configure:
   - **Variable name**: Binding name for `env` object
   - **Secret name**: Select existing secret
4. Click **Deploy** (immediate) or **Save version** (gradual)

**Method 2: Create secret from Worker settings**
1. Follow Method 1 steps
2. If secret doesn't exist, click **Create secret** in dropdown
3. Creates account-level secret and adds binding simultaneously

### Deployment Options

- **Deploy**: Immediate 100% rollout
- **Save version**: Create version for gradual deployment

## Security Best Practices

### Secret Rotation

```typescript
// Design for rotation - don't hardcode assumptions
interface Env {
  PRIMARY_KEY: { get(): Promise<string> };
  FALLBACK_KEY?: { get(): Promise<string> }; // Optional fallback during rotation
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    let apiKey = await env.PRIMARY_KEY.get();
    
    // Attempt with primary key
    let response = await fetch("https://api.example.com/data", {
      headers: { "Authorization": `Bearer ${apiKey}` }
    });
    
    // Fallback to old key if new key fails (during rotation)
    if (!response.ok && env.FALLBACK_KEY) {
      apiKey = await env.FALLBACK_KEY.get();
      response = await fetch("https://api.example.com/data", {
        headers: { "Authorization": `Bearer ${apiKey}` }
      });
    }
    
    return response;
  }
}
```

**Rotation workflow:**
1. Create new secret with different name (`api_key_v2`)
2. Add as fallback binding to Worker
3. Deploy and verify new secret works
4. Update primary binding to new secret
5. Deploy
6. Remove old secret after verification period

### Audit and Monitoring

```typescript
interface Env {
  API_KEY: { get(): Promise<string> };
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const startTime = Date.now();
    let secretAccessSuccess = false;
    
    try {
      const apiKey = await env.API_KEY.get();
      secretAccessSuccess = true;
      
      // Use secret
      const response = await fetch("https://api.example.com", {
        headers: { "Authorization": `Bearer ${apiKey}` }
      });
      
      // Log usage
      ctx.waitUntil(
        fetch("https://logging.example.com/log", {
          method: "POST",
          body: JSON.stringify({
            event: "secret_used",
            secret_name: "API_KEY",
            timestamp: new Date().toISOString(),
            duration_ms: Date.now() - startTime,
            success: response.ok
          })
        })
      );
      
      return response;
    } catch (error) {
      // Log secret access failure
      ctx.waitUntil(
        fetch("https://logging.example.com/log", {
          method: "POST",
          body: JSON.stringify({
            event: "secret_access_failed",
            secret_name: "API_KEY",
            error: error instanceof Error ? error.message : "Unknown error",
            timestamp: new Date().toISOString()
          })
        })
      );
      
      return new Response("Internal error", { status: 500 });
    }
  }
}
```

### Never Log Secret Values

```typescript
// ❌ WRONG - logs secret value
const secret = await env.API_KEY.get();
console.log(`Using secret: ${secret}`);

// ✅ CORRECT - log metadata only
console.log("Retrieved API_KEY from Secrets Store");
```

## Comparison: Secrets Store vs Worker Secrets

| Feature | Secrets Store | Worker Secrets |
|---------|---------------|----------------|
| **Scope** | Account-level | Per-Worker |
| **Reusability** | Multiple Workers | Single Worker |
| **Access pattern** | `await env.BINDING.get()` | `env.SECRET_NAME` |
| **Management** | Centralized dashboard | Per-Worker settings |
| **Wrangler commands** | `secrets-store` | `secret` |
| **Local dev** | Separate local secrets | `.dev.vars` or `.env` |
| **Gradual deployment** | Supported | Supported |
| **Limits** | 100 per account | Per-Worker limit |
| **Use case** | Shared credentials | Worker-specific config |

**When to use Secrets Store:**
- Multiple Workers use same API key
- Centralized secret management required
- Compliance requires audit trail
- Team collaboration on secrets

**When to use Worker Secrets:**
- Secret unique to one Worker
- Simple single-Worker projects
- No need for cross-Worker sharing

## Migration Patterns

### From Worker Secrets to Secrets Store

**Before (Worker Secrets):**
```typescript
// wrangler.toml - no secrets_store_secrets
// Secret managed via: wrangler secret put API_KEY

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const apiKey = env.API_KEY; // Direct access
    // ... use apiKey
  }
}
```

**After (Secrets Store):**
```typescript
// wrangler.toml
secrets_store_secrets = [
  { binding = "API_KEY", store_id = "abc123", secret_name = "shared_api_key" }
]

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const apiKey = await env.API_KEY.get(); // Async access
    // ... use apiKey
  }
}
```

**Migration steps:**
1. Create secret in Secrets Store
2. Add `secrets_store_secrets` binding to `wrangler.toml`
3. Update code to use `await env.BINDING.get()`
4. Test in staging environment
5. Deploy to production
6. Delete old Worker secret via `wrangler secret delete API_KEY`

### Sharing Secrets Across Workers

```typescript
// worker-1/wrangler.toml
secrets_store_secrets = [
  { binding = "SHARED_DB", store_id = "abc123", secret_name = "postgres_url" }
]

// worker-2/wrangler.toml
secrets_store_secrets = [
  { binding = "DB_CONN", store_id = "abc123", secret_name = "postgres_url" }
]
```

Both Workers access same secret with different binding names:

```typescript
// worker-1/src/index.ts
const dbUrl = await env.SHARED_DB.get();

// worker-2/src/index.ts
const dbUrl = await env.DB_CONN.get();
```

## Troubleshooting

### "Secret not found" Error

**Symptoms:**
```
Error: Secret with name 'my_secret' not found in store
```

**Solutions:**
1. Verify secret exists: `wrangler secrets-store secret list <store-id> --remote`
2. Check `secret_name` matches exactly (case-sensitive)
3. Ensure secret has `workers` scope
4. Verify `store_id` is correct

### Local Development: Production Secrets Not Accessible

**Symptoms:**
```
Error: Cannot access secret 'API_KEY' in local development
```

**Solutions:**
1. Create local-only secret (no `--remote` flag):
   ```bash
   wrangler secrets-store secret create <store-id> --name API_KEY --scopes workers
   ```
2. Use same `store_id` and `secret_name` in `wrangler.toml`
3. Keep production and local secrets separate

### Type Errors with Binding

**Symptoms:**
```typescript
// Error: Property 'get' does not exist on type 'string'
const key = await env.API_KEY.get();
```

**Solutions:**
```typescript
// Define correct type for Secrets Store binding
interface Env {
  API_KEY: {
    get(): Promise<string>;
  };
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const key = await env.API_KEY.get(); // Type-safe
    // ...
  }
}
```

### Deployment Fails: "Binding already exists"

**Symptoms:**
```
Error: Binding 'API_KEY' already exists
```

**Solutions:**
1. Remove duplicate binding from dashboard **Settings** → **Bindings**
2. Check for conflicts between `wrangler.toml` and dashboard configuration
3. Remove old Worker secret with same name: `wrangler secret delete API_KEY`

### Quota Exceeded

**Symptoms:**
```
Error: Account secret quota exceeded (100/100)
```

**Solutions:**
1. Check quota: `wrangler secrets-store quota --remote`
2. Delete unused secrets
3. Consolidate duplicate secrets
4. Contact Cloudflare support for quota increase

## TypeScript Patterns

### Strongly-Typed Bindings

```typescript
// env.d.ts
interface SecretsStoreBinding {
  get(): Promise<string>;
}

interface Env {
  // Secrets Store bindings
  STRIPE_API_KEY: SecretsStoreBinding;
  DATABASE_URL: SecretsStoreBinding;
  SENDGRID_TOKEN: SecretsStoreBinding;
  
  // Regular Worker secrets (direct access)
  WORKER_SECRET: string;
  
  // Other bindings
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Type-safe access
    const stripeKey = await env.STRIPE_API_KEY.get();
    const dbUrl = await env.DATABASE_URL.get();
    
    // Direct access for Worker secrets
    const workerSecret = env.WORKER_SECRET;
    
    return Response.json({ ok: true });
  }
}
```

### Helper Functions

```typescript
// secrets.ts
import type { SecretsStoreBinding } from "./env";

export async function getSecretWithFallback(
  primary: SecretsStoreBinding,
  fallback?: SecretsStoreBinding
): Promise<string> {
  try {
    return await primary.get();
  } catch (error) {
    if (fallback) {
      console.warn("Primary secret failed, using fallback");
      return await fallback.get();
    }
    throw error;
  }
}

export async function getAllSecrets(
  secrets: Record<string, SecretsStoreBinding>
): Promise<Record<string, string>> {
  const entries = await Promise.all(
    Object.entries(secrets).map(async ([key, binding]) => [
      key,
      await binding.get()
    ])
  );
  return Object.fromEntries(entries);
}

// Usage
interface Env {
  API_KEY_PRIMARY: SecretsStoreBinding;
  API_KEY_FALLBACK: SecretsStoreBinding;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const apiKey = await getSecretWithFallback(
      env.API_KEY_PRIMARY,
      env.API_KEY_FALLBACK
    );
    
    return Response.json({ ok: true });
  }
}
```

## Integration Patterns

### With D1 Database

```typescript
import { drizzle } from "drizzle-orm/d1";

interface Env {
  DB: D1Database;
  DB_CREDENTIALS: { get(): Promise<string> };
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Get connection credentials
    const credentials = await env.DB_CREDENTIALS.get();
    const { username, password } = JSON.parse(credentials);
    
    // Use with D1
    const db = drizzle(env.DB);
    // ... perform queries
    
    return Response.json({ ok: true });
  }
}
```

### With KV Namespace

```typescript
interface Env {
  CACHE: KVNamespace;
  ENCRYPTION_KEY: { get(): Promise<string> };
}

async function encryptValue(value: string, key: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(value);
  const keyMaterial = encoder.encode(key);
  
  const cryptoKey = await crypto.subtle.importKey(
    "raw",
    keyMaterial,
    { name: "AES-GCM" },
    false,
    ["encrypt"]
  );
  
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const encrypted = await crypto.subtle.encrypt(
    { name: "AES-GCM", iv },
    cryptoKey,
    data
  );
  
  // Combine IV and encrypted data
  const combined = new Uint8Array(iv.length + encrypted.byteLength);
  combined.set(iv);
  combined.set(new Uint8Array(encrypted), iv.length);
  
  return btoa(String.fromCharCode(...combined));
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const encryptionKey = await env.ENCRYPTION_KEY.get();
    
    // Encrypt sensitive data before storing in KV
    const sensitiveData = "user-ssn-123-45-6789";
    const encrypted = await encryptValue(sensitiveData, encryptionKey);
    
    await env.CACHE.put("user:123:ssn", encrypted);
    
    return Response.json({ ok: true });
  }
}
```

### With Service Bindings

```typescript
// auth-worker/wrangler.toml
secrets_store_secrets = [
  { binding = "JWT_SECRET", store_id = "abc123", secret_name = "jwt_signing_key" }
]

// api-worker/wrangler.toml
services = [
  { binding = "AUTH", service = "auth-worker" }
]

// auth-worker/src/index.ts
interface Env {
  JWT_SECRET: { get(): Promise<string> };
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const secret = await env.JWT_SECRET.get();
    
    // Sign JWT
    const token = await signJWT({ userId: 123 }, secret);
    
    return Response.json({ token });
  }
}

// api-worker/src/index.ts
interface Env {
  AUTH: Fetcher;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Call auth service (which uses Secrets Store)
    const authResponse = await env.AUTH.fetch(
      new Request("https://auth/verify", {
        headers: request.headers
      })
    );
    
    return authResponse;
  }
}
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Deploy Worker

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install dependencies
        run: npm ci
      
      - name: Create Secrets Store secret
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
        run: |
          echo "${{ secrets.API_KEY_VALUE }}" | \
          npx wrangler secrets-store secret create ${{ secrets.STORE_ID }} \
            --name API_KEY \
            --scopes workers \
            --remote
      
      - name: Deploy Worker
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
        run: npx wrangler deploy
```

### GitLab CI/CD

```yaml
deploy:
  image: node:20
  script:
    - npm ci
    - |
      echo "$API_KEY_VALUE" | \
      npx wrangler secrets-store secret create $STORE_ID \
        --name API_KEY \
        --scopes workers \
        --remote
    - npx wrangler deploy
  only:
    - main
  variables:
    CLOUDFLARE_API_TOKEN: $CLOUDFLARE_API_TOKEN
```

## References

- **Docs**: https://developers.cloudflare.com/secrets-store/
- **Workers Integration**: https://developers.cloudflare.com/secrets-store/integrations/workers/
- **API Reference**: https://developers.cloudflare.com/api/resources/secrets_store/
- **Wrangler Commands**: https://developers.cloudflare.com/workers/wrangler/commands/#secrets-store
- **Access Control**: https://developers.cloudflare.com/secrets-store/access-control/

## Key Takeaways

1. **Async access required**: Always use `await env.BINDING.get()`
2. **Account-level**: Secrets shared across Workers, not per-Worker
3. **Local vs production**: Separate secrets for dev (no `--remote`) vs prod (`--remote`)
4. **Type safety**: Define `{ get(): Promise<string> }` interface
5. **Security**: Never log secret values, only metadata
6. **Limits**: 100 secrets per account during beta
7. **Scopes**: Set `workers` scope when creating secrets
8. **Rotation**: Design for rotation with fallback bindings
9. **Dashboard vs Wrangler**: Both create same account-level secrets
10. **No direct access**: Secret values never returned by API after creation
