# Cloudflare Durable Objects Storage

Expert guidance for Cloudflare Durable Objects Storage API - covering SQLite and KV storage backends, transactions, concurrency patterns, PITR, and best practices.

## When to Use

- Implementing persistent storage in Durable Objects
- Working with SQLite or KV storage APIs
- Designing data models for Durable Objects
- Implementing transactions and point-in-time recovery
- Debugging storage-related issues
- Migrating between storage backends
- Optimizing storage performance

## Core Concepts

### Storage Backends

Durable Objects support two storage backends:

1. **SQLite-backed (RECOMMENDED)**: Modern backend with SQL API, synchronous KV API, and PITR
2. **KV-backed (Legacy)**: Original backend with async KV API only

**Always use SQLite-backed Durable Objects for new projects.**

### Accessing Storage

Storage is accessed via `ctx.storage` passed to the Durable Object constructor:

```typescript
export class MyDurableObject extends DurableObject {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    // ctx.storage provides storage API access
  }
}
```

### Configuration in wrangler.toml/wrangler.jsonc

**SQLite-backed (use `new_sqlite_classes`):**

```jsonc
{
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["MyDurableObject"]
    }
  ]
}
```

```toml
[[migrations]]
tag = "v1"
new_sqlite_classes = ["MyDurableObject"]
```

**KV-backed (legacy, use `new_classes`):**

```jsonc
{
  "migrations": [
    {
      "tag": "v1",
      "new_classes": ["MyDurableObject"]
    }
  ]
}
```

## SQLite Storage API

### SQL API (`ctx.storage.sql`)

The SQL API provides full SQLite functionality with extensions.

#### Basic Usage

```typescript
export class MyDurableObject extends DurableObject {
  sql: SqlStorage;
  
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    this.sql = ctx.storage.sql;
    
    // Initialize schema
    this.sql.exec(`
      CREATE TABLE IF NOT EXISTS users(
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        email TEXT UNIQUE
      );
    `);
  }
  
  async addUser(name: string, email: string) {
    this.sql.exec(
      "INSERT INTO users (name, email) VALUES (?, ?)",
      name,
      email
    );
  }
}
```

#### Query Execution with `exec()`

```typescript
// Basic query
const cursor = this.sql.exec("SELECT * FROM users");

// With parameter binding (prevents SQL injection)
const cursor = this.sql.exec(
  "SELECT * FROM users WHERE email = ?",
  email
);

// Multiple statements (binding applies to last statement)
this.sql.exec(`
  CREATE TABLE IF NOT EXISTS products(id INTEGER PRIMARY KEY);
  INSERT INTO products (id) VALUES (?);
`, 123);
```

#### Working with Cursors

The `exec()` method returns an `SqlStorageCursor` with several consumption methods:

```typescript
// Iterate over rows
for (let row of cursor) {
  console.log(row); // { id: 1, name: "Alice", email: "..." }
}

// Convert to array
const users = this.sql.exec("SELECT * FROM users").toArray();
// [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }]

// Get single row (throws if != 1 row)
const user = this.sql.exec(
  "SELECT * FROM users WHERE id = ?", 
  userId
).one();

// Iterate manually
const cursor = this.sql.exec("SELECT * FROM users");
const result = cursor.next();
if (!result.done) {
  console.log(result.value); // First row object
}
```

#### Raw Array Format

```typescript
const cursor = this.sql.exec("SELECT * FROM users");

// Get raw arrays instead of objects
for (let row of cursor.raw()) {
  console.log(row); // [1, "Alice", "alice@example.com"]
}

// Get column names
const columnNames = cursor.columnNames; // ["id", "name", "email"]

// Mixed usage (cursor and raw() iterate same results)
const rawResult = cursor.raw().next(); // First row as array
const remaining = cursor.toArray(); // Remaining rows as objects
```

#### Cursor Properties

```typescript
const cursor = this.sql.exec("SELECT * FROM users");

// Access billing metrics
console.log(cursor.rowsRead);    // Number of rows read (for billing)
console.log(cursor.rowsWritten); // Number of rows written (for billing)
console.log(cursor.columnNames); // Column names array
```

#### TypeScript Types

Provide type safety for query results:

```typescript
type User = {
  id: number;
  name: string;
  email: string;
};

// Type parameter for object results
const user = this.sql.exec<User>(
  "SELECT * FROM users WHERE id = ?",
  userId
).one();
// user is type User

// Type parameter for raw array results
type UserRow = [id: number, name: string, email: string];
const cursor = this.sql.exec("SELECT * FROM users")
  .raw<UserRow>();
for (let row of cursor) {
  // row is type UserRow
}
```

**Important**: Type parameters do NOT validate at runtime - ensure query matches your type.

#### Supported SQLite Extensions

- **FTS5**: Full-text search (`fts5vocab` included)
- **JSON**: JSON functions/operators
- **Math**: Math functions

#### Indexes

Create indexes for read-heavy workloads:

```typescript
this.sql.exec(`
  CREATE INDEX IF NOT EXISTS idx_users_email 
  ON users(email);
`);
```

**Trade-off**: Each row update to an indexed column counts as ≥1 additional row written.

#### Database Size

```typescript
const sizeInBytes = this.ctx.storage.sql.databaseSize;
```

#### SQL Limits

| Limit | Value |
|-------|-------|
| Max columns per table | 100 |
| Max rows per table | Unlimited (subject to 10GB per-object limit) |
| Max string/BLOB/row size | 2 MB |
| Max SQL statement length | 100 KB |
| Max bound parameters | 100 |
| Max SQL function arguments | 32 |
| Max LIKE/GLOB pattern | 50 bytes |

**Note**: Cannot use transaction-related SQL statements like `BEGIN TRANSACTION` or `SAVEPOINT` directly. Use `ctx.storage.transaction()` or `ctx.storage.transactionSync()` instead.

### Synchronous KV API (`ctx.storage.kv`)

SQLite-backed Durable Objects support synchronous KV operations (stored in hidden `__cf_kv` table):

```typescript
// Get (returns undefined if missing)
const value = this.ctx.storage.kv.get("counter");

// Put (any structured-clone-compatible value)
this.ctx.storage.kv.put("counter", 42);
this.ctx.storage.kv.put("user", { name: "Alice", age: 30 });

// Delete (returns true if existed)
const existed = this.ctx.storage.kv.delete("counter");

// List
for (let [key, value] of this.ctx.storage.kv.list()) {
  console.log(key, value);
}

// List with options
const entries = this.ctx.storage.kv.list({
  start: "user:",        // Inclusive start
  startAfter: "user:a",  // Exclusive start (mutually exclusive with start)
  end: "user:z",         // Exclusive end
  prefix: "user:",       // Filter by prefix
  reverse: true,         // Descending order
  limit: 100            // Max entries
});
```

**Limits**: Key + value combined cannot exceed 2 MB.

### Asynchronous KV API (`ctx.storage`)

Both SQLite-backed and KV-backed Durable Objects support async KV operations:

```typescript
// Get single key
const value = await this.ctx.storage.get("key");

// Get multiple keys (returns Map, max 128 keys)
const map = await this.ctx.storage.get(["key1", "key2", "key3"]);

// Put single key-value
await this.ctx.storage.put("key", value);

// Put multiple (max 128 pairs)
await this.ctx.storage.put({
  "key1": "value1",
  "key2": { nested: "object" }
});

// Delete single key (returns boolean)
const existed = await this.ctx.storage.delete("key");

// Delete multiple (returns count, max 128 keys)
const count = await this.ctx.storage.delete(["key1", "key2"]);

// List all
const map = await this.ctx.storage.list();

// List with options
const map = await this.ctx.storage.list({
  start: "user:",
  startAfter: "user:a",
  end: "user:z",
  prefix: "user:",
  reverse: true,
  limit: 100
});
```

#### Async Options

```typescript
// Get options
await this.ctx.storage.get("key", {
  allowConcurrency: true,  // Allow concurrent events during operation
  noCache: true           // Don't cache this key
});

// Put/delete options
await this.ctx.storage.put("key", value, {
  allowUnconfirmed: true,  // Don't wait for disk confirmation
  noCache: true           // Discard from memory after write
});

// List options
await this.ctx.storage.list({
  prefix: "user:",
  allowConcurrency: true,
  noCache: true
});
```

**Key differences** (SQLite vs KV backend):
- **SQLite**: Key + value max 2 MB combined
- **KV**: Key max 2 KiB, value max 128 KiB

### Transactions

#### Synchronous Transactions (SQLite only)

Use `transactionSync()` for SQL operations:

```typescript
const result = this.ctx.storage.transactionSync(() => {
  this.sql.exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", 100, 1);
  this.sql.exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", 100, 2);
  return "transfer complete";
});
// Returns "transfer complete"
// If callback throws, transaction rolls back
```

**Requirements**:
- Callback MUST be synchronous (no `async`, no `await`, no Promises)
- Only for SQL queries or synchronous KV operations

#### Asynchronous Transactions

Use `transaction()` for async operations:

```typescript
await this.ctx.storage.transaction(async (txn) => {
  const value = await txn.get("counter");
  await txn.put("counter", value + 1);
  
  // Explicit rollback
  if (value > 100) {
    txn.rollback();
    return;
  }
  
  // Otherwise commits on successful completion
});
```

**For SQLite-backed objects**: Use `ctx.storage` directly in the callback - the `txn` parameter is obsolete:

```typescript
await this.ctx.storage.transaction(async () => {
  // Use ctx.storage directly
  const value = await this.ctx.storage.get("counter");
  await this.ctx.storage.put("counter", value + 1);
});
```

**Automatic write coalescing**: Multiple `put()`/`delete()` calls without intervening `await` are automatically atomic:

```typescript
// These are automatically coalesced into one atomic operation
const val = await this.ctx.storage.get("foo");
this.ctx.storage.delete("foo");
this.ctx.storage.put("bar", val);
// No possibility of data loss between delete and put
```

### Point-in-Time Recovery (PITR)

SQLite-backed Durable Objects support 30-day PITR:

```typescript
// Get current bookmark
const bookmark = await this.ctx.storage.getCurrentBookmark();

// Get bookmark for specific time
const twoDaysAgo = Date.now() - (2 * 24 * 60 * 60 * 1000);
const bookmark = await this.ctx.storage.getBookmarkForTime(twoDaysAgo);

// Schedule restoration on next session
const undoBookmark = await this.ctx.storage.onNextSessionRestoreBookmark(bookmark);
this.ctx.abort(); // Restart to apply restoration

// Later, undo the restoration
await this.ctx.storage.onNextSessionRestoreBookmark(undoBookmark);
this.ctx.abort();
```

**Bookmarks** are lexically comparable strings (earlier time < later time).

**Note**: PITR not supported in local development.

### Alarms

Schedule future execution:

```typescript
// Set alarm (Date or milliseconds since epoch)
await this.ctx.storage.setAlarm(Date.now() + 60000); // 1 minute from now
await this.ctx.storage.setAlarm(new Date("2026-01-15T10:00:00Z"));

// Get current alarm (null if not set)
const alarmTime = await this.ctx.storage.getAlarm();
if (alarmTime) {
  console.log("Alarm set for:", new Date(alarmTime));
}

// Delete alarm
await this.ctx.storage.deleteAlarm();

// Implement alarm handler
async alarm() {
  // Executes when alarm fires
  console.log("Alarm fired!");
  await this.doScheduledWork();
}
```

**Alarm behavior**:
- Stored persistently
- Survives Durable Object restarts
- Millisecond granularity
- Usually executes within few milliseconds of scheduled time
- Can be delayed up to 1 minute during maintenance/failover
- `deleteAll()` does NOT delete alarms - use `deleteAlarm()` explicitly

### Delete All Storage

```typescript
// Delete entire SQLite database (atomic)
await this.ctx.storage.deleteAll();

// Also delete alarm
await this.ctx.storage.deleteAlarm();
await this.ctx.storage.deleteAll();
```

**Important**:
- SQLite: Atomic deletion (all-or-nothing)
- KV: Can partially fail, leaving some data
- Does NOT delete alarms automatically
- Calling `deleteAll()` ensures no storage billing

### Synchronization

```typescript
// Wait for all pending writes to complete
await this.ctx.storage.sync();
```

## Concurrency Model

Durable Objects use **input gates** and **output gates** to prevent race conditions while maintaining performance.

### Input Gates

**Rule**: While a storage operation is executing, no new events (requests, fetch responses, etc.) are delivered except storage completion events.

```typescript
// NO RACE CONDITION (thanks to input gates)
async getUniqueNumber() {
  let val = await this.ctx.storage.get("counter");
  // No other request can be processed here
  await this.ctx.storage.put("counter", val + 1);
  return val;
}
```

Concurrent requests are serialized automatically:
- Request 1 calls `get()` → blocks
- Request 2 delivery is delayed until Request 1's storage operations complete
- No interleaving = no race conditions

**Caveat**: Concurrent calls from the same event are NOT protected:

```typescript
// STILL HAS RACE CONDITION
async handler() {
  // Both calls initiated without await between them
  const promise1 = this.getUniqueNumber();
  const promise2 = this.getUniqueNumber();
  const [val1, val2] = await Promise.all([promise1, promise2]);
  // val1 === val2 (duplicate!)
}
```

### Output Gates

**Rule**: Outgoing network messages (responses, fetch requests) are held until all pending storage writes complete. If writes fail, messages are discarded and the object restarts.

```typescript
async incrementCounter() {
  let val = await this.ctx.storage.get("counter");
  this.ctx.storage.put("counter", val + 1); // Don't await
  return new Response(String(val));
  // Response is NOT sent until put() confirms success
  // If put() fails, response is discarded and object restarts
}
```

**Benefits**:
- Safe to not `await` writes
- Responses never lie about persisted state
- Outgoing `fetch()` calls also delayed until writes confirm

### Automatic Caching

Built-in in-memory cache (several MB) for storage operations:

- `get()`: Returns instantly if cached
- `put()`: Writes to cache instantly (output gate delays network confirmation)
- Writes are coalesced automatically

**Result**: Simple code is fast AND correct:

```typescript
// Fast AND correct (no manual caching needed)
async getUniqueNumber() {
  let val = await this.ctx.storage.get("counter"); // Cached after first call
  await this.ctx.storage.put("counter", val + 1);  // Instant cache write
  return val;
}
```

### Bypass Options

Opt out of gates/caching when needed:

```typescript
// Allow concurrency during this operation
await this.ctx.storage.get("key", { allowConcurrency: true });

// Don't cache this key
await this.ctx.storage.get("key", { noCache: true });

// Allow network messages before write confirms
await this.ctx.storage.put("key", value, { allowUnconfirmed: true });

// Don't cache this write
await this.ctx.storage.put("key", value, { noCache: true });
```

## Common Patterns

### Initialize from Storage

Use `blockConcurrencyWhile()` to initialize in-memory state from storage:

```typescript
export class Counter extends DurableObject {
  value: number;
  
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    
    ctx.blockConcurrencyWhile(async () => {
      this.value = (await ctx.storage.get("value")) || 0;
    });
  }
  
  async increment() {
    this.value++;
    await this.ctx.storage.put("value", this.value);
    return this.value;
  }
}
```

### In-Memory Caching Pattern

```typescript
export class UserCache extends DurableObject {
  cache = new Map<string, User>();
  
  async getUser(id: string): Promise<User> {
    if (this.cache.has(id)) {
      return this.cache.get(id)!;
    }
    
    const user = await this.ctx.storage.get<User>(`user:${id}`);
    if (user) {
      this.cache.set(id, user);
    }
    return user;
  }
  
  async updateUser(id: string, data: Partial<User>) {
    const user = await this.getUser(id);
    const updated = { ...user, ...data };
    
    this.cache.set(id, updated);
    await this.ctx.storage.put(`user:${id}`, updated);
    return updated;
  }
}
```

### SQL Schema Migration

```typescript
export class MyDurableObject extends DurableObject {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    this.sql = ctx.storage.sql;
    
    // Check schema version
    this.sql.exec(`
      CREATE TABLE IF NOT EXISTS _meta(
        key TEXT PRIMARY KEY,
        value TEXT
      );
    `);
    
    const result = this.sql.exec(
      "SELECT value FROM _meta WHERE key = 'schema_version'"
    ).toArray();
    
    const version = result[0]?.value || "0";
    
    if (version === "0") {
      this.migrateToV1();
    }
    if (version === "1") {
      this.migrateToV2();
    }
  }
  
  private migrateToV1() {
    this.sql.exec(`
      CREATE TABLE users(id INTEGER PRIMARY KEY, name TEXT);
      INSERT OR REPLACE INTO _meta VALUES ('schema_version', '1');
    `);
  }
  
  private migrateToV2() {
    this.sql.exec(`
      ALTER TABLE users ADD COLUMN email TEXT;
      UPDATE _meta SET value = '2' WHERE key = 'schema_version';
    `);
  }
}
```

### Batch Processing with Alarms

```typescript
export class BatchProcessor extends DurableObject {
  pending: string[] = [];
  
  async addItem(item: string) {
    this.pending.push(item);
    
    // Schedule alarm if not set
    const alarmTime = await this.ctx.storage.getAlarm();
    if (!alarmTime) {
      await this.ctx.storage.setAlarm(Date.now() + 5000); // 5 seconds
    }
  }
  
  async alarm() {
    // Process batch
    const items = [...this.pending];
    this.pending = [];
    
    await this.processBatch(items);
  }
  
  private async processBatch(items: string[]) {
    // Batch processing logic
    this.sql.exec(`
      INSERT INTO processed_items (item, timestamp)
      VALUES ${items.map(() => "(?, ?)").join(", ")}
    `, ...items.flatMap(item => [item, Date.now()]));
  }
}
```

### Rate Limiting

```typescript
export class RateLimiter extends DurableObject {
  async checkLimit(key: string, limit: number, window: number): Promise<boolean> {
    const now = Date.now();
    const windowStart = now - window;
    
    // Clean old entries
    this.sql.exec(
      "DELETE FROM requests WHERE key = ? AND timestamp < ?",
      key,
      windowStart
    );
    
    // Count recent requests
    const count = this.sql.exec(
      "SELECT COUNT(*) as count FROM requests WHERE key = ? AND timestamp >= ?",
      key,
      windowStart
    ).one().count;
    
    if (count >= limit) {
      return false; // Rate limit exceeded
    }
    
    // Record this request
    this.sql.exec(
      "INSERT INTO requests (key, timestamp) VALUES (?, ?)",
      key,
      now
    );
    
    return true;
  }
}
```

### Data Location Control

```typescript
// Restrict to jurisdiction (GDPR/FedRAMP compliance)
const euNamespace = env.MY_DURABLE_OBJECT.jurisdiction("eu");
const euId = euNamespace.newUniqueId();
const stub = euNamespace.get(euId);

// Location hint (best effort, not guaranteed)
const stub = env.MY_DURABLE_OBJECT.get(id, {
  locationHint: "enam" // Eastern North America
});
```

**Jurisdictions**: `eu` (European Union), `fedramp` (FedRAMP-compliant)

**Location hints**: `wnam`, `enam`, `sam`, `weur`, `eeur`, `apac`, `oc`, `afr`, `me`

## Limits

### SQLite-backed Durable Objects

| Limit | Value |
|-------|-------|
| Storage per account | Unlimited (Paid) / 5 GB (Free) |
| Storage per object | 10 GB |
| Key + value size | 2 MB combined |
| WebSocket message size | 32 MiB (received) |
| CPU per request | 30s (default) / up to 5 min |
| Max Durable Object classes | 500 (Paid) / 100 (Free) |

**SQL-specific limits** (see SQL Limits table above)

### KV-backed Durable Objects (Legacy)

| Limit | Value |
|-------|-------|
| Storage per account | 50 GB (can request increase) |
| Storage per object | Unlimited |
| Key size | 2 KiB |
| Value size | 128 KiB |
| CPU per request | 30s |

### Increase CPU Limit

Set in wrangler configuration:

```jsonc
{
  "limits": {
    "cpu_ms": 300000  // 5 minutes = 300,000 ms
  }
}
```

```toml
[limits]
cpu_ms = 300_000
```

## Wrangler Commands

```bash
# Create new Durable Object class
npx wrangler init

# Local development
npx wrangler dev

# Deploy
npx wrangler deploy

# List Durable Objects
npx wrangler d1 list  # (not applicable - D1 is different)

# Get Durable Object info (via API)
curl -X GET "https://api.cloudflare.com/client/v4/accounts/{account_id}/workers/durable_objects/namespaces" \
  -H "Authorization: Bearer {api_token}"
```

## Best Practices

### Storage Backend Selection

✅ **Use SQLite-backed Durable Objects** for all new projects
- Richer feature set (SQL, PITR, synchronous KV)
- Better performance
- 10 GB per object limit (vs unlimited for KV, but 10 GB is ample)

❌ **Avoid KV-backed Durable Objects** unless maintaining legacy code

### Schema Design

✅ **Use SQL for structured data**, KV for simple key-value
✅ **Create indexes** for frequently queried columns
✅ **Normalize data** to reduce redundancy
✅ **Use parameter binding** (prevents SQL injection)

```typescript
// ✅ GOOD: Parameter binding
this.sql.exec("SELECT * FROM users WHERE email = ?", email);

// ❌ BAD: String concatenation (SQL injection risk)
this.sql.exec(`SELECT * FROM users WHERE email = '${email}'`);
```

### Performance Optimization

✅ **Leverage automatic caching** - simple code is fast
✅ **Batch operations** - insert/update multiple rows in one statement
✅ **Use synchronous KV API** (`ctx.storage.kv`) for simple key-value
✅ **Don't await writes unnecessarily** - output gates protect you

```typescript
// ✅ GOOD: Fast, correct
async increment() {
  let val = await this.ctx.storage.get("counter");
  this.ctx.storage.put("counter", val + 1); // Don't await
  return new Response(String(val));
}

// ❌ SLOWER: Unnecessary await
async increment() {
  let val = await this.ctx.storage.get("counter");
  await this.ctx.storage.put("counter", val + 1); // Slower
  return new Response(String(val));
}
```

### Data Integrity

✅ **Trust input/output gates** for most cases
✅ **Use explicit transactions** for complex multi-step operations
✅ **Initialize state in `blockConcurrencyWhile()`**
✅ **Delete alarms AND storage** when cleaning up objects

```typescript
// ✅ GOOD: Complete cleanup
async cleanup() {
  await this.ctx.storage.deleteAlarm();
  await this.ctx.storage.deleteAll();
}

// ❌ BAD: Alarm remains, causes billing
async cleanup() {
  await this.ctx.storage.deleteAll(); // Alarm not deleted!
}
```

### Avoid Common Pitfalls

❌ **Don't use transaction SQL statements directly** (`BEGIN TRANSACTION`, `SAVEPOINT`)
- Use `ctx.storage.transaction()` or `transactionSync()` instead

❌ **Don't initiate concurrent storage operations from same event**
```typescript
// ❌ BAD: Race condition
const p1 = this.getUniqueNumber();
const p2 = this.getUniqueNumber();
await Promise.all([p1, p2]); // Can return duplicates!

// ✅ GOOD: Sequential
const n1 = await this.getUniqueNumber();
const n2 = await this.getUniqueNumber();
```

❌ **Don't assume TypeScript types validate at runtime**
```typescript
// Types are hints only - query must actually return User shape
const user = this.sql.exec<User>("SELECT * FROM users WHERE id = ?", id).one();
```

## Troubleshooting

### "oldString not found in content" errors
- Ensure exact match with file content
- Check for hidden characters or different indentation

### Slow performance
- Check if using async KV API when sync KV API (`ctx.storage.kv`) is available
- Verify automatic caching is enabled (default)
- Consider whether you're awaiting writes unnecessarily
- Add indexes for frequently filtered columns

### Race conditions still occurring
- Check for concurrent calls initiated from same event (not protected by input gates)
- Verify initialization uses `blockConcurrencyWhile()`
- Consider using explicit transactions for complex operations

### Storage billing higher than expected
- Check `rowsRead` and `rowsWritten` on cursors
- Verify unused Durable Objects have had `deleteAll()` called
- Confirm alarms are deleted with `deleteAlarm()`
- Review indexes (each indexed column update = ≥1 additional row written)

### "Durable Object is overloaded" errors
- Single Durable Object soft limit: ~1,000 requests/second
- Consider sharding across multiple objects
- Optimize request processing time
- Check for CPU-intensive operations

## References

- [Durable Objects Storage API Docs](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)
- [Access Durable Objects Storage Best Practices](https://developers.cloudflare.com/durable-objects/best-practices/access-durable-objects-storage/)
- [Durable Objects: Easy, Fast, Correct — Choose Three](https://blog.cloudflare.com/durable-objects-easy-fast-correct-choose-three/)
- [Zero-latency SQLite storage in every Durable Object](https://blog.cloudflare.com/sqlite-in-durable-objects/)
- [Durable Objects Examples](https://developers.cloudflare.com/durable-objects/examples/)
- [Durable Objects Limits](https://developers.cloudflare.com/durable-objects/platform/limits/)

## Summary

Cloudflare Durable Objects Storage provides:
- **Two backends**: SQLite (recommended) and KV (legacy)
- **Multiple APIs**: SQL, synchronous KV, asynchronous KV, PITR, alarms
- **Automatic correctness**: Input/output gates prevent race conditions
- **Automatic performance**: Built-in caching makes simple code fast
- **Strong consistency**: Each object is single-threaded with private storage
- **30-day PITR**: Restore SQLite-backed objects to any point in past 30 days

**Key takeaway**: Write simple, intuitive code - the platform handles concurrency and performance automatically.
