# Cloudflare Containers Skill

**APPLIES TO: Cloudflare Containers ONLY - NOT general Cloudflare Workers**

Use when working with Cloudflare Containers: deploying containerized apps on Workers platform, configuring container-enabled Durable Objects, managing container lifecycle, or implementing stateful/stateless container patterns.

## Core Concepts

### Container-Enabled Durable Objects
Each Container runs alongside a Durable Object that manages its lifecycle. Key points:
- Container = Durable Object + containerized workload
- One container instance per Durable Object instance
- Must use SQLite-backed DOs (`new_sqlite_classes` not `new_classes`)
- Container class extends `DurableObject` (or use `@cloudflare/containers` helper)

### Container Package (`@cloudflare/containers`)
Recommended approach - provides idiomatic API over raw Durable Object container methods:

```typescript
import { Container, getContainer, getRandom } from "@cloudflare/containers";

export class MyContainer extends Container {
  defaultPort = 8080;
  sleepAfter = "5m";
  envVars = {
    MESSAGE: "I'm a container!"
  };
  
  override onStart() {
    console.log("Container started");
  }
  
  override onStop() {
    console.log("Container stopped");
  }
  
  override onError(error: unknown) {
    console.error("Container error:", error);
  }
}
```

Key Container class properties:
- `defaultPort` - Port for `fetch()`/`containerFetch()` to use, blocks until container listens
- `sleepAfter` - Idle timeout (e.g., "10s", "5m", "2h")
- `envVars` - Environment variables passed on start
- `onStart()` - Hook when container starts
- `onStop()` - Hook when container stops  
- `onError(error)` - Hook on container error

## Wrangler Configuration

### Basic Container Config

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2026-01-10",
  "containers": [
    {
      "class_name": "MyContainer",
      "image": "./Dockerfile",  // or path to directory with Dockerfile
      "max_instances": 10
    }
  ],
  "durable_objects": {
    "bindings": [
      {
        "name": "MY_CONTAINER",
        "class_name": "MyContainer"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["MyContainer"]  // Must use new_sqlite_classes
    }
  ]
}
```

### TOML Format

```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2026-01-10"

[[containers]]
class_name = "MyContainer"
image = "./Dockerfile"
max_instances = 10

[[durable_objects.bindings]]
name = "MY_CONTAINER"
class_name = "MyContainer"

[[migrations]]
tag = "v1"
new_sqlite_classes = ["MyContainer"]
```

Key config requirements:
- `image` - Path to Dockerfile or directory containing one
- `class_name` - Must match Durable Object class name
- `max_instances` - Max simultaneous container instances
- Must have corresponding DO binding and migration with `new_sqlite_classes`

## Container Images

### Requirements
- Must run on `linux/amd64` architecture
- Can use any language/runtime
- Should expose ports for communication
- Use standard Dockerfile

### Example Dockerfile (Go Server)

```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

### Auto-Generated Environment Variables
Containers automatically receive:
- `CLOUDFLARE_DEPLOYMENT_ID` - Unique instance identifier
- Any custom vars from `envVars` or `startOptions`

## Routing Patterns

### Stateful Routing (Individual Instances)
Route to specific container instances by name/ID:

```typescript
export default {
  async fetch(request: Request, env: Env) {
    const url = new URL(request.url);
    const sessionId = url.searchParams.get("session");
    
    // Each session gets its own container
    const container = env.MY_CONTAINER.getByName(sessionId);
    return container.fetch(request);
  }
}
```

### Stateless Routing (Load Balancing)
Distribute requests across multiple interchangeable instances:

```typescript
import { getRandom } from "@cloudflare/containers";

export default {
  async fetch(request: Request, env: Env) {
    // Route to one of 5 random instances
    const container = await getRandom(env.MY_CONTAINER, 5);
    return container.fetch(request);
  }
}
```

**Note:** `getRandom` is temporary - native autoscaling/load balancing coming soon.

### Hybrid Pattern
Combine stateful and stateless routing:

```typescript
export default {
  async fetch(request: Request, env: Env) {
    const url = new URL(request.url);
    
    if (url.pathname.startsWith("/session/")) {
      // Stateful: route by session ID
      const sessionId = url.pathname.split("/")[2];
      return env.MY_CONTAINER.getByName(sessionId).fetch(request);
    }
    
    // Stateless: load balance
    return (await getRandom(env.MY_CONTAINER, 3)).fetch(request);
  }
}
```

## Container Lifecycle Management

### Starting Containers

**Automatic start (recommended):**
```typescript
// fetch() automatically starts if not running
const container = env.MY_CONTAINER.getByName("id");
return container.fetch(request);
```

**Manual start with options:**
```typescript
const container = env.MY_CONTAINER.getByName("id");
await container.startAndWaitForPorts({
  startOptions: {
    env: {
      CUSTOM_VAR: "value",
      API_KEY: env.SECRET
    },
    entrypoint: ["python", "server.py"],
    enableInternet: false
  }
});
```

**Using raw Durable Object API:**
```typescript
export class MyContainer extends DurableObject {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    
    this.ctx.blockConcurrencyWhile(async () => {
      this.ctx.container.start({
        env: { FOO: "bar" },
        entrypoint: ["node", "server.js"],
        enableInternet: false
      });
    });
  }
}
```

### Stopping Containers

**Automatic (idle timeout):**
Containers stop after `sleepAfter` period with no requests.

**Manual stop:**
```typescript
await container.destroy("Manually stopped");
```

**Send signals:**
```typescript
const SIGTERM = 15;
const SIGKILL = 9;
this.ctx.container.signal(SIGTERM);
```

### Monitoring Container Status

```typescript
export class MyContainer extends Container {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    
    this.ctx.container.monitor()
      .then(() => console.log("Container exited"))
      .catch((err) => console.error("Container error:", err));
  }
}
```

## Communication with Containers

### HTTP Requests

**Default port (recommended):**
```typescript
// Uses defaultPort from Container class
const container = env.MY_CONTAINER.getByName("id");
const response = await container.fetch(request);
```

**Specific port:**
```typescript
const port = this.ctx.container.getTcpPort(8080);
const response = await port.fetch("http://container/api", {
  method: "POST",
  body: JSON.stringify(data)
});
```

### TCP Connections

```typescript
const port = this.ctx.container.getTcpPort(8080);
const conn = port.connect('10.0.0.1:8080');
await conn.opened;

try {
  if (request.body) {
    await request.body.pipeTo(conn.writable);
  }
  return new Response(conn.readable);
} catch (err) {
  return new Response("Failed to proxy", { status: 502 });
}
```

### WebSocket Forwarding

```typescript
export default {
  async fetch(request: Request, env: Env) {
    const upgradeHeader = request.headers.get("Upgrade");
    if (upgradeHeader === "websocket") {
      const container = env.MY_CONTAINER.getByName("ws-session");
      return container.fetch(request);
    }
    return new Response("Not a WebSocket", { status: 400 });
  }
}
```

## Environment Variables and Secrets

### Hard-coded Environment Variables

```typescript
export class MyContainer extends Container {
  envVars = {
    ENV_VAR: "static-value",
    NODE_ENV: "production"
  };
}
```

### Worker Secrets

**Set secret:**
```bash
wrangler secret put MY_SECRET
```

**Pass to container:**
```typescript
import { env } from "cloudflare:workers";

export class MyContainer extends Container {
  envVars = {
    MY_SECRET: env.MY_SECRET
  };
}
```

### Secret Store

**Create store and secret:**
```bash
wrangler secrets-store store create my-store --remote
wrangler secrets-store secret create my-store --name DB_PASSWORD --scopes workers --remote
```

**Config binding:**
```jsonc
{
  "secrets_store_secrets": [
    {
      "binding": "SECRET_STORE",
      "store_id": "my-store",
      "secret_name": "DB_PASSWORD"
    }
  ]
}
```

**Pass to container (async):**
```typescript
const container = env.MY_CONTAINER.getByName("id");
await container.startAndWaitForPorts({
  startOptions: {
    env: {
      DB_PASSWORD: await env.SECRET_STORE.get()
    }
  }
});
```

### KV Values

**Create KV namespace:**
```bash
wrangler kv namespace create DEMO_KV
wrangler kv key put --binding DEMO_KV CONFIG '{"timeout":30}'
```

**Config binding:**
```jsonc
{
  "kv_namespaces": [
    {
      "binding": "DEMO_KV",
      "id": "your-kv-namespace-id"
    }
  ]
}
```

**Pass to container:**
```typescript
const config = await env.DEMO_KV.get("CONFIG", "json");
const container = env.MY_CONTAINER.getByName("id");

await container.startAndWaitForPorts({
  startOptions: {
    env: {
      CONFIG_JSON: JSON.stringify(config),
      API_URL: await env.DEMO_KV.get("API_URL")
    }
  }
});
```

### Per-Instance Environment Variables

```typescript
export default {
  async fetch(request: Request, env: Env) {
    const instanceOne = env.MY_CONTAINER.getByName("foo");
    const instanceTwo = env.MY_CONTAINER.getByName("bar");
    
    // Different env vars per instance
    await instanceOne.startAndWaitForPorts({
      startOptions: {
        envVars: {
          INSTANCE_NAME: "foo",
          CONFIG: await env.DEMO_KV.get("foo-config")
        }
      }
    });
    
    await instanceTwo.startAndWaitForPorts({
      startOptions: {
        envVars: {
          INSTANCE_NAME: "bar",
          CONFIG: await env.DEMO_KV.get("bar-config")
        }
      }
    });
    
    // Route based on logic
    return instanceOne.fetch(request);
  }
}
```

### Build-time Environment Variables

Use `image_vars` in wrangler config for variables only available during image build:

```jsonc
{
  "containers": [
    {
      "class_name": "MyContainer",
      "image": "./Dockerfile",
      "image_vars": {
        "BUILD_VERSION": "1.2.3",
        "BUILD_ENV": "production"
      }
    }
  ]
}
```

## Wrangler Commands

### Build and Deploy

```bash
# Deploy Worker + Container (builds and pushes image automatically)
wrangler deploy

# Build container image only
wrangler containers build ./path/to/dockerfile -t myapp:latest

# Build and push image
wrangler containers build ./path/to/dockerfile -t myapp:latest --push

# Push existing image to Cloudflare registry
wrangler containers push myapp:latest
```

### Container Management

```bash
# List containers
wrangler containers list

# Get container info
wrangler containers info <CONTAINER_ID>

# Delete container
wrangler containers delete <CONTAINER_ID>
```

### Image Management

```bash
# List images in registry
wrangler containers images list

# Filter images
wrangler containers images list --filter "myapp:.*"

# Delete image
wrangler containers images delete myapp:v1

# JSON output
wrangler containers images list --json
```

### External Registry Configuration

**Configure ECR registry:**
```bash
wrangler containers registries configure <ECR_DOMAIN> \
  --public-credential <AWS_ACCESS_KEY_ID> \
  --secret-store-id <STORE_ID> \
  --secret-name <SECRET_NAME>
```

**List configured registries:**
```bash
wrangler containers registries list
wrangler containers registries list --json
```

**Delete registry:**
```bash
wrangler containers registries delete <DOMAIN>
```

**Non-interactive secret setup:**
```bash
echo "secret-value" | wrangler containers registries configure <DOMAIN> \
  --public-credential <PUBLIC_CRED>
```

### Development

```bash
# Deploy will build image via Docker - ensure Docker is running
docker info  # Check Docker status first

# First deploy takes longer (image build + push + provisioning)
# Subsequent deploys are faster (cached layers)
# Wait several minutes after first deploy before container receives requests
```

## Common Use Cases

### Stateful Backend (Session-based)

```typescript
import { Container } from "@cloudflare/containers";

export class SessionBackend extends Container {
  defaultPort = 3000;
  sleepAfter = "30m";
}

export default {
  async fetch(request: Request, env: Env) {
    const { sessionId } = await request.json();
    // Each session gets dedicated container instance
    const container = env.SESSION_BACKEND.getByName(sessionId);
    return container.fetch(request);
  }
}
```

### Short-Lived Code Execution

```typescript
export class CodeSandbox extends Container {
  defaultPort = 8080;
  sleepAfter = "5m";  // Quick cleanup
}

export default {
  async fetch(request: Request, env: Env) {
    const { code, executionId } = await request.json();
    
    const container = env.CODE_SANDBOX.getByName(executionId);
    await container.startAndWaitForPorts({
      startOptions: {
        envVars: {
          USER_CODE: Buffer.from(code).toString('base64'),
          TIMEOUT: "30000"
        }
      }
    });
    
    return container.fetch(new Request("http://container/execute", {
      method: "POST"
    }));
  }
}
```

### Resource-Intensive Processing

```typescript
export class VideoProcessor extends Container {
  defaultPort = 8080;
  sleepAfter = "2h";
  
  envVars = {
    MAX_MEMORY: "4096",
    FFMPEG_THREADS: "4"
  };
}

export default {
  async fetch(request: Request, env: Env) {
    const { videoId } = await request.json();
    
    // One container per video processing job
    const container = env.VIDEO_PROCESSOR.getByName(videoId);
    return container.fetch(request);
  }
}
```

### Scheduled Container Jobs (Cron)

```typescript
export default {
  async scheduled(event: ScheduledEvent, env: Env) {
    const jobId = `cron-${Date.now()}`;
    const container = env.BATCH_JOB.getByName(jobId);
    
    await container.startAndWaitForPorts({
      startOptions: {
        envVars: {
          JOB_TYPE: "daily-report",
          EXECUTION_TIME: new Date().toISOString()
        }
      }
    });
    
    await container.fetch(new Request("http://container/run-job"));
  }
}
```

### R2 FUSE Mount Example

Mount R2 buckets as filesystems in containers:

```typescript
import { Container } from "@cloudflare/containers";

export class R2FileProcessor extends Container {
  defaultPort = 8080;
  sleepAfter = "10m";
  
  async onStart() {
    // Container should mount R2 bucket via FUSE in Dockerfile/entrypoint
    console.log("R2-backed container started");
  }
}

export default {
  async fetch(request: Request, env: Env) {
    const { bucketPath } = await request.json();
    
    const container = env.R2_PROCESSOR.getByName("processor");
    await container.startAndWaitForPorts({
      startOptions: {
        envVars: {
          R2_BUCKET_NAME: env.R2_BUCKET.name,
          R2_ACCESS_KEY: env.R2_ACCESS_KEY,
          R2_SECRET_KEY: env.R2_SECRET_KEY,
          TARGET_PATH: bucketPath
        }
      }
    });
    
    return container.fetch(request);
  }
}
```

## Raw Durable Object Container API

If not using `@cloudflare/containers` package, use raw DO container methods:

### Properties
- `this.ctx.container.running` - Boolean, true if container running

### Methods

**`start(options?)`** - Boot container
```typescript
this.ctx.container.start({
  env: { FOO: "bar" },
  entrypoint: ["node", "server.js"],
  enableInternet: false
});
```

**`destroy(error?)`** - Stop container
```typescript
await this.ctx.container.destroy("Manual stop");
```

**`signal(signal)`** - Send IPC signal
```typescript
const SIGTERM = 15;
this.ctx.container.signal(SIGTERM);
```

**`getTcpPort(port)`** - Get TCP port for communication
```typescript
const port = this.ctx.container.getTcpPort(8080);
const res = await port.fetch("http://container/api");
```

**`monitor()`** - Promise that resolves on exit, rejects on error
```typescript
this.ctx.container.monitor()
  .then(() => console.log("Exited"))
  .catch((err) => console.error("Error:", err));
```

## Platform Details

### Limits
- Container images must be linux/amd64
- Max instances controlled by `max_instances` in config
- Containers run until `sleepAfter` idle timeout or manual stop
- First deployment takes several minutes to provision

### Location Selection
- Containers likely run near requesting Worker instance
- Not guaranteed to be co-located
- Future: latency-aware routing and autoscaling (unreleased)

### Image Registry
- Cloudflare-managed registry automatically integrated
- Supports external registries (currently Amazon ECR)
- Images cached between deployments (layer reuse)

### Deployment Process
When `wrangler deploy` runs:
1. Wrangler builds image using Docker
2. Image pushed to Cloudflare registry (or external)
3. Worker deployed with container configuration
4. Network provisioned to spawn container instances
5. First deploy: wait several minutes before containers ready

### Scaling (Current)
- Manual scaling via `getContainer()` with unique IDs
- `getRandom()` helper for simple load balancing (temporary)
- Each instance runs until `sleepAfter` timeout or manual stop

### Scaling (Upcoming)
Future autoscaling features:
- `autoscale = true` on Container class
- Automatic instance launch based on traffic/resources
- Latency-aware routing with `getContainer()`
- Custom readiness checks
- Automatic instance shutdown when no longer needed

## Best Practices

1. **Use `@cloudflare/containers` package** - Cleaner API than raw Durable Object methods

2. **Set appropriate `sleepAfter`** - Balance resource usage vs cold start latency
   - Short-lived jobs: "5m"
   - Session-based: "30m" - "2h"
   - Long-running services: "2h" or longer

3. **Choose routing pattern based on use case:**
   - Stateless services → `getRandom()` load balancing
   - Stateful sessions → `getByName()` with session/user ID
   - Short-lived jobs → Unique IDs with explicit lifecycle control

4. **Pass secrets securely:**
   - Use Worker Secrets or Secret Store, not hard-coded values
   - Read KV/secrets asynchronously when starting containers
   - Don't log sensitive environment variables

5. **Design for container restarts:**
   - Containers can stop after idle timeout
   - Persist critical state to Durable Object storage, D1, or R2
   - Handle cold starts gracefully

6. **Monitor container lifecycle:**
   - Use `onStart`/`onStop`/`onError` hooks
   - Log container events for debugging
   - Use `monitor()` for long-running jobs

7. **Optimize Docker images:**
   - Use multi-stage builds to reduce size
   - Leverage layer caching for faster deploys
   - Only include necessary dependencies

8. **Check deployment status:**
   - Run `wrangler containers list` after deploy
   - Wait for "ready" status before sending traffic
   - First deploy takes longer than subsequent deploys

9. **Development workflow:**
   - Ensure Docker is running (`docker info`)
   - Test Dockerfile locally before deploying
   - Use `wrangler containers build` to validate image builds

10. **Plan for future autoscaling:**
    - Current `getRandom()` is temporary
    - Design stateless containers for future automatic scaling
    - Prepare for latency-aware routing features

## Type Definitions

```typescript
// Env bindings
interface Env {
  MY_CONTAINER: DurableObjectNamespace;
  MY_SECRET: string;
  SECRET_STORE: SecretStoreBinding;
  DEMO_KV: KVNamespace;
  R2_BUCKET: R2Bucket;
}

// Container class configuration
class MyContainer extends Container {
  defaultPort: number;           // Port to use for fetch()
  sleepAfter: string;            // Idle timeout: "10s", "5m", "2h"
  envVars: Record<string, string>; // Environment variables
  
  override onStart(): void;
  override onStop(): void;
  override onError(error: unknown): void;
}

// Start options
interface StartOptions {
  env?: Record<string, string>;     // Environment variables
  entrypoint?: string[];            // Command to run
  enableInternet?: boolean;         // Allow internet access
}

// Container fetch method
interface ContainerInstance {
  fetch(request: Request): Promise<Response>;
  startAndWaitForPorts(options?: {
    startOptions?: StartOptions;
  }): Promise<void>;
  destroy(error?: string): Promise<void>;
}

// Helper functions
function getContainer(
  namespace: DurableObjectNamespace,
  id: string
): ContainerInstance;

function getRandom(
  namespace: DurableObjectNamespace,
  count: number
): Promise<ContainerInstance>;
```

## Resources

- Cloudflare Containers Docs: https://developers.cloudflare.com/containers/
- Container Package (GitHub): https://github.com/cloudflare/containers
- Container Package (NPM): https://www.npmjs.com/package/@cloudflare/containers
- Wrangler Commands: https://developers.cloudflare.com/workers/wrangler/commands/#containers
- Durable Objects: https://developers.cloudflare.com/durable-objects/
- Examples: https://developers.cloudflare.com/containers/examples/

## Troubleshooting

**Docker not running:**
```bash
# Check Docker status
docker info

# If fails, start Docker Desktop or Colima
```

**Container not receiving requests after deploy:**
- Wait several minutes after first deploy for provisioning
- Check deployment status: `wrangler containers list`
- Ensure container is listening on configured `defaultPort`

**Build failures:**
- Verify Dockerfile syntax and paths
- Test build locally: `docker build -t test .`
- Check image targets `linux/amd64` architecture

**Container errors:**
- Implement `onError()` hook to log errors
- Check container logs in Cloudflare dashboard
- Use `monitor()` to catch runtime errors

**Slow deployments:**
- First deploy is always slower (provisioning)
- Optimize Docker image size and layers
- Use multi-stage builds for faster layer caching
