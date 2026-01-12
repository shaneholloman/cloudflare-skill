# Cloudflare Bindings Skill

Expert guidance on Cloudflare Workers Bindings - the runtime APIs that connect Workers to Cloudflare platform resources.

## Core Concepts

**What are Bindings?**
- Runtime variables injecting capabilities into Workers
- Provide authenticated, optimized access to Cloudflare resources
- No manual credential management required
- Better performance than REST APIs for Worker-to-resource communication
- Permissions + API in one construct

**Binding Philosophy**
- Declared in `wrangler.toml` or `wrangler.jsonc` (recommended for new projects)
- Available via `env` parameter in handler functions
- Available as `this.env` in WorkerEntrypoint/DurableObject/Workflow classes
- Can be imported from `cloudflare:workers` for top-level scope access
- Strongly typed via `wrangler types`

## Accessing Bindings

### Method 1: Handler Parameter (Most Common)
```typescript
export default {
  async fetch(request: Request, env: Env) {
    await env.MY_BUCKET.put('key', 'value');
    const data = await env.MY_KV.get('key');
    return new Response(data);
  }
}
```

### Method 2: Class Property
```typescript
export class MyDurableObject extends DurableObject {
  async someMethod() {
    const value = await this.env.MY_KV.get('key');
    return value;
  }
}
```

### Method 3: Global Import (Top-Level Scope)
```typescript
import { env } from 'cloudflare:workers';
import ApiClient from 'example-api-client';

// OK: Read env vars/secrets in top-level scope
const apiClient = new ApiClient({ apiKey: env.API_KEY });
const LOG_LEVEL = env.LOG_LEVEL || 'info';

// ERROR: Cannot perform I/O outside request context
// const data = await env.KV.get('key'); // ❌ Will fail

export default {
  async fetch(request: Request) {
    // OK: I/O allowed inside request context
    const data = await env.KV.get('key'); // ✅ Works
    return new Response(data);
  }
}
```

**Top-Level Import Limitations:**
- Environment variables and secrets: ✅ Accessible
- Durable Object stubs: ✅ Can get stub
- KV/R2/D1 operations: ❌ Cannot perform I/O
- Service bindings: ❌ Cannot call
- Queue producers: ❌ Cannot send

### Overriding `env` Values (Testing)
```typescript
import { env, withEnv } from 'cloudflare:workers';

export default {
  async fetch(request: Request) {
    console.log(env.NAME); // "Alice"
    
    withEnv({ NAME: "Bob" }, () => {
      console.log(env.NAME); // "Bob"
    });
    
    console.log(env.NAME); // "Alice"
  }
}
```

## Global Scope Pollution Warning

**❌ AVOID:**
```typescript
let client = undefined;

export default {
  fetch(request: Request, env: Env) {
    // Client might not update when env.MY_SECRET changes
    client ??= new Client(env.MY_SECRET);
  }
}
```

**✅ CORRECT:**
```typescript
export default {
  fetch(request: Request, env: Env) {
    // New instance per request = always up-to-date
    const client = new Client(env.MY_SECRET);
  }
}
```

When bindings change during deployment, existing isolates may be reused. Creating derived objects in global scope can cause stale values to persist.

## Binding Types Reference

### Environment Variables & Secrets

**Environment Variables** - Plain text values in configuration
```jsonc
// wrangler.jsonc
{
  "vars": {
    "API_HOST": "example.com",
    "API_ACCOUNT_ID": "user123",
    "SERVICE_CONFIG": {
      "URL": "api.example.com",
      "ID": 123
    }
  }
}
```

**Secrets** - Encrypted values (never in config files)
```bash
# Set via CLI only
npx wrangler secret put API_KEY
```

**Usage:**
```typescript
export default {
  async fetch(request: Request, env: Env) {
    const host = env.API_HOST;        // From vars
    const key = env.API_KEY;          // From secret
    const config = env.SERVICE_CONFIG; // Object from vars
  }
}
```

### KV (Key-Value Storage)

**Configuration:**
```jsonc
{
  "kv_namespaces": [
    {
      "binding": "MY_KV",
      "id": "abcdef1234567890",
      "preview_id": "fedcba0987654321"  // Optional, for wrangler dev --remote
    }
  ]
}
```

**Auto-Provisioning (Beta):**
```jsonc
{
  "kv_namespaces": [
    {
      "binding": "MY_KV"
      // ID auto-created on deploy
    }
  ]
}
```

**Common Patterns:**
```typescript
// Read
const value = await env.MY_KV.get('key');
const json = await env.MY_KV.get('key', { type: 'json' });
const buffer = await env.MY_KV.get('key', { type: 'arrayBuffer' });
const stream = await env.MY_KV.get('key', { type: 'stream' });

// Write
await env.MY_KV.put('key', 'value');
await env.MY_KV.put('key', JSON.stringify(obj), {
  expirationTtl: 3600,
  metadata: { user: 'alice' }
});

// Delete
await env.MY_KV.delete('key');

// List
const { keys } = await env.MY_KV.list({ prefix: 'user:' });
```

**Best Practices:**
- Eventually consistent reads (optimized for high read volume)
- Use for configuration, cached data, session storage
- Max value size: 25 MiB
- Keys limited to 512 bytes

### R2 (Object Storage)

**Configuration:**
```jsonc
{
  "r2_buckets": [
    {
      "binding": "MY_BUCKET",
      "bucket_name": "my-bucket",
      "jurisdiction": "eu",  // Optional: eu, fedramp
      "preview_bucket_name": "my-bucket-preview"  // Optional
    }
  ]
}
```

**Auto-Provisioning:**
```jsonc
{
  "r2_buckets": [
    {
      "binding": "MY_BUCKET"
      // bucket_name auto-generated on deploy
    }
  ]
}
```

**Common Patterns:**
```typescript
// Write
await env.MY_BUCKET.put('file.txt', 'content', {
  httpMetadata: {
    contentType: 'text/plain',
    cacheControl: 'max-age=3600'
  },
  customMetadata: { uploadedBy: 'alice' }
});

// Read
const obj = await env.MY_BUCKET.get('file.txt');
if (obj) {
  const text = await obj.text();
  console.log(obj.httpMetadata);
  console.log(obj.customMetadata);
}

// Head (metadata only)
const obj = await env.MY_BUCKET.head('file.txt');
if (obj) {
  console.log(obj.size, obj.uploaded);
}

// Range read
const obj = await env.MY_BUCKET.get('file.txt', {
  range: { offset: 0, length: 1024 }
});

// Conditional get
const obj = await env.MY_BUCKET.get('file.txt', {
  onlyIf: { etagMatches: 'abc123' }
});

// List
const { objects, truncated, cursor } = await env.MY_BUCKET.list({
  prefix: 'uploads/',
  limit: 100,
  include: ['httpMetadata', 'customMetadata']
});

// Delete
await env.MY_BUCKET.delete('file.txt');
await env.MY_BUCKET.delete(['file1.txt', 'file2.txt']); // Bulk (max 1000)

// Multipart upload (for large files)
const upload = await env.MY_BUCKET.createMultipartUpload('large.bin');
const part1 = await upload.uploadPart(1, chunk1);
const part2 = await upload.uploadPart(2, chunk2);
await upload.complete([part1, part2]);
```

**Best Practices:**
- S3-compatible API available (separate from Workers binding)
- No egress fees
- Strongly consistent
- Use multipart uploads for files >100MB

### D1 (Relational Database)

**Configuration:**
```jsonc
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-db",
      "database_id": "abc-123-def",
      "preview_database_id": "preview-abc-123",  // Optional
      "migrations_dir": "migrations"              // Optional, default: "migrations"
    }
  ]
}
```

**Common Patterns:**
```typescript
// Prepare + run (safe from SQL injection)
const { results } = await env.DB.prepare(
  'SELECT * FROM users WHERE email = ?'
).bind('user@example.com').run();

// Multiple parameters
await env.DB.prepare(
  'INSERT INTO users (name, email) VALUES (?, ?)'
).bind('Alice', 'alice@example.com').run();

// Named parameters (requires compatibility_flags)
await env.DB.prepare(
  'SELECT * FROM users WHERE email = :email'
).bind({ email: 'user@example.com' }).run();

// Get first row
const user = await env.DB.prepare(
  'SELECT * FROM users WHERE id = ?'
).bind(123).first();

// Get raw array results
const rows = await env.DB.prepare(
  'SELECT name, email FROM users'
).raw();
// Returns: [['Alice', 'alice@example.com'], ['Bob', 'bob@example.com']]

// Batch (transactional)
const results = await env.DB.batch([
  env.DB.prepare('INSERT INTO users (name) VALUES (?)').bind('Alice'),
  env.DB.prepare('INSERT INTO users (name) VALUES (?)').bind('Bob')
]);

// Exec (no parameters, no prepared statement)
await env.DB.exec('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)');

// Sessions (read replicas)
const session = env.DB.withSession('first-primary');
const { results } = await session.prepare(
  'SELECT * FROM users'
).run();
```

**TypeScript Support:**
```typescript
type User = {
  id: number;
  name: string;
  email: string;
};

const result = await env.DB.prepare(
  'SELECT * FROM users LIMIT 10'
).run<User>();

result.results.forEach((user: User) => {
  console.log(user.name);
});
```

**Type Conversions:**
- JavaScript `null` → SQLite `NULL` → JavaScript `null`
- JavaScript `Number` → SQLite `REAL` or `INTEGER` → JavaScript `Number`
- JavaScript `String` → SQLite `TEXT` → JavaScript `String`
- JavaScript `Boolean` → SQLite `INTEGER` (0/1) → JavaScript `Number`
- JavaScript `ArrayBuffer` → SQLite `BLOB` → JavaScript `Array`

**Best Practices:**
- Always use prepared statements with `.bind()`
- Use STRICT tables to enforce types
- Max DB size: 10 GB (paid), 500 MB (free)
- Use batching for transactions

### Durable Objects

**Configuration:**
```jsonc
{
  "durable_objects": {
    "bindings": [
      {
        "name": "COUNTER",
        "class_name": "Counter"
      },
      {
        "name": "EXTERNAL_DO",
        "class_name": "ExternalClass",
        "script_name": "other-worker",
        "environment": "production"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["Counter"]
    },
    {
      "tag": "v2",
      "renamed_classes": [
        { "from": "Counter", "to": "CounterV2" }
      ]
    }
  ]
}
```

**Common Patterns:**
```typescript
// Get stub
const id = env.COUNTER.idFromName('global-counter');
const stub = env.COUNTER.get(id);

// RPC call
const count = await stub.increment();

// HTTP call
const response = await stub.fetch('https://fake-host/increment');

// From string ID
const id = env.COUNTER.idFromString('000000...');

// From request URL
const id = env.COUNTER.idFromName(new URL(request.url).pathname);
```

**Durable Object Class:**
```typescript
export class Counter extends DurableObject {
  async increment(): Promise<number> {
    let count = (await this.ctx.storage.get<number>('count')) || 0;
    count++;
    await this.ctx.storage.put('count', count);
    return count;
  }
  
  async fetch(request: Request): Promise<Response> {
    const count = await this.increment();
    return Response.json({ count });
  }
}
```

**Best Practices:**
- One instance per unique ID (strong consistency)
- Storage: SQLite-backed transactional storage
- Use for coordination, locks, real-time state
- RPC calls are more efficient than fetch()

### Queues

**Producer Configuration:**
```jsonc
{
  "queues": {
    "producers": [
      {
        "binding": "MY_QUEUE",
        "queue": "my-queue-name",
        "delivery_delay": 60  // Optional: delay messages by N seconds
      }
    ]
  }
}
```

**Consumer Configuration:**
```jsonc
{
  "queues": {
    "consumers": [
      {
        "queue": "my-queue-name",
        "max_batch_size": 10,
        "max_batch_timeout": 30,
        "max_retries": 3,
        "dead_letter_queue": "my-dlq",
        "max_concurrency": 5,
        "retry_delay": 120
      }
    ]
  }
}
```

**Producer Patterns:**
```typescript
// Send single message
await env.MY_QUEUE.send({
  type: 'user-signup',
  userId: 123
});

// Send with delay
await env.MY_QUEUE.send(
  { type: 'reminder' },
  { delaySeconds: 3600 }
);

// Send batch
await env.MY_QUEUE.sendBatch([
  { body: { type: 'email', to: 'alice@example.com' } },
  { body: { type: 'email', to: 'bob@example.com' }, delaySeconds: 60 }
]);
```

**Consumer Patterns:**
```typescript
export default {
  async queue(batch: MessageBatch<any>, env: Env): Promise<void> {
    for (const message of batch.messages) {
      console.log(message.body);
      message.ack(); // Explicit ack
    }
    
    // Or retry all
    batch.retryAll({ delaySeconds: 300 });
  }
}
```

**Best Practices:**
- Guaranteed delivery with at-least-once semantics
- Use batching for throughput
- Dead letter queues for failed messages
- Messages up to 128 KB

### Service Bindings

**Configuration:**
```jsonc
{
  "services": [
    {
      "binding": "AUTH_SERVICE",
      "service": "auth-worker",
      "entrypoint": "AuthService"  // Optional: named entrypoint
    },
    {
      "binding": "STAGING_API",
      "service": "api-worker-staging"  // Format: worker-name-environment
    }
  ]
}
```

**HTTP Pattern:**
```typescript
// Call another worker (no Internet roundtrip)
const response = await env.AUTH_SERVICE.fetch('https://fake-host/verify', {
  method: 'POST',
  body: JSON.stringify({ token: 'abc123' })
});

const { valid } = await response.json();
```

**RPC Pattern (Recommended):**
```typescript
// Target worker
export class AuthService extends WorkerEntrypoint {
  async verifyToken(token: string): Promise<boolean> {
    // verification logic
    return true;
  }
}

export default AuthService;

// Calling worker
const valid = await env.AUTH_SERVICE.verifyToken('abc123');
```

**Best Practices:**
- Zero latency between Workers
- RPC is more efficient than fetch
- Named entrypoints enable multiple services per Worker
- Use for microservices architecture

### Vectorize (Vector Database)

**Configuration:**
```jsonc
{
  "vectorize": [
    {
      "binding": "VECTOR_INDEX",
      "index_name": "embeddings-index"
    }
  ]
}
```

**Common Patterns:**
```typescript
// Insert vectors
await env.VECTOR_INDEX.insert([
  {
    id: 'doc1',
    values: [0.1, 0.2, 0.3, ...],
    metadata: { title: 'Document 1' }
  }
]);

// Query
const results = await env.VECTOR_INDEX.query([0.15, 0.25, 0.35, ...], {
  topK: 10,
  returnMetadata: true,
  filter: { category: 'tech' }
});
```

### Hyperdrive (Database Accelerator)

**Configuration:**
```jsonc
{
  "compatibility_flags": ["nodejs_compat_v2"],
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "hyperdrive-config-id"
    }
  ]
}
```

**Common Patterns:**
```typescript
import postgres from 'postgres';

export default {
  async fetch(request: Request, env: Env) {
    const sql = postgres(env.HYPERDRIVE.connectionString);
    const result = await sql`SELECT * FROM users LIMIT 10`;
    return Response.json(result);
  }
}
```

**Best Practices:**
- Connection pooling for Postgres databases
- Reduces latency to external databases
- Requires `nodejs_compat_v2` compatibility flag

### Workers AI

**Configuration:**
```jsonc
{
  "ai": {
    "binding": "AI"
  }
}
```

**Common Patterns:**
```typescript
// Text generation
const response = await env.AI.run('@cf/meta/llama-3-8b-instruct', {
  prompt: 'Write a haiku about Workers'
});

// Text embedding
const { data } = await env.AI.run('@cf/baai/bge-base-en-v1.5', {
  text: ['Hello world', 'Goodbye world']
});

// Image classification
const response = await env.AI.run('@cf/microsoft/resnet-50', {
  image: imageBuffer
});
```

### Analytics Engine

**Configuration:**
```jsonc
{
  "analytics_engine_datasets": [
    {
      "binding": "ANALYTICS",
      "dataset": "my-dataset"  // Optional, defaults to binding name
    }
  ]
}
```

**Common Patterns:**
```typescript
env.ANALYTICS.writeDataPoint({
  indexes: ['api_endpoint'],
  blobs: ['POST', '/api/users'],
  doubles: [response_time_ms]
});
```

### Browser Rendering

**Configuration:**
```jsonc
{
  "browser": {
    "binding": "BROWSER"
  }
}
```

**Common Patterns:**
```typescript
const browser = await puppeteer.launch(env.BROWSER);
const page = await browser.newPage();
await page.goto('https://example.com');
const screenshot = await page.screenshot();
await browser.close();
```

### mTLS Certificates

**Configuration:**
```jsonc
{
  "mtls_certificates": [
    {
      "binding": "CLIENT_CERT",
      "certificate_id": "cert-id-from-wrangler"
    }
  ]
}
```

**Common Patterns:**
```typescript
const response = await env.CLIENT_CERT.fetch('https://secure-api.com/data');
```

### Email Bindings

**Configuration:**
```jsonc
{
  "send_email": [
    {
      "name": "EMAIL",
      "destination_address": "notify@example.com"
    }
  ]
}
```

### Dispatch Namespaces (Workers for Platforms)

**Configuration:**
```jsonc
{
  "dispatch_namespaces": [
    {
      "binding": "DISPATCHER",
      "namespace": "customer-workers",
      "outbound": {
        "service": "outbound-worker",
        "parameters": ["params_object"]
      }
    }
  ]
}
```

**Common Patterns:**
```typescript
// Get customer worker stub
const worker = env.DISPATCHER.get('customer-123');
const response = await worker.fetch(request);
```

## Wrangler Configuration Patterns

### JSON vs TOML

**JSON (Recommended for new projects):**
```jsonc
// wrangler.jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-01-01",
  "vars": {
    "API_URL": "https://api.example.com"
  }
}
```

**TOML (Legacy):**
```toml
# wrangler.toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[vars]
API_URL = "https://api.example.com"
```

### Environment-Specific Configuration

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-01-01",
  
  // Production bindings
  "vars": {
    "ENV": "production"
  },
  "kv_namespaces": [
    {
      "binding": "CACHE",
      "id": "prod-kv-id"
    }
  ],
  
  // Environment overrides
  "env": {
    "staging": {
      "vars": {
        "ENV": "staging"
      },
      "kv_namespaces": [
        {
          "binding": "CACHE",
          "id": "staging-kv-id"
        }
      ]
    },
    "dev": {
      "vars": {
        "ENV": "development"
      }
    }
  }
}
```

**Deploy to environment:**
```bash
npx wrangler deploy --env staging
```

### Inheritable vs Non-Inheritable Keys

**Inheritable** (defined at top-level, inherited by environments):
- `name`, `main`, `compatibility_date`
- `compatibility_flags`, `workers_dev`
- `routes`, `route`, `triggers`
- `tsconfig`, `build`, `minify`
- `limits`, `observability`

**Non-Inheritable** (must be redefined per environment):
- `vars`
- `durable_objects`, `kv_namespaces`, `r2_buckets`
- `d1_databases`, `vectorize`, `hyperdrive`
- `services`, `queues`, `analytics_engine_datasets`

**Top-Level Only** (cannot be in environments):
- `migrations`, `keep_vars`, `send_metrics`, `site`

## Local Development

### Default Mode (Local Simulators)
```bash
npx wrangler dev
```
- KV/R2/D1 use local storage
- Changes don't affect production
- Faster startup

### Remote Mode (Real Resources)
```bash
npx wrangler dev --remote
```
- Uses `preview_id`, `preview_bucket_name`, `preview_database_id`
- Requires preview bindings in config
- Affects real (non-production) resources

### Remote Bindings (Hybrid)
```bash
npx wrangler dev --remote-binding MY_KV
```
- Local worker, remote specific binding
- Best of both worlds

## TypeScript Integration

### Generate Types
```bash
npx wrangler types
```

Generates `worker-configuration.d.ts`:
```typescript
interface Env {
  MY_KV: KVNamespace;
  MY_BUCKET: R2Bucket;
  DB: D1Database;
  COUNTER: DurableObjectNamespace;
  MY_QUEUE: Queue;
  API_KEY: string;
  API_URL: string;
}
```

### Usage
```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // env is now fully typed
    const value = await env.MY_KV.get('key');
    await env.MY_QUEUE.send({ type: 'test' });
    
    return new Response('OK');
  }
}
```

## Common Anti-Patterns

### ❌ Hardcoding Credentials
```typescript
// DON'T
const apiKey = 'sk_live_abc123';
```
**✅ Use secrets:**
```bash
npx wrangler secret put API_KEY
```

### ❌ Using REST API from Worker
```typescript
// DON'T
await fetch('https://api.cloudflare.com/client/v4/accounts/.../kv/...');
```
**✅ Use bindings:**
```typescript
await env.MY_KV.get('key');
```

### ❌ Polling KV/D1 for Changes
```typescript
// DON'T
setInterval(() => {
  const config = await env.KV.get('config');
}, 1000);
```
**✅ Use Durable Objects for real-time state**

### ❌ Storing Large Data in env.vars
```typescript
// DON'T
{ "vars": { "HUGE_CONFIG": "..." } } // Max 5KB per var
```
**✅ Use KV/R2 for large data**

## Troubleshooting

### Binding Not Found
```
Error: env.MY_KV is undefined
```
**Solutions:**
1. Check binding name matches config
2. Run `npx wrangler types` to regenerate types
3. Verify binding exists in current environment
4. Check for typos in `wrangler.jsonc`

### Type Errors
```
Property 'MY_KV' does not exist on type 'Env'
```
**Solution:** Run `npx wrangler types`

### Preview Binding Required
```
Error: preview_id is required for --remote
```
**Solution:** Add preview_id to config or use local mode

### Stale Binding Values
```
Secret updated but Worker still uses old value
```
**Solution:** Avoid caching `env` values in global scope

## Security Best Practices

1. **Never commit secrets to config files**
   - Use `npx wrangler secret put` for sensitive values
   - Secrets are encrypted at rest and in transit

2. **Use least privilege bindings**
   - Only bind resources the Worker needs
   - Use separate namespaces/buckets per environment

3. **Validate binding input**
   - User input → binding calls = potential injection
   - Sanitize keys, validate data types

4. **Audit binding access**
   - Review which Workers have access to sensitive bindings
   - Use Cloudflare Access for additional controls

## Performance Optimization

### KV
- Cache frequently accessed values in Worker memory
- Use list() cursor for pagination (don't fetch all keys)
- Batch writes when possible

### R2
- Use streaming for large objects
- Set appropriate cache headers
- Use multipart uploads for >100MB files

### D1
- Use batching for transactions
- Prepared statements are cached
- Index frequently queried columns

### Durable Objects
- Minimize instance creation (reuse IDs)
- Use transactional storage for consistency
- Prefer RPC over fetch() for lower latency

## Reference

**Official Docs:**
- Bindings Overview: https://developers.cloudflare.com/workers/runtime-apis/bindings/
- Configuration: https://developers.cloudflare.com/workers/wrangler/configuration/
- KV API: https://developers.cloudflare.com/kv/api/
- R2 API: https://developers.cloudflare.com/r2/api/workers/
- D1 API: https://developers.cloudflare.com/d1/worker-api/

**Wrangler Commands:**
```bash
npx wrangler types              # Generate TypeScript types
npx wrangler dev                # Local development
npx wrangler dev --remote       # Remote development
npx wrangler deploy             # Deploy to production
npx wrangler deploy --env staging  # Deploy to environment
npx wrangler secret put NAME    # Set secret
npx wrangler kv:namespace list  # List KV namespaces
npx wrangler r2 bucket list     # List R2 buckets
npx wrangler d1 list            # List D1 databases
```
