# Cloudflare Workers KV Skill

Expert guidance for Cloudflare Workers KV - a global, low-latency key-value data storage system.

## Core Concepts

### What is Workers KV?
Workers KV is an eventually-consistent, globally-distributed key-value store designed for read-heavy workloads with low latency. Optimized for storing configuration data, user preferences, authentication tokens, cached API responses, and routing data.

### Key Characteristics
- **Eventually consistent**: Writes visible immediately in same location, up to 60s globally
- **Read-optimized**: High read volumes with low latency
- **Global distribution**: Data replicated across Cloudflare's edge network
- **25 MiB value limit**: Per key-value pair
- **512 byte key limit**: Maximum key size

## Configuration

### 1. Create KV Namespace

```bash
# Create namespace
npx wrangler kv namespace create MY_NAMESPACE

# Output: namespace ID to add to wrangler config
# { binding = "MY_NAMESPACE", id = "abc123..." }
```

### 2. Bind to Worker

**wrangler.jsonc:**
```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-11",
  "kv_namespaces": [
    {
      "binding": "MY_KV",
      "id": "abc123xyz789"
    }
  ]
}
```

**wrangler.toml:**
```toml
name = "my-worker"
main = "src/index.ts"

[[kv_namespaces]]
binding = "MY_KV"
id = "abc123xyz789"
```

### 3. TypeScript Types

```typescript
interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Access KV via env.MY_KV
  }
} satisfies ExportedHandler<Env>;
```

## Core Operations

### Writing Data

#### Basic Put
```typescript
// String value
await env.MY_KV.put("user:123", "John Doe");

// JSON value (serialize manually)
await env.MY_KV.put("config:app", JSON.stringify({ theme: "dark", lang: "en" }));

// Binary data
const buffer = new ArrayBuffer(8);
await env.MY_KV.put("binary:data", buffer);

// ReadableStream
const stream = response.body;
await env.MY_KV.put("large:file", stream);
```

#### Put with Options
```typescript
// With expiration (seconds since UNIX epoch)
await env.MY_KV.put("session:abc", token, {
  expiration: Math.floor(Date.now() / 1000) + 3600 // 1 hour
});

// With TTL (seconds from now, minimum 60)
await env.MY_KV.put("cache:api", data, {
  expirationTtl: 300 // 5 minutes
});

// With metadata (max 1024 bytes serialized)
await env.MY_KV.put("user:profile", userData, {
  metadata: { 
    version: 2, 
    lastUpdated: Date.now(),
    source: "api"
  }
});

// Combined
await env.MY_KV.put("temp:data", value, {
  expirationTtl: 3600,
  metadata: { temporary: true }
});
```

### Reading Data

#### Single Key Get
```typescript
// Default: returns string or null
const value = await env.MY_KV.get("user:123");

// JSON type (auto-parses)
const config = await env.MY_KV.get<AppConfig>("config:app", "json");

// ArrayBuffer for binary
const buffer = await env.MY_KV.get("binary:data", "arrayBuffer");

// Stream for large values (most efficient)
const stream = await env.MY_KV.get("large:file", "stream");

// With cache TTL (seconds, min 60)
const value = await env.MY_KV.get("hot:key", {
  type: "text",
  cacheTtl: 300 // Cache at edge for 5 minutes
});
```

#### Bulk Get (up to 100 keys)
```typescript
// Get multiple keys at once (more efficient)
const values = await env.MY_KV.get([
  "user:1", 
  "user:2", 
  "user:3"
]);

// Returns Map<string, string | null>
const user1 = values.get("user:1"); // string | null

// With JSON type
const configs = await env.MY_KV.get<Config>([
  "config:app",
  "config:feature"
], "json");

// Returns Map<string, Config | null>
```

#### Get with Metadata
```typescript
// Single key
const result = await env.MY_KV.getWithMetadata("user:profile");
// { value: string | null, metadata: any | null }

if (result.value && result.metadata) {
  const { version, lastUpdated } = result.metadata;
}

// Multiple keys
const results = await env.MY_KV.getWithMetadata([
  "key1",
  "key2"
]);
// Returns Map<string, { value, metadata }>

// With type
const result = await env.MY_KV.getWithMetadata<UserData>(
  "user:123",
  "json"
);
```

### Deleting Data

```typescript
// Delete single key
await env.MY_KV.delete("user:123");

// Delete returns void, always succeeds (even if key doesn't exist)
```

### Listing Keys

```typescript
// List all keys
const keys = await env.MY_KV.list();
// { keys: [...], list_complete: boolean, cursor?: string }

// With prefix filter
const userKeys = await env.MY_KV.list({ prefix: "user:" });

// Pagination
let cursor: string | undefined;
let allKeys = [];

do {
  const result = await env.MY_KV.list({ cursor });
  allKeys.push(...result.keys);
  cursor = result.cursor;
} while (!result.list_complete);

// Limit results
const limited = await env.MY_KV.list({ limit: 100 });
```

## Common Patterns

### 1. Caching API Responses

```typescript
async function getCachedData(
  env: Env, 
  key: string, 
  fetcher: () => Promise<any>
): Promise<any> {
  // Check cache first
  const cached = await env.MY_KV.get(key, "json");
  if (cached) return cached;
  
  // Fetch fresh data
  const data = await fetcher();
  
  // Cache for 5 minutes
  await env.MY_KV.put(key, JSON.stringify(data), {
    expirationTtl: 300
  });
  
  return data;
}

// Usage
const apiData = await getCachedData(
  env,
  "cache:api:users",
  () => fetch("https://api.example.com/users").then(r => r.json())
);
```

### 2. User Session Management

```typescript
interface Session {
  userId: string;
  expiresAt: number;
  metadata: Record<string, any>;
}

async function createSession(env: Env, userId: string): Promise<string> {
  const sessionId = crypto.randomUUID();
  const expiresAt = Date.now() + (24 * 60 * 60 * 1000); // 24 hours
  
  await env.SESSIONS.put(
    `session:${sessionId}`,
    JSON.stringify({ userId, expiresAt }),
    { 
      expirationTtl: 86400, // 24 hours
      metadata: { createdAt: Date.now() }
    }
  );
  
  return sessionId;
}

async function getSession(env: Env, sessionId: string): Promise<Session | null> {
  const data = await env.SESSIONS.get<Session>(`session:${sessionId}`, "json");
  
  if (!data || data.expiresAt < Date.now()) {
    return null;
  }
  
  return data;
}

async function destroySession(env: Env, sessionId: string): Promise<void> {
  await env.SESSIONS.delete(`session:${sessionId}`);
}
```

### 3. Feature Flags / Configuration

```typescript
interface FeatureFlags {
  [key: string]: boolean;
}

async function getFeatureFlags(env: Env): Promise<FeatureFlags> {
  const flags = await env.CONFIG.get<FeatureFlags>(
    "features:flags",
    { type: "json", cacheTtl: 600 } // Cache 10 minutes
  );
  
  return flags || {};
}

async function isFeatureEnabled(
  env: Env, 
  feature: string
): Promise<boolean> {
  const flags = await getFeatureFlags(env);
  return flags[feature] || false;
}

// Usage in Worker
export default {
  async fetch(request, env, ctx): Promise<Response> {
    if (await isFeatureEnabled(env, "beta_feature")) {
      return handleBetaFeature(request);
    }
    return handleStandardFlow(request);
  }
} satisfies ExportedHandler<Env>;
```

### 4. Rate Limiting

```typescript
async function rateLimit(
  env: Env,
  identifier: string,
  limit: number,
  windowSeconds: number
): Promise<boolean> {
  const key = `ratelimit:${identifier}`;
  const now = Date.now();
  
  const data = await env.MY_KV.get<{ count: number, resetAt: number }>(
    key, 
    "json"
  );
  
  if (!data || data.resetAt < now) {
    // New window
    await env.MY_KV.put(
      key,
      JSON.stringify({ count: 1, resetAt: now + windowSeconds * 1000 }),
      { expirationTtl: windowSeconds }
    );
    return true;
  }
  
  if (data.count >= limit) {
    return false; // Rate limited
  }
  
  // Increment counter
  await env.MY_KV.put(
    key,
    JSON.stringify({ count: data.count + 1, resetAt: data.resetAt }),
    { expirationTtl: Math.ceil((data.resetAt - now) / 1000) }
  );
  
  return true;
}
```

### 5. A/B Testing

```typescript
interface ABTest {
  variants: string[];
  weights: number[];
}

async function getVariant(
  env: Env,
  userId: string,
  testName: string
): Promise<string> {
  // Check if user already assigned
  const assigned = await env.AB_TESTS.get(`test:${testName}:user:${userId}`);
  if (assigned) return assigned;
  
  // Get test configuration
  const test = await env.AB_TESTS.get<ABTest>(
    `test:${testName}:config`,
    { type: "json", cacheTtl: 3600 }
  );
  
  if (!test) return "control";
  
  // Assign variant based on hash of userId
  const hash = await hashString(userId);
  const random = (hash % 100) / 100;
  
  let cumulative = 0;
  let variant = test.variants[0];
  
  for (let i = 0; i < test.variants.length; i++) {
    cumulative += test.weights[i];
    if (random < cumulative) {
      variant = test.variants[i];
      break;
    }
  }
  
  // Store assignment (30 days)
  await env.AB_TESTS.put(
    `test:${testName}:user:${userId}`,
    variant,
    { expirationTtl: 2592000 }
  );
  
  return variant;
}

async function hashString(str: string): Promise<number> {
  const encoder = new TextEncoder();
  const data = encoder.encode(str);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = new Uint8Array(hashBuffer);
  return hashArray[0] + (hashArray[1] << 8);
}
```

### 6. Coalescing Cold Keys (Reduce Cardinality)

```typescript
// Instead of many individual keys
// await env.KV.put("user:123:name", "John");
// await env.KV.put("user:123:email", "john@example.com");
// await env.KV.put("user:123:role", "admin");

// Coalesce into single key-value object
interface UserProfile {
  name: string;
  email: string;
  role: string;
}

await env.USERS.put(
  "user:123:profile",
  JSON.stringify({
    name: "John",
    email: "john@example.com", 
    role: "admin"
  })
);

// Benefits:
// - Cold keys benefit from hot key cache
// - Single read instead of multiple
// - Reduced KV operations count

// Trade-off:
// - Harder to update individual fields
// - Larger value size
// - Need locking for concurrent updates
```

## Wrangler CLI Usage

### Namespace Management

```bash
# Create namespace
npx wrangler kv namespace create MY_NAMESPACE

# Create preview namespace (for local dev)
npx wrangler kv namespace create MY_NAMESPACE --preview

# List all namespaces
npx wrangler kv namespace list

# Delete namespace
npx wrangler kv namespace delete --namespace-id=abc123

# Rename namespace  
npx wrangler kv namespace rename "Old Name" --namespace-id=abc123 --new-name="New Name"
```

### Key Operations

```bash
# Put key-value
npx wrangler kv key put --binding=MY_KV "mykey" "myvalue"

# Put from file
npx wrangler kv key put --binding=MY_KV "file:data" --path=./data.json

# Put with TTL (seconds from now)
npx wrangler kv key put --binding=MY_KV "temp" "value" --ttl=3600

# Put with metadata
npx wrangler kv key put --binding=MY_KV "key" "value" --metadata='{"version":1}'

# Get key
npx wrangler kv key get --binding=MY_KV "mykey"

# Get as text (decode UTF-8)
npx wrangler kv key get --binding=MY_KV "mykey" --text

# List keys
npx wrangler kv key list --binding=MY_KV

# List with prefix
npx wrangler kv key list --binding=MY_KV --prefix="user:"

# Delete key
npx wrangler kv key delete --binding=MY_KV "mykey"

# Use namespace-id instead of binding
npx wrangler kv key get --namespace-id=abc123 "mykey"

# Remote vs Local (for wrangler dev)
npx wrangler kv key put --binding=MY_KV "key" "value" --remote
npx wrangler kv key get --binding=MY_KV "key" --local
```

### Bulk Operations

```bash
# Bulk put from JSON file (max 10,000 keys, <100MB total)
# Format: [{"key": "key1", "value": "value1"}, ...]
npx wrangler kv bulk put data.json --binding=MY_KV

# With TTL
npx wrangler kv bulk put data.json --binding=MY_KV --ttl=3600

# Bulk get from file of keys
# Format: ["key1", "key2", ...]
npx wrangler kv bulk get keys.json --binding=MY_KV

# Bulk delete from file of keys
npx wrangler kv bulk delete keys.json --binding=MY_KV --force
```

### Local Development

```bash
# Dev with local KV (default, isolated from production)
npx wrangler dev

# Dev with remote KV (connects to production KV)
npx wrangler dev --remote

# Or set in wrangler.jsonc:
# "kv_namespaces": [
#   { "binding": "MY_KV", "id": "...", "remote": true }
# ]
```

## REST API Usage

### Using Cloudflare SDK (TypeScript)

```typescript
import Cloudflare from 'cloudflare';

const client = new Cloudflare({
  apiEmail: process.env.CLOUDFLARE_EMAIL,
  apiKey: process.env.CLOUDFLARE_API_KEY,
});

const accountId = 'YOUR_ACCOUNT_ID';
const namespaceId = 'YOUR_NAMESPACE_ID';

// Write key-value
await client.kv.namespaces.values.update(
  namespaceId,
  'mykey',
  {
    account_id: accountId,
    value: 'myvalue',
  }
);

// Read key-value
const value = await client.kv.namespaces.values.get(
  namespaceId,
  'mykey',
  { account_id: accountId }
);

// Delete key-value
await client.kv.namespaces.values.delete(
  namespaceId,
  'mykey',
  { account_id: accountId }
);

// List namespaces
for await (const namespace of client.kv.namespaces.list({ 
  account_id: accountId 
})) {
  console.log(namespace.id, namespace.title);
}

// List keys in namespace
const keys = await client.kv.namespaces.keys.list(
  namespaceId,
  { account_id: accountId }
);
```

### Using cURL

```bash
# Put value
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/storage/kv/namespaces/$NAMESPACE_ID/values/$KEY_NAME" \
  -X PUT \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: text/plain" \
  --data "value here"

# Get value  
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/storage/kv/namespaces/$NAMESPACE_ID/values/$KEY_NAME" \
  -H "Authorization: Bearer $API_TOKEN"

# Delete value
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/storage/kv/namespaces/$NAMESPACE_ID/values/$KEY_NAME" \
  -X DELETE \
  -H "Authorization: Bearer $API_TOKEN"

# List keys
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/storage/kv/namespaces/$NAMESPACE_ID/keys" \
  -H "Authorization: Bearer $API_TOKEN"

# Bulk write (max 10,000 keys)
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/storage/kv/namespaces/$NAMESPACE_ID/bulk" \
  -X PUT \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '[{"key":"key1","value":"value1"},{"key":"key2","value":"value2"}]'
```

## Best Practices

### 1. Handle Eventual Consistency

```typescript
// ❌ BAD: Immediate read after write may not see new value globally
await env.MY_KV.put("key", "value");
const value = await env.MY_KV.get("key"); // May be stale in other regions

// ✅ GOOD: Design for eventual consistency
// Option 1: Return confirmation without reading
await env.MY_KV.put("key", "value");
return new Response("Updated", { status: 200 });

// Option 2: Use local cache or variable
const newValue = "updated";
await env.MY_KV.put("key", newValue);
return new Response(newValue); // Use local value

// Option 3: Accept 60s propagation delay
await env.MY_KV.put("key", "value");
// Other regions see change within 60s
```

### 2. Avoid Concurrent Writes to Same Key

```typescript
// ❌ BAD: Rate limit violation (max 1 write/second per key)
await Promise.all([
  env.MY_KV.put("counter", "1"),
  env.MY_KV.put("counter", "2"),
  env.MY_KV.put("counter", "3"),
]); // Throws 429 errors

// ✅ GOOD: Single write per key per second
await env.MY_KV.put("counter", "3");

// ✅ GOOD: Use unique keys for concurrent writes
await Promise.all([
  env.MY_KV.put("counter:1", "1"),
  env.MY_KV.put("counter:2", "2"),
  env.MY_KV.put("counter:3", "3"),
]);

// ✅ GOOD: Retry with exponential backoff
async function putWithRetry(
  kv: KVNamespace,
  key: string,
  value: string,
  maxAttempts = 5
) {
  let delay = 1000;
  
  for (let i = 0; i < maxAttempts; i++) {
    try {
      await kv.put(key, value);
      return;
    } catch (err) {
      if (err.message.includes("429") && i < maxAttempts - 1) {
        await new Promise(resolve => setTimeout(resolve, delay));
        delay *= 2; // Exponential backoff
      } else {
        throw err;
      }
    }
  }
}
```

### 3. Use Bulk Operations When Possible

```typescript
// ❌ BAD: Multiple individual gets (uses more Worker operations)
const user1 = await env.USERS.get("user:1");
const user2 = await env.USERS.get("user:2");
const user3 = await env.USERS.get("user:3");
// 3 operations against 1,000 limit

// ✅ GOOD: Single bulk get
const users = await env.USERS.get(["user:1", "user:2", "user:3"]);
// 1 operation against 1,000 limit

// Note: Bulk write NOT available in Workers, only via Wrangler/API
```

### 4. Choose Appropriate cacheTtl

```typescript
// For frequently-read, rarely-updated data
const config = await env.CONFIG.get("app:settings", {
  type: "json",
  cacheTtl: 3600 // Cache 1 hour at edge
});

// For real-time data (default: 60s minimum)
const status = await env.STATUS.get("system:health", {
  cacheTtl: 60 // Minimum cache, faster updates
});

// Trade-off:
// - Higher cacheTtl = better performance, staler data
// - Lower cacheTtl = fresher data, more KV reads
```

### 5. Handle Null Values Properly

```typescript
// ❌ BAD: No null check
const value = await env.MY_KV.get("key");
const result = value.toUpperCase(); // Error if null

// ✅ GOOD: Always check for null
const value = await env.MY_KV.get("key");

if (value === null) {
  return new Response("Not found", { status: 404 });
}

return new Response(value);

// ✅ GOOD: Provide default
const value = (await env.MY_KV.get("config")) ?? "default-config";
```

### 6. Structure Keys Hierarchically

```typescript
// ✅ GOOD: Use prefixes for organization and filtering
"user:123:profile"
"user:123:settings"
"cache:api:users"
"session:abc-def-ghi"
"feature:flags:beta"

// List by prefix
const userKeys = await env.MY_KV.list({ prefix: "user:123:" });
const cacheKeys = await env.MY_KV.list({ prefix: "cache:" });

// Easy to bulk delete by prefix pattern
```

### 7. Use Metadata for Versioning

```typescript
interface DataWithVersion {
  value: any;
  metadata: {
    version: number;
    schemaVersion: number;
    updatedAt: number;
  };
}

// Write with metadata
await env.MY_KV.put(
  "data:key",
  JSON.stringify(data),
  {
    metadata: {
      version: 2,
      schemaVersion: 1,
      updatedAt: Date.now()
    }
  }
);

// Read and check version
const result = await env.MY_KV.getWithMetadata("data:key", "json");

if (result.metadata?.schemaVersion !== 1) {
  // Handle migration
}
```

### 8. Error Handling

```typescript
export default {
  async fetch(request, env, ctx): Promise<Response> {
    try {
      const value = await env.MY_KV.get("key");
      
      if (value === null) {
        return new Response("Not found", { status: 404 });
      }
      
      return new Response(value);
    } catch (err) {
      console.error("KV error:", err);
      
      const message = err instanceof Error 
        ? err.message 
        : "Unknown KV error";
        
      return new Response(message, { 
        status: 500,
        headers: { "Content-Type": "text/plain" }
      });
    }
  }
} satisfies ExportedHandler<Env>;
```

## Limits & Quotas

| Feature | Free Plan | Paid Plan |
|---------|-----------|-----------|
| **Reads/day** | 100,000 | Unlimited |
| **Writes/day** (different keys) | 1,000 | Unlimited |
| **Writes to same key** | 1/second | 1/second |
| **Operations per Worker invocation** | 1,000 | 1,000 |
| **Namespaces per account** | 1,000 | 1,000 |
| **Storage per account** | 1 GB | Unlimited |
| **Storage per namespace** | 1 GB | Unlimited |
| **Keys per namespace** | Unlimited | Unlimited |
| **Key size** | 512 bytes | 512 bytes |
| **Value size** | 25 MiB | 25 MiB |
| **Metadata size** | 1024 bytes | 1024 bytes |
| **Minimum cacheTtl** | 60 seconds | 60 seconds |
| **Bulk get/put keys** | 100 / 10,000 | 100 / 10,000 |

**Important Notes:**
- Bulk requests count as **1 operation** against the 1,000 limit
- REST API subject to [Cloudflare API rate limits](https://developers.cloudflare.com/fundamentals/api/reference/limits)
- Writes to same key limited to **1 per second** (applies to all plans)

## Pricing (Paid Plans)

- **Reads:** $0.50 per 10 million reads
- **Writes:** $5.00 per 1 million writes  
- **Deletes:** $5.00 per 1 million deletes
- **Storage:** $0.50 per GB-month (stored data)

## Migration & Data Import

### Import from JSON

```bash
# Prepare data.json
# [
#   {"key": "user:1", "value": "John"},
#   {"key": "user:2", "value": "Jane", "expiration_ttl": 3600}
# ]

npx wrangler kv bulk put data.json --binding=MY_KV
```

### Export Data

```bash
# List all keys
npx wrangler kv key list --binding=MY_KV > keys.json

# Bulk get values
npx wrangler kv bulk get keys.json --binding=MY_KV > data.json
```

### Migrate Between Namespaces

```typescript
async function migrateNamespace(
  source: KVNamespace,
  dest: KVNamespace
): Promise<void> {
  let cursor: string | undefined;
  
  do {
    const result = await source.list({ cursor, limit: 1000 });
    
    // Copy each key
    for (const { name, metadata } of result.keys) {
      const value = await source.get(name);
      if (value !== null) {
        await dest.put(name, value, { metadata });
      }
    }
    
    cursor = result.cursor;
  } while (!result.list_complete);
}
```

## Testing

### Unit Testing with Miniflare

```typescript
import { beforeAll, describe, expect, it } from "vitest";
import { unstable_dev } from "wrangler";
import type { UnstableDevWorker } from "wrangler";

describe("Worker KV tests", () => {
  let worker: UnstableDevWorker;

  beforeAll(async () => {
    worker = await unstable_dev("src/index.ts", {
      experimental: { disableExperimentalWarning: true },
    });
  });

  afterAll(async () => {
    await worker.stop();
  });

  it("should store and retrieve value", async () => {
    const resp = await worker.fetch("/set?key=test&value=hello");
    expect(resp.status).toBe(200);
    
    const getResp = await worker.fetch("/get?key=test");
    expect(await getResp.text()).toBe("hello");
  });
});
```

### Local Development with Persist

```bash
# Persist KV data between dev sessions
npx wrangler dev --persist-to=.wrangler/state

# Data stored in: .wrangler/state/v3/kv/
```

## Troubleshooting

### Issue: Key not found immediately after write

**Cause:** Eventual consistency - writes propagate globally in ~60s

**Solution:** Design for eventual consistency, don't read immediately after write in different locations

### Issue: 429 Rate Limit Errors

**Cause:** More than 1 write per second to same key

**Solution:** Use unique keys or implement retry with exponential backoff

### Issue: Worker exceeds 1,000 operations limit

**Cause:** Too many individual KV operations

**Solution:** Use bulk get operations (count as 1 operation)

### Issue: Local dev not seeing production data

**Cause:** `wrangler dev` uses local KV by default

**Solution:** Add `"remote": true` to binding or use `--remote` flag

### Issue: Large response truncated

**Cause:** Value > 25 MiB limit

**Solution:** Use streaming with `get("key", "stream")` or split into multiple keys

### Issue: Stale data after update

**Cause:** High `cacheTtl` value caching old data at edge

**Solution:** Reduce `cacheTtl` or wait for cache expiration

## When NOT to Use KV

- ❌ **Strong consistency required** - Use Durable Objects
- ❌ **Write-heavy workloads** - Use D1 or Durable Objects  
- ❌ **Relational queries** - Use D1 (SQLite)
- ❌ **Large file storage (>25 MiB)** - Use R2
- ❌ **Real-time atomic operations** - Use Durable Objects
- ❌ **Complex transactions** - Use D1 or Durable Objects

## When TO Use KV

- ✅ **Read-heavy workloads** (caching, configs)
- ✅ **Global distribution needed**
- ✅ **Eventually consistent data acceptable**
- ✅ **Key-value access patterns**
- ✅ **Low-latency reads critical**
- ✅ **Simple data structures**

## Additional Resources

- [Official KV Documentation](https://developers.cloudflare.com/kv/)
- [KV API Reference](https://developers.cloudflare.com/kv/api/)
- [Wrangler KV Commands](https://developers.cloudflare.com/kv/reference/kv-commands/)
- [REST API Documentation](https://developers.cloudflare.com/api/resources/kv/)
- [KV Pricing](https://developers.cloudflare.com/kv/platform/pricing/)
- [How KV Works](https://developers.cloudflare.com/kv/concepts/how-kv-works/)
