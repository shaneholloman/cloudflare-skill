# Cloudflare Queues Skill

## Overview

Cloudflare Queues is a flexible, distributed message queuing system that integrates with Cloudflare Workers. It enables reliable asynchronous processing, decoupling of application components, and message batching with guaranteed delivery.

**Key capabilities:**
- Message queue for async processing with guaranteed at-least-once delivery
- Push-based (Worker) and pull-based (HTTP) consumers
- Configurable batching, retries, and delays (up to 12 hours)
- Dead Letter Queues (DLQ) for failed messages
- Support for multiple content types (json, text, bytes, v8)

## Core Concepts

### 1. Queues
A queue is an auto-scaling buffer that stores messages until consumed. Each queue can have:
- Multiple producer Workers writing to it
- Only ONE consumer (either Worker or HTTP pull)
- Max 10,000 queues per account

### 2. Producers
Producers publish messages to queues via Worker bindings:
- Any Worker can produce to multiple queues
- Multiple Workers can produce to the same queue
- Messages can be up to 128 KB each
- Max throughput: 5,000 messages/second per queue

### 3. Consumers
Two consumer types:
- **Push-based (Worker)**: Automatically invoked when messages available; auto-scales
- **Pull-based (HTTP)**: External services pull messages over HTTP; controlled rate

### 4. Messages
- JSON-serializable objects (default: `json` content type)
- Support for text, bytes, json, and v8 (structured clone) formats
- Retention: 4 days default (configurable to 14 days)
- Messages delivered in batches (configurable size/timeout)

## Wrangler Configuration

### Producer Configuration

**wrangler.jsonc:**
```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "queues": {
    "producers": [
      {
        "queue": "my-queue-name",
        "binding": "MY_QUEUE",
        "delivery_delay": 60  // Optional: delay all messages by N seconds (max 43200)
      }
    ]
  }
}
```

**wrangler.toml:**
```toml
[[queues.producers]]
  queue = "my-queue-name"
  binding = "MY_QUEUE"
  delivery_delay = 60  # Optional: default delay in seconds
```

### Consumer Configuration (Push-based)

**wrangler.jsonc:**
```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "queues": {
    "consumers": [
      {
        "queue": "my-queue-name",
        "max_batch_size": 10,           // 1-100, default 10
        "max_batch_timeout": 5,         // 0-60 seconds, default 5
        "max_retries": 3,               // default 3, max 100
        "dead_letter_queue": "my-dlq",  // optional DLQ
        "retry_delay": 300              // optional: delay retries by N seconds
      }
    ]
  }
}
```

**wrangler.toml:**
```toml
[[queues.consumers]]
  queue = "my-queue-name"
  max_batch_size = 10           # default: 10, max: 100
  max_batch_timeout = 5         # default: 5 seconds, max: 60
  max_retries = 3               # default: 3, max: 100
  dead_letter_queue = "my-dlq"  # optional
  retry_delay = 300             # optional: delay retries in seconds
```

### Consumer Configuration (Pull-based)

**wrangler.jsonc:**
```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "queues": {
    "consumers": [
      {
        "queue": "my-queue-name",
        "type": "http_pull",
        "visibility_timeout_ms": 5000,  // default 30000, max 12 hours
        "max_retries": 5,
        "dead_letter_queue": "my-dlq",
        "retry_delay": 60
      }
    ]
  }
}
```

**wrangler.toml:**
```toml
[[queues.consumers]]
  queue = "my-queue-name"
  type = "http_pull"
  visibility_timeout_ms = 5000  # default: 30000ms
  max_retries = 5
  dead_letter_queue = "my-dlq"
  retry_delay = 60
```

## Wrangler CLI Commands

### Queue Management
```bash
# Create a queue
npx wrangler queues create <QUEUE_NAME>

# Create queue with custom retention
npx wrangler queues create <QUEUE_NAME> --retention-period-hours=336  # 14 days

# Create queue with delivery delay
npx wrangler queues create <QUEUE_NAME> --delivery-delay-secs=300

# List all queues
npx wrangler queues list

# Delete a queue
npx wrangler queues delete <QUEUE_NAME>
```

### Consumer Management
```bash
# Add push-based (Worker) consumer
npx wrangler queues consumer add <QUEUE_NAME> <WORKER_SCRIPT_NAME>

# Add with options
npx wrangler queues consumer add <QUEUE_NAME> <WORKER_SCRIPT_NAME> \
  --batch-size=50 \
  --batch-timeout=10 \
  --max-retries=5 \
  --dead-letter-queue=my-dlq \
  --retry-delay-secs=60

# Add pull-based (HTTP) consumer
npx wrangler queues consumer http add <QUEUE_NAME>
npx wrangler queues consumer http add <QUEUE_NAME> --retry-delay-secs=60

# Remove consumer
npx wrangler queues consumer worker remove <QUEUE_NAME> <WORKER_SCRIPT_NAME>
npx wrangler queues consumer http remove <QUEUE_NAME>
```

### Queue Operations
```bash
# Pause a queue (stops message delivery to consumers)
npx wrangler queues pause <QUEUE_NAME>

# Resume a paused queue
npx wrangler queues resume <QUEUE_NAME>

# Purge all messages from a queue
npx wrangler queues purge <QUEUE_NAME>
```

## Code Patterns

### Producer: Sending Messages

**Basic send:**
```typescript
export interface Env {
  MY_QUEUE: Queue<any>;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Send a single message
    await env.MY_QUEUE.send({
      url: request.url,
      method: request.method,
      timestamp: Date.now()
    });
    
    return new Response('Message sent', { status: 202 });
  }
} satisfies ExportedHandler<Env>;
```

**Send with delay:**
```typescript
// Delay message by 10 minutes
await env.MY_QUEUE.send(message, { 
  delaySeconds: 600 
});

// No delay (override queue-level delay)
await env.MY_QUEUE.send(message, { 
  delaySeconds: 0 
});
```

**Send batch:**
```typescript
const messages = [
  { body: 'message 1' },
  { body: 'message 2' },
  { body: 'message 3', options: { delaySeconds: 300 } }
];

// Send up to 100 messages (or 256 KB total)
await env.MY_QUEUE.sendBatch(messages);

// Send batch with global delay
await env.MY_QUEUE.sendBatch(messages, { delaySeconds: 120 });
```

**Content types:**
```typescript
// JSON (default) - auto-serialized
await env.MY_QUEUE.send({ key: 'value' }, { contentType: 'json' });

// Text
await env.MY_QUEUE.send('plain string', { contentType: 'text' });

// Bytes
const buffer = new Uint8Array([1, 2, 3]);
await env.MY_QUEUE.send(buffer, { contentType: 'bytes' });

// V8 (structured clone) - supports Date, Map, etc.
await env.MY_QUEUE.send(new Date(), { contentType: 'v8' });
```

**Non-blocking send with waitUntil:**
```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Send without blocking response
    ctx.waitUntil(env.MY_QUEUE.send({ data: 'async' }));
    
    return new Response('Accepted', { status: 202 });
  }
};
```

### Consumer: Processing Messages (Push-based)

**Basic consumer:**
```typescript
export interface Env {
  MY_QUEUE: Queue<any>;
}

export default {
  async queue(batch: MessageBatch, env: Env, ctx: ExecutionContext): Promise<void> {
    // Process each message
    for (const message of batch.messages) {
      console.log('Message ID:', message.id);
      console.log('Body:', message.body);
      console.log('Timestamp:', message.timestamp);
      console.log('Attempts:', message.attempts);
    }
    
    // All messages auto-acknowledged if no error thrown
  }
} satisfies ExportedHandler<Env>;
```

**Explicit acknowledgement:**
```typescript
async queue(batch: MessageBatch, env: Env, ctx: ExecutionContext): Promise<void> {
  for (const message of batch.messages) {
    try {
      await processMessage(message.body);
      message.ack();  // Explicitly acknowledge success
    } catch (error) {
      console.error(`Failed to process message ${message.id}:`, error);
      // Don't ack - will be retried
    }
  }
}
```

**Retry with delay:**
```typescript
async queue(batch: MessageBatch, env: Env, ctx: ExecutionContext): Promise<void> {
  for (const message of batch.messages) {
    try {
      await callUpstreamAPI(message.body);
      message.ack();
    } catch (error) {
      if (error.status === 429) {  // Rate limited
        // Retry after 10 minutes
        message.retry({ delaySeconds: 600 });
      } else {
        // Immediate retry (next batch)
        message.retry();
      }
    }
  }
}
```

**Exponential backoff:**
```typescript
function calculateExponentialBackoff(attempts: number, baseSeconds: number): number {
  return Math.min(baseSeconds ** attempts, 43200);  // max 12 hours
}

async queue(batch: MessageBatch, env: Env, ctx: ExecutionContext): Promise<void> {
  const BASE_DELAY = 30;
  
  for (const message of batch.messages) {
    try {
      await processMessage(message.body);
      message.ack();
    } catch (error) {
      const delay = calculateExponentialBackoff(message.attempts, BASE_DELAY);
      message.retry({ delaySeconds: delay });
    }
  }
}
```

**Batch operations:**
```typescript
async queue(batch: MessageBatch, env: Env, ctx: ExecutionContext): Promise<void> {
  console.log(`Queue: ${batch.queue}`);
  console.log(`Batch size: ${batch.messages.length}`);
  
  try {
    // Process all messages together
    await bulkProcess(batch.messages);
    
    // Acknowledge entire batch
    batch.ackAll();
  } catch (error) {
    // Retry entire batch with delay
    batch.retryAll({ delaySeconds: 300 });
  }
}
```

**Multiple queues, single consumer:**
```typescript
export default {
  async queue(batch: MessageBatch, env: Env, ctx: ExecutionContext): Promise<void> {
    switch (batch.queue) {
      case 'high-priority-queue':
        await processUrgent(batch.messages);
        break;
      case 'low-priority-queue':
        await processDeferred(batch.messages);
        break;
      case 'email-queue':
        await sendEmails(batch.messages);
        break;
      default:
        console.warn(`Unknown queue: ${batch.queue}`);
        batch.retryAll();
    }
  }
};
```

**Async processing with waitUntil:**
```typescript
async queue(batch: MessageBatch, env: Env, ctx: ExecutionContext): Promise<void> {
  for (const message of batch.messages) {
    // Use waitUntil for async operations like logging
    ctx.waitUntil(
      logToAnalytics(message.id, message.body)
    );
    
    await processMessage(message.body);
    message.ack();
  }
}
```

### Consumer: Pull-based (HTTP)

**Pull messages:**
```typescript
const CF_ACCOUNT_ID = 'your-account-id';
const QUEUE_ID = 'your-queue-id';
const QUEUES_API_TOKEN = 'your-api-token';

// Pull up to 50 messages with 6-second visibility timeout
const response = await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/queues/${QUEUE_ID}/messages/pull`,
  {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      'authorization': `Bearer ${QUEUES_API_TOKEN}`
    },
    body: JSON.stringify({
      visibility_timeout_ms: 6000,
      batch_size: 50
    })
  }
);

const data = await response.json();
// data.result.messages contains array of messages
```

**Acknowledge messages:**
```typescript
const leaseIdsToAck = messages
  .filter(msg => processedSuccessfully.includes(msg.id))
  .map(msg => ({ lease_id: msg.lease_id }));

const leaseIdsToRetry = messages
  .filter(msg => needsRetry.includes(msg.id))
  .map(msg => ({ lease_id: msg.lease_id, delay_seconds: 600 }));

await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/queues/${QUEUE_ID}/messages/ack`,
  {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      'authorization': `Bearer ${QUEUES_API_TOKEN}`
    },
    body: JSON.stringify({
      acks: leaseIdsToAck,
      retries: leaseIdsToRetry
    })
  }
);
```

## Common Use Cases

### 1. Async Task Processing
```typescript
// Producer: Accept request, queue work
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { userId, reportType } = await request.json();
    
    await env.REPORT_QUEUE.send({
      userId,
      reportType,
      requestedAt: Date.now()
    });
    
    return Response.json({ 
      message: 'Report queued for generation',
      status: 'pending'
    });
  }
};

// Consumer: Process report generation
export default {
  async queue(batch: MessageBatch, env: Env): Promise<void> {
    for (const message of batch.messages) {
      const { userId, reportType } = message.body;
      
      const report = await generateReport(userId, reportType, env);
      await env.REPORTS_BUCKET.put(`${userId}/${reportType}.pdf`, report);
      
      message.ack();
    }
  }
};
```

### 2. Buffering API Calls
```typescript
// Producer: Queue log entries
async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
  ctx.waitUntil(
    env.LOGS_QUEUE.send({
      method: request.method,
      url: request.url,
      cf: request.cf,
      timestamp: Date.now()
    })
  );
  
  return handleRequest(request);
}

// Consumer: Batch write to external API
async queue(batch: MessageBatch, env: Env): Promise<void> {
  const logs = batch.messages.map(m => m.body);
  
  // Send 100 logs at once instead of 100 individual requests
  await fetch(env.LOG_ANALYTICS_ENDPOINT, {
    method: 'POST',
    body: JSON.stringify({ logs })
  });
  
  batch.ackAll();
}
```

### 3. Rate Limiting Upstream Calls
```typescript
async queue(batch: MessageBatch, env: Env, ctx: ExecutionContext): Promise<void> {
  for (const message of batch.messages) {
    try {
      await callRateLimitedAPI(message.body);
      message.ack();
    } catch (error) {
      if (error.status === 429) {
        // Retry-After header suggests 60 seconds
        const retryAfter = parseInt(error.headers.get('Retry-After') || '60');
        message.retry({ delaySeconds: retryAfter });
      } else {
        throw error;  // Will cause batch retry
      }
    }
  }
}
```

### 4. Event-Driven Workflows
```typescript
// R2 event notification → Queue → Worker
// Configured via R2 bucket event notifications

export default {
  async queue(batch: MessageBatch, env: Env): Promise<void> {
    for (const message of batch.messages) {
      const event = message.body;
      
      if (event.action === 'PutObject') {
        // New file uploaded
        await processNewFile(event.object.key, env);
      } else if (event.action === 'DeleteObject') {
        // File deleted
        await cleanupReferences(event.object.key, env);
      }
      
      message.ack();
    }
  }
};
```

### 5. Dead Letter Queue Pattern
```typescript
// Main queue consumer
export default {
  async queue(batch: MessageBatch, env: Env): Promise<void> {
    for (const message of batch.messages) {
      try {
        await riskyOperation(message.body);
        message.ack();
      } catch (error) {
        console.error(`Failed after ${message.attempts} attempts:`, error);
        // After max_retries, automatically goes to DLQ
      }
    }
  }
};

// DLQ consumer for manual inspection/reprocessing
export default {
  async queue(batch: MessageBatch, env: Env): Promise<void> {
    for (const message of batch.messages) {
      // Log to external monitoring
      await logFailedMessage(message);
      
      // Store for manual review
      await env.FAILED_MESSAGES_KV.put(
        message.id,
        JSON.stringify(message.body)
      );
      
      message.ack();
    }
  }
};
```

## Architecture Patterns

### Pattern 1: Producer-Consumer Decoupling
```
[User Request] → [Producer Worker] → [Queue] → [Consumer Worker] → [External API/Storage]
```

**Benefits:**
- Request handler returns immediately
- Consumer scales independently
- Retry logic isolated from user-facing code

### Pattern 2: Fan-out with Multiple Queues
```
[Producer] → [Queue 1] → [Consumer 1] → [Processing A]
          → [Queue 2] → [Consumer 2] → [Processing B]
          → [Queue 3] → [Consumer 3] → [Processing C]
```

**Example:**
```typescript
async fetch(request: Request, env: Env): Promise<Response> {
  const event = await request.json();
  
  // Send to multiple queues for different processing
  await Promise.all([
    env.ANALYTICS_QUEUE.send(event),
    env.NOTIFICATIONS_QUEUE.send(event),
    env.AUDIT_LOG_QUEUE.send(event)
  ]);
  
  return Response.json({ status: 'processed' });
}
```

### Pattern 3: Priority Queues
```
[Producer] → [High Priority Queue] → [Consumer] (small batch timeout)
          → [Low Priority Queue] → [Consumer] (large batch timeout)
```

**Configuration:**
```jsonc
// High priority: process faster
{
  "queue": "high-priority",
  "max_batch_size": 5,
  "max_batch_timeout": 1
}

// Low priority: batch more
{
  "queue": "low-priority",
  "max_batch_size": 100,
  "max_batch_timeout": 30
}
```

### Pattern 4: Delayed Job Processing
```
[Scheduler] → [Queue with delay] → [Consumer at future time]
```

**Example:**
```typescript
// Schedule email for 1 hour from now
await env.EMAIL_QUEUE.send({
  to: user.email,
  template: 'welcome',
  userId: user.id
}, { delaySeconds: 3600 });
```

## TypeScript Types

```typescript
// Environment with queue bindings
export interface Env {
  MY_QUEUE: Queue<MyMessageType>;
  ANALYTICS_QUEUE: Queue<AnalyticsEvent>;
  // Other bindings...
}

// Message types
interface MyMessageType {
  id: string;
  action: 'create' | 'update' | 'delete';
  data: Record<string, any>;
  timestamp: number;
}

// Queue producer methods
interface Queue<Body = unknown> {
  send(body: Body, options?: QueueSendOptions): Promise<void>;
  sendBatch(
    messages: Iterable<MessageSendRequest<Body>>,
    options?: QueueSendBatchOptions
  ): Promise<void>;
}

// Consumer message batch
interface MessageBatch<Body = unknown> {
  readonly queue: string;
  readonly messages: Message<Body>[];
  ackAll(): void;
  retryAll(options?: QueueRetryOptions): void;
}

// Individual message
interface Message<Body = unknown> {
  readonly id: string;
  readonly timestamp: Date;
  readonly body: Body;
  readonly attempts: number;
  ack(): void;
  retry(options?: QueueRetryOptions): void;
}

// Options
interface QueueSendOptions {
  contentType?: 'text' | 'bytes' | 'json' | 'v8';
  delaySeconds?: number;  // 0-43200 (12 hours)
}

interface QueueRetryOptions {
  delaySeconds?: number;
}

interface MessageSendRequest<Body = unknown> {
  body: Body;
  options?: QueueSendOptions;
}
```

## Best Practices

### 1. Message Size and Batching
- Keep messages < 64 KB to minimize costs (charged per 64 KB chunk)
- Use batch sends for bulk operations (up to 100 messages or 256 KB)
- Configure `max_batch_size` and `max_batch_timeout` based on processing needs

### 2. Idempotency
- Design consumers to handle duplicate messages (at-least-once delivery)
- Use message IDs for deduplication if needed
- Store processed message IDs in KV/D1 for tracking

```typescript
async queue(batch: MessageBatch, env: Env): Promise<void> {
  for (const message of batch.messages) {
    // Check if already processed
    const processed = await env.PROCESSED_KV.get(message.id);
    if (processed) {
      message.ack();
      continue;
    }
    
    await processMessage(message.body);
    
    // Mark as processed
    await env.PROCESSED_KV.put(message.id, '1', { expirationTtl: 86400 });
    message.ack();
  }
}
```

### 3. Error Handling
- Use explicit ack/retry for fine-grained control
- Configure DLQ to capture permanently failed messages
- Log failures with context for debugging

```typescript
async queue(batch: MessageBatch, env: Env): Promise<void> {
  for (const message of batch.messages) {
    try {
      await processMessage(message.body);
      message.ack();
    } catch (error) {
      console.error('Processing failed:', {
        messageId: message.id,
        attempts: message.attempts,
        error: error.message
      });
      
      if (message.attempts >= 3) {
        // Last attempt before DLQ - log critical error
        await logCriticalFailure(message, error, env);
      }
      
      // Don't call retry() - let it retry automatically
    }
  }
}
```

### 4. Content Types
- Use `json` (default) for pull consumers and dashboard visibility
- Use `text` for simple strings
- Use `bytes` for binary data
- Avoid `v8` with pull consumers (not decodable)

### 5. Performance Optimization
- Use `waitUntil()` for non-blocking sends in request handlers
- Batch database writes in consumers instead of individual writes
- Configure appropriate CPU limits for heavy processing (up to 5 minutes)

```jsonc
{
  "limits": {
    "cpu_ms": 300000  // 5 minutes
  }
}
```

### 6. Cost Optimization
- Operations: write + read + delete = 3 ops per message
- Retries add read operations
- DLQ writes add operations
- Formula: `((messages × 3) - 1M) / 1M × $0.40` per month
- Batch aggressively to reduce processing overhead

### 7. Monitoring
- Track queue backlog metrics
- Monitor consumer lag and processing time
- Set alerts for DLQ message accumulation
- Use `message.attempts` for retry pattern analysis

## Limits

| Limit | Value |
|-------|-------|
| Max queues per account | 10,000 |
| Max message size | 128 KB |
| Max batch size (consumer) | 100 messages |
| Max batch size (sendBatch) | 100 messages or 256 KB |
| Max batch timeout | 60 seconds |
| Max throughput per queue | 5,000 msgs/sec |
| Message retention | 4-14 days (configurable) |
| Max backlog per queue | 25 GB |
| Max consumer concurrency | 250 (push-based) |
| Consumer CPU time | 30s default, 5m max |
| Max delay | 12 hours (43,200 seconds) |
| Max retries | 100 (default 3) |

## Pricing

- **Included:** 1M operations/month
- **Additional:** $0.40 per million operations
- **Operation:** Each 64 KB chunk read/written/deleted
- **No egress charges**

**Cost examples:**
- 1M messages/day (30 days): ~$35.60/month
- 100M messages/month (127 KB each): ~$239.60/month

## Local Development

```bash
# Start local dev server with queues
npx wrangler dev

# Queues work automatically in local mode
# Producer sends work in local dev
# Consumer processes in local dev
```

**Note:** Local queues are ephemeral - cleared on restart.

## Troubleshooting

### Messages not being delivered
- Check consumer is properly configured in wrangler.jsonc/toml
- Verify queue name matches between producer and consumer
- Check if queue is paused: `npx wrangler queues list`
- Review consumer errors in logs

### Consumer throwing errors
- Check CPU time limits (may need to increase)
- Verify external API timeouts are reasonable
- Use explicit ack/retry instead of batch operations for partial failures

### High costs
- Reduce retry counts if appropriate
- Optimize message sizes (keep < 64 KB)
- Increase batch sizes to reduce processing frequency
- Check for retry loops

### Messages going to DLQ
- Review consumer error logs for root cause
- Check if external dependencies are available
- Verify message format matches consumer expectations
- Consider increasing retry delay

## References

- **Official Docs:** https://developers.cloudflare.com/queues/
- **JavaScript APIs:** https://developers.cloudflare.com/queues/configuration/javascript-apis/
- **Wrangler Config:** https://developers.cloudflare.com/workers/wrangler/configuration/#queues
- **REST API:** https://developers.cloudflare.com/api/resources/queues/
- **Pricing:** https://developers.cloudflare.com/queues/platform/pricing/
- **Limits:** https://developers.cloudflare.com/queues/platform/limits/

## Quick Reference

```bash
# Create queue
npx wrangler queues create my-queue

# Add consumer
npx wrangler queues consumer add my-queue my-worker

# Deploy
npx wrangler deploy

# Pause queue
npx wrangler queues pause my-queue

# Purge queue
npx wrangler queues purge my-queue
```

```typescript
// Send message
await env.MY_QUEUE.send({ data: 'value' });

// Send with delay
await env.MY_QUEUE.send(msg, { delaySeconds: 300 });

// Process messages
async queue(batch: MessageBatch, env: Env): Promise<void> {
  for (const msg of batch.messages) {
    await process(msg.body);
    msg.ack();
  }
}
```
