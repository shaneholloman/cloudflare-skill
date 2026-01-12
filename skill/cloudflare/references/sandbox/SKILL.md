# Cloudflare Sandbox SDK Skill

## Overview

Cloudflare Sandbox SDK enables secure, isolated code execution in containers on Cloudflare's edge. Run untrusted code, execute commands, manage files, expose services, and integrate with AI agents—all within isolated sandbox environments backed by Durable Objects and Cloudflare Containers.

**Use for**: AI code execution, interactive dev environments, data analysis platforms, CI/CD systems, code interpreters, multi-tenant code execution.

---

## Core Architecture

### Sandbox Instance Model
- Each sandbox = Durable Object + Container
- Persistent across requests (same ID = same sandbox)
- Isolated filesystem, processes, network namespace
- Configurable sleep/wake behavior for cost optimization

### Client Architecture Pattern
```
SandboxClient (aggregator)
├── CommandClient     → exec(), streaming
├── FileClient        → read/write/list/delete
├── ProcessClient     → background processes
├── PortClient        → expose services, preview URLs
├── GitClient         → clone repos
├── UtilityClient     → health, sessions
└── InterpreterClient → code execution (Python/JS)
```

### Execution Modes
1. **Foreground** (exec): Blocking, captures output, returns when done
2. **Background** (execStream/startProcess): Non-blocking, uses FIFOs for streaming, concurrent operation

---

## Essential Setup Patterns

### 1. Basic Worker Setup

```typescript
import { getSandbox, proxyToSandbox, type Sandbox } from '@cloudflare/sandbox';

export { Sandbox } from '@cloudflare/sandbox';

type Env = {
  Sandbox: DurableObjectNamespace<Sandbox>;
};

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // CRITICAL: proxyToSandbox MUST be called first for preview URLs
    const proxyResponse = await proxyToSandbox(request, env);
    if (proxyResponse) return proxyResponse;

    const sandbox = getSandbox(env.Sandbox, 'my-sandbox');
    
    // Your sandbox logic here
    const result = await sandbox.exec('python3 -c "print(2 + 2)"');
    return Response.json({ output: result.stdout });
  }
};
```

**Key Rules**:
- ALWAYS call `proxyToSandbox()` first (required for preview URLs)
- Same sandbox ID = reuse existing container
- Export `Sandbox` type from SDK for Durable Object registration

### 2. Wrangler Configuration

**wrangler.jsonc** (recommended):
```jsonc
{
  "name": "my-sandbox-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-01-01",
  
  "containers": [
    {
      "class_name": "Sandbox",
      "image": "./Dockerfile",
      "instance_type": "lite",        // lite | standard | heavy
      "max_instances": 5              // concurrent sandboxes limit
    }
  ],
  
  "durable_objects": {
    "bindings": [
      {
        "class_name": "Sandbox",
        "name": "Sandbox"             // matches Env.Sandbox binding
      }
    ]
  },
  
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["Sandbox"]  // required once
    }
  ]
}
```

**Instance Types**:
- `lite`: 256MB RAM, 0.5 vCPU (default, cost-effective)
- `standard`: 512MB RAM, 1 vCPU
- `heavy`: 1GB RAM, 2 vCPU

**wrangler.toml** alternative:
```toml
name = "my-sandbox-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[containers]]
class_name = "Sandbox"
image = "./Dockerfile"
instance_type = "lite"
max_instances = 5

[[durable_objects.bindings]]
class_name = "Sandbox"
name = "Sandbox"

[[migrations]]
tag = "v1"
new_sqlite_classes = ["Sandbox"]
```

### 3. Dockerfile Configuration

**Basic Dockerfile**:
```dockerfile
FROM docker.io/cloudflare/sandbox:latest

# Python is pre-installed
# Add custom dependencies
RUN pip3 install --no-cache-dir pandas numpy matplotlib

# Expose ports for local dev (required for wrangler dev)
EXPOSE 8080 3000

# Entrypoint is already set by base image
```

**Custom Python Packages**:
```dockerfile
FROM docker.io/cloudflare/sandbox:latest

# Scientific computing stack
RUN pip3 install --no-cache-dir \
    jupyter-server \
    ipykernel \
    matplotlib \
    numpy \
    pandas \
    seaborn \
    plotly \
    scipy \
    scikit-learn
```

**Node.js Support**:
```dockerfile
FROM docker.io/cloudflare/sandbox:latest

# Node is pre-installed
# Add global packages
RUN npm install -g typescript ts-node
```

**CRITICAL for local dev**: `EXPOSE` directives required in Dockerfile for `wrangler dev` port access. Production auto-exposes all ports.

---

## getSandbox Configuration

```typescript
const sandbox = getSandbox(env.Sandbox, 'sandbox-id', {
  normalizeId: true,         // lowercase ID (required for preview URLs)
  sleepAfter: '30m',         // sleep after inactivity: '5m', '1h', '2d'
  keepAlive: false,          // false = auto-timeout, true = never sleep
  
  containerTimeouts: {
    instanceGetTimeoutMS: 30000,  // 30s for provisioning
    portReadyTimeoutMS: 90000     // 90s for container startup
  }
});
```

**Sleep Configuration**:
- `sleepAfter`: Duration string (e.g., '5m', '30m', '1h', '2d')
- `keepAlive: false`: Enables auto-sleep (default)
- `keepAlive: true`: Sandbox never sleeps (higher cost)
- Sleeping sandboxes wake automatically on next request (cold start)

**Normalization**:
- `normalizeId: true`: Required for preview URLs (converts to lowercase, URL-safe)
- Use when exposing ports with preview URLs

---

## Command Execution Patterns

### 1. Basic Execution

```typescript
const result = await sandbox.exec('python3 script.py');

// Result interface
{
  stdout: string,      // captured stdout
  stderr: string,      // captured stderr
  exitCode: number,    // 0 = success
  success: boolean,    // exitCode === 0
  duration: number     // execution time (ms)
}
```

### 2. Streaming Execution

```typescript
const result = await sandbox.exec('npm install', {
  stream: true,
  
  onOutput: (stream, data) => {
    console.log(`[${stream}]`, data);  // stream = 'stdout' | 'stderr'
  },
  
  onComplete: (result) => {
    console.log('Exit code:', result.exitCode);
  },
  
  onError: (error) => {
    console.error('Execution failed:', error);
  }
});
```

**Use streaming for**:
- Long-running commands (builds, installs)
- Real-time feedback to users
- Progress monitoring

### 3. Working Directory & Environment

```typescript
// Change directory, set env vars
const result = await sandbox.exec('python3 test.py', {
  cwd: '/workspace/project',
  env: {
    API_KEY: 'secret',
    DEBUG: 'true'
  }
});
```

### 4. Complex Commands

```typescript
// Multi-line scripts
const result = await sandbox.exec(`
  cd /workspace
  git clone https://github.com/user/repo.git
  cd repo
  npm install
  npm test
`);

// Pipe and redirection
const result = await sandbox.exec('cat data.csv | python3 process.py > output.json');
```

---

## File System Operations

### 1. Read Files

```typescript
// Read file
const file = await sandbox.readFile('/workspace/data.txt');
// Returns: { content: string, path: string }

// Read binary file
const imageFile = await sandbox.readFile('/workspace/image.png');
// content is base64-encoded for binary files
```

### 2. Write Files

```typescript
// Write text file
await sandbox.writeFile('/workspace/config.json', JSON.stringify(config));

// Write with explicit encoding
await sandbox.writeFile('/workspace/script.py', pythonCode, { encoding: 'utf-8' });

// Create nested directories automatically
await sandbox.writeFile('/workspace/deep/nested/file.txt', 'content');
```

### 3. List Directory Contents

```typescript
const files = await sandbox.listFiles('/workspace');

// Returns array of:
[
  {
    name: 'file.txt',
    path: '/workspace/file.txt',
    type: 'file',        // 'file' | 'directory'
    size: 1024,          // bytes
    modified: 1234567890 // Unix timestamp
  }
]
```

### 4. Delete Files

```typescript
// Delete file
await sandbox.deleteFile('/workspace/temp.txt');

// Delete directory recursively
await sandbox.deleteFile('/workspace/temp-dir', { recursive: true });
```

### 5. Directory Operations

```typescript
// Create directory
await sandbox.mkdir('/workspace/new-dir', { recursive: true });

// Check if path exists
const exists = await sandbox.pathExists('/workspace/file.txt');
```

---

## Background Processes

### 1. Start Process

```typescript
const process = await sandbox.startProcess('python3 -m http.server 8080', {
  processId: 'web-server',       // unique ID for this process
  cwd: '/workspace/public',
  env: { PORT: '8080' }
});

// Returns: { id: string, pid: number, command: string }
```

### 2. Process Management

```typescript
// List running processes
const processes = await sandbox.listProcesses();
// Returns: [{ id, pid, command, status }]

// Get process info
const info = await sandbox.getProcess('web-server');

// Stop process
await sandbox.stopProcess('web-server');

// Get process logs
const logs = await sandbox.getProcessLogs('web-server');
// Returns: { stdout: string, stderr: string }
```

### 3. Process Patterns

**Long-running services**:
```typescript
// Start API server
await sandbox.startProcess('node server.js', {
  processId: 'api',
  env: { PORT: '3000', NODE_ENV: 'production' }
});

// Expose to internet (see Port Exposure section)
const exposed = await sandbox.exposePort(3000, { name: 'api' });
```

**Background jobs**:
```typescript
// Start data processing job
await sandbox.startProcess('python3 process_data.py', {
  processId: 'data-job',
  cwd: '/workspace/jobs'
});

// Check status periodically
const status = await sandbox.getProcess('data-job');
if (status.status === 'exited') {
  const logs = await sandbox.getProcessLogs('data-job');
  console.log(logs.stdout);
}
```

---

## Port Exposure & Preview URLs

### Prerequisites
- Custom domain with wildcard DNS: `*.yourdomain.com → worker.yourdomain.com`
- `.workers.dev` domains do NOT support preview URLs
- `normalizeId: true` in getSandbox options
- `proxyToSandbox()` called first in fetch handler

### 1. Expose Port

```typescript
const exposed = await sandbox.exposePort(8080, {
  name: 'web-app',           // optional, for identification
  hostname: request.hostname // use request hostname
});

// Returns:
{
  url: 'https://8080-my-sandbox-abc123.yourdomain.com',
  port: 8080,
  name: 'web-app',
  status: 'active'
}
```

### 2. Check Port Status

```typescript
// Check if port is exposed
const isExposed = await sandbox.isPortExposed(8080);

// Get specific port info
const portInfo = await sandbox.getExposedPort(8080);

// List all exposed ports
const allPorts = await sandbox.getExposedPorts(request.hostname);
```

### 3. Unexpose Port

```typescript
await sandbox.unexposePort(8080);
```

### 4. Complete Example

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // REQUIRED: Handle preview URL routing
    const proxyResponse = await proxyToSandbox(request, env);
    if (proxyResponse) return proxyResponse;

    const url = new URL(request.url);
    const sandbox = getSandbox(env.Sandbox, 'dev-env', {
      normalizeId: true  // REQUIRED for preview URLs
    });

    if (url.pathname === '/start-server') {
      // Start web server
      await sandbox.writeFile('/workspace/index.html', '<h1>Hello!</h1>');
      await sandbox.startProcess('python3 -m http.server 8080', {
        processId: 'server',
        cwd: '/workspace'
      });

      // Wait for server to be ready
      await new Promise(resolve => setTimeout(resolve, 2000));

      // Expose port
      const exposed = await sandbox.exposePort(8080, {
        name: 'preview',
        hostname: request.hostname
      });

      return Response.json({
        message: 'Server started',
        url: exposed.url
      });
    }

    return new Response('Try /start-server');
  }
};
```

**Production Deployment Guide**: See https://developers.cloudflare.com/sandbox/guides/production-deployment/

---

## Sessions (Isolated Execution Contexts)

Sessions provide isolated execution contexts within a sandbox. Each session maintains its own:
- Shell state
- Environment variables
- Working directory
- Process namespace

### 1. Create Session

```typescript
const session = await sandbox.createSession({
  id: 'user-123',              // unique session ID
  name: 'User Workspace',      // optional display name
  cwd: '/workspace/user123',   // working directory
  env: {
    USER_ID: '123',
    API_KEY: 'secret'
  }
});

// Session has full sandbox API bound to its context
await session.exec('echo $USER_ID');
await session.writeFile('config.txt', 'data');
```

### 2. Retrieve Session

```typescript
// Get existing session
const session = await sandbox.getSession('user-123');

// List all sessions
const sessions = await sandbox.listSessions();
```

### 3. Session Operations

```typescript
// Execute in session context
const result = await session.exec('pwd');  // uses session's cwd

// File ops in session context
await session.writeFile('data.txt', 'content');  // relative to session cwd
const file = await session.readFile('data.txt');

// Start process in session
const process = await session.startProcess('python3 worker.py', {
  processId: 'worker-1'
});

// Get process logs
const logs = await session.getProcessLogs('worker-1');
```

### 4. Delete Session

```typescript
const result = await sandbox.deleteSession('user-123');
// Returns: { success: boolean, sessionId: string, timestamp: number }
```

### 5. Multi-Tenant Pattern

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const userId = request.headers.get('X-User-ID');
    const sandbox = getSandbox(env.Sandbox, 'multi-tenant');
    
    // Each user gets isolated session
    let session;
    try {
      session = await sandbox.getSession(userId);
    } catch {
      session = await sandbox.createSession({
        id: userId,
        cwd: `/workspace/users/${userId}`,
        env: { USER_ID: userId }
      });
    }

    // User's code runs in their session
    const code = await request.text();
    const result = await session.exec(`python3 -c "${code}"`);
    
    return Response.json({ output: result.stdout });
  }
};
```

---

## Code Interpreter Pattern

For rich Python/JavaScript code execution with output capture:

### 1. Basic Code Interpreter

```typescript
const result = await sandbox.interpret('python', {
  code: `
import matplotlib.pyplot as plt
plt.plot([1, 2, 3], [4, 5, 6])
plt.savefig('plot.png')
print("Chart created")
  `,
  files: {
    'data.csv': 'name,value\nalice,10\nbob,20'
  }
});

// Returns:
{
  outputs: [
    { type: 'text', content: 'Chart created' },
    { type: 'image', path: 'plot.png', format: 'png' }
  ],
  files: ['plot.png'],
  error: null
}
```

### 2. Jupyter Integration

**Dockerfile**:
```dockerfile
FROM docker.io/cloudflare/sandbox:latest

RUN pip3 install --no-cache-dir \
    jupyter-server \
    ipykernel \
    matplotlib \
    pandas

EXPOSE 8888
```

**Worker**:
```typescript
// Start Jupyter
await sandbox.startProcess('jupyter notebook --ip=0.0.0.0 --port=8888 --no-browser', {
  processId: 'jupyter',
  cwd: '/workspace'
});

const exposed = await sandbox.exposePort(8888, { name: 'jupyter' });
return Response.json({ url: exposed.url });
```

---

## Git Operations

```typescript
// Clone repository
await sandbox.exec('git clone https://github.com/user/repo.git /workspace/repo');

// Clone with specific branch
await sandbox.exec('git clone -b main --single-branch https://github.com/user/repo.git /workspace/repo');

// Authenticated clone (use secrets)
const token = env.GITHUB_TOKEN;
await sandbox.exec(`git clone https://${token}@github.com/user/private-repo.git`);

// Git operations
await sandbox.exec('git pull', { cwd: '/workspace/repo' });
await sandbox.exec('git checkout -b feature', { cwd: '/workspace/repo' });
```

---

## Error Handling

### 1. Command Errors

```typescript
const result = await sandbox.exec('python3 invalid.py');

if (!result.success) {
  console.error('Command failed');
  console.error('Exit code:', result.exitCode);
  console.error('Stderr:', result.stderr);
}
```

### 2. SDK Errors

```typescript
try {
  await sandbox.readFile('/nonexistent');
} catch (error) {
  if (error.code === 'FILE_NOT_FOUND') {
    // Handle missing file
  } else if (error.code === 'CONTAINER_NOT_READY') {
    // Container still provisioning (wait and retry)
  } else if (error.code === 'TIMEOUT') {
    // Operation timed out
  }
  throw error;
}
```

### 3. Retry Pattern

```typescript
async function execWithRetry(sandbox, command, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await sandbox.exec(command);
    } catch (error) {
      if (error.code === 'CONTAINER_NOT_READY' && i < maxRetries - 1) {
        await new Promise(resolve => setTimeout(resolve, 2000));
        continue;
      }
      throw error;
    }
  }
}
```

---

## Common Use Cases

### 1. AI Code Execution Agent

```typescript
// Give LLM a Python REPL
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { code } = await request.json();
    const sandbox = getSandbox(env.Sandbox, 'ai-agent');

    // Execute user-provided code safely
    const result = await sandbox.exec(`python3 -c "${code.replace(/"/g, '\\"')}"`);

    return Response.json({
      output: result.stdout,
      error: result.stderr,
      success: result.success
    });
  }
};
```

### 2. Interactive Development Environment

```typescript
// VS Code Server in sandbox
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const proxyResponse = await proxyToSandbox(request, env);
    if (proxyResponse) return proxyResponse;

    const sandbox = getSandbox(env.Sandbox, 'ide', { normalizeId: true });

    if (request.url.endsWith('/start')) {
      // Install and start code-server
      await sandbox.exec('curl -fsSL https://code-server.dev/install.sh | sh');
      await sandbox.startProcess('code-server --bind-addr 0.0.0.0:8080', {
        processId: 'vscode'
      });

      const exposed = await sandbox.exposePort(8080);
      return Response.json({ url: exposed.url });
    }

    return new Response('Try /start');
  }
};
```

### 3. CI/CD Pipeline

```typescript
// Run tests in sandbox
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { repo, branch } = await request.json();
    const sandbox = getSandbox(env.Sandbox, `ci-${repo}-${Date.now()}`);

    // Clone and test
    await sandbox.exec(`git clone -b ${branch} ${repo} /workspace/repo`);
    
    const install = await sandbox.exec('npm install', {
      cwd: '/workspace/repo',
      stream: true,
      onOutput: (stream, data) => console.log(data)
    });

    if (!install.success) {
      return Response.json({ success: false, error: 'Install failed' });
    }

    const test = await sandbox.exec('npm test', { cwd: '/workspace/repo' });

    return Response.json({
      success: test.success,
      output: test.stdout,
      exitCode: test.exitCode
    });
  }
};
```

### 4. Data Analysis Platform

```typescript
// Jupyter notebook execution
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { notebook } = await request.json();
    const sandbox = getSandbox(env.Sandbox, 'data-analysis');

    // Write notebook
    await sandbox.writeFile('/workspace/analysis.ipynb', JSON.stringify(notebook));

    // Execute notebook
    const result = await sandbox.exec(
      'jupyter nbconvert --to notebook --execute analysis.ipynb --output results.ipynb',
      { cwd: '/workspace' }
    );

    // Read results
    const output = await sandbox.readFile('/workspace/results.ipynb');

    return Response.json({
      success: result.success,
      notebook: JSON.parse(output.content)
    });
  }
};
```

### 5. Code Snippet Executor with Multiple Languages

```typescript
const languageConfigs = {
  python: { cmd: 'python3', ext: 'py' },
  javascript: { cmd: 'node', ext: 'js' },
  typescript: { cmd: 'ts-node', ext: 'ts' },
  bash: { cmd: 'bash', ext: 'sh' }
};

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { language, code } = await request.json();
    const config = languageConfigs[language];
    
    if (!config) {
      return Response.json({ error: 'Unsupported language' }, { status: 400 });
    }

    const sandbox = getSandbox(env.Sandbox, 'code-runner');
    const filename = `/workspace/script.${config.ext}`;

    await sandbox.writeFile(filename, code);
    const result = await sandbox.exec(`${config.cmd} ${filename}`);

    return Response.json({
      output: result.stdout,
      error: result.stderr,
      exitCode: result.exitCode
    });
  }
};
```

---

## Storage Integration (R2/S3)

Mount object storage as filesystem:

```typescript
const sandbox = getSandbox(env.Sandbox, 'data-processor');

// Mount R2 bucket
await sandbox.mountStorage({
  type: 'r2',
  bucket: env.DATA_BUCKET,
  mountPoint: '/data',
  readOnly: false
});

// Access files in mounted storage
const result = await sandbox.exec('python3 process.py', {
  cwd: '/data/datasets'
});

// Files written to /data persist in R2
await sandbox.writeFile('/data/output/results.csv', csvData);
```

**Configuration**:
```jsonc
{
  "r2_buckets": [
    {
      "binding": "DATA_BUCKET",
      "bucket_name": "my-data-bucket"
    }
  ]
}
```

---

## Local Development with Wrangler

### 1. Start Dev Server

```bash
npm run dev
# or
wrangler dev
```

**First run**: Builds Docker container (2-3 min)
**Subsequent runs**: Uses cache (fast)

### 2. Port Exposure in Dev

**CRITICAL**: Dockerfile must include `EXPOSE` for ports used in `wrangler dev`:

```dockerfile
FROM docker.io/cloudflare/sandbox:latest
EXPOSE 8080 3000 8888
```

Without `EXPOSE`, you'll get "Connection refused: container port not found"

**Production**: All ports auto-accessible (no EXPOSE needed)

### 3. Test Locally

```bash
# Execute code
curl http://localhost:8787/run

# File operations
curl http://localhost:8787/file

# Start service
curl http://localhost:8787/start-server
```

---

## Production Deployment

### 1. Deploy

```bash
wrangler deploy
```

This:
1. Builds Docker image
2. Pushes to Cloudflare Container Registry
3. Deploys Worker globally

**First deploy**: Wait 2-3 min for container provisioning

### 2. Check Container Status

```bash
wrangler containers list
```

### 3. Monitor Logs

```bash
wrangler tail
```

### 4. Environment Variables & Secrets

```bash
# Set secret
wrangler secret put GITHUB_TOKEN

# Use in Worker
const token = env.GITHUB_TOKEN;
```

**wrangler.jsonc**:
```jsonc
{
  "vars": {
    "ENVIRONMENT": "production",
    "API_URL": "https://api.example.com"
  }
}
```

---

## Performance & Optimization

### 1. Sandbox ID Strategy

```typescript
// ❌ BAD: Creates new sandbox every time (slow, expensive)
const sandbox = getSandbox(env.Sandbox, `user-${Date.now()}`);

// ✅ GOOD: Reuse sandbox per user
const sandbox = getSandbox(env.Sandbox, `user-${userId}`);

// ✅ GOOD: Reuse for temporary tasks
const sandbox = getSandbox(env.Sandbox, 'shared-runner');
```

### 2. Sleep Configuration

```typescript
// Cost-optimized: Sleep after 30min inactivity
const sandbox = getSandbox(env.Sandbox, 'id', {
  sleepAfter: '30m',
  keepAlive: false
});

// Always-on (higher cost, faster response)
const sandbox = getSandbox(env.Sandbox, 'id', {
  keepAlive: true
});
```

### 3. Pre-warm Containers

```typescript
// Cron trigger to keep sandbox warm
export default {
  async scheduled(event: ScheduledEvent, env: Env) {
    const sandbox = getSandbox(env.Sandbox, 'main');
    await sandbox.exec('echo "keepalive"');  // Wake sandbox
  }
};
```

**wrangler.jsonc**:
```jsonc
{
  "triggers": {
    "crons": ["*/5 * * * *"]  // Every 5 minutes
  }
}
```

### 4. Increase max_instances for High Traffic

```jsonc
{
  "containers": [
    {
      "class_name": "Sandbox",
      "max_instances": 50  // Allow 50 concurrent sandboxes
    }
  ]
}
```

---

## Security Best Practices

### 1. Sandbox Isolation
- Each sandbox = isolated container (filesystem, network, processes)
- Use unique sandbox IDs per tenant for multi-tenant apps
- Sandboxes cannot communicate directly

### 2. Input Validation

```typescript
// ❌ DANGEROUS: Command injection
const result = await sandbox.exec(`python3 -c "${userCode}"`);

// ✅ SAFE: Write to file, execute file
await sandbox.writeFile('/workspace/user_code.py', userCode);
const result = await sandbox.exec('python3 /workspace/user_code.py');
```

### 3. Resource Limits

```typescript
// Timeout long-running commands
const result = await sandbox.exec('python3 script.py', {
  timeout: 30000  // 30 seconds
});
```

### 4. Secrets Management

```typescript
// ❌ NEVER hardcode secrets
const token = 'ghp_abc123';

// ✅ Use environment secrets
const token = env.GITHUB_TOKEN;

// Pass to sandbox via exec env
const result = await sandbox.exec('git clone ...', {
  env: { GIT_TOKEN: token }
});
```

### 5. Preview URL Security

Preview URLs include auto-generated tokens:
```
https://8080-sandbox-abc123def456.yourdomain.com
```

Token changes on each expose operation, preventing unauthorized access.

---

## Troubleshooting

### Container Not Ready
**Error**: `CONTAINER_NOT_READY`
**Cause**: Container still provisioning (first request or after sleep)
**Fix**: Retry after 2-3 seconds

```typescript
async function execWithRetry(sandbox, cmd) {
  for (let i = 0; i < 3; i++) {
    try {
      return await sandbox.exec(cmd);
    } catch (e) {
      if (e.code === 'CONTAINER_NOT_READY') {
        await new Promise(r => setTimeout(r, 2000));
        continue;
      }
      throw e;
    }
  }
}
```

### Port Exposure Fails in Dev
**Error**: "Connection refused: container port not found"
**Cause**: Missing `EXPOSE` directive in Dockerfile
**Fix**: Add `EXPOSE <port>` to Dockerfile

### Preview URLs Not Working
**Checklist**:
1. Custom domain configured? (not `.workers.dev`)
2. Wildcard DNS set up? (`*.domain.com → worker.domain.com`)
3. `normalizeId: true` in getSandbox?
4. `proxyToSandbox()` called first in fetch?

### Slow First Request
**Cause**: Cold start (container provisioning)
**Solutions**:
- Use `sleepAfter` instead of creating new sandboxes
- Pre-warm with cron triggers
- Set `keepAlive: true` for critical sandboxes

### File Not Persisting
**Cause**: Files in `/tmp` or other ephemeral paths
**Fix**: Use `/workspace` for persistent files

---

## Quick Reference

### Essential Imports
```typescript
import { getSandbox, proxyToSandbox, type Sandbox } from '@cloudflare/sandbox';
export { Sandbox } from '@cloudflare/sandbox';
```

### Core APIs
- `getSandbox(namespace, id, options?)` → Get/create sandbox
- `sandbox.exec(command, options?)` → Execute command
- `sandbox.readFile(path)` → Read file
- `sandbox.writeFile(path, content)` → Write file
- `sandbox.listFiles(path)` → List directory
- `sandbox.startProcess(command, options)` → Background process
- `sandbox.exposePort(port, options)` → Get preview URL
- `sandbox.createSession(options)` → Isolated session

### Configuration Keys
- `wrangler.jsonc`: `containers`, `durable_objects`, `migrations`
- Dockerfile: `FROM cloudflare/sandbox:latest`, `EXPOSE`, `RUN`
- getSandbox: `normalizeId`, `sleepAfter`, `keepAlive`

### Common Patterns
- Always call `proxyToSandbox()` first
- Same ID = reuse sandbox
- Use `/workspace` for files
- `normalizeId: true` for preview URLs
- Retry on `CONTAINER_NOT_READY`

---

## Additional Resources

- **Official Docs**: https://developers.cloudflare.com/sandbox/
- **API Reference**: https://developers.cloudflare.com/sandbox/api/
- **Examples Repo**: https://github.com/cloudflare/sandbox-sdk/tree/main/examples
- **npm Package**: https://www.npmjs.com/package/@cloudflare/sandbox
- **Discord Support**: https://discord.cloudflare.com

---

## Unresolved Questions

None - comprehensive coverage of Cloudflare Sandbox SDK achieved.
