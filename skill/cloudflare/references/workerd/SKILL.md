# Cloudflare Workerd Runtime Skill

**Description**: Expert guidance for Cloudflare Workerd - the JavaScript/Wasm runtime powering Cloudflare Workers. Covers configuration, architecture, bindings, APIs, and common patterns. SPECIFICALLY for the Workerd runtime itself, NOT the full Cloudflare Workers platform.

**When to use**: Building/configuring Workerd applications, debugging Workers runtime issues, setting up local development, working with workerd.capnp configs, or implementing Workerd-specific patterns.

---

## Core Concepts

### What is Workerd?

Workerd is:
- JavaScript/Wasm server runtime based on V8
- The same code that powers Cloudflare Workers
- Usable as: application server, dev tool, or programmable HTTP proxy
- Standards-based (web platform APIs like `fetch()`)
- Capability-oriented (explicit bindings vs global namespaces)

### Key Design Principles

1. **Server-first**: Built for servers, not CLIs/GUIs
2. **Standards-based**: Web platform APIs (`fetch()`, Web Crypto, etc.)
3. **Nanoservices**: Microservices with local function call performance
4. **Homogeneous deployment**: Deploy all nanoservices everywhere
5. **Capability bindings**: Explicit connections, immune to SSRF
6. **Always backwards compatible**: Version number = max compatibility date

### Architecture Overview

```
Config (workerd.capnp)
├── Services (named workers/endpoints)
│   ├── Worker (JS/Wasm code + bindings)
│   ├── Network (internet access)
│   ├── ExternalServer (proxy to backend)
│   └── DiskDirectory (static files)
├── Sockets (HTTP/HTTPS listeners)
└── Extensions (global capabilities)
```

---

## Configuration (workerd.capnp)

### Basic Structure

```capnp
using Workerd = import "/workerd/workerd.capnp";

const config :Workerd.Config = (
  services = [
    (name = "main", worker = .mainWorker),
  ],
  sockets = [
    (name = "http", address = "*:8080", http = (), service = "main"),
  ]
);

const mainWorker :Workerd.Worker = (
  serviceWorkerScript = embed "worker.js",
  compatibilityDate = "2024-01-15",
);
```

### Services

Services are named endpoints that can be:

**Worker Service**:
```capnp
(name = "api", worker = (
  modules = [
    (name = "index.js", esModule = embed "index.js"),
  ],
  compatibilityDate = "2024-01-15",
  bindings = [...],
))
```

**Network Service** (internet access):
```capnp
(name = "internet", network = (
  allow = ["public"],
  tlsOptions = (trustBrowserCas = true)
))
```

**External Server** (reverse proxy):
```capnp
(name = "backend", external = (
  address = "backend.example.com:443",
  http = (style = tls)
))
```

**Disk Directory** (static files):
```capnp
(name = "assets", disk = (
  path = "/var/www/static",
  writable = false
))
```

### Sockets

HTTP socket:
```capnp
(name = "http", address = "*:8080", http = (), service = "main")
```

HTTPS socket:
```capnp
(
  name = "https",
  address = "*:443",
  https = (
    options = (),
    tlsOptions = (
      keypair = (
        privateKey = embed "key.pem",
        certificateChain = embed "cert.pem"
      )
    )
  ),
  service = "main"
)
```

Unix socket:
```capnp
(name = "app", address = "unix:/tmp/app.sock", http = (), service = "main")
```

### Worker Formats

**Service Worker Syntax**:
```capnp
serviceWorkerScript = embed "worker.js"
```

**ES Modules Syntax** (recommended):
```capnp
modules = [
  (name = "index.js", esModule = embed "src/index.js"),
  (name = "utils.js", esModule = embed "src/utils.js"),
  (name = "wasm.wasm", wasm = embed "build/module.wasm"),
  (name = "data.json", json = embed "data.json"),
]
```

**CommonJS Modules**:
```capnp
(name = "legacy.js", commonJsModule = embed "legacy.js", namedExports = ["foo", "bar"])
```

---

## Bindings

Bindings provide capability-based access to resources. For ES modules, available as `env` object; for service workers, as globals.

### Common Binding Types

**Text/Data/JSON**:
```capnp
bindings = [
  (name = "API_KEY", text = "secret-key"),
  (name = "CONFIG", json = embed "config.json"),
  (name = "WASM_DATA", data = embed "data.bin"),
]
```

**Environment Variables** (from system):
```capnp
(name = "DATABASE_URL", fromEnvironment = "DATABASE_URL")
```

**Service Binding** (nanoservices):
```capnp
(name = "AUTH_SERVICE", service = "auth-worker")
```

**Service with Entrypoint**:
```capnp
(name = "API", service = (
  name = "backend",
  entrypoint = "adminApi",  # Named export
  props = (json = '{"role":"admin"}')  # ctx.props
))
```

**Durable Objects**:
```capnp
(name = "ROOM", durableObjectNamespace = (
  serviceName = "room-service",
  className = "Room"
))
```

**KV Namespace**:
```capnp
(name = "CACHE", kvNamespace = "kv-service")
```

**R2 Bucket**:
```capnp
(name = "STORAGE", r2Bucket = "r2-service")
```

**Queue**:
```capnp
(name = "TASKS", queue = "queue-service")
```

**Analytics Engine**:
```capnp
(name = "ANALYTICS", analyticsEngine = "analytics-service")
```

**Memory Cache**:
```capnp
(name = "FAST_CACHE", memoryCache = (
  id = "shared-cache",
  limits = (maxKeys = 1000, maxValueSize = 1048576)
))
```

**Worker Loader** (dynamic loading):
```capnp
(name = "LOADER", workerLoader = (id = "dynamic-workers"))
```

**Wrapped Bindings** (middleware):
```capnp
(name = "TRACED_API", wrapped = (
  moduleName = "tracing-module",
  entrypoint = "makeTracer",
  innerBindings = [
    (name = "backend", service = "backend-service")
  ]
))
```

**Crypto Key**:
```capnp
(name = "SIGNING_KEY", cryptoKey = (
  format = raw,
  algorithm = (name = "HMAC", hash = "SHA-256"),
  keyData = embed "key.bin",
  usages = [sign, verify],
  extractable = false
))
```

**Unsafe Eval** (restricted):
```capnp
(name = "UnsafeEval", unsafeEval = ())
```

### Parameter Bindings (for inheritance)

```capnp
# Base worker with parameters
const baseWorker :Workerd.Worker = (
  modules = [(name = "index.js", esModule = embed "base.js")],
  compatibilityDate = "2024-01-15",
  bindings = [
    (name = "API_ENDPOINT", parameter = (type = text)),
    (name = "DATABASE", parameter = (type = service)),
  ]
);

# Derived worker fills in parameters
const derivedWorker :Workerd.Worker = (
  inherit = "base-service",
  bindings = [
    (name = "API_ENDPOINT", text = "https://api.example.com"),
    (name = "DATABASE", service = "postgres-service"),
  ]
);
```

---

## Compatibility Dates & Flags

### Setting Compatibility Date

```capnp
compatibilityDate = "2024-01-15"
```

Always set to current date for new projects. Update carefully by reviewing [compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/).

### Common Compatibility Flags

```capnp
compatibilityFlags = [
  "nodejs_compat",           # Node.js API compatibility
  "streams_enable_constructors",
  "transformstream_enable_standard_constructor",
]
```

### Backwards Compatibility

- Workerd supports old compatibility dates forever
- Version number = compatibility date (e.g., v1.20240115.0)
- Update code when bumping compatibilityDate
- No breaking changes without developer contact

---

## Common Patterns

### Multi-Service Architecture

```capnp
const config :Workerd.Config = (
  services = [
    # Frontend
    (name = "frontend", worker = (
      modules = [(name = "index.js", esModule = embed "frontend/index.js")],
      compatibilityDate = "2024-01-15",
      bindings = [
        (name = "API", service = "api"),
      ]
    )),
    
    # API Backend
    (name = "api", worker = (
      modules = [(name = "index.js", esModule = embed "api/index.js")],
      compatibilityDate = "2024-01-15",
      bindings = [
        (name = "DB", service = "postgres"),
        (name = "CACHE", kvNamespace = "kv"),
      ]
    )),
    
    # Database proxy
    (name = "postgres", external = (
      address = "db.internal:5432",
      http = ()
    )),
    
    # KV storage
    (name = "kv", disk = (path = "/var/kv", writable = true)),
  ],
  
  sockets = [
    (name = "http", address = "*:8080", http = (), service = "frontend"),
  ]
);
```

### Durable Objects Setup

```capnp
const config :Workerd.Config = (
  services = [
    (name = "app", worker = (
      modules = [
        (name = "index.js", esModule = embed "index.js"),
        (name = "room.js", esModule = embed "room.js"),
      ],
      compatibilityDate = "2024-01-15",
      bindings = [
        (name = "ROOMS", durableObjectNamespace = "Room"),
      ],
      durableObjectNamespaces = [
        (className = "Room", uniqueKey = "app-v1"),
      ],
      durableObjectStorage = (localDisk = "/var/do"),
    )),
  ],
  sockets = [
    (name = "http", address = "*:8080", http = (), service = "app"),
  ]
);
```

### Development vs Production

```capnp
# Development config
const devWorker :Workerd.Worker = (
  modules = [(name = "index.js", esModule = embed "src/index.js")],
  compatibilityDate = "2024-01-15",
  bindings = [
    (name = "API_URL", text = "http://localhost:3000"),
    (name = "DEBUG", text = "true"),
  ]
);

# Production inherits and overrides
const prodWorker :Workerd.Worker = (
  inherit = "dev-service",
  bindings = [
    (name = "API_URL", text = "https://api.production.com"),
    (name = "DEBUG", text = "false"),
  ]
);
```

### External HTTP Proxy

```capnp
const config :Workerd.Config = (
  services = [
    (name = "proxy", worker = (
      serviceWorkerScript = embed "proxy.js",
      compatibilityDate = "2024-01-15",
      bindings = [
        (name = "BACKEND", service = "backend"),
      ]
    )),
    (name = "backend", external = (
      address = "internal-service:8080",
      http = ()
    )),
  ],
  sockets = [
    (name = "http", address = "*:80", http = (), service = "proxy"),
  ]
);
```

---

## Worker Code Patterns

### ES Modules (Recommended)

```javascript
// index.js
export default {
  async fetch(request, env, ctx) {
    // env contains bindings
    const value = await env.KV.get("key");
    const response = await env.API_SERVICE.fetch(request);
    
    // ctx.waitUntil for background tasks
    ctx.waitUntil(logRequest(request));
    
    return new Response("Hello");
  },
  
  // Named entrypoint
  async adminApi(request, env, ctx) {
    return new Response("Admin API");
  },
  
  // Queue consumer
  async queue(batch, env, ctx) {
    for (const msg of batch.messages) {
      await processMessage(msg.body);
    }
  },
  
  // Scheduled handler
  async scheduled(event, env, ctx) {
    ctx.waitUntil(runScheduledTask(env));
  }
};

async function logRequest(req) {
  // Background task
}
```

### Service Worker Syntax

```javascript
// worker.js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  // Bindings are globals
  const value = await KV.get("key");
  const response = await API_SERVICE.fetch(request);
  return new Response("Hello");
}
```

### Durable Objects

```javascript
// room.js
export class Room {
  constructor(state, env) {
    this.state = state;
    this.env = env;
  }
  
  async fetch(request) {
    const url = new URL(request.url);
    
    if (url.pathname === "/state") {
      const value = await this.state.storage.get("counter");
      return new Response(value || "0");
    }
    
    if (url.pathname === "/increment") {
      const value = (await this.state.storage.get("counter")) || 0;
      await this.state.storage.put("counter", value + 1);
      return new Response(String(value + 1));
    }
    
    return new Response("Not found", {status: 404});
  }
}
```

### RPC between Services

```javascript
// api.js
export default {
  async fetch(request, env, ctx) {
    // Call another service via RPC
    const authService = env.AUTH;
    const user = await authService.validateToken(request.headers.get("Authorization"));
    
    return new Response(`Hello ${user.name}`);
  }
};

// auth.js
export default {
  async validateToken(token) {
    // Return structured data directly
    return {id: 123, name: "Alice"};
  }
};
```

### Accessing Bindings

```typescript
// TypeScript interface for bindings
interface Env {
  // Service bindings
  API: Fetcher;
  
  // KV/R2/DO
  CACHE: KVNamespace;
  STORAGE: R2Bucket;
  ROOMS: DurableObjectNamespace;
  
  // Primitives
  API_KEY: string;
  CONFIG: {apiUrl: string; debug: boolean};
  
  // Special
  LOADER: WorkerLoader;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Type-safe access
    const data = await env.CACHE.get("key");
    return new Response(data);
  }
};
```

---

## CLI Commands

### Serve Configuration

```bash
# Basic serve
workerd serve config.capnp

# With named constant
workerd serve config.capnp myConfig

# Override socket address
workerd serve config.capnp --socket-addr http=*:3000

# Inherit socket from parent (systemd)
workerd serve config.capnp --socket-fd http=3

# Verbose logging
workerd serve config.capnp --verbose

# Set compatibility date from CLI
workerd serve config.capnp --compat-date=2024-01-15
```

### Compile Binary

```bash
# Create standalone binary
workerd compile config.capnp constantName -o my-server

# Run compiled binary
./my-server
```

### Test Workers

```bash
# Run tests
workerd test config.capnp

# Specific test file
workerd test config.capnp --test-only=my-test.js
```

---

## Integration with Wrangler

### Local Development

```bash
# Use custom workerd binary
export MINIFLARE_WORKERD_PATH="/path/to/workerd"

# Run wrangler dev
wrangler dev
```

### Wrangler Config (wrangler.toml)

```toml
name = "my-worker"
main = "src/index.js"
compatibility_date = "2024-01-15"
compatibility_flags = ["nodejs_compat"]

[[kv_namespaces]]
binding = "CACHE"
id = "abc123"

[[r2_buckets]]
binding = "STORAGE"
bucket_name = "my-bucket"

[[durable_objects.bindings]]
name = "ROOMS"
class_name = "Room"
script_name = "my-worker"

[env.production]
vars = { API_URL = "https://api.prod.com" }
```

---

## Production Deployment

### Systemd Service

```ini
# /etc/systemd/system/workerd.service
[Unit]
Description=workerd runtime
After=network-online.target
Requires=workerd.socket

[Service]
Type=exec
ExecStart=/usr/bin/workerd serve /etc/workerd/config.capnp \
  --socket-fd http=3 --socket-fd https=4
Restart=always
User=nobody
Group=nogroup
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/workerd.socket
[Unit]
Description=sockets for workerd
PartOf=workerd.service

[Socket]
ListenStream=0.0.0.0:80
ListenStream=0.0.0.0:443

[Install]
WantedBy=sockets.target
```

### Docker

```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates
COPY workerd /usr/local/bin/
COPY config.capnp /etc/workerd/
COPY src/ /etc/workerd/src/
EXPOSE 8080
CMD ["workerd", "serve", "/etc/workerd/config.capnp"]
```

### Environment Variables

```capnp
# Use system env vars
bindings = [
  (name = "DATABASE_URL", fromEnvironment = "DATABASE_URL"),
  (name = "API_KEY", fromEnvironment = "API_KEY"),
]
```

```bash
# Set in environment
export DATABASE_URL="postgres://..."
export API_KEY="secret"
workerd serve config.capnp
```

---

## Runtime APIs

Workerd implements web standard APIs. Key categories:

### Fetch API
- `fetch()` - HTTP requests
- `Request`, `Response`, `Headers`
- `AbortController`, `AbortSignal`

### Streams API
- `ReadableStream`, `WritableStream`, `TransformStream`
- Byte streams, BYOB readers

### Web Crypto
- `crypto.subtle` - cryptographic operations
- `crypto.randomUUID()`, `crypto.getRandomValues()`

### Encoding
- `TextEncoder`, `TextDecoder`
- `atob()`, `btoa()`

### Web Standards
- `URL`, `URLSearchParams`
- `Blob`, `File`, `FormData`
- `WebSocket`
- `EventSource` (SSE)
- `HTMLRewriter`

### Performance
- `performance.now()`, `performance.timeOrigin`
- `setTimeout()`, `setInterval()`, `queueMicrotask()`

### Console
- `console.log()`, `console.error()`, etc.

### Node.js Compatibility
When `nodejs_compat` flag enabled:
- `node:*` module imports
- `process.env`, `Buffer`
- Subset of Node.js APIs

---

## Debugging & Logging

### Structured Logging

```capnp
logging = (
  structuredLogging = true,
  stdoutPrefix = "OUT: ",
  stderrPrefix = "ERR: "
)
```

### Console Logging

```javascript
export default {
  async fetch(request, env, ctx) {
    console.log("Request received", {
      method: request.method,
      url: request.url
    });
    
    console.error("Error occurred", error);
    console.warn("Warning message");
    
    return new Response("OK");
  }
};
```

### V8 Flags

```capnp
v8Flags = [
  "--expose-gc",           # Expose garbage collector
  "--max-old-space-size=2048"  # Heap size
]
```

**Warning**: Use V8 flags at your own risk. Can break everything.

---

## Common Pitfalls

### 1. Compatibility Date
❌ **Wrong**: Omitting compatibility date
```capnp
const worker :Workerd.Worker = (
  serviceWorkerScript = embed "worker.js"
)
```

✅ **Correct**: Always specify
```capnp
const worker :Workerd.Worker = (
  serviceWorkerScript = embed "worker.js",
  compatibilityDate = "2024-01-15"
)
```

### 2. Global Outbound
❌ **Wrong**: Assuming fetch() works without config
```javascript
await fetch("https://api.example.com")  // May not work
```

✅ **Correct**: Configure network service
```capnp
bindings = [
  (name = "API", service = (
    name = "internet",
    network = (allow = ["public"])
  ))
]
```

### 3. Binding Types
❌ **Wrong**: Mixing binding types
```capnp
(name = "CONFIG", text = '{"key":"value"}')  # String, not parsed
```

✅ **Correct**: Use json for objects
```capnp
(name = "CONFIG", json = '{"key":"value"}')  # Parsed object
```

### 4. Service vs Namespace
❌ **Wrong**: Confusing service binding with DO namespace
```capnp
(name = "ROOM", service = "room-service")  # Just a Fetcher
```

✅ **Correct**: Use durableObjectNamespace
```capnp
(name = "ROOM", durableObjectNamespace = "Room")
```

### 5. Module Names
❌ **Wrong**: Incorrect module references
```capnp
modules = [(name = "src/index.js", esModule = embed "src/index.js")]
```

✅ **Correct**: Use consistent naming
```capnp
modules = [(name = "index.js", esModule = embed "src/index.js")]
```

---

## Best Practices

### 1. Use ES Modules
Prefer modules over service worker syntax:
```javascript
export default {
  async fetch(request, env, ctx) {
    // Modern, typed, composable
  }
};
```

### 2. Explicit Bindings
Never rely on global namespace. Use capability bindings:
```capnp
bindings = [
  (name = "DB", service = "postgres"),
  (name = "CACHE", kvNamespace = "kv"),
]
```

### 3. Type Safety
Define TypeScript interfaces for bindings:
```typescript
interface Env {
  DB: Fetcher;
  CACHE: KVNamespace;
  API_KEY: string;
}
```

### 4. Service Isolation
Split concerns into separate services:
```capnp
services = [
  (name = "frontend", worker = ...),
  (name = "api", worker = ...),
  (name = "admin", worker = ...),
]
```

### 5. Compatibility Date Strategy
- New projects: Use current date
- Updates: Review flags, test carefully
- Production: Pin to tested date
- Never skip compatibility date

### 6. Background Tasks
Use `ctx.waitUntil()` for async work:
```javascript
ctx.waitUntil(
  logToAnalytics(request)
);
```

### 7. Error Handling
Handle errors gracefully:
```javascript
try {
  return await handleRequest(request, env);
} catch (error) {
  console.error("Request failed", error);
  return new Response("Internal Error", {status: 500});
}
```

### 8. Resource Limits
Configure appropriate limits:
```capnp
memoryCache = (
  id = "cache",
  limits = (
    maxKeys = 1000,
    maxValueSize = 1048576,  # 1MB
    maxTotalValueSize = 10485760  # 10MB
  )
)
```

---

## Security Considerations

### 1. Capability-Based Access
Only grant necessary bindings:
```capnp
# ❌ Don't grant everything
bindings = [
  (name = "EVERYTHING", service = "internet")
]

# ✅ Grant specific services
bindings = [
  (name = "API", service = "api-backend"),
  (name = "CACHE", kvNamespace = "cache-only"),
]
```

### 2. Secret Management
Never hardcode secrets in config:
```capnp
# ❌ Wrong
(name = "API_KEY", text = "secret-key-here")

# ✅ Correct
(name = "API_KEY", fromEnvironment = "API_KEY")
```

### 3. Network Restrictions
Limit network access:
```capnp
network = (
  allow = ["public"],  # Or specific CIDR blocks
  deny = ["local"]
)
```

### 4. TLS Configuration
Use secure TLS settings:
```capnp
tlsOptions = (
  trustBrowserCas = true,
  minVersion = tls12,
  cipherList = "HIGH:!aNULL:!MD5"
)
```

### 5. Crypto Keys
Use non-extractable keys:
```capnp
cryptoKey = (
  ...,
  extractable = false,  # Prevent key extraction
  usages = [sign, verify]
)
```

---

## Troubleshooting

### Issue: Worker not responding
**Check**:
1. Socket configuration correct?
2. Service name matches?
3. Worker has fetch handler?
4. Port available/not blocked?

### Issue: Binding not found
**Check**:
1. Binding name matches code?
2. Service exists in config?
3. Module vs service worker syntax?

### Issue: Module not found
**Check**:
1. Module name in config matches import?
2. File path correct in embed?
3. ES module syntax valid?

### Issue: Compatibility errors
**Check**:
1. Compatibility date set?
2. API available on that date?
3. Flags needed?

### Issue: Performance problems
**Try**:
1. Enable --verbose logging
2. Check V8 flags
3. Review memory limits
4. Profile with DevTools

---

## Reference Links

- [Workerd GitHub](https://github.com/cloudflare/workerd)
- [Workers Documentation](https://developers.cloudflare.com/workers/)
- [Compatibility Dates](https://developers.cloudflare.com/workers/configuration/compatibility-dates/)
- [Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/)
- [workerd.capnp Schema](https://github.com/cloudflare/workerd/blob/main/src/workerd/server/workerd.capnp)

---

## Quick Reference

### Most Common Config Patterns

```capnp
# Simple HTTP worker
const config :Workerd.Config = (
  services = [(name = "main", worker = .mainWorker)],
  sockets = [(name = "http", address = "*:8080", http = (), service = "main")]
);

const mainWorker :Workerd.Worker = (
  modules = [(name = "index.js", esModule = embed "src/index.js")],
  compatibilityDate = "2024-01-15",
  bindings = [
    (name = "API_KEY", fromEnvironment = "API_KEY"),
    (name = "CACHE", kvNamespace = "kv"),
  ]
);

# Multi-service with DO
const config :Workerd.Config = (
  services = [
    (name = "app", worker = (
      modules = [(name = "index.js", esModule = embed "index.js")],
      compatibilityDate = "2024-01-15",
      bindings = [(name = "ROOMS", durableObjectNamespace = "Room")],
      durableObjectNamespaces = [(className = "Room", uniqueKey = "v1")],
      durableObjectStorage = (localDisk = "/var/do")
    ))
  ],
  sockets = [(name = "http", address = "*:8080", http = (), service = "app")]
);
```

---

## Skill Metadata

- **Workerd Version Coverage**: v1.20240101.0 - v1.20260111.0
- **Last Updated**: 2026-01-11
- **Focuses**: Configuration, Architecture, Bindings, Patterns
- **NOT Covered**: Full Cloudflare Workers platform features (Pages, D1, Vectorize, AI)
