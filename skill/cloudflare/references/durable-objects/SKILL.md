# Cloudflare Durable Objects Skill

Expert guidance for building stateful applications with Cloudflare Durable Objects.

## What are Durable Objects?

Durable Objects combine compute with storage in a globally-unique, strongly-consistent package. Key characteristics:

- **Globally unique instances**: Each Durable Object has a unique ID, allowing coordination between multiple clients
- **Co-located storage**: Fast, strongly-consistent storage lives with the compute
- **Automatic placement**: Objects spawn near where first requested
- **Stateful serverless**: Maintain in-memory state and persistent storage
- **Single-threaded**: Each instance processes requests serially (no race conditions)

## Core Architecture Patterns

### 1. Durable Object Class Structure

All Durable Objects extend the base `DurableObject` class:

```typescript
import { DurableObject } from "cloudflare:workers";

export class MyDurableObject extends DurableObject<Env> {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    // ctx: Access to storage, WebSockets, alarms
    // env: Bindings (KV, R2, other DOs, secrets)
  }

  // Public methods become RPC endpoints
  async myMethod(arg: string): Promise<string> {
    return `Received: ${arg}`;
  }

  // Special handlers (optional)
  async alarm() { /* triggered by scheduled alarms */ }
  async webSocketMessage(ws: WebSocket, message: string | ArrayBuffer) { }
  async webSocketClose(ws: WebSocket, code: number, reason: string, wasClean: boolean) { }
  async webSocketError(ws: WebSocket, error: unknown) { }
}
```

### 2. Accessing Durable Objects from Workers

Workers instantiate Durable Objects via bindings and stubs:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Get Durable Object stub by ID
    const id = env.MY_DO.idFromName("unique-name");
    const stub = env.MY_DO.get(id);

    // Call RPC methods directly (recommended)
    const result = await stub.myMethod("hello");

    // Or use fetch handler (legacy/HTTP patterns)
    const response = await stub.fetch(request);

    return new Response(result);
  }
} satisfies ExportedHandler<Env>;
```

### 3. ID Generation Strategies

```typescript
// Named objects (deterministic, human-readable coordination)
const id = env.MY_DO.idFromName("chat-room-123");

// Random ID (unique per object, use for sharding)
const id = env.MY_DO.newUniqueId();

// ID from string (derive from user ID, etc.)
const id = env.MY_DO.idFromString(userProvidedId);

// Jurisdiction-specific (data locality)
const id = env.MY_DO.idFromName("user-data", { jurisdiction: "eu" });
```

## Storage API

### SQLite Storage (Recommended)

Modern Durable Objects use SQLite for structured storage:

```typescript
export class Counter extends DurableObject<Env> {
  async increment(): Promise<number> {
    // Execute SQL directly
    const result = this.ctx.storage.sql.exec(
      `INSERT INTO counters (id, value) VALUES (1, 1)
       ON CONFLICT(id) DO UPDATE SET value = value + 1
       RETURNING value`
    ).one();

    return result.value;
  }

  async getCount(): Promise<number> {
    const result = this.ctx.storage.sql.exec(
      "SELECT value FROM counters WHERE id = 1"
    ).one();

    return result?.value ?? 0;
  }
}
```

#### SQL API Methods

```typescript
// Execute query, returns cursor
const cursor = this.ctx.storage.sql.exec(
  "SELECT * FROM users WHERE age > ?",
  18
);

// Iterate results
for (const row of cursor) {
  console.log(row.name, row.age);
}

// Get all results as array
const users = cursor.toArray();

// Get exactly one row (throws if 0 or >1)
const user = this.ctx.storage.sql.exec("SELECT * FROM users WHERE id = ?", 1).one();

// Get raw arrays instead of objects
const rawCursor = cursor.raw();
const rawRows = rawCursor.toArray(); // [[1, 'Alice'], [2, 'Bob']]

// Check query stats
console.log(cursor.rowsRead, cursor.rowsWritten);
console.log(cursor.columnNames); // ['id', 'name', 'age']

// Get database size
const sizeBytes = this.ctx.storage.sql.databaseSize;
```

#### SQL Schema Best Practices

```typescript
constructor(ctx: DurableObjectState, env: Env) {
  super(ctx, env);

  // Initialize schema idempotently
  this.ctx.storage.sql.exec(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY,
      name TEXT NOT NULL,
      created_at INTEGER DEFAULT (unixepoch())
    );
    CREATE INDEX IF NOT EXISTS idx_name ON users(name);
  `);
}
```

### Synchronous KV API (SQLite-backed objects)

For simple key-value storage on SQLite objects:

```typescript
// Direct synchronous access (no await needed)
const value = this.ctx.storage.kv.get("myKey");
this.ctx.storage.kv.put("myKey", { foo: "bar" });
this.ctx.storage.kv.delete("myKey");

// List keys
for (const [key, value] of this.ctx.storage.kv.list({ prefix: "user:" })) {
  console.log(key, value);
}
```

### Asynchronous KV API

Legacy or for advanced use cases:

```typescript
// Single operations
await this.ctx.storage.put("key", value);
const value = await this.ctx.storage.get("key");
await this.ctx.storage.delete("key");

// Batch operations
await this.ctx.storage.put({ key1: "a", key2: "b" }); // up to 128 keys
const values = await this.ctx.storage.get(["key1", "key2"]); // returns Map
await this.ctx.storage.delete(["key1", "key2"]);

// List with options
const entries = await this.ctx.storage.list({
  prefix: "user:",
  start: "user:100",
  end: "user:200",
  limit: 50,
  reverse: true
});

// Delete all data (SQL + KV)
await this.ctx.storage.deleteAll();
```

### Transactions

```typescript
// Synchronous transactions (SQLite only)
const result = this.ctx.storage.transactionSync(() => {
  // All SQL operations in this function are atomic
  this.ctx.storage.sql.exec("UPDATE accounts SET balance = balance - 100 WHERE id = 1");
  this.ctx.storage.sql.exec("UPDATE accounts SET balance = balance + 100 WHERE id = 2");
  return true;
});

// Async transactions (any storage)
await this.ctx.storage.transaction(async (txn) => {
  const balance = await txn.get("balance");
  await txn.put("balance", balance + 100);
  // Throw to rollback, or call txn.rollback()
});
```

### Point-in-Time Recovery (SQLite)

Restore Durable Object storage to any point in last 30 days:

```typescript
// Get current bookmark
const now = await this.ctx.storage.getCurrentBookmark();

// Get bookmark for specific time
const yesterday = await this.ctx.storage.getBookmarkForTime(Date.now() - 86400000);

// Schedule restore on next session
const undoBookmark = await this.ctx.storage.onNextSessionRestoreBookmark(yesterday);
// Must call ctx.abort() to restart and complete recovery
```

## Alarms

Schedule future execution for individual Durable Objects:

```typescript
export class ScheduledTask extends DurableObject<Env> {
  async scheduleTask(delayMs: number) {
    // Set alarm for future execution
    await this.ctx.storage.setAlarm(Date.now() + delayMs);
  }

  // Called when alarm fires
  async alarm() {
    console.log("Alarm fired!");

    // Do work (send notification, process batch, etc.)
    await this.processScheduledWork();

    // Optionally reschedule
    await this.ctx.storage.setAlarm(Date.now() + 3600000); // 1 hour
  }

  async checkAlarm() {
    const scheduledTime = await this.ctx.storage.getAlarm();
    return scheduledTime ? new Date(scheduledTime) : null;
  }

  async cancelAlarm() {
    await this.ctx.storage.deleteAlarm();
  }
}
```

### Multiple Events with Single Alarm

Each DO can only have one alarm, but you can manage many events:

```typescript
async scheduleEvent(id: string, runAt: number, repeatMs?: number) {
  await this.ctx.storage.put(`event:${id}`, { id, runAt, repeatMs });

  const currentAlarm = await this.ctx.storage.getAlarm();
  if (!currentAlarm || runAt < currentAlarm) {
    await this.ctx.storage.setAlarm(runAt);
  }
}

async alarm() {
  const now = Date.now();
  const events = await this.ctx.storage.list({ prefix: "event:" });
  let nextAlarm: number | null = null;

  for (const [key, event] of events) {
    if (event.runAt <= now) {
      await this.processEvent(event);

      if (event.repeatMs) {
        event.runAt = now + event.repeatMs;
        await this.ctx.storage.put(key, event);
      } else {
        await this.ctx.storage.delete(key);
      }
    }

    if (event.runAt > now && (!nextAlarm || event.runAt < nextAlarm)) {
      nextAlarm = event.runAt;
    }
  }

  if (nextAlarm) await this.ctx.storage.setAlarm(nextAlarm);
}
```

## WebSockets

Durable Objects excel at managing WebSocket connections for real-time coordination.

### WebSocket Hibernation (Recommended)

Hibernation allows DOs to sleep while maintaining WebSocket connections (no duration charges while idle):

```typescript
export class ChatRoom extends DurableObject<Env> {
  async fetch(request: Request): Promise<Response> {
    const upgradeHeader = request.headers.get("Upgrade");
    if (upgradeHeader !== "websocket") {
      return new Response("Expected WebSocket", { status: 426 });
    }

    const [client, server] = Object.values(new WebSocketPair());

    // Accept WebSocket with hibernation support
    this.ctx.acceptWebSocket(server);

    // Optionally attach metadata that survives hibernation
    server.serializeAttachment({ userId: "123", joinedAt: Date.now() });

    return new Response(null, { status: 101, webSocket: client });
  }

  // Called when message received (wakes from hibernation)
  async webSocketMessage(ws: WebSocket, message: string | ArrayBuffer) {
    const data = ws.deserializeAttachment();
    console.log("Message from", data.userId);

    // Broadcast to all connected clients
    for (const client of this.ctx.getWebSockets()) {
      client.send(message);
    }
  }

  // Called when client closes connection
  async webSocketClose(ws: WebSocket, code: number, reason: string, wasClean: boolean) {
    ws.close(code, "Server closing");
  }

  // Called on WebSocket error
  async webSocketError(ws: WebSocket, error: unknown) {
    console.error("WebSocket error:", error);
  }
}
```

### WebSocket Management

```typescript
// Get all connected WebSockets
const sockets = this.ctx.getWebSockets();

// Filter with tags
this.ctx.acceptWebSocket(server, ["room:lobby", "user:123"]);
const lobbySockets = this.ctx.getWebSockets("room:lobby");

// Get tags for a socket
const tags = this.ctx.getTags(ws);

// Send to specific socket
ws.send("Hello!");
ws.send(new Uint8Array([1, 2, 3])); // Binary

// Close socket
ws.close(1000, "Normal closure");

// Persist/restore state across hibernation
ws.serializeAttachment({ userId: "abc", permissions: ["read"] });
const data = ws.deserializeAttachment(); // Returns persisted object
```

### Standard WebSocket API (No Hibernation)

If you don't need hibernation, use standard event listeners:

```typescript
async fetch(request: Request): Promise<Response> {
  const [client, server] = Object.values(new WebSocketPair());

  server.accept();
  this.currentConnections++;

  server.addEventListener("message", (event) => {
    console.log("Received:", event.data);
    server.send(`Echo: ${event.data}`);
  });

  server.addEventListener("close", (event) => {
    this.currentConnections--;
    server.close(event.code, "Closing");
  });

  return new Response(null, { status: 101, webSocket: client });
}
```

## Wrangler Configuration

### Basic Setup

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-04-03",

  // Durable Object bindings
  "durable_objects": {
    "bindings": [
      {
        "name": "MY_DO",
        "class_name": "MyDurableObject"
      },
      {
        "name": "EXTERNAL_DO",
        "class_name": "ExternalDO",
        "script_name": "other-worker" // From another Worker
      }
    ]
  },

  // Migrations (required for DO lifecycle changes)
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["MyDurableObject"] // Create new SQLite-backed DO
    }
  ]
}
```

### Migration Types

```jsonc
{
  "migrations": [
    // Create new DO class
    {
      "tag": "v1",
      "new_sqlite_classes": ["MyDO"] // Recommended
      // or "new_classes": ["MyDO"] // Legacy KV-backed (Paid plan only)
    },

    // Rename class (same Worker)
    {
      "tag": "v2",
      "renamed_classes": [
        { "from": "OldName", "to": "NewName" }
      ]
    },

    // Transfer class (between Workers)
    {
      "tag": "v3",
      "transferred_classes": [
        {
          "from": "SourceClass",
          "from_script": "old-worker",
          "to": "DestClass"
        }
      ]
    },

    // Delete class (destroys all instances and data!)
    {
      "tag": "v4",
      "deleted_classes": ["ObsoleteClass"]
    }
  ]
}
```

### Advanced Configuration

```jsonc
{
  // Increase CPU limit (default 30s, max 300s)
  "limits": {
    "cpu_ms": 300000 // 5 minutes
  },

  // Multiple environments
  "env": {
    "production": {
      "durable_objects": {
        "bindings": [
          {
            "name": "MY_DO",
            "class_name": "MyDO",
            "environment": "production" // Separate DO instances per env
          }
        ]
      }
    },
    "staging": {
      "durable_objects": {
        "bindings": [
          {
            "name": "MY_DO",
            "class_name": "MyDO",
            "environment": "staging"
          }
        ]
      }
    }
  }
}
```

## Common Patterns

### 1. Rate Limiting

```typescript
export class RateLimiter extends DurableObject<Env> {
  async checkLimit(key: string, limit: number, windowMs: number): Promise<boolean> {
    const now = Date.now();
    const windowStart = now - windowMs;

    // Clean old entries and count recent requests
    const requests = await this.ctx.storage.sql.exec(
      "SELECT COUNT(*) as count FROM requests WHERE key = ? AND timestamp > ?",
      key, windowStart
    ).one();

    if (requests.count >= limit) {
      return false; // Rate limit exceeded
    }

    // Record this request
    this.ctx.storage.sql.exec(
      "INSERT INTO requests (key, timestamp) VALUES (?, ?)",
      key, now
    );

    return true;
  }
}
```

### 2. Distributed Lock

```typescript
export class Lock extends DurableObject<Env> {
  private held = false;

  async acquire(timeoutMs: number = 5000): Promise<boolean> {
    if (this.held) return false;

    this.held = true;

    // Auto-release after timeout
    await this.ctx.storage.setAlarm(Date.now() + timeoutMs);

    return true;
  }

  async release(): Promise<void> {
    this.held = false;
    await this.ctx.storage.deleteAlarm();
  }

  async alarm() {
    this.held = false; // Auto-release on timeout
  }
}
```

### 3. Real-time Collaboration

```typescript
export class CollabDoc extends DurableObject<Env> {
  async fetch(request: Request): Promise<Response> {
    const [client, server] = Object.values(new WebSocketPair());
    this.ctx.acceptWebSocket(server);

    // Send current document state to new client
    const doc = this.ctx.storage.kv.get("document") || "";
    server.send(JSON.stringify({ type: "init", content: doc }));

    return new Response(null, { status: 101, webSocket: client });
  }

  async webSocketMessage(ws: WebSocket, message: string) {
    const data = JSON.parse(message);

    if (data.type === "edit") {
      // Update document
      this.ctx.storage.kv.put("document", data.content);

      // Broadcast to all clients except sender
      for (const client of this.ctx.getWebSockets()) {
        if (client !== ws) {
          client.send(message);
        }
      }
    }
  }
}
```

### 4. Session Management

```typescript
export class UserSession extends DurableObject<Env> {
  async createSession(userId: string, data: object): Promise<string> {
    const sessionId = crypto.randomUUID();
    const expiresAt = Date.now() + 86400000; // 24 hours

    this.ctx.storage.sql.exec(
      "INSERT INTO sessions (id, user_id, data, expires_at) VALUES (?, ?, ?, ?)",
      sessionId, userId, JSON.stringify(data), expiresAt
    );

    // Set alarm to clean up expired session
    await this.ctx.storage.setAlarm(expiresAt);

    return sessionId;
  }

  async getSession(sessionId: string): Promise<object | null> {
    const row = this.ctx.storage.sql.exec(
      "SELECT data, expires_at FROM sessions WHERE id = ? AND expires_at > ?",
      sessionId, Date.now()
    ).one();

    return row ? JSON.parse(row.data) : null;
  }

  async alarm() {
    // Clean expired sessions
    this.ctx.storage.sql.exec(
      "DELETE FROM sessions WHERE expires_at <= ?",
      Date.now()
    );
  }
}
```

### 5. Request Deduplication

```typescript
export class Deduplicator extends DurableObject<Env> {
  private pending = new Map<string, Promise<Response>>();

  async deduplicatedFetch(url: string): Promise<Response> {
    // If request already in flight, return same promise
    if (this.pending.has(url)) {
      return this.pending.get(url)!;
    }

    // Start new request
    const promise = fetch(url).finally(() => {
      this.pending.delete(url);
    });

    this.pending.set(url, promise);
    return promise;
  }
}
```

## Wrangler CLI Commands

```bash
# Development
npx wrangler dev                    # Local development with DOs
npx wrangler dev --remote           # Test against production DOs

# Deployment
npx wrangler deploy                 # Deploy Worker + DOs
npx wrangler deploy --dry-run       # Validate without deploying

# Migrations
npx wrangler deploy                 # Automatically applies migrations in wrangler.toml

# Inspect objects (requires Durable Objects API)
npx wrangler durable-objects list   # List DO namespaces
npx wrangler durable-objects info <namespace> <id>  # Get info about specific object

# Delete objects
npx wrangler durable-objects delete <namespace> <id>
```

## Best Practices

### Design Patterns

1. **Keep objects focused**: Each DO class should handle one type of coordination
2. **Use named IDs for coordination**: `idFromName()` for chat rooms, game lobbies, etc.
3. **Use random IDs for sharding**: `newUniqueId()` for distributing load
4. **Minimize constructor work**: It runs on every wake-up (including after hibernation)
5. **Leverage hibernation**: Use WebSocket hibernation to reduce costs for idle connections

### Storage

1. **Prefer SQLite over KV**: Better performance, structured data, transactions
2. **Create indexes for queries**: But remember each index update counts as a write
3. **Batch operations**: Use transactions or multi-key operations
4. **Set alarms for cleanup**: Don't let storage grow unbounded
5. **Use PITR for recovery**: Store critical bookmarks before risky operations

### Performance

1. **One DO = 1000 req/s max**: Shard across multiple DOs for higher throughput
2. **Minimize storage operations**: Cache frequently-read data in memory
3. **Use `allowConcurrency: true`** for independent reads (breaks serialization guarantee)
4. **Avoid long-running operations**: CPU resets to 30s per incoming request
5. **Use alarms for deferred work**: Don't block request handlers

### Reliability

1. **Handle overload errors**: DOs return 503 when overwhelmed
2. **Implement retry logic**: With exponential backoff
3. **Design for cold starts**: DOs can be evicted and restarted
4. **Test migration paths**: Especially rename/transfer operations
5. **Monitor alarm retries**: They auto-retry 6 times with exponential backoff

### Security

1. **Validate inputs in Workers**: Before calling DO methods
2. **Don't trust DO names**: Users can craft arbitrary names
3. **Rate limit DO creation**: Prevent abuse via unlimited DO spawning
4. **Use jurisdiction tags**: For data locality requirements
5. **Encrypt sensitive data**: Before storing in DO storage

## Limits

### SQLite-backed Durable Objects

- **Storage per DO**: 10 GB
- **Storage per account**: Unlimited (Paid) / 5 GB (Free)
- **Key + value size**: 2 MB combined
- **CPU per request**: 30s default, configurable to 300s
- **Max DO classes**: 500 (Paid) / 100 (Free)
- **SQL columns per table**: 100
- **SQL statement length**: 100 KB
- **WebSocket message size**: 32 MiB (received)

### General Limits

- **Requests per DO**: ~1000/s soft limit per instance
- **Number of DOs**: Unlimited
- **WebSocket connections**: Unlimited per DO (within memory limits)
- **Memory per DO**: 128 MB (shared with CPU execution)

## Troubleshooting

### Common Issues

**Durable Object is overloaded**
- Too many requests to single instance
- Solution: Shard across multiple DOs

**Storage quota exceeded**
- Free plan: 5 GB total
- Solution: Upgrade plan or implement data cleanup

**CPU time exceeded**
- Processing took >30s between requests
- Solution: Increase `limits.cpu_ms` or break work into chunks

**WebSockets disconnecting**
- DO was evicted (no requests for too long)
- Solution: Use WebSocket hibernation or implement reconnection logic

**Migration failed**
- Invalid migration configuration
- Solution: Check tag uniqueness, class name references

## RPC vs Fetch Handler

Modern Durable Objects use RPC (compatibility_date >= 2024-04-03):

```typescript
// RPC (Recommended)
stub.myMethod(arg1, arg2)           // Type-safe, direct method calls

// Fetch (Legacy/HTTP patterns)
stub.fetch(new Request("http://do/endpoint"))
```

Use fetch when:
- Need HTTP request/response semantics
- Working with legacy code
- Proxying real HTTP requests

Use RPC for:
- Everything else (simpler, faster, type-safe)

## TypeScript Types

```typescript
import { DurableObject } from "cloudflare:workers";

interface Env {
  MY_DO: DurableObjectNamespace<MyDurableObject>;
  KV: KVNamespace;
  SECRET: string;
}

export class MyDurableObject extends DurableObject<Env> {
  // ctx: DurableObjectState - access to storage, alarms, WebSockets
  // env: Env - bindings available to this DO
}

// Workers runtime types
type DurableObjectId = /* opaque */;
type DurableObjectStub<T> = T & { fetch(request: Request): Promise<Response> };
type DurableObjectNamespace<T> = {
  newUniqueId(options?: { jurisdiction?: string }): DurableObjectId;
  idFromName(name: string): DurableObjectId;
  idFromString(id: string): DurableObjectId;
  get(id: DurableObjectId): DurableObjectStub<T>;
};
```

## Resources

- [Official Docs](https://developers.cloudflare.com/durable-objects/)
- [API Reference](https://developers.cloudflare.com/durable-objects/api/)
- [Example Apps](https://developers.cloudflare.com/durable-objects/examples/)
- [Limits](https://developers.cloudflare.com/durable-objects/platform/limits/)
- [Pricing](https://developers.cloudflare.com/durable-objects/platform/pricing/)

## Quick Reference

```typescript
// Get DO stub
const id = env.MY_DO.idFromName("room-123");
const stub = env.MY_DO.get(id);

// Call RPC method
await stub.myMethod();

// Storage
this.ctx.storage.sql.exec("SELECT * FROM users").toArray();
this.ctx.storage.kv.get("key");
await this.ctx.storage.put("key", value);

// Alarms
await this.ctx.storage.setAlarm(Date.now() + 3600000);

// WebSockets
this.ctx.acceptWebSocket(server);
this.ctx.getWebSockets();
ws.send("message");

// PITR
await this.ctx.storage.getCurrentBookmark();
await this.ctx.storage.onNextSessionRestoreBookmark(bookmark);
```
