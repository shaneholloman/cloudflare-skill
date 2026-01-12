# Cloudflare Wrangler Skill

## Overview
Comprehensive guide for Cloudflare Wrangler CLI - the command-line interface for managing, developing, and deploying Cloudflare Workers. This skill covers configuration, commands, API usage, bindings, and best practices.

## Core Concepts

### What is Wrangler?
Wrangler is the official Cloudflare Developer Platform CLI that allows you to:
- Create, develop, and deploy Workers
- Manage bindings (KV, D1, R2, Durable Objects, etc.)
- Configure routing and environments
- Run local development servers
- Execute migrations and manage resources
- Perform integration testing

### Installation
```bash
npm install wrangler --save-dev
# or globally
npm install -g wrangler
```

Run commands via package manager:
```bash
# npm
npx wrangler <command>

# pnpm
pnpm wrangler <command>

# yarn
yarn wrangler <command>
```

## Configuration

### wrangler.jsonc vs wrangler.toml
As of v3.91.0, Wrangler supports both JSON and TOML config formats. **JSON (wrangler.jsonc) is recommended for new projects.**

#### wrangler.jsonc (Recommended)
```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.js",
  "compatibility_date": "2025-01-01",
  "compatibility_flags": [],
  "workers_dev": true,
  "route": {
    "pattern": "example.org/*",
    "zone_name": "example.org"
  },
  "vars": {
    "API_KEY": "dev-key",
    "DEBUG": true
  },
  "kv_namespaces": [
    {
      "binding": "MY_KV",
      "id": "abc123",
      "preview_id": "def456"
    }
  ],
  "env": {
    "staging": {
      "name": "my-worker-staging",
      "vars": {
        "API_KEY": "staging-key"
      }
    },
    "production": {
      "name": "my-worker-prod",
      "vars": {
        "API_KEY": "prod-key"
      }
    }
  }
}
```

#### wrangler.toml
```toml
name = "my-worker"
main = "src/index.js"
compatibility_date = "2025-01-01"
workers_dev = true

[vars]
API_KEY = "dev-key"
DEBUG = true

[[kv_namespaces]]
binding = "MY_KV"
id = "abc123"
preview_id = "def456"

[env.staging]
name = "my-worker-staging"
vars = { API_KEY = "staging-key" }

[env.production]
name = "my-worker-prod"
vars = { API_KEY = "prod-key" }
```

### Key Configuration Fields

#### Required Fields
```jsonc
{
  "name": "my-worker",              // Worker name (alphanumeric + dashes)
  "main": "src/index.ts",           // Entrypoint file
  "compatibility_date": "2025-01-01" // Runtime version
}
```

#### Top-Level Only Keys (Cannot be overridden per environment)
- `keep_vars`: Preserve dashboard variables on deploy
- `migrations`: Durable Object migrations
- `send_metrics`: Wrangler telemetry (default: true)
- `site`: Workers Sites config (deprecated - use Pages/Assets)

#### Inheritable Keys (Can be overridden per environment)
- `name`, `main`, `compatibility_date`
- `account_id`: Account ID or use `CLOUDFLARE_ACCOUNT_ID` env var
- `workers_dev`: Enable `*.workers.dev` subdomain
- `preview_urls`: Enable preview URLs (default: same as workers_dev)
- `route` / `routes`: Routing configuration
- `tsconfig`: Custom TypeScript config path
- `triggers`: Cron schedules
- `build`: Custom build configuration
- `minify`: Minify script before upload
- `logpush`: Enable Workers Logpush
- `limits`: Runtime limits (CPU, etc.)
- `observability`: Logging configuration
- `assets`: Static assets configuration

#### Non-Inheritable Keys (Must be specified per environment)
- `define`: Value substitution map
- `vars`: Environment variables
- `durable_objects`: Durable Object bindings
- `kv_namespaces`: KV bindings
- `r2_buckets`: R2 bindings
- `vectorize`: Vectorize index bindings
- `services`: Service bindings
- `tail_consumers`: Tail Worker bindings

### Environments
Configure different environments (dev, staging, production):

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-01",
  "vars": {
    "ENVIRONMENT": "development"
  },
  "env": {
    "staging": {
      "name": "my-worker-staging",
      "vars": {
        "ENVIRONMENT": "staging"
      },
      "route": {
        "pattern": "staging.example.com/*",
        "zone_name": "example.com"
      }
    },
    "production": {
      "name": "my-worker-prod",
      "vars": {
        "ENVIRONMENT": "production"
      },
      "route": {
        "pattern": "example.com/*",
        "zone_name": "example.com"
      }
    }
  }
}
```

Deploy to specific environment:
```bash
wrangler deploy --env staging
wrangler deploy --env production
```

### Routing Options

#### 1. Custom Domains (Recommended)
```jsonc
{
  "routes": [
    {
      "pattern": "shop.example.com",
      "custom_domain": true
    }
  ]
}
```

#### 2. Zone-Based Routes
```jsonc
{
  "routes": [
    {
      "pattern": "subdomain.example.com/*",
      "zone_id": "YOUR_ZONE_ID"
    }
  ]
}
```

Or using zone name:
```jsonc
{
  "routes": [
    {
      "pattern": "subdomain.example.com/*",
      "zone_name": "example.com"
    }
  ]
}
```

#### 3. workers.dev Subdomain
```jsonc
{
  "workers_dev": true  // Enables *.workers.dev subdomain
}
```

### Automatic Resource Provisioning
**Beta feature** - Wrangler can auto-provision KV, R2, and D1 resources:

```jsonc
{
  "kv_namespaces": [
    {
      "binding": "MY_KV"  // No ID needed - auto-created on deploy
    }
  ],
  "r2_buckets": [
    {
      "binding": "MY_BUCKET"  // No bucket_name needed
    }
  ],
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-database"
      // No database_id - auto-created on deploy
    }
  ]
}
```

IDs are written back to config file on deploy.

## Essential Commands

### Project Initialization
```bash
# Create new project (launches C3 - create-cloudflare-cli)
wrangler init [name] [--yes]

# Fetch Worker from dashboard
wrangler init --from-dash worker-name
```

### Development
```bash
# Start local dev server
wrangler dev

# Dev with remote resources
wrangler dev --remote

# Dev with specific environment
wrangler dev --env staging

# Dev on specific port
wrangler dev --port 8787

# Dev with live reload
wrangler dev --live-reload
```

### Deployment
```bash
# Deploy to production
wrangler deploy

# Deploy to specific environment
wrangler deploy --env staging

# Deploy with dry run (no actual deploy)
wrangler deploy --dry-run

# Deploy keeping dashboard variables
wrangler deploy --keep-vars
```

### Version Management
```bash
# List recent versions
wrangler versions list

# View specific version
wrangler versions view <version-id>

# List deployments
wrangler deployments list

# Rollback to previous deployment
wrangler rollback [deployment-id]
```

### Resource Management

#### KV Operations
```bash
# Create namespace
wrangler kv namespace create MY_KV
wrangler kv namespace create MY_KV --env production

# List namespaces
wrangler kv namespace list

# Delete namespace
wrangler kv namespace delete --namespace-id=<id>

# Put key
wrangler kv key put "key" "value" --namespace-id=<id>

# Get key
wrangler kv key get "key" --namespace-id=<id>

# Delete key
wrangler kv key delete "key" --namespace-id=<id>

# List keys
wrangler kv key list --namespace-id=<id>

# Bulk operations
wrangler kv bulk put data.json --namespace-id=<id>
wrangler kv bulk delete keys.json --namespace-id=<id>
```

#### D1 Operations
```bash
# Create database
wrangler d1 create my-database

# List databases
wrangler d1 list

# Get database info
wrangler d1 info my-database

# Execute SQL
wrangler d1 execute my-database --command "SELECT * FROM users"
wrangler d1 execute my-database --file schema.sql

# Remote execution
wrangler d1 execute my-database --remote --command "SELECT * FROM users"

# Local execution (for dev)
wrangler d1 execute my-database --local --command "SELECT * FROM users"

# Export database
wrangler d1 export my-database --output backup.sql

# Migrations
wrangler d1 migrations create my-database "create_users_table"
wrangler d1 migrations list my-database
wrangler d1 migrations apply my-database

# Time Travel (restore to point in time)
wrangler d1 time-travel info my-database --timestamp "2025-01-01T00:00:00Z"
wrangler d1 time-travel restore my-database --timestamp "2025-01-01T00:00:00Z"
```

#### R2 Operations
```bash
# Create bucket
wrangler r2 bucket create my-bucket

# List buckets
wrangler r2 bucket list

# Delete bucket
wrangler r2 bucket delete my-bucket

# Put object
wrangler r2 object put my-bucket/file.txt --file local-file.txt

# Get object
wrangler r2 object get my-bucket/file.txt

# Delete object
wrangler r2 object delete my-bucket/file.txt

# List objects
wrangler r2 object list my-bucket
```

#### Durable Objects
```bash
# List Durable Objects
wrangler durable-objects namespace list

# Delete Durable Object instance
wrangler durable-objects delete-namespace <namespace-id>
```

#### Queues
```bash
# Create queue
wrangler queues create my-queue

# List queues
wrangler queues list

# Delete queue
wrangler queues delete my-queue

# Consumer configuration in wrangler.jsonc
{
  "queues": {
    "producers": [
      {
        "binding": "MY_QUEUE",
        "queue": "my-queue",
        "delivery_delay": 60
      }
    ],
    "consumers": [
      {
        "queue": "my-queue",
        "max_batch_size": 10,
        "max_batch_timeout": 30,
        "max_retries": 3,
        "dead_letter_queue": "my-dlq"
      }
    ]
  }
}
```

#### Vectorize
```bash
# Create index
wrangler vectorize create my-index --dimensions 768 --metric cosine

# List indexes
wrangler vectorize list

# Get index info
wrangler vectorize get my-index

# Delete index
wrangler vectorize delete my-index
```

#### Hyperdrive
```bash
# Create config
wrangler hyperdrive create my-hyperdrive \
  --connection-string "postgresql://user:pass@host:5432/db"

# List configs
wrangler hyperdrive list

# Get config
wrangler hyperdrive get <id>

# Update config
wrangler hyperdrive update <id> --origin-host new-host

# Delete config
wrangler hyperdrive delete <id>
```

### Secrets Management
```bash
# Set secret (interactive)
wrangler secret put SECRET_NAME

# Set secret (from stdin)
echo "secret-value" | wrangler secret put SECRET_NAME

# List secrets
wrangler secret list

# Delete secret
wrangler secret delete SECRET_NAME

# Bulk operations
wrangler secret bulk secrets.json
```

### Logging & Monitoring
```bash
# Tail logs in real-time
wrangler tail

# Tail with filters
wrangler tail --format pretty
wrangler tail --status error
wrangler tail --method POST
wrangler tail --sampling-rate 0.5

# Tail specific environment
wrangler tail --env production
```

### Authentication
```bash
# Login via OAuth
wrangler login

# Logout
wrangler logout

# Check authentication
wrangler whoami
```

## Bindings Configuration

### Environment Variables
```jsonc
{
  "vars": {
    "API_URL": "https://api.example.com",
    "API_VERSION": "v1",
    "FEATURE_FLAGS": {
      "newFeature": true,
      "betaAccess": false
    }
  }
}
```

Access in Worker:
```typescript
export default {
  async fetch(request: Request, env: Env) {
    console.log(env.API_URL);
    console.log(env.FEATURE_FLAGS.newFeature);
    return new Response("OK");
  }
};
```

### KV Namespaces
```jsonc
{
  "kv_namespaces": [
    {
      "binding": "CACHE",
      "id": "abc123",
      "preview_id": "def456"
    },
    {
      "binding": "SESSIONS",
      "id": "ghi789"
    }
  ]
}
```

Usage:
```typescript
export default {
  async fetch(request: Request, env: Env) {
    // Get value
    const cached = await env.CACHE.get("key");
    
    // Put value
    await env.CACHE.put("key", "value", {
      expirationTtl: 3600
    });
    
    // Delete
    await env.CACHE.delete("key");
    
    // List keys
    const keys = await env.CACHE.list({ prefix: "user:" });
    
    return new Response(cached);
  }
};
```

### D1 Databases
```jsonc
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "production-db",
      "database_id": "abc-123-def",
      "preview_database_id": "xyz-789-uvw"
    }
  ]
}
```

Usage:
```typescript
export default {
  async fetch(request: Request, env: Env) {
    // Query
    const result = await env.DB.prepare(
      "SELECT * FROM users WHERE id = ?"
    ).bind(1).first();
    
    // Multiple queries in batch
    const results = await env.DB.batch([
      env.DB.prepare("INSERT INTO users (name) VALUES (?)").bind("Alice"),
      env.DB.prepare("INSERT INTO users (name) VALUES (?)").bind("Bob")
    ]);
    
    // Transaction
    const info = await env.DB.prepare(
      "INSERT INTO users (name, email) VALUES (?, ?)"
    ).bind("Charlie", "charlie@example.com").run();
    
    return Response.json(result);
  }
};
```

### R2 Buckets
```jsonc
{
  "r2_buckets": [
    {
      "binding": "ASSETS",
      "bucket_name": "my-assets",
      "preview_bucket_name": "my-assets-preview",
      "jurisdiction": "eu"
    }
  ]
}
```

Usage:
```typescript
export default {
  async fetch(request: Request, env: Env) {
    // Put object
    await env.ASSETS.put("file.txt", "content", {
      httpMetadata: {
        contentType: "text/plain"
      }
    });
    
    // Get object
    const object = await env.ASSETS.get("file.txt");
    if (object) {
      const content = await object.text();
    }
    
    // Delete object
    await env.ASSETS.delete("file.txt");
    
    // List objects
    const list = await env.ASSETS.list({ prefix: "images/" });
    
    return new Response("OK");
  }
};
```

### Durable Objects
```jsonc
{
  "durable_objects": {
    "bindings": [
      {
        "name": "COUNTER",
        "class_name": "Counter",
        "script_name": "my-worker"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["Counter"]
    }
  ]
}
```

Usage:
```typescript
export class Counter {
  state: DurableObjectState;
  
  constructor(state: DurableObjectState) {
    this.state = state;
  }
  
  async fetch(request: Request) {
    let count = await this.state.storage.get<number>("count") || 0;
    count++;
    await this.state.storage.put("count", count);
    return new Response(String(count));
  }
}

export default {
  async fetch(request: Request, env: Env) {
    const id = env.COUNTER.idFromName("global");
    const stub = env.COUNTER.get(id);
    return stub.fetch(request);
  }
};
```

### Service Bindings
```jsonc
{
  "services": [
    {
      "binding": "AUTH_SERVICE",
      "service": "auth-worker",
      "entrypoint": "default"
    }
  ]
}
```

Usage:
```typescript
export default {
  async fetch(request: Request, env: Env) {
    // Call another Worker
    const response = await env.AUTH_SERVICE.fetch(
      new Request("http://internal/verify", {
        method: "POST",
        body: JSON.stringify({ token: "..." })
      })
    );
    
    return response;
  }
};
```

### Queues
```jsonc
{
  "queues": {
    "producers": [
      {
        "binding": "TASKS",
        "queue": "task-queue"
      }
    ],
    "consumers": [
      {
        "queue": "task-queue",
        "max_batch_size": 10,
        "max_batch_timeout": 30
      }
    ]
  }
}
```

Producer:
```typescript
export default {
  async fetch(request: Request, env: Env) {
    await env.TASKS.send({
      type: "email",
      to: "user@example.com"
    });
    
    // Batch send
    await env.TASKS.sendBatch([
      { body: { task: 1 } },
      { body: { task: 2 } }
    ]);
    
    return new Response("Queued");
  }
};
```

Consumer:
```typescript
export default {
  async queue(batch: MessageBatch, env: Env) {
    for (const message of batch.messages) {
      console.log(message.body);
      message.ack();
    }
  }
};
```

### Vectorize
```jsonc
{
  "vectorize": [
    {
      "binding": "VECTORS",
      "index_name": "embeddings"
    }
  ]
}
```

Usage:
```typescript
export default {
  async fetch(request: Request, env: Env) {
    // Insert vectors
    await env.VECTORS.insert([
      { id: "1", values: [0.1, 0.2, 0.3] },
      { id: "2", values: [0.4, 0.5, 0.6] }
    ]);
    
    // Query
    const results = await env.VECTORS.query([0.15, 0.25, 0.35], {
      topK: 5
    });
    
    return Response.json(results);
  }
};
```

### Hyperdrive
```jsonc
{
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "hyperdrive-id"
    }
  ],
  "compatibility_flags": ["nodejs_compat_v2"]
}
```

Usage:
```typescript
import { Client } from "pg";

export default {
  async fetch(request: Request, env: Env) {
    const client = new Client({
      connectionString: env.HYPERDRIVE.connectionString
    });
    
    await client.connect();
    const result = await client.query("SELECT * FROM users");
    await client.end();
    
    return Response.json(result.rows);
  }
};
```

### Workers AI
```jsonc
{
  "ai": {
    "binding": "AI"
  }
}
```

Usage:
```typescript
export default {
  async fetch(request: Request, env: Env) {
    const response = await env.AI.run(
      "@cf/meta/llama-2-7b-chat-int8",
      {
        prompt: "What is the meaning of life?"
      }
    );
    
    return Response.json(response);
  }
};
```

## Wrangler Programmatic API

### unstable_startWorker (Recommended for Testing)
```typescript
import { unstable_startWorker } from "wrangler";
import { describe, it, before, after } from "node:test";
import assert from "node:assert";

describe("worker", () => {
  let worker;
  
  before(async () => {
    worker = await unstable_startWorker({
      config: "wrangler.jsonc"
    });
  });
  
  after(async () => {
    await worker.dispose();
  });
  
  it("should respond with 200", async () => {
    const response = await worker.fetch("http://example.com");
    assert.strictEqual(response.status, 200);
  });
});
```

### getPlatformProxy
Emulate Workers bindings in Node.js:

```typescript
import { getPlatformProxy } from "wrangler";

const { env, dispose } = await getPlatformProxy<Env>({
  configPath: "wrangler.jsonc",
  environment: "production",
  persist: { path: ".wrangler/state" }
});

// Use bindings
const value = await env.MY_KV.get("key");
await env.DB.prepare("SELECT * FROM users").all();

// Cleanup
await dispose();
```

### unstable_dev (Legacy)
```typescript
import { unstable_dev } from "wrangler";

const worker = await unstable_dev("src/index.ts", {
  config: "wrangler.jsonc",
  experimental: { disableExperimentalWarning: true }
});

const response = await worker.fetch();
const text = await response.text();

await worker.stop();
```

## Advanced Patterns

### Custom Build Configuration
```jsonc
{
  "build": {
    "command": "npm run build",
    "cwd": "./",
    "watch_dir": ["src", "lib"]
  }
}
```

### Observability & Logging
```jsonc
{
  "observability": {
    "enabled": true,
    "head_sampling_rate": 0.1  // Log 10% of requests
  },
  "logpush": true  // Enable Logpush
}
```

### Runtime Limits
```jsonc
{
  "limits": {
    "cpu_ms": 100  // Max 100ms CPU time per invocation
  }
}
```

### Cron Triggers
```jsonc
{
  "triggers": {
    "crons": ["0 0 * * *", "*/15 * * * *"]
  }
}
```

Handler:
```typescript
export default {
  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext) {
    console.log("Cron triggered at", event.scheduledTime);
    // Do work
  }
};
```

### Email Bindings
```jsonc
{
  "send_email": [
    {
      "name": "EMAIL",
      "destination_address": "admin@example.com"
    }
  ]
}
```

Usage:
```typescript
import { EmailMessage } from "cloudflare:email";

export default {
  async fetch(request: Request, env: Env) {
    const message = new EmailMessage(
      "sender@example.com",
      "admin@example.com",
      "Alert from Worker"
    );
    message.setBody("text/plain", "Something happened!");
    
    await env.EMAIL.send(message);
    return new Response("Email sent");
  }
};
```

### Workers for Platforms (Dispatch Namespaces)
```jsonc
{
  "dispatch_namespaces": [
    {
      "binding": "DISPATCHER",
      "namespace": "customer-workers",
      "outbound": {
        "service": "outbound-worker",
        "parameters": ["param1"]
      }
    }
  ]
}
```

### Static Assets
```jsonc
{
  "assets": {
    "directory": "./public",
    "binding": "ASSETS",
    "html_handling": "auto-trailing-slash",
    "not_found_handling": "404-page"
  }
}
```

### mTLS Certificates
```jsonc
{
  "mtls_certificates": [
    {
      "binding": "CERT",
      "certificate_id": "cert-uuid"
    }
  ]
}
```

Upload certificate:
```bash
wrangler mtls-certificate upload --cert cert.pem --key key.pem
```

## Development Best Practices

### 1. Use TypeScript
```typescript
interface Env {
  MY_KV: KVNamespace;
  DB: D1Database;
  API_KEY: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Type-safe access to env
    return new Response("OK");
  }
} satisfies ExportedHandler<Env>;
```

### 2. Generate Types from Config
```bash
wrangler types
```

This creates `worker-configuration.d.ts` with types from your wrangler config.

### 3. Environment-Specific Configuration
```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-01",
  "vars": {
    "LOG_LEVEL": "debug"
  },
  "env": {
    "production": {
      "vars": {
        "LOG_LEVEL": "error"
      },
      "observability": {
        "enabled": true,
        "head_sampling_rate": 1.0
      }
    }
  }
}
```

### 4. Use Local Development for Fast Iteration
```bash
# Local with hot reload
wrangler dev

# Test against remote bindings when needed
wrangler dev --remote
```

### 5. Manage Secrets Properly
Never commit secrets to version control. Use:
- `wrangler secret put` for production secrets
- `.dev.vars` file for local secrets (gitignored)

`.dev.vars`:
```
SECRET_KEY=local-dev-key
DATABASE_PASSWORD=local-password
```

### 6. Test with Real Bindings
```typescript
import { unstable_startWorker } from "wrangler";

const worker = await unstable_startWorker({
  config: "wrangler.jsonc"
});

// Tests run against real local bindings
const response = await worker.fetch("/api/users");
```

### 7. Use Migrations for Durable Objects
```jsonc
{
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["MyDurableObject"]
    },
    {
      "tag": "v2",
      "renamed_classes": [
        { "from": "MyDurableObject", "to": "MyDO" }
      ]
    }
  ]
}
```

### 8. Monitor with Tail
```bash
# Watch logs in development
wrangler tail --format pretty

# Filter production errors
wrangler tail --env production --status error
```

### 9. Version Control Your Deployments
```bash
# Create new version
wrangler deploy

# List versions
wrangler versions list

# Rollback if needed
wrangler rollback
```

### 10. Use System Environment Variables
Wrangler respects these environment variables:
- `CLOUDFLARE_API_TOKEN`
- `CLOUDFLARE_ACCOUNT_ID`
- `WRANGLER_LOG`
- `WRANGLER_SEND_METRICS`

## Common Workflows

### New Worker Project
```bash
# Initialize
wrangler init my-worker
cd my-worker

# Develop
wrangler dev

# Deploy
wrangler deploy
```

### Adding KV Storage
```bash
# Create namespace
wrangler kv namespace create MY_KV
wrangler kv namespace create MY_KV --preview

# Add to wrangler.jsonc
# Deploy
wrangler deploy
```

### Adding D1 Database
```bash
# Create database
wrangler d1 create my-db

# Create migration
wrangler d1 migrations create my-db "initial_schema"

# Edit migration file, then apply
wrangler d1 migrations apply my-db --local

# Deploy
wrangler deploy

# Apply migrations to production
wrangler d1 migrations apply my-db --remote
```

### Multi-Environment Deployment
```bash
# Deploy to staging
wrangler deploy --env staging

# Test staging
curl https://my-worker-staging.workers.dev

# Deploy to production
wrangler deploy --env production
```

### Debugging Worker Issues
```bash
# Check configuration
wrangler whoami

# Validate Worker
wrangler check

# Watch logs
wrangler tail --format pretty

# Test locally
wrangler dev

# Check recent deployments
wrangler deployments list
```

## Common Gotchas

### 1. Binding IDs vs Names
- Bindings use `binding` (name in code) and `id`/`database_id`/`bucket_name` (resource ID)
- Preview bindings need separate IDs: `preview_id`, `preview_database_id`, etc.

### 2. Environment Inheritance
- Non-inheritable keys (bindings, vars) must be redefined per environment
- Inheritable keys (routes, compatibility_date) can be overridden

### 3. Local vs Remote Development
- `wrangler dev` (default): Local simulation, fast, limited accuracy
- `wrangler dev --remote`: Remote execution, slower, production-accurate

### 4. Compatibility Dates
Always set `compatibility_date` to avoid unexpected runtime changes:
```jsonc
{
  "compatibility_date": "2025-01-01"
}
```

### 5. Durable Objects Need script_name
When using Durable Objects with `getPlatformProxy`, always specify `script_name`:
```jsonc
{
  "durable_objects": {
    "bindings": [
      {
        "name": "MY_DO",
        "class_name": "MyDO",
        "script_name": "my-worker"  // Required
      }
    ]
  }
}
```

### 6. Secrets in Local Development
Secrets set with `wrangler secret put` only work in deployed Workers. For local dev, use `.dev.vars`.

### 7. Node.js Compatibility
Some bindings (like Hyperdrive with `pg`) require Node.js compatibility:
```jsonc
{
  "compatibility_flags": ["nodejs_compat_v2"]
}
```

## Performance Tips

### 1. Minimize Bundle Size
```jsonc
{
  "minify": true,
  "no_bundle": false
}
```

### 2. Use KV for Caching
```typescript
const cached = await env.CACHE.get("key", { cacheTtl: 3600 });
if (cached) return new Response(cached);
```

### 3. Batch Database Operations
```typescript
const results = await env.DB.batch([
  env.DB.prepare("SELECT * FROM users"),
  env.DB.prepare("SELECT * FROM posts")
]);
```

### 4. Use R2 Multipart Uploads for Large Files
```typescript
const upload = await env.ASSETS.createMultipartUpload("large-file.bin");
// Upload parts...
await upload.complete(parts);
```

### 5. Leverage Edge Caching
```typescript
return new Response(data, {
  headers: {
    "Cache-Control": "public, max-age=3600"
  }
});
```

## Troubleshooting

### Authentication Issues
```bash
# Re-authenticate
wrangler logout
wrangler login

# Check credentials
wrangler whoami
```

### Configuration Errors
```bash
# Validate config schema
wrangler check

# Use JSON config for schema validation
# (wrangler.jsonc with $schema)
```

### Binding Not Available
- Check binding is in config
- For environments, ensure binding is defined for that environment
- Local dev: some bindings need `--remote`

### Deployment Failures
```bash
# Check logs
wrangler tail

# Try dry run
wrangler deploy --dry-run

# Check account limits
wrangler whoami
```

### Local Development Issues
```bash
# Clear local state
rm -rf .wrangler/state

# Use remote bindings
wrangler dev --remote

# Check persistence location
wrangler dev --persist-to ./local-state
```

## Resources

- Official Docs: https://developers.cloudflare.com/workers/wrangler/
- Configuration: https://developers.cloudflare.com/workers/wrangler/configuration/
- Commands: https://developers.cloudflare.com/workers/wrangler/commands/
- API: https://developers.cloudflare.com/workers/wrangler/api/
- Examples: https://github.com/cloudflare/workers-sdk/tree/main/templates
- Discord: https://discord.gg/cloudflaredev

## Quick Reference

### Essential Commands
```bash
wrangler init              # Create project
wrangler dev              # Local development
wrangler deploy           # Deploy to production
wrangler tail             # View logs
wrangler versions list    # List versions
wrangler rollback         # Rollback deployment
```

### Resource Commands
```bash
wrangler kv namespace create NAME    # Create KV
wrangler d1 create NAME              # Create D1
wrangler r2 bucket create NAME       # Create R2
wrangler queues create NAME          # Create Queue
wrangler vectorize create NAME       # Create Vectorize
```

### Config Essentials
```jsonc
{
  "name": "worker-name",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-01",
  "vars": { /* env vars */ },
  "kv_namespaces": [{ "binding": "KV", "id": "..." }],
  "d1_databases": [{ "binding": "DB", "database_id": "..." }]
}
```
