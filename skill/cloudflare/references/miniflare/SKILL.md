# Miniflare Skill

**Version:** 1.0.0  
**Purpose:** Expert guidance for Cloudflare Miniflare - local development simulator for Cloudflare Workers

## Overview

Miniflare is a simulator for developing and testing Cloudflare Workers locally. It runs Workers code in a sandbox implementing Workers' runtime APIs, enabling fully-local development without internet connectivity.

**Key Features:**
- Full-featured: supports KV, Durable Objects, R2, D1, WebSockets, Queues, and more
- Fully-local: test and develop without internet, with instant reload
- TypeScript-native with detailed logging and source map support
- Advanced testing: dispatch events without HTTP requests, simulate Worker connections

**When to Use:**
- Writing integration tests for Workers
- Advanced use cases requiring fine-grained control
- Testing Worker bindings and storage locally
- Simulating multiple Workers with service bindings

**Note:** Most users should use Wrangler for standard development. Miniflare is for advanced use cases and testing.

---

## Installation & Setup

### Installation

```bash
npm i -D miniflare
```

### Basic Setup

Miniflare requires ES modules. Set in `package.json`:

```json
{
  "type": "module"
}
```

### Basic Initialization

```js
import { Miniflare } from "miniflare";

const mf = new Miniflare({
  modules: true,
  script: `
    export default {
      async fetch(request, env, ctx) {
        return new Response("Hello Miniflare!");
      }
    }
  `,
});

const res = await mf.dispatchFetch("http://localhost:8787/");
console.log(await res.text()); // Hello Miniflare!
await mf.dispose();
```

---

## Core Configuration

### Script Loading Patterns

**Inline Script:**
```js
const mf = new Miniflare({
  modules: true,
  script: `export default { ... }`,
});
```

**File-based Script:**
```js
const mf = new Miniflare({
  scriptPath: "worker.js",
});
```

**Multiple Modules with Auto-crawling:**
```js
const mf = new Miniflare({
  scriptPath: "src/index.js",
  modules: true,
  modulesRules: [
    { type: "ESModule", include: ["**/*.js"] },
    { type: "Text", include: ["**/*.txt"] },
  ],
});
```

**Explicit Module List:**
```js
const mf = new Miniflare({
  modules: [
    { type: "ESModule", path: "src/index.js" },
    { type: "ESModule", path: "src/utils.js" },
    { type: "Text", path: "data.txt" },
  ],
});
```

### Compatibility & Standards

```js
const mf = new Miniflare({
  compatibilityDate: "2021-11-23",
  compatibilityFlags: ["formdata_parser_supports_files"],
  upstream: "https://example.com", // Upstream origin for requests
});
```

---

## Event Dispatching

### Fetch Events

**Direct dispatch (no HTTP server):**
```js
const res = await mf.dispatchFetch("http://localhost:8787/path", {
  method: "POST",
  headers: { "Authorization": "Bearer token" },
  body: JSON.stringify({ data: "value" }),
});
```

**With custom Host header for routing:**
```js
const res = await mf.dispatchFetch("http://localhost:8787/", {
  headers: { "Host": "api.example.com" },
});
```

### Scheduled Events

```js
const worker = await mf.getWorker();
const result = await worker.scheduled({
  cron: "30 * * * *",
});
// result: { outcome: "ok", noRetry: false }
```

### Queue Events

```js
const worker = await mf.getWorker();
const result = await worker.queue("queue-name", [
  { id: "msg1", timestamp: new Date(), body: "data", attempts: 1 },
  { id: "msg2", timestamp: new Date(), body: { json: true }, attempts: 1 },
]);
// result: { outcome: "ok", retryAll: false, ackAll: false, ... }
```

---

## HTTP Server

### Starting Server

```js
const mf = new Miniflare({
  scriptPath: "worker.js",
  port: 8787,
  host: "127.0.0.1",
});
await mf.ready; // Wait for server to be ready
console.log("Listening on :8787");
```

### HTTPS Server

**Self-signed certificate:**
```js
const mf = new Miniflare({
  https: true, // Uses default self-signed cert
});
```

**Custom certificate from files:**
```js
const mf = new Miniflare({
  httpsKeyPath: "./key.pem",
  httpsCertPath: "./cert.pem",
});
```

**Custom certificate from strings:**
```js
const mf = new Miniflare({
  httpsKey: "-----BEGIN RSA PRIVATE KEY-----...",
  httpsCert: "-----BEGIN CERTIFICATE-----...",
});
```

### Request.cf Object

**Auto-fetch from Cloudflare:**
```js
const mf = new Miniflare({
  cf: true, // Default: fetches from Cloudflare endpoint
});
```

**Custom cf object:**
```js
const mf = new Miniflare({
  cf: "cf.json", // Load from file
});
```

**Disable cf object:**
```js
const mf = new Miniflare({
  cf: false,
});
```

---

## Storage Bindings

### KV Namespaces

**Basic setup:**
```js
const mf = new Miniflare({
  kvNamespaces: ["TEST_NAMESPACE", "CACHE"],
  kvPersist: "./kv-data", // Persist to disk (optional)
});
```

**Access in Worker:**
```js
export default {
  async fetch(request, env) {
    await env.TEST_NAMESPACE.put("key", "value");
    const value = await env.TEST_NAMESPACE.get("key");
    return new Response(value);
  },
};
```

**Manipulate outside Worker (testing):**
```js
const ns = await mf.getKVNamespace("TEST_NAMESPACE");
await ns.put("count", "1");
const value = await ns.get("count");

// Use in test:
const res = await mf.dispatchFetch("http://localhost/");
console.log(await res.text()); // Uses value from KV
```

### R2 Buckets

**Basic setup:**
```js
const mf = new Miniflare({
  r2Buckets: ["BUCKET", "IMAGES"],
  r2Persist: "./r2-data", // Persist to disk (optional)
});
```

**Access in Worker:**
```js
export default {
  async fetch(request, env) {
    await env.BUCKET.put("file.txt", "content");
    const object = await env.BUCKET.get("file.txt");
    return new Response(await object.text());
  },
};
```

**Manipulate outside Worker (testing):**
```js
const bucket = await mf.getR2Bucket("BUCKET");
await bucket.put("count", "1");
const object = await bucket.get("count");
const text = await object.text();
```

### Durable Objects

**Basic setup:**
```js
const mf = new Miniflare({
  modules: true,
  durableObjects: {
    COUNTER: "Counter", // className
    API_OBJECT: { className: "ApiObject", scriptName: "api-worker" },
  },
  durableObjectsPersist: "./do-data", // Persist to disk (optional)
  script: `
    export class Counter {
      constructor(state) {
        this.storage = state.storage;
      }
      
      async fetch(request) {
        let count = (await this.storage.get("count")) || 0;
        count++;
        await this.storage.put("count", count);
        return new Response(count.toString());
      }
    }
    
    export default {
      async fetch(request, env) {
        const id = env.COUNTER.idFromName("global");
        const stub = env.COUNTER.get(id);
        return stub.fetch(request);
      }
    }
  `,
});
```

**Manipulate outside Worker (testing):**
```js
const ns = await mf.getDurableObjectNamespace("COUNTER");
const id = ns.idFromName("test");
const stub = ns.get(id);
const res = await stub.fetch("http://localhost/");

// Or access storage directly:
const storage = await mf.getDurableObjectStorage(id);
await storage.put("key", "value");
const value = await storage.get("key");
```

### D1 Databases

**Basic setup:**
```js
const mf = new Miniflare({
  d1Databases: ["DB"],
  d1Persist: "./d1-data", // Persist to disk (optional)
});
```

**Access outside Worker (testing):**
```js
const db = await mf.getD1Database("DB");
await db.exec(`CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)`);
await db.prepare("INSERT INTO users (name) VALUES (?)").bind("Alice").run();
```

### Cache API

**Configuration:**
```js
const mf = new Miniflare({
  cache: true, // Enabled by default
  cachePersist: "./cache-data", // Persist to disk (optional)
  cacheWarnUsage: true, // Warn on cache usage (for *.workers.dev domains)
});
```

**Access outside Worker (testing):**
```js
const caches = await mf.getCaches();
const defaultCache = caches.default;
const namedCache = await caches.open("my-cache");

await defaultCache.put("http://example.com", new Response("cached"));
const cached = await defaultCache.match("http://example.com");
```

---

## Bindings

### Environment Variables & Secrets

```js
const mf = new Miniflare({
  bindings: {
    SECRET_KEY: "my-secret-value",
    API_URL: "https://api.example.com",
    DEBUG: true,
  },
});
```

**Access bindings in tests:**
```js
const bindings = await mf.getBindings();
console.log(bindings.SECRET_KEY); // "my-secret-value"
```

### WASM Modules

```js
const mf = new Miniflare({
  wasmBindings: {
    ADD_MODULE: "./add.wasm",
  },
});
```

### Text & Data Blobs

```js
const mf = new Miniflare({
  textBlobBindings: {
    TEXT: "./data.txt",
  },
  dataBlobBindings: {
    DATA: "./data.bin",
  },
});
```

### Queue Producers

**Setup:**
```js
const mf = new Miniflare({
  queueProducers: ["QUEUE"],
});
```

**Access outside Worker (testing):**
```js
const producer = await mf.getQueueProducer("QUEUE");
await producer.send({ body: "message data" });
```

---

## Multiple Workers & Service Bindings

### Multiple Workers

```js
const mf = new Miniflare({
  // Shared config
  host: "0.0.0.0",
  port: 8787,
  kvPersist: true,
  
  workers: [
    {
      name: "main-worker",
      kvNamespaces: { DATA: "shared-data" },
      serviceBindings: {
        API: "api-worker",
        // Custom function binding
        async EXTERNAL(request) {
          return new Response("External response");
        },
      },
      modules: true,
      script: `
        export default {
          async fetch(request, env) {
            const apiRes = await env.API.fetch(request);
            const data = await apiRes.json();
            return Response.json(data);
          }
        }
      `,
    },
    {
      name: "api-worker",
      kvNamespaces: { DATA: "shared-data" },
      script: `
        addEventListener("fetch", (event) => {
          event.respondWith(new Response(JSON.stringify({ ok: true })));
        })
      `,
    },
  ],
});
```

### Routing

**Route-based Worker selection:**
```js
const mf = new Miniflare({
  workers: [
    {
      name: "api",
      scriptPath: "./api/worker.js",
      routes: ["http://127.0.0.1/api/*", "api.example.com/*"],
    },
    {
      name: "web",
      scriptPath: "./web/worker.js",
      routes: ["example.com/*"],
    },
  ],
});
```

**Note:** Update `/etc/hosts` (Linux/macOS) or `C:\Windows\System32\drivers\etc\hosts` (Windows):
```
127.0.0.1 api.example.com
127.0.0.1 example.com
```

**Dispatch with routing:**
```js
// Dispatches to "api" worker
const res = await mf.dispatchFetch("http://api.example.com/users");

// Or with Host header
const res = await mf.dispatchFetch("http://localhost:8787/users", {
  headers: { "Host": "api.example.com" },
});
```

---

## Testing Patterns

### Basic Test Setup (node:test)

```js
import assert from "node:assert";
import test, { after, before, describe } from "node:test";
import { Miniflare } from "miniflare";

describe("worker", () => {
  let mf;

  before(async () => {
    mf = new Miniflare({
      modules: true,
      scriptPath: "src/index.js",
      kvNamespaces: ["TEST_KV"],
      bindings: { API_KEY: "test-key" },
    });
    await mf.ready;
  });

  test("fetch returns hello", async () => {
    const res = await mf.dispatchFetch("http://localhost/");
    assert.strictEqual(await res.text(), "Hello World");
  });

  test("kv operations", async () => {
    const kv = await mf.getKVNamespace("TEST_KV");
    await kv.put("key", "value");
    
    const res = await mf.dispatchFetch("http://localhost/kv");
    assert.strictEqual(await res.text(), "value");
  });

  test("environment bindings", async () => {
    const bindings = await mf.getBindings();
    assert.strictEqual(bindings.API_KEY, "test-key");
  });

  after(async () => {
    await mf.dispose();
  });
});
```

### Testing with Custom Build

```js
import { spawnSync } from "node:child_process";
import test, { before } from "node:test";

before(() => {
  // Build TypeScript/bundled Worker before tests
  spawnSync("npx wrangler build", {
    shell: true,
    stdio: "pipe",
  });
});
```

### Testing Durable Objects

```js
test("durable object state", async () => {
  const ns = await mf.getDurableObjectNamespace("COUNTER");
  const id = ns.idFromName("test-counter");
  const stub = ns.get(id);
  
  // Make requests to DO
  const res1 = await stub.fetch("http://localhost/increment");
  assert.strictEqual(await res1.text(), "1");
  
  const res2 = await stub.fetch("http://localhost/increment");
  assert.strictEqual(await res2.text(), "2");
  
  // Access storage directly
  const storage = await mf.getDurableObjectStorage(id);
  const count = await storage.get("count");
  assert.strictEqual(count, 2);
});
```

### Testing Queue Handlers

```js
test("queue message processing", async () => {
  const worker = await mf.getWorker();
  
  const result = await worker.queue("my-queue", [
    {
      id: "msg1",
      timestamp: new Date(),
      body: { userId: 123 },
      attempts: 1,
    },
  ]);
  
  assert.strictEqual(result.outcome, "ok");
  assert.strictEqual(result.ackAll, false);
  
  // Verify side effects
  const kv = await mf.getKVNamespace("QUEUE_LOG");
  const log = await kv.get("msg1");
  assert.ok(log);
});
```

### Testing Scheduled Events

```js
test("scheduled cron handler", async () => {
  const worker = await mf.getWorker();
  
  const result = await worker.scheduled({
    scheduledTime: new Date("2024-01-01T00:00:00Z"),
    cron: "0 0 * * *",
  });
  
  assert.strictEqual(result.outcome, "ok");
  
  // Verify scheduled task side effects
  const kv = await mf.getKVNamespace("JOBS");
  const lastRun = await kv.get("last-run");
  assert.ok(lastRun);
});
```

---

## Advanced Patterns

### Reloading & Updating Options

```js
const mf = new Miniflare({
  scriptPath: "worker.js",
  bindings: { VERSION: "1.0" },
});

// Update options and reload
await mf.setOptions({
  scriptPath: "worker.js",
  bindings: { VERSION: "2.0" },
});
```

### File Watching (Manual)

```js
import { watch } from "fs";
import { Miniflare } from "miniflare";

const config = {
  scriptPath: "worker.js",
  bindings: { KEY: "value" },
};

const mf = new Miniflare(config);

watch("worker.js", async () => {
  console.log("Reloading worker...");
  await mf.setOptions(config);
});
```

### Logging Configuration

```js
import { Miniflare, Log, LogLevel } from "miniflare";

const mf = new Miniflare({
  scriptPath: "worker.js",
  log: new Log(LogLevel.DEBUG), // DEBUG, INFO, WARN, ERROR
});
```

**Log Levels:**
- `LogLevel.DEBUG` - Detailed debugging info
- `LogLevel.INFO` - General information
- `LogLevel.WARN` - Warnings
- `LogLevel.ERROR` - Errors only

### Live Reload (Development)

```js
const mf = new Miniflare({
  scriptPath: "worker.js",
  liveReload: true, // Auto-reload HTML pages on worker reload
});
```

### Workers Site

```js
const mf = new Miniflare({
  scriptPath: "worker.js",
  sitePath: "./public",
  siteInclude: ["**/*.html", "**/*.css", "**/*.js"],
  siteExclude: ["node_modules/**"],
});
```

---

## Common Patterns & Best Practices

### Pattern: Shared Storage Between Workers

```js
const mf = new Miniflare({
  kvPersist: "./data",
  workers: [
    {
      name: "writer",
      kvNamespaces: { DATA: "shared" },
      script: `/* writes to KV */`,
    },
    {
      name: "reader",
      kvNamespaces: { DATA: "shared" }, // Same namespace
      script: `/* reads from KV */`,
    },
  ],
});
```

### Pattern: Testing Error Handling

```js
test("handles 404", async () => {
  const res = await mf.dispatchFetch("http://localhost/not-found");
  assert.strictEqual(res.status, 404);
});

test("handles invalid input", async () => {
  const res = await mf.dispatchFetch("http://localhost/api", {
    method: "POST",
    body: "invalid json",
  });
  assert.strictEqual(res.status, 400);
});
```

### Pattern: Isolated Test Data

```js
describe("user tests", () => {
  let mf;
  
  beforeEach(async () => {
    // Create fresh instance per test
    mf = new Miniflare({
      scriptPath: "worker.js",
      kvNamespaces: ["USERS"],
      // Use in-memory storage (no persist)
    });
  });
  
  afterEach(async () => {
    await mf.dispose(); // Clean up
  });
  
  test("create user", async () => {
    // Fresh KV namespace for this test
  });
});
```

### Pattern: Mock External Services

```js
const mf = new Miniflare({
  workers: [
    {
      name: "main",
      serviceBindings: {
        EXTERNAL_API: "mock-api",
      },
      script: `/* main worker */`,
    },
    {
      name: "mock-api",
      script: `
        addEventListener("fetch", (event) => {
          event.respondWith(
            Response.json({ mocked: true })
          );
        })
      `,
    },
  ],
});
```

### Pattern: Integration Test Setup

```js
// test-utils.js
export async function createTestWorker(overrides = {}) {
  const mf = new Miniflare({
    scriptPath: "dist/worker.js",
    kvNamespaces: ["TEST_KV"],
    r2Buckets: ["TEST_BUCKET"],
    bindings: {
      ENVIRONMENT: "test",
      ...overrides.bindings,
    },
    ...overrides,
  });
  
  await mf.ready;
  return mf;
}

// test.js
import { createTestWorker } from "./test-utils.js";

test("my test", async () => {
  const mf = await createTestWorker({
    bindings: { CUSTOM: "value" },
  });
  
  try {
    const res = await mf.dispatchFetch("http://localhost/");
    assert.ok(res.ok);
  } finally {
    await mf.dispose();
  }
});
```

---

## Troubleshooting

### Issue: Module Resolution Errors

**Problem:** `Cannot find module` errors

**Solution:**
```js
// Use absolute paths or modulesRules
const mf = new Miniflare({
  scriptPath: "./src/index.js",
  modules: true,
  modulesRules: [
    { type: "ESModule", include: ["**/*.js"], fallthrough: true },
  ],
});
```

### Issue: Persistence Not Working

**Problem:** Data not persisting between runs

**Solution:**
```js
// Ensure persist paths are set correctly
const mf = new Miniflare({
  kvPersist: "./data/kv",           // Directory, not file
  r2Persist: "./data/r2",
  durableObjectsPersist: "./data/do",
  cachePersist: "./data/cache",
});
```

### Issue: TypeScript Workers

**Problem:** Cannot directly run TypeScript

**Solution:**
```js
// Build before running tests
import { spawnSync } from "node:child_process";

before(() => {
  const result = spawnSync("npm run build", { shell: true });
  if (result.error) throw result.error;
});

// Then use built output
const mf = new Miniflare({
  scriptPath: "dist/worker.js",
});
```

### Issue: Request.cf Undefined

**Problem:** `request.cf` is undefined in worker

**Solution:**
```js
// Enable cf object fetching
const mf = new Miniflare({
  cf: true, // Fetch from Cloudflare
  // Or provide custom cf object
  cf: "./cf.json",
});
```

### Issue: Port Already in Use

**Problem:** `EADDRINUSE` error

**Solution:**
```js
// Don't specify port for testing (no HTTP server needed)
const mf = new Miniflare({
  scriptPath: "worker.js",
  // Don't set port/host - use dispatchFetch instead
});

// Use dispatchFetch, not HTTP server
const res = await mf.dispatchFetch("http://localhost/");
```

### Issue: Durable Object Not Found

**Problem:** `ReferenceError: Counter is not defined`

**Solution:**
```js
// Ensure DO class is exported from script
const mf = new Miniflare({
  modules: true, // Required for DOs
  script: `
    export class Counter { /* ... */ } // Must export
    export default { /* ... */ }
  `,
  durableObjects: {
    COUNTER: "Counter", // Must match export name
  },
});
```

---

## Complete API Reference

### Miniflare Constructor Options

```typescript
interface MiniflareOptions {
  // Script configuration
  script?: string;
  scriptPath?: string;
  modules?: boolean | Module[];
  modulesRules?: ModuleRule[];
  
  // Compatibility
  compatibilityDate?: string;
  compatibilityFlags?: string[];
  
  // Server
  host?: string;
  port?: number;
  https?: boolean;
  httpsKey?: string;
  httpsKeyPath?: string;
  httpsCert?: string;
  httpsCertPath?: string;
  
  // Request config
  cf?: boolean | string;
  upstream?: string;
  liveReload?: boolean;
  
  // Bindings
  bindings?: Record<string, any>;
  wasmBindings?: Record<string, string>;
  textBlobBindings?: Record<string, string>;
  dataBlobBindings?: Record<string, string>;
  serviceBindings?: Record<string, string | Function>;
  
  // Storage
  kvNamespaces?: string[];
  kvPersist?: boolean | string;
  
  r2Buckets?: string[];
  r2Persist?: boolean | string;
  
  durableObjects?: Record<string, string | DurableObjectOptions>;
  durableObjectsPersist?: boolean | string;
  
  d1Databases?: string[];
  d1Persist?: boolean | string;
  
  cache?: boolean;
  cachePersist?: boolean | string;
  cacheWarnUsage?: boolean;
  
  // Workers Site
  sitePath?: string;
  siteInclude?: string[];
  siteExclude?: string[];
  
  // Multiple workers
  workers?: WorkerOptions[];
  name?: string;
  routes?: string[];
  
  // Logging
  log?: Log;
}
```

### Miniflare Instance Methods

```typescript
class Miniflare {
  // Lifecycle
  constructor(options: MiniflareOptions);
  ready: Promise<void>;
  dispose(): Promise<void>;
  setOptions(options: MiniflareOptions): Promise<void>;
  
  // Event dispatching
  dispatchFetch(url: string, init?: RequestInit): Promise<Response>;
  getWorker(): Promise<Worker>;
  
  // Bindings access
  getBindings(): Promise<Record<string, any>>;
  getKVNamespace(name: string): Promise<KVNamespace>;
  getR2Bucket(name: string): Promise<R2Bucket>;
  getDurableObjectNamespace(name: string): Promise<DurableObjectNamespace>;
  getDurableObjectStorage(id: DurableObjectId): Promise<DurableObjectStorage>;
  getD1Database(name: string): Promise<D1Database>;
  getCaches(): Promise<CacheStorage>;
  getQueueProducer(name: string): Promise<QueueProducer>;
}

interface Worker {
  scheduled(options: ScheduledOptions): Promise<ScheduledResult>;
  queue(name: string, messages: QueueMessage[]): Promise<QueueResult>;
}
```

---

## Key Differences from Production

### Behavior Differences

1. **No actual Cloudflare edge:** Runs in workerd locally, not on Cloudflare's global network
2. **Persistence:** Storage is local filesystem or in-memory, not distributed
3. **Request.cf object:** Fetched from cached endpoint or mocked, not real edge metadata
4. **Performance:** Local performance doesn't reflect edge performance
5. **Caching:** Cache behavior may differ slightly from production

### Not Supported in Miniflare

- Cloudflare Analytics Engine
- Cloudflare Images
- Live production data
- True global distribution
- Some advanced Workers features (check docs for specifics)

---

## Migration Notes

### From Wrangler Dev to Miniflare

**Wrangler (development):**
```bash
wrangler dev
```

**Miniflare (testing):**
```js
const mf = new Miniflare({
  scriptPath: "dist/worker.js",
  // Manually configure all bindings from wrangler.toml
  kvNamespaces: ["KV"],
  bindings: { API_KEY: "..." },
});
```

**Note:** Miniflare doesn't read `wrangler.toml` - configure everything via API.

### From Miniflare 2 to Miniflare 3

Major changes:
- Different API surface
- Better workerd integration
- Changed persistence options
- See [official migration guide](https://developers.cloudflare.com/workers/testing/vitest-integration/migration-guides/migrate-from-miniflare-2/)

---

## Additional Resources

- [Miniflare Docs](https://developers.cloudflare.com/workers/testing/miniflare/)
- [Miniflare GitHub](https://github.com/cloudflare/workers-sdk/tree/main/packages/miniflare)
- [Workers Docs](https://developers.cloudflare.com/workers/)
- [Vitest Integration](https://developers.cloudflare.com/workers/testing/vitest-integration/) (recommended for most users)
- [wrangler CLI](https://developers.cloudflare.com/workers/wrangler/)

---

## When to Use This Skill

**Use this skill when:**
- Setting up integration tests for Cloudflare Workers
- Need direct access to Worker bindings in tests
- Testing multiple Workers with service bindings
- Advanced use cases requiring fine-grained control over Worker execution
- Need to dispatch events without HTTP requests
- Testing storage operations (KV, R2, Durable Objects, D1)

**Don't use this skill for:**
- Standard Wrangler development workflow questions
- Cloudflare platform questions (DNS, CDN, etc.)
- Workers runtime API questions (use Workers docs)
- Production deployment questions
- Questions about other Cloudflare products

This skill focuses ONLY on Miniflare - the local development simulator. For broader Cloudflare Workers questions, consult the Workers documentation.
