# Cloudflare Pulumi Provider Skill

Expert guidance for using the Cloudflare Pulumi Provider (@pulumi/cloudflare) to manage Cloudflare infrastructure as code.

## Overview

The Cloudflare Pulumi Provider enables programmatic management of Cloudflare resources including Workers, Pages, D1 databases, KV namespaces, R2 buckets, DNS records, queues, and more. This skill covers TypeScript/JavaScript patterns (primary), with notes on Python, Go, and other languages.

**Provider Package:**
- TypeScript/JavaScript: `@pulumi/cloudflare`
- Python: `pulumi-cloudflare`
- Go: `github.com/pulumi/pulumi-cloudflare/sdk/v6/go/cloudflare`
- .NET: `Pulumi.Cloudflare`

**Current Version:** v6.x

## Provider Configuration

### Authentication Methods

Three mutually exclusive authentication methods:

1. **API Token (Recommended)**
```typescript
import * as cloudflare from "@pulumi/cloudflare";

const provider = new cloudflare.Provider("cf-provider", {
    apiToken: process.env.CLOUDFLARE_API_TOKEN,
});
```

Environment variable: `CLOUDFLARE_API_TOKEN`

2. **API Key (Legacy)**
```typescript
const provider = new cloudflare.Provider("cf-provider", {
    apiKey: process.env.CLOUDFLARE_API_KEY,
    email: process.env.CLOUDFLARE_EMAIL,
});
```

Environment variables: `CLOUDFLARE_API_KEY`, `CLOUDFLARE_EMAIL`

Note: API keys are legacy; use API tokens for new projects.

3. **API User Service Key (Restricted)**
```typescript
const provider = new cloudflare.Provider("cf-provider", {
    apiUserServiceKey: process.env.CLOUDFLARE_API_USER_SERVICE_KEY,
});
```

Environment variable: `CLOUDFLARE_API_USER_SERVICE_KEY`

### Pulumi.yaml Configuration

```yaml
name: my-cloudflare-app
runtime: nodejs
config:
  cloudflare:apiToken:
    value: ${CLOUDFLARE_API_TOKEN}
```

### Account ID Management

Most Cloudflare resources require an `accountId`. Common pattern:

```typescript
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config("cloudflare");
const accountId = config.require("accountId");

// Use in resources
const kv = new cloudflare.WorkersKvNamespace("my-kv", {
    accountId: accountId,
    title: "my-kv-namespace",
});
```

Store in `Pulumi.<stack>.yaml`:
```yaml
config:
  cloudflare:accountId: "your-account-id-here"
```

## Core Resource Types

### Workers (cloudflare.WorkerScript)

Deploy Cloudflare Workers:

```typescript
import * as cloudflare from "@pulumi/cloudflare";
import * as fs from "fs";

const worker = new cloudflare.WorkerScript("my-worker", {
    accountId: accountId,
    name: "my-worker",
    content: fs.readFileSync("./dist/worker.js", "utf8"),
    
    // Module format (ES modules)
    module: true,
    
    // Compatibility settings
    compatibilityDate: "2024-01-01",
    compatibilityFlags: ["nodejs_compat"],
    
    // Bindings
    kvNamespaceBindings: [{
        name: "MY_KV",
        namespaceId: myKv.id,
    }],
    r2BucketBindings: [{
        name: "MY_BUCKET",
        bucketName: myBucket.name,
    }],
    d1DatabaseBindings: [{
        name: "DB",
        databaseId: myDb.id,
    }],
    queueBindings: [{
        name: "MY_QUEUE",
        queue: myQueue.id,
    }],
    
    // Service bindings
    serviceBindings: [{
        name: "OTHER_SERVICE",
        service: otherWorker.name,
    }],
    
    // Environment variables
    plainTextBindings: [{
        name: "ENV_VAR",
        text: "value",
    }],
    secretTextBindings: [{
        name: "API_KEY",
        text: apiKeySecret,
    }],
});
```

**Key patterns:**
- Use `module: true` for ES modules (modern)
- Set `compatibilityDate` to lock in behavior
- Add `compatibilityFlags` for Node.js, etc.
- Bindings connect Workers to other resources

### Workers KV (cloudflare.WorkersKvNamespace)

Key-value storage:

```typescript
const kv = new cloudflare.WorkersKvNamespace("my-kv", {
    accountId: accountId,
    title: "my-kv-namespace",
});

// Write values during deployment
const kvValue = new cloudflare.WorkersKvValue("config-value", {
    accountId: accountId,
    namespaceId: kv.id,
    key: "config",
    value: JSON.stringify({ foo: "bar" }),
});
```

### R2 Buckets (cloudflare.R2Bucket)

Object storage compatible with S3 API:

```typescript
const bucket = new cloudflare.R2Bucket("my-bucket", {
    accountId: accountId,
    name: "my-bucket",
    location: "auto", // or specific location like "wnam"
});

// Bind to Worker
const worker = new cloudflare.WorkerScript("my-worker", {
    accountId: accountId,
    name: "my-worker",
    content: workerCode,
    r2BucketBindings: [{
        name: "MY_BUCKET",
        bucketName: bucket.name,
    }],
});
```

### D1 Databases (cloudflare.D1Database)

SQLite-compatible serverless database:

```typescript
const db = new cloudflare.D1Database("my-db", {
    accountId: accountId,
    name: "my-database",
});

// Run migrations (no native support, use custom resource or API)
// Bind to Worker
const worker = new cloudflare.WorkerScript("my-worker", {
    accountId: accountId,
    name: "my-worker",
    content: workerCode,
    d1DatabaseBindings: [{
        name: "DB",
        databaseId: db.id,
    }],
});
```

**D1 Migration Pattern:**
Use Cloudflare API directly or wrangler for migrations:

```typescript
import * as command from "@pulumi/command";

const migration = new command.local.Command("d1-migration", {
    create: pulumi.interpolate`wrangler d1 execute ${db.name} --command "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)"`,
}, { dependsOn: [db] });
```

### Queues (cloudflare.Queue)

Message queues for async processing:

```typescript
const queue = new cloudflare.Queue("my-queue", {
    accountId: accountId,
    name: "my-queue",
});

// Producer worker
const producer = new cloudflare.WorkerScript("producer", {
    accountId: accountId,
    name: "producer",
    content: producerCode,
    queueBindings: [{
        name: "MY_QUEUE",
        queue: queue.id,
    }],
});

// Consumer worker
const consumer = new cloudflare.WorkerScript("consumer", {
    accountId: accountId,
    name: "consumer",
    content: consumerCode,
    queueConsumers: [{
        queue: queue.name,
        maxBatchSize: 10,
        maxRetries: 3,
        maxWaitTimeMs: 5000,
    }],
});
```

### Pages Projects (cloudflare.PagesProject)

Static site hosting with Workers integration:

```typescript
const pagesProject = new cloudflare.PagesProject("my-site", {
    accountId: accountId,
    name: "my-site",
    productionBranch: "main",
    
    buildConfig: {
        buildCommand: "npm run build",
        destinationDir: "dist",
        rootDir: "",
    },
    
    source: {
        type: "github",
        config: {
            owner: "my-org",
            repoName: "my-repo",
            productionBranch: "main",
            prCommentsEnabled: true,
            deploymentsEnabled: true,
        },
    },
    
    deploymentConfigs: {
        production: {
            environmentVariables: {
                NODE_VERSION: "18",
            },
            kvNamespaces: {
                MY_KV: myKv.id,
            },
            d1Databases: {
                DB: myDb.id,
            },
            r2Buckets: {
                MY_BUCKET: myBucket.name,
            },
        },
    },
});
```

### DNS Records (cloudflare.DnsRecord)

Manage DNS entries:

```typescript
// Get zone ID
const zone = cloudflare.getZone({
    name: "example.com",
});

const record = new cloudflare.DnsRecord("www", {
    zoneId: zone.then(z => z.id),
    name: "www",
    type: "A",
    content: "192.0.2.1",
    ttl: 3600,
    proxied: true, // Enable Cloudflare proxy
});

// CNAME record
const cname = new cloudflare.DnsRecord("api", {
    zoneId: zone.then(z => z.id),
    name: "api",
    type: "CNAME",
    content: worker.subdomain, // Point to Worker
    proxied: true,
});
```

### Workers Domains/Routes

Custom domains for Workers:

```typescript
// Worker route (pattern-based)
const route = new cloudflare.WorkerRoute("my-route", {
    zoneId: zoneId,
    pattern: "example.com/api/*",
    scriptName: worker.name,
});

// Workers domain (dedicated subdomain)
const domain = new cloudflare.WorkersDomain("my-domain", {
    accountId: accountId,
    hostname: "api.example.com",
    service: worker.name,
    zoneId: zoneId,
});
```

## Advanced Patterns

### Resource Dependencies

Pulumi automatically tracks dependencies through outputs:

```typescript
const kv = new cloudflare.WorkersKvNamespace("kv", {
    accountId: accountId,
    title: "my-kv",
});

// Worker depends on KV (implicit via kv.id)
const worker = new cloudflare.WorkerScript("worker", {
    accountId: accountId,
    name: "my-worker",
    content: workerCode,
    kvNamespaceBindings: [{
        name: "MY_KV",
        namespaceId: kv.id, // Creates dependency
    }],
});
```

### Outputs and Exports

Export resource identifiers for use elsewhere:

```typescript
export const kvId = kv.id;
export const bucketName = bucket.name;
export const workerUrl = worker.subdomain;
export const dbId = db.id;
```

### Component Resources

Encapsulate related resources:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as cloudflare from "@pulumi/cloudflare";

interface WorkerAppArgs {
    accountId: pulumi.Input<string>;
    workerCode: string;
    domain: string;
}

class WorkerApp extends pulumi.ComponentResource {
    public readonly worker: cloudflare.WorkerScript;
    public readonly kv: cloudflare.WorkersKvNamespace;
    public readonly domain: cloudflare.WorkersDomain;

    constructor(name: string, args: WorkerAppArgs, opts?: pulumi.ComponentResourceOptions) {
        super("custom:cloudflare:WorkerApp", name, {}, opts);

        const defaultOpts = { parent: this };

        this.kv = new cloudflare.WorkersKvNamespace(`${name}-kv`, {
            accountId: args.accountId,
            title: `${name}-kv`,
        }, defaultOpts);

        this.worker = new cloudflare.WorkerScript(`${name}-worker`, {
            accountId: args.accountId,
            name: `${name}-worker`,
            content: args.workerCode,
            module: true,
            kvNamespaceBindings: [{
                name: "KV",
                namespaceId: this.kv.id,
            }],
        }, defaultOpts);

        this.domain = new cloudflare.WorkersDomain(`${name}-domain`, {
            accountId: args.accountId,
            hostname: args.domain,
            service: this.worker.name,
        }, defaultOpts);

        this.registerOutputs({
            workerId: this.worker.id,
            kvId: this.kv.id,
            domainId: this.domain.id,
        });
    }
}

// Usage
const app = new WorkerApp("my-app", {
    accountId: accountId,
    workerCode: fs.readFileSync("./dist/worker.js", "utf8"),
    domain: "api.example.com",
});
```

### Transform Pattern

Modify resource args before creation (common in higher-level frameworks like SST):

```typescript
import { Transform } from "@pulumi/pulumi";

interface BucketArgs {
    accountId: pulumi.Input<string>;
    transform?: {
        bucket?: Transform<cloudflare.R2BucketArgs>;
    };
}

function createBucket(name: string, args: BucketArgs) {
    const bucketArgs: cloudflare.R2BucketArgs = {
        accountId: args.accountId,
        name: name,
        location: "auto",
    };
    
    // Apply transform if provided
    const finalArgs = args.transform?.bucket?.(bucketArgs) ?? bucketArgs;
    
    return new cloudflare.R2Bucket(name, finalArgs);
}
```

### Secrets Management

Handle sensitive values:

```typescript
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();
const apiKey = config.requireSecret("apiKey"); // Encrypted in state

const worker = new cloudflare.WorkerScript("worker", {
    accountId: accountId,
    name: "my-worker",
    content: workerCode,
    secretTextBindings: [{
        name: "API_KEY",
        text: apiKey,
    }],
});
```

## Integration with Wrangler

### Local Development

Use wrangler for local dev, Pulumi for deployment:

**wrangler.toml:**
```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

[[kv_namespaces]]
binding = "MY_KV"
id = "preview_id"  # For local dev
```

**Match Pulumi bindings:**
```typescript
const worker = new cloudflare.WorkerScript("worker", {
    accountId: accountId,
    name: "my-worker",
    content: workerCode,
    compatibilityDate: "2024-01-01",
    compatibilityFlags: ["nodejs_compat"],
    kvNamespaceBindings: [{
        name: "MY_KV", // Must match wrangler.toml binding
        namespaceId: kv.id,
    }],
});
```

### Migration from Wrangler

Convert wrangler.toml to Pulumi:

```toml
# wrangler.toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[vars]
ENVIRONMENT = "production"

[[kv_namespaces]]
binding = "MY_KV"
id = "abc123"

[[r2_buckets]]
binding = "MY_BUCKET"
bucket_name = "my-bucket"
```

Becomes:

```typescript
const kv = new cloudflare.WorkersKvNamespace("kv", {
    accountId: accountId,
    title: "my-kv",
});

const bucket = new cloudflare.R2Bucket("bucket", {
    accountId: accountId,
    name: "my-bucket",
});

const worker = new cloudflare.WorkerScript("worker", {
    accountId: accountId,
    name: "my-worker",
    content: workerCode,
    compatibilityDate: "2024-01-01",
    plainTextBindings: [{
        name: "ENVIRONMENT",
        text: "production",
    }],
    kvNamespaceBindings: [{
        name: "MY_KV",
        namespaceId: kv.id,
    }],
    r2BucketBindings: [{
        name: "MY_BUCKET",
        bucketName: bucket.name,
    }],
});
```

## Integration with Cloudflare APIs

### Using Outputs with API Calls

Access resource IDs after creation:

```typescript
const db = new cloudflare.D1Database("db", {
    accountId: accountId,
    name: "my-db",
});

// Use output in API call
db.id.apply(async (dbId) => {
    const response = await fetch(
        `https://api.cloudflare.com/client/v4/accounts/${accountId}/d1/database/${dbId}/query`,
        {
            method: "POST",
            headers: {
                "Authorization": `Bearer ${apiToken}`,
                "Content-Type": "application/json",
            },
            body: JSON.stringify({
                sql: "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)",
            }),
        }
    );
    return response.json();
});
```

### Custom Dynamic Providers

For resources not yet in Pulumi provider:

```typescript
import * as pulumi from "@pulumi/pulumi";

class D1MigrationProvider implements pulumi.dynamic.ResourceProvider {
    async create(inputs: any): Promise<pulumi.dynamic.CreateResult> {
        const response = await fetch(
            `https://api.cloudflare.com/client/v4/accounts/${inputs.accountId}/d1/database/${inputs.databaseId}/query`,
            {
                method: "POST",
                headers: {
                    "Authorization": `Bearer ${inputs.apiToken}`,
                    "Content-Type": "application/json",
                },
                body: JSON.stringify({ sql: inputs.sql }),
            }
        );
        
        const result = await response.json();
        return { id: `${inputs.databaseId}-${Date.now()}`, outs: result };
    }

    async update(id: string, olds: any, news: any): Promise<pulumi.dynamic.UpdateResult> {
        // Run new migrations if SQL changed
        if (olds.sql !== news.sql) {
            await this.create(news);
        }
        return {};
    }

    async delete(id: string, props: any): Promise<void> {
        // Optionally run down migrations
    }
}

class D1Migration extends pulumi.dynamic.Resource {
    constructor(name: string, args: any, opts?: pulumi.CustomResourceOptions) {
        super(new D1MigrationProvider(), name, args, opts);
    }
}

// Usage
const migration = new D1Migration("migration", {
    accountId: accountId,
    databaseId: db.id,
    apiToken: apiToken,
    sql: "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)",
}, { dependsOn: [db] });
```

## Multi-Language Support

### Python

```python
import pulumi
import pulumi_cloudflare as cloudflare

config = pulumi.Config()
account_id = config.require("accountId")

kv = cloudflare.WorkersKvNamespace("my-kv",
    account_id=account_id,
    title="my-kv-namespace")

worker = cloudflare.WorkerScript("my-worker",
    account_id=account_id,
    name="my-worker",
    content=open("./dist/worker.js").read(),
    module=True,
    kv_namespace_bindings=[
        cloudflare.WorkerScriptKvNamespaceBindingArgs(
            name="MY_KV",
            namespace_id=kv.id,
        )
    ])

pulumi.export("kv_id", kv.id)
pulumi.export("worker_name", worker.name)
```

### Go

```go
package main

import (
    "io/ioutil"
    
    "github.com/pulumi/pulumi-cloudflare/sdk/v6/go/cloudflare"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi/config"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        cfg := config.New(ctx, "")
        accountId := cfg.Require("accountId")
        
        kv, err := cloudflare.NewWorkersKvNamespace(ctx, "my-kv", &cloudflare.WorkersKvNamespaceArgs{
            AccountId: pulumi.String(accountId),
            Title:     pulumi.String("my-kv-namespace"),
        })
        if err != nil {
            return err
        }
        
        workerCode, err := ioutil.ReadFile("./dist/worker.js")
        if err != nil {
            return err
        }
        
        worker, err := cloudflare.NewWorkerScript(ctx, "my-worker", &cloudflare.WorkerScriptArgs{
            AccountId: pulumi.String(accountId),
            Name:      pulumi.String("my-worker"),
            Content:   pulumi.String(string(workerCode)),
            Module:    pulumi.Bool(true),
            KvNamespaceBindings: cloudflare.WorkerScriptKvNamespaceBindingArray{
                &cloudflare.WorkerScriptKvNamespaceBindingArgs{
                    Name:        pulumi.String("MY_KV"),
                    NamespaceId: kv.ID(),
                },
            },
        })
        if err != nil {
            return err
        }
        
        ctx.Export("kvId", kv.ID())
        ctx.Export("workerName", worker.Name)
        return nil
    })
}
```

## Common Use Cases

### Full-Stack Worker App

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as cloudflare from "@pulumi/cloudflare";
import * as fs from "fs";

const config = new pulumi.Config();
const accountId = config.require("accountId");
const zoneId = config.require("zoneId");
const domain = config.require("domain");

// Storage resources
const kv = new cloudflare.WorkersKvNamespace("cache", {
    accountId: accountId,
    title: "api-cache",
});

const db = new cloudflare.D1Database("db", {
    accountId: accountId,
    name: "app-database",
});

const bucket = new cloudflare.R2Bucket("assets", {
    accountId: accountId,
    name: "app-assets",
});

// API Worker
const apiWorker = new cloudflare.WorkerScript("api", {
    accountId: accountId,
    name: "api-worker",
    content: fs.readFileSync("./dist/api.js", "utf8"),
    module: true,
    compatibilityDate: "2024-01-01",
    compatibilityFlags: ["nodejs_compat"],
    
    kvNamespaceBindings: [{
        name: "CACHE",
        namespaceId: kv.id,
    }],
    d1DatabaseBindings: [{
        name: "DB",
        databaseId: db.id,
    }],
    r2BucketBindings: [{
        name: "ASSETS",
        bucketName: bucket.name,
    }],
    
    plainTextBindings: [{
        name: "ENVIRONMENT",
        text: pulumi.getStack(),
    }],
});

// Custom domain
const apiDomain = new cloudflare.WorkersDomain("api-domain", {
    accountId: accountId,
    hostname: `api.${domain}`,
    service: apiWorker.name,
    zoneId: zoneId,
});

export const apiUrl = pulumi.interpolate`https://api.${domain}`;
export const kvId = kv.id;
export const dbId = db.id;
export const bucketName = bucket.name;
```

### Multi-Environment Setup

```typescript
// Pulumi.dev.yaml
config:
  cloudflare:accountId: "dev-account-id"
  cloudflare:apiToken: "dev-token"
  app:domain: "dev.example.com"

// Pulumi.prod.yaml
config:
  cloudflare:accountId: "prod-account-id"
  cloudflare:apiToken: "prod-token"
  app:domain: "example.com"

// index.ts
const config = new pulumi.Config();
const stack = pulumi.getStack();
const accountId = config.require("accountId");
const domain = config.require("domain");

const worker = new cloudflare.WorkerScript(`worker-${stack}`, {
    accountId: accountId,
    name: `my-worker-${stack}`,
    content: workerCode,
    plainTextBindings: [{
        name: "ENVIRONMENT",
        text: stack,
    }],
});
```

### Queue-Based Processing

```typescript
const queue = new cloudflare.Queue("processing-queue", {
    accountId: accountId,
    name: "image-processing",
});

// Producer: API receives requests
const apiWorker = new cloudflare.WorkerScript("api", {
    accountId: accountId,
    name: "api-worker",
    content: apiCode,
    queueBindings: [{
        name: "PROCESSING_QUEUE",
        queue: queue.id,
    }],
});

// Consumer: Process async
const processorWorker = new cloudflare.WorkerScript("processor", {
    accountId: accountId,
    name: "processor-worker",
    content: processorCode,
    queueConsumers: [{
        queue: queue.name,
        maxBatchSize: 10,
        maxRetries: 3,
        maxWaitTimeMs: 5000,
    }],
    r2BucketBindings: [{
        name: "OUTPUT_BUCKET",
        bucketName: outputBucket.name,
    }],
});
```

## Configuration Best Practices

### 1. Use Stack Configuration
Store environment-specific values in stack config:

```yaml
config:
  cloudflare:accountId: "abc123"
  cloudflare:apiToken:
    secure: "encrypted-value"
  app:domain: "example.com"
  app:zoneId: "xyz789"
```

### 2. Explicit Provider Configuration
Use explicit provider when managing multiple accounts:

```typescript
const devProvider = new cloudflare.Provider("dev", {
    apiToken: devToken,
});

const prodProvider = new cloudflare.Provider("prod", {
    apiToken: prodToken,
});

const devWorker = new cloudflare.WorkerScript("dev-worker", {
    accountId: devAccountId,
    name: "worker",
    content: workerCode,
}, { provider: devProvider });

const prodWorker = new cloudflare.WorkerScript("prod-worker", {
    accountId: prodAccountId,
    name: "worker",
    content: workerCode,
}, { provider: prodProvider });
```

### 3. Resource Naming Conventions
Include stack/environment in resource names:

```typescript
const stack = pulumi.getStack();

const kv = new cloudflare.WorkersKvNamespace(`${stack}-kv`, {
    accountId: accountId,
    title: `${stack}-my-kv`,
});
```

### 4. Protect Production Resources
Use `protect` to prevent accidental deletion:

```typescript
const prodDb = new cloudflare.D1Database("prod-db", {
    accountId: accountId,
    name: "production-database",
}, { protect: true }); // Cannot be deleted without removing protect
```

### 5. Use dependsOn for Ordering
Explicit dependencies when needed:

```typescript
const migration = new command.local.Command("migration", {
    create: pulumi.interpolate`wrangler d1 execute ${db.name} --file ./schema.sql`,
}, { dependsOn: [db] });

const worker = new cloudflare.WorkerScript("worker", {
    accountId: accountId,
    name: "worker",
    content: workerCode,
    d1DatabaseBindings: [{
        name: "DB",
        databaseId: db.id,
    }],
}, { dependsOn: [migration] }); // Ensure migrations run first
```

## Common Patterns & Idioms

### Dynamic Worker Content from Build Output

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as command from "@pulumi/command";
import * as fs from "fs";

// Build worker
const build = new command.local.Command("build-worker", {
    create: "npm run build",
    dir: "./worker",
});

// Read built output
const workerContent = build.stdout.apply(() => 
    fs.readFileSync("./worker/dist/index.js", "utf8")
);

const worker = new cloudflare.WorkerScript("worker", {
    accountId: accountId,
    name: "my-worker",
    content: workerContent,
}, { dependsOn: [build] });
```

### Conditional Resources

```typescript
const stack = pulumi.getStack();
const isProd = stack === "prod";

// Only create analytics in production
const analytics = isProd ? new cloudflare.WorkersKvNamespace("analytics", {
    accountId: accountId,
    title: "analytics",
}) : undefined;

const worker = new cloudflare.WorkerScript("worker", {
    accountId: accountId,
    name: "worker",
    content: workerCode,
    kvNamespaceBindings: analytics ? [{
        name: "ANALYTICS",
        namespaceId: analytics.id,
    }] : [],
});
```

### Resource Tagging Pattern

While Cloudflare doesn't have native tags, use naming:

```typescript
function createResource(name: string, type: string) {
    const stack = pulumi.getStack();
    const fullName = `${stack}-${type}-${name}`;
    
    return new cloudflare.WorkersKvNamespace(fullName, {
        accountId: accountId,
        title: fullName,
    });
}

const userCache = createResource("user-cache", "kv");
const sessionCache = createResource("session-cache", "kv");
```

## Troubleshooting

### Common Errors

**1. Account ID Missing**
```
error: Missing required property 'accountId'
```
Solution: Add to config or pass explicitly.

**2. Binding Name Mismatch**
Worker expects `MY_KV` but binding uses different name.
Solution: Match binding names in Pulumi and worker code.

**3. Resource Not Found**
```
error: resource 'abc123' not found
```
Solution: Ensure resources exist in correct account/zone.

**4. API Token Permissions**
Solution: Verify token has required permissions (Workers, KV, R2, etc.).

### Debugging

Enable verbose logging:

```bash
pulumi up --logtostderr -v=9
```

Preview changes without applying:

```bash
pulumi preview
```

View resource state:

```bash
pulumi stack export
```

## Architecture References

### Microservices with Service Bindings

```typescript
// Auth service
const authWorker = new cloudflare.WorkerScript("auth", {
    accountId: accountId,
    name: "auth-service",
    content: authCode,
    d1DatabaseBindings: [{
        name: "USERS_DB",
        databaseId: usersDb.id,
    }],
});

// API service (calls auth)
const apiWorker = new cloudflare.WorkerScript("api", {
    accountId: accountId,
    name: "api-service",
    content: apiCode,
    serviceBindings: [{
        name: "AUTH",
        service: authWorker.name,
    }],
});

// In API worker code:
// const authResponse = await env.AUTH.fetch("/verify", { ... });
```

### Event-Driven with Queues

```typescript
// Event bus
const eventQueue = new cloudflare.Queue("events", {
    accountId: accountId,
    name: "event-bus",
});

// Producers
const producers = ["api", "webhook", "cron"].map(name =>
    new cloudflare.WorkerScript(`${name}-producer`, {
        accountId: accountId,
        name: `${name}-producer`,
        content: producerCode,
        queueBindings: [{
            name: "EVENTS",
            queue: eventQueue.id,
        }],
    })
);

// Consumers
const consumers = ["email", "analytics", "cache"].map(name =>
    new cloudflare.WorkerScript(`${name}-consumer`, {
        accountId: accountId,
        name: `${name}-consumer`,
        content: consumerCode,
        queueConsumers: [{
            queue: eventQueue.name,
            maxBatchSize: 10,
        }],
    })
);
```

### CDN with Dynamic Content

```typescript
// Static assets in R2
const staticBucket = new cloudflare.R2Bucket("static", {
    accountId: accountId,
    name: "static-assets",
});

// Dynamic content worker
const appWorker = new cloudflare.WorkerScript("app", {
    accountId: accountId,
    name: "app-worker",
    content: appCode,
    r2BucketBindings: [{
        name: "STATIC",
        bucketName: staticBucket.name,
    }],
    kvNamespaceBindings: [{
        name: "CACHE",
        namespaceId: cache.id,
    }],
});

// Route: /api/* -> worker, /* -> R2 (via worker)
const route = new cloudflare.WorkerRoute("route", {
    zoneId: zoneId,
    pattern: `${domain}/*`,
    scriptName: appWorker.name,
});
```

## Additional Resources

- **Pulumi Registry:** https://www.pulumi.com/registry/packages/cloudflare/
- **API Docs:** https://www.pulumi.com/registry/packages/cloudflare/api-docs/
- **GitHub:** https://github.com/pulumi/pulumi-cloudflare
- **Cloudflare Docs:** https://developers.cloudflare.com/
- **Workers Docs:** https://developers.cloudflare.com/workers/
- **Wrangler Docs:** https://developers.cloudflare.com/workers/wrangler/

## Quick Reference

### Essential Imports
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as cloudflare from "@pulumi/cloudflare";
```

### Common Resource Types
- `cloudflare.Provider` - Provider configuration
- `cloudflare.WorkerScript` - Cloudflare Worker
- `cloudflare.WorkersKvNamespace` - KV namespace
- `cloudflare.R2Bucket` - R2 bucket
- `cloudflare.D1Database` - D1 database
- `cloudflare.Queue` - Queue
- `cloudflare.PagesProject` - Pages project
- `cloudflare.DnsRecord` - DNS record
- `cloudflare.WorkerRoute` - Worker route
- `cloudflare.WorkersDomain` - Worker custom domain

### Key Properties
- `accountId` - Required for most resources
- `zoneId` - Required for DNS/domain resources
- `name`/`title` - Resource identifier
- `*Bindings` - Connect resources to Workers

### Essential Patterns
1. Use stack config for environment values
2. Use `Output` for dependent values
3. Use `apply()` for transformations
4. Use `protect` for production resources
5. Match binding names across code/config
