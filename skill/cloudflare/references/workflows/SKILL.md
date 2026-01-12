# Cloudflare Workflows Skill

**Skill for developing with Cloudflare Workflows - durable multi-step applications with automatic retries, state persistence, and long-running execution.**

## Overview

Cloudflare Workflows enable building durable multi-step applications that:
- Chain together multiple steps with automatic retry logic
- Persist state between steps (minutes, hours, or weeks)
- Handle failures gracefully without losing progress
- Wait for external events or approvals
- Sleep for extended periods without consuming resources

**Available on:** Free and Paid Workers plans

## Core Concepts

### Workflow Definition

A Workflow is a class that extends `WorkflowEntrypoint` and defines a `run` method:

```typescript
import { WorkflowEntrypoint, WorkflowStep, WorkflowEvent } from 'cloudflare:workers';

type Env = {
  MY_WORKFLOW: Workflow;
  // Other bindings (KV, R2, D1, AI, etc.)
};

type Params = {
  // User-defined parameters
  userId: string;
  data: Record<string, unknown>;
};

export class MyWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    // Access bindings via this.env
    // Access params via event.payload
    
    const result = await step.do('step name', async () => {
      // Step logic here
      return { someData: 'value' };
    });
    
    // Use result in subsequent steps
  }
}
```

### Workflow Instance

Each invocation of a Workflow creates an **instance** with:
- Unique ID (auto-generated or user-provided)
- Independent state and lifecycle
- Parallel execution capability

### Steps

Steps are independently retriable units of work. Use `step.do()` to wrap:
- API calls
- Database queries (D1, KV, R2)
- AI model invocations
- Third-party service calls
- Data processing

**Key characteristics:**
- Each step can retry independently
- State returned from a step is persisted
- Failed steps don't force re-execution of successful steps
- Step name acts as a "cache key" for state

## Configuration

### Wrangler Configuration

Define Workflows in `wrangler.toml` or `wrangler.jsonc`:

```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2024-10-22"

[[workflows]]
name = "my-workflow"           # Workflow name
binding = "MY_WORKFLOW"        # Binding variable name in code
class_name = "MyWorkflow"      # Class name in your code

# Optional: for cross-script calls
# script_name = "other-worker"
```

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-10-22",
  "workflows": [
    {
      "name": "my-workflow",
      "binding": "MY_WORKFLOW",
      "class_name": "MyWorkflow"
    }
  ]
}
```

### CPU Limits

Increase CPU time per invocation (default 30s, max 5 minutes):

```toml
[limits]
cpu_ms = 300_000  # 5 minutes
```

```jsonc
{
  "limits": {
    "cpu_ms": 300000
  }
}
```

## Step Patterns

### Basic Step

```typescript
const data = await step.do('fetch user data', async () => {
  const user = await this.env.DB.prepare(
    'SELECT * FROM users WHERE id = ?'
  ).bind(event.payload.userId).first();
  return user;
});
```

### Step with Retry Configuration

```typescript
const result = await step.do(
  'call external API',
  {
    retries: {
      limit: 10,              // Max attempts (default: 5, or Infinity)
      delay: '10 seconds',    // Delay between retries (default: 10000ms)
      backoff: 'exponential'  // 'constant' | 'linear' | 'exponential'
    },
    timeout: '30 minutes'     // Timeout per attempt (default: '10 minutes')
  },
  async () => {
    const response = await fetch('https://api.example.com/data');
    if (!response.ok) throw new Error('API call failed');
    return response.json();
  }
);
```

### Parallel Steps

```typescript
// Execute steps concurrently
const [user, settings, logs] = await Promise.all([
  step.do('fetch user', async () => {
    return await this.env.KV.get(`user:${event.payload.userId}`);
  }),
  step.do('fetch settings', async () => {
    return await this.env.KV.get(`settings:${event.payload.userId}`);
  }),
  step.do('fetch logs', async () => {
    return await this.env.R2_BUCKET.get(`logs/${event.payload.userId}.json`);
  })
]);
```

### Conditional Steps

```typescript
const config = await step.do('fetch config', async () => {
  return await this.env.KV.get('feature-flags', { type: 'json' });
});

// ✅ Condition based on step output (deterministic)
if (config.enableEmailNotifications) {
  await step.do('send email', async () => {
    await sendEmail(event.payload.userEmail);
  });
}
```

### Dynamic Steps (Loops)

```typescript
const fileList = await step.do('get file list', async () => {
  return await this.env.BUCKET.list();
});

// Process each file in a separate step
for (const file of fileList.objects) {
  await step.do(`process ${file.key}`, async () => {
    const object = await this.env.BUCKET.get(file.key);
    const data = await object.arrayBuffer();
    // Process data...
    return { processed: true };
  });
}
```

## Sleep and Scheduling

### Sleep for Duration

```typescript
// Sleep for a relative period
await step.sleep('wait 1 hour', '1 hour');
await step.sleep('wait 30 days', '30 days');
await step.sleep('wait 5 seconds', 5000); // milliseconds

// Accepted units: second, minute, hour, day, week, month, year
```

### Sleep Until Date

```typescript
// Sleep until specific timestamp
const launchDate = Date.parse('24 Oct 2024 13:00:00 UTC');
await step.sleepUntil('wait until launch', launchDate);

// Or with Date object
const deadline = new Date('2024-12-31T23:59:59Z');
await step.sleepUntil('wait until deadline', deadline);
```

**Important:** Workflows in `waiting` state (sleeping) don't count toward concurrency limits.

## Events and Parameters

### Passing Parameters

**From Worker:**
```typescript
export default {
  async fetch(req: Request, env: Env) {
    const payload = {
      userId: 'user123',
      email: 'user@example.com',
      timestamp: Date.now()
    };
    
    const instance = await env.MY_WORKFLOW.create({
      id: crypto.randomUUID(),
      params: payload
    });
    
    return Response.json({ id: instance.id });
  }
};
```

**Via Wrangler CLI:**
```bash
npx wrangler workflows trigger my-workflow '{"userId":"user123","email":"user@example.com"}'
```

### Accessing Parameters

```typescript
export class MyWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    // Access parameters
    const userId = event.payload.userId;
    const email = event.payload.email;
    const instanceId = event.instanceId;
    const createdAt = event.timestamp;
    
    // Use in steps...
  }
}
```

### Wait for External Events

```typescript
export class MyWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    // Process initial data
    await step.do('initial processing', async () => {
      // ... processing logic
    });
    
    // Wait for external event (e.g., webhook, approval)
    const webhookData = await step.waitForEvent<WebhookPayload>(
      'wait for stripe webhook',
      { 
        type: 'stripe-webhook',
        timeout: '24 hours'  // Default: 24 hours, max: 365 days
      }
    );
    
    // Continue with webhook data
    await step.do('process payment', async () => {
      // Use webhookData...
    });
  }
}
```

### Send Events to Running Workflows

**From Worker:**
```typescript
export default {
  async fetch(req: Request, env: Env) {
    const instanceId = new URL(req.url).searchParams.get('instanceId');
    const webhookPayload = await req.json();
    
    const instance = await env.MY_WORKFLOW.get(instanceId);
    
    // Send event with matching type
    await instance.sendEvent({
      type: 'stripe-webhook',
      payload: webhookPayload
    });
    
    return Response.json({ status: await instance.status() });
  }
};
```

**Timeout Handling:**
```typescript
try {
  const event = await step.waitForEvent('wait for approval', {
    type: 'approval',
    timeout: '1 hour'
  });
  // Handle received event
} catch (e) {
  // Timeout occurred
  console.log('No approval received, proceeding with default');
}
```

## Triggering Workflows

### From Worker (HTTP Handler)

```typescript
export default {
  async fetch(req: Request, env: Env) {
    // Create new instance
    const instance = await env.MY_WORKFLOW.create({
      id: crypto.randomUUID(),  // Optional: auto-generated if omitted
      params: { userId: 'user123' }
    });
    
    return Response.json({
      id: instance.id,
      status: await instance.status()
    });
  }
};
```

### From Queue Consumer

```typescript
export default {
  async queue(batch: MessageBatch<any>, env: Env) {
    for (const message of batch.messages) {
      await env.MY_WORKFLOW.create({
        id: `job-${message.id}`,
        params: message.body
      });
    }
  }
};
```

### From Cron Trigger

```typescript
export default {
  async scheduled(event: ScheduledEvent, env: Env) {
    // Trigger daily cleanup workflow
    await env.CLEANUP_WORKFLOW.create({
      id: `cleanup-${Date.now()}`,
      params: { timestamp: event.scheduledTime }
    });
  }
};
```

### From Another Workflow

```typescript
export class ParentWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    const result = await step.do('initial work', async () => {
      return { fileKey: 'output.pdf' };
    });
    
    // Trigger child workflow (non-blocking)
    const childInstance = await step.do('trigger child workflow', async () => {
      return await this.env.CHILD_WORKFLOW.create({
        id: `child-${event.instanceId}`,
        params: { fileKey: result.fileKey }
      });
    });
    
    // Parent continues immediately
    await step.do('continue other work', async () => {
      console.log(`Started child: ${childInstance.id}`);
    });
  }
}
```

### Batch Creation

```typescript
// Create up to 100 instances at once
const instances = [
  { id: 'user1', params: { name: 'John' } },
  { id: 'user2', params: { name: 'Jane' } },
  { id: 'user3', params: { name: 'Alice' } }
];

const created = await env.MY_WORKFLOW.createBatch(instances);
// Idempotent: skips existing IDs within retention period
```

## Instance Management

### Get Instance Status

```typescript
const instance = await env.MY_WORKFLOW.get('instance-id');
const status = await instance.status();

// status.status: 'queued' | 'running' | 'paused' | 'errored' | 'terminated' | 'complete' | 'waiting' | 'waitingForPause' | 'unknown'
// status.error: { name: string, message: string } | undefined
// status.output: any (returned from run method)
```

### Pause/Resume

```typescript
const instance = await env.MY_WORKFLOW.get('instance-id');

await instance.pause();   // Pause running instance
await instance.resume();  // Resume paused instance
```

### Terminate

```typescript
const instance = await env.MY_WORKFLOW.get('instance-id');

// Cannot be resumed after termination
await instance.terminate();
```

### Restart

```typescript
const instance = await env.MY_WORKFLOW.get('instance-id');

// Cancels in-progress steps, erases state, runs from beginning
await instance.restart();
```

## Error Handling

### Non-Retryable Errors

```typescript
import { NonRetryableError } from 'cloudflare:workflows';

await step.do('validate payment', async () => {
  if (!event.payload.paymentMethod) {
    // Fail immediately without retrying
    throw new NonRetryableError('Payment method is required');
  }
  
  const response = await fetch('https://payment-api.example.com/charge', {
    method: 'POST',
    body: JSON.stringify(event.payload)
  });
  
  if (response.status === 401) {
    // Authentication failures shouldn't retry
    throw new NonRetryableError('Invalid API credentials');
  }
  
  if (!response.ok) {
    // Retryable error
    throw new Error(`Payment failed: ${response.statusText}`);
  }
  
  return response.json();
});
```

### Catching Step Errors

```typescript
try {
  await step.do('risky operation', async () => {
    // Operation that might fail
    throw new NonRetryableError('Something went wrong');
  });
} catch (e) {
  console.log(`Step failed: ${e.message}`);
  
  // Run cleanup step
  await step.do('cleanup', async () => {
    // Cleanup logic...
  });
}

// Workflow continues execution
await step.do('next step', async () => {
  // More work...
});
```

### Idempotency Checks

```typescript
await step.do('charge customer', async () => {
  const customerId = event.payload.customerId;
  
  // Check if already charged (idempotency)
  const subscription = await fetch(
    `https://payment.processor/subscriptions/${customerId}`
  ).then(res => res.json());
  
  if (subscription.charged) {
    return subscription; // Already charged, return early
  }
  
  // Perform non-idempotent operation
  return await fetch(
    `https://payment.processor/subscriptions/${customerId}`,
    {
      method: 'POST',
      body: JSON.stringify({ amount: 10.0 })
    }
  ).then(res => res.json());
});
```

## Wrangler CLI Usage

### Create Project

```bash
npm create cloudflare@latest my-workflow -- --template "cloudflare/workflows-starter"
cd my-workflow
```

### Deploy

```bash
npx wrangler deploy
```

### List Workflows

```bash
npx wrangler workflows list
```

### Trigger Instance

```bash
# With parameters
npx wrangler workflows trigger my-workflow '{"userId":"user123"}'

# Without parameters
npx wrangler workflows trigger my-workflow
```

### List Instances

```bash
npx wrangler workflows instances list my-workflow
```

### Describe Instance

```bash
# By ID
npx wrangler workflows instances describe my-workflow instance-id

# Latest instance
npx wrangler workflows instances describe my-workflow latest
```

### Pause/Resume/Terminate

```bash
npx wrangler workflows instances pause my-workflow instance-id
npx wrangler workflows instances resume my-workflow instance-id
npx wrangler workflows instances terminate my-workflow instance-id
```

## REST API

### Create Instance

```bash
curl -X POST \
  "https://api.cloudflare.com/client/v4/accounts/{account_id}/workflows/{workflow_name}/instances" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "custom-instance-id",
    "params": {"userId": "user123"}
  }'
```

### Get Instance Status

```bash
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/workflows/{workflow_name}/instances/{instance_id}/status" \
  -H "Authorization: Bearer {api_token}"
```

### Send Event to Instance

```bash
curl -X POST \
  "https://api.cloudflare.com/client/v4/accounts/{account_id}/workflows/{workflow_name}/instances/{instance_id}/events" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "approval",
    "payload": {"approved": true}
  }'
```

## Common Patterns

### Image Processing Pipeline

```typescript
export class ImageProcessingWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    // Fetch image from R2
    const imageData = await step.do('fetch image', async () => {
      const object = await this.env.BUCKET.get(event.payload.imageKey);
      return await object.arrayBuffer();
    });
    
    // Generate AI description
    const description = await step.do('generate description', async () => {
      const imageArray = Array.from(new Uint8Array(imageData));
      return await this.env.AI.run('@cf/llava-hf/llava-1.5-7b-hf', {
        image: imageArray,
        prompt: 'Describe this image',
        max_tokens: 50
      });
    });
    
    // Wait for approval
    await step.waitForEvent('await approval', {
      event: 'approved',
      timeout: '24 hours'
    });
    
    // Publish to public bucket
    await step.do('publish', async () => {
      await this.env.BUCKET.put(`public/${event.payload.imageKey}`, imageData);
    });
  }
}
```

### User Lifecycle Management

```typescript
export class UserLifecycleWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    // Send welcome email
    await step.do('send welcome email', async () => {
      await sendEmail(event.payload.email, 'Welcome!');
    });
    
    // Wait 7 days
    await step.sleep('wait for trial period', '7 days');
    
    // Check if user has converted
    const hasConverted = await step.do('check conversion', async () => {
      const user = await this.env.DB.prepare(
        'SELECT subscription_status FROM users WHERE id = ?'
      ).bind(event.payload.userId).first();
      return user.subscription_status === 'active';
    });
    
    if (!hasConverted) {
      // Send trial expiration email
      await step.do('send trial expiration email', async () => {
        await sendEmail(event.payload.email, 'Your trial is ending');
      });
    }
  }
}
```

### Data Pipeline with Retry Logic

```typescript
export class DataPipelineWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    // Extract data from source
    const rawData = await step.do('extract data', {
      retries: { limit: 10, delay: '30 seconds', backoff: 'exponential' },
      timeout: '5 minutes'
    }, async () => {
      const response = await fetch(event.payload.sourceUrl);
      if (!response.ok) throw new Error('Failed to fetch data');
      return response.json();
    });
    
    // Transform data
    const transformed = await step.do('transform data', async () => {
      return rawData.map(item => ({
        id: item.id,
        normalized: normalizeData(item)
      }));
    });
    
    // Store large dataset in R2 (avoid 1 MiB step limit)
    const dataRef = await step.do('store transformed data', async () => {
      const key = `processed/${Date.now()}.json`;
      await this.env.BUCKET.put(key, JSON.stringify(transformed));
      return { key };
    });
    
    // Load into database in batches
    await step.do('load to database', {
      retries: { limit: 5, delay: '1 minute', backoff: 'linear' }
    }, async () => {
      const stored = await this.env.BUCKET.get(dataRef.key);
      const data = await stored.json();
      
      // Batch insert
      for (let i = 0; i < data.length; i += 100) {
        const batch = data.slice(i, i + 100);
        await this.env.DB.batch(
          batch.map(item => 
            this.env.DB.prepare('INSERT INTO records VALUES (?, ?)').bind(item.id, item.normalized)
          )
        );
      }
    });
  }
}
```

### Human-in-the-Loop Approval

```typescript
export class ApprovalWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    // Create approval request
    await step.do('create approval request', async () => {
      await this.env.DB.prepare(
        'INSERT INTO approvals (id, user_id, status) VALUES (?, ?, ?)'
      ).bind(event.instanceId, event.payload.userId, 'pending').run();
    });
    
    // Wait for approval (with timeout)
    try {
      const approval = await step.waitForEvent<{ approved: boolean }>(
        'wait for approval',
        { type: 'approval-response', timeout: '48 hours' }
      );
      
      if (approval.approved) {
        await step.do('process approval', async () => {
          // Approval logic...
        });
      } else {
        await step.do('handle rejection', async () => {
          // Rejection logic...
        });
      }
    } catch (e) {
      // Timeout: auto-reject
      await step.do('auto reject', async () => {
        await this.env.DB.prepare(
          'UPDATE approvals SET status = ? WHERE id = ?'
        ).bind('auto-rejected', event.instanceId).run();
      });
    }
  }
}
```

## Best Practices

### ✅ DO

1. **Make steps granular and self-contained**
   - One API call per step (unless proving idempotency)
   - Separate unrelated operations into different steps
   - Each step = one unit of work

2. **Ensure idempotency**
   - Check if operation already completed before executing
   - Use idempotency keys for external APIs
   - Design steps to be safely retried

3. **Name steps deterministically**
   - Use static names or names based on step outputs
   - Step names act as cache keys
   - Avoid `Date.now()`, `Math.random()` in step names

4. **Return state from steps**
   - Persist data by returning from `step.do()`
   - Build top-level state from step returns only
   - Don't rely on variables outside steps

5. **Always `await` steps**
   - Use `await step.do()` and `await step.sleep()`
   - Avoid dangling promises
   - Ensure errors are caught

6. **Use deterministic conditionals**
   - Base conditions on `event.payload` or step outputs
   - Wrap non-deterministic values in steps
   - Ensure reproducible control flow

7. **Store large data externally**
   - Keep step returns under 1 MiB
   - Use R2/KV for large datasets
   - Return references, not full data

8. **Batch instance creation**
   - Use `createBatch()` for multiple instances
   - Improves throughput and rate limits
   - Up to 100 instances per call

### ❌ DON'T

1. **Don't wrap entire logic in one step**
   - Breaks durability guarantees
   - Loses granular retry control
   - Harder to debug

2. **Don't store state outside steps**
   - In-memory variables lost on hibernation
   - Not persisted across engine lifetimes
   - Use step returns instead

3. **Don't mutate incoming events**
   - Events are immutable
   - Changes not persisted across steps
   - Return new state from steps

4. **Don't use non-deterministic logic outside steps**
   - `Math.random()`, `Date.now()` outside steps
   - Can cause different behavior on restart
   - Wrap in `step.do()` instead

5. **Don't perform side effects outside steps**
   - May execute multiple times on restart
   - Not protected by durability guarantees
   - Logs outside steps may duplicate

6. **Don't name steps non-deterministically**
   - Prevents state caching
   - Forces unnecessary re-execution
   - Use dynamic names only deterministically

7. **Don't forget to handle timeouts**
   - `waitForEvent` throws on timeout
   - Wrap in try-catch if continuing is acceptable
   - Set appropriate timeout durations

8. **Don't reuse instance IDs**
   - IDs must be unique within retention period
   - Use composite IDs if mapping to users
   - Store ID mappings in database

## Limits

| Resource | Free Plan | Paid Plan |
|----------|-----------|-----------|
| **CPU time per step** | 10 ms | 30 seconds (default), configurable to 5 minutes |
| **Max step state** | 1 MiB | 1 MiB |
| **Max event payload** | 1 MiB | 1 MiB |
| **Max instance state** | 100 MB | 1 GB |
| **Max sleep duration** | 365 days | 365 days |
| **Max steps per workflow** | 1,024 | 1,024 |
| **Workflow executions** | 100,000/day | Unlimited |
| **Concurrent instances** | 25 | 10,000 |
| **Instance creation rate** | 100/second | 100/second |
| **Max queued instances** | 100,000 | 1,000,000 |
| **State retention** | 3 days | 30 days |
| **Max workflow name** | 64 characters | 64 characters |
| **Max instance ID** | 100 characters | 100 characters |
| **Subrequests per instance** | 50 | 1,000 |

**Notes:**
- `step.sleep()` doesn't count toward step limit
- Waiting instances don't count toward concurrency
- Only actively `running` instances count toward concurrency
- CPU time = active processing, not I/O wait time

## Pricing

Workflows use **Workers Standard pricing**:

| Metric | Free Plan | Paid Plan |
|--------|-----------|-----------|
| **Requests** | 100,000/day | 10M included/month + $0.30/million |
| **CPU time** | 10 ms/invocation | 30M CPU-ms included/month + $0.02/million CPU-ms |
| **Storage** | 1 GB | 1 GB included/month + $0.20/GB-month |

**Storage:**
- Billed as GB-month (averaged over 30 days)
- Includes running, errored, sleeping, completed instances
- Retention: 3 days (Free), 30 days (Paid)
- Delete instances to free up storage

## TypeScript Types

```typescript
// Import types
import { 
  WorkflowEntrypoint, 
  WorkflowStep, 
  WorkflowEvent,
  NonRetryableError 
} from 'cloudflare:workers';

// Define parameter types
interface MyParams {
  userId: string;
  email: string;
  metadata?: Record<string, string>;
}

// Define environment
interface Env {
  MY_WORKFLOW: Workflow;
  KV: KVNamespace;
  DB: D1Database;
  BUCKET: R2Bucket;
  AI: Ai;
}

// Workflow class
export class MyWorkflow extends WorkflowEntrypoint<Env, MyParams> {
  async run(event: WorkflowEvent<MyParams>, step: WorkflowStep) {
    // Typed access to event.payload
    const userId: string = event.payload.userId;
    
    // Typed return from steps
    const user: User = await step.do('fetch user', async () => {
      return await this.env.KV.get<User>(`user:${userId}`, { type: 'json' });
    });
  }
}

// Binding type
interface Env {
  MY_WORKFLOW: Workflow<MyParams>;
}

// Create instance with typed params
const instance = await env.MY_WORKFLOW.create({
  id: 'user123',
  params: { userId: 'user123', email: 'user@example.com' }
});
```

## Debugging and Observability

### Logs

```typescript
await step.do('process data', async () => {
  console.log('Processing started');  // Logged once per successful step
  // Processing logic...
  console.log('Processing completed');
});

// ⚠️ Logs outside steps may appear multiple times
console.log('Workflow started');  // May duplicate on restart
```

### Instance Status

```bash
# CLI
npx wrangler workflows instances describe my-workflow instance-id

# Shows:
# - Status (queued, running, paused, etc.)
# - Step details and timing
# - Errors and retries
# - State output
```

### Error Inspection

```typescript
const instance = await env.MY_WORKFLOW.get('instance-id');
const status = await instance.status();

if (status.status === 'errored') {
  console.error('Workflow failed:', status.error);
  // status.error: { name: string, message: string }
}
```

## Integration with Cloudflare Services

### Workers AI

```typescript
await step.do('generate text', async () => {
  return await this.env.AI.run('@cf/meta/llama-2-7b-chat-int8', {
    prompt: 'Write a story about Cloudflare'
  });
});
```

### D1 Database

```typescript
await step.do('query database', async () => {
  const result = await this.env.DB.prepare(
    'SELECT * FROM users WHERE email = ?'
  ).bind(event.payload.email).first();
  return result;
});
```

### R2 Storage

```typescript
await step.do('store file', async () => {
  await this.env.BUCKET.put(
    `files/${event.payload.filename}`,
    event.payload.fileData
  );
});
```

### KV

```typescript
await step.do('get from KV', async () => {
  return await this.env.KV.get('config', { type: 'json' });
});
```

### Vectorize

```typescript
await step.do('insert vectors', async () => {
  const embeddings = await this.env.AI.run('@cf/baai/bge-base-en-v1.5', {
    text: [event.payload.text]
  });
  
  await this.env.VECTORIZE.insert([{
    id: event.payload.id,
    values: embeddings.data[0]
  }]);
});
```

### Queues

Trigger Workflows from Queue consumers:

```typescript
export default {
  async queue(batch: MessageBatch, env: Env) {
    for (const message of batch.messages) {
      await env.PROCESSING_WORKFLOW.create({
        id: `msg-${message.id}`,
        params: message.body
      });
    }
  }
};
```

## Advanced Patterns

### Promise.race with Steps

```typescript
// ⚠️ Wrap Promise.race in step.do for deterministic caching
const result = await step.do('race steps', async () => {
  return await Promise.race([
    step.do('option A', async () => {
      await sleep(1000);
      return 'A';
    }),
    step.do('option B', async () => {
      return 'B';
    })
  ]);
});
```

### Cross-Script Workflow Bindings

**billing-worker (defines workflow):**
```toml
name = "billing-worker"
main = "src/billing.ts"

[[workflows]]
name = "billing-workflow"
binding = "BILLING"
class_name = "BillingWorkflow"
```

**web-api-worker (calls workflow):**
```toml
name = "web-api-worker"
main = "src/api.ts"

[[workflows]]
name = "billing-workflow"
binding = "BILLING"
class_name = "BillingWorkflow"
script_name = "billing-worker"  # Reference other script
```

### Multiple Workflows in One Script

```typescript
export class UserOnboarding extends WorkflowEntrypoint<Env, UserParams> {
  async run(event: WorkflowEvent<UserParams>, step: WorkflowStep) {
    // Onboarding logic...
  }
}

export class DataProcessing extends WorkflowEntrypoint<Env, DataParams> {
  async run(event: WorkflowEvent<DataParams>, step: WorkflowStep) {
    // Processing logic...
  }
}
```

```toml
[[workflows]]
name = "user-onboarding"
binding = "USER_ONBOARDING"
class_name = "UserOnboarding"

[[workflows]]
name = "data-processing"
binding = "DATA_PROCESSING"
class_name = "DataProcessing"
```

## References

- [Official Documentation](https://developers.cloudflare.com/workflows/)
- [Get Started Guide](https://developers.cloudflare.com/workflows/get-started/guide/)
- [Workers API Reference](https://developers.cloudflare.com/workflows/build/workers-api/)
- [REST API Reference](https://developers.cloudflare.com/api/resources/workflows/)
- [Examples](https://developers.cloudflare.com/workflows/examples/)
- [Limits](https://developers.cloudflare.com/workflows/reference/limits/)
- [Pricing](https://developers.cloudflare.com/workflows/reference/pricing/)
