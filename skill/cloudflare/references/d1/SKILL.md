# Cloudflare D1 Database Skill

Expert guidance for Cloudflare D1, a serverless SQLite database designed for horizontal scale-out across multiple databases.

## Overview

D1 is Cloudflare's managed, serverless database with:
- SQLite SQL semantics and compatibility
- Built-in disaster recovery via Time Travel (30-day point-in-time recovery)
- Horizontal scale-out architecture (10 GB per database)
- Worker and HTTP API access
- Pricing based on query and storage costs only

**Architecture Philosophy**: D1 is optimized for per-user, per-tenant, or per-entity database patterns rather than single large databases.

## Configuration

### wrangler.toml Setup

```toml
name = "your-worker-name"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"                    # Environment variable name in Worker
database_name = "your-db-name"    # Human-readable name
database_id = "your-database-id"  # UUID from D1 dashboard/CLI
migrations_dir = "migrations"     # Optional: path to migration files
```

**Multiple databases**:
```toml
[[d1_databases]]
binding = "DB"
database_name = "main-db"
database_id = "xxx-xxx-xxx"

[[d1_databases]]
binding = "ANALYTICS_DB"
database_name = "analytics-db"
database_id = "yyy-yyy-yyy"
```

### TypeScript Environment Types

```typescript
interface Env {
  DB: D1Database;
  ANALYTICS_DB?: D1Database;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Access via env.DB
  }
}
```

## Database Operations

### Creating a Database

```bash
# Create new database
wrangler d1 create <database-name>

# Output includes database_id for wrangler.toml
```

### Querying Patterns

#### Basic Prepared Statements (ALWAYS use these)

```typescript
// ❌ NEVER: Direct string interpolation (SQL injection risk)
const result = await env.DB.prepare(`SELECT * FROM users WHERE id = ${userId}`).all();

// ✅ CORRECT: Prepared statements with bind()
const result = await env.DB.prepare(
  'SELECT * FROM users WHERE id = ?'
).bind(userId).all();

// Multiple parameters
const result = await env.DB.prepare(
  'SELECT * FROM users WHERE email = ? AND active = ?'
).bind(email, true).all();
```

#### Query Execution Methods

```typescript
// .all() - Returns all rows
const { results, success, meta } = await env.DB.prepare(
  'SELECT * FROM users WHERE active = ?'
).bind(true).all();
// results: Array of row objects
// success: boolean
// meta: { duration: number, rows_read: number, rows_written: number }

// .first() - Returns first row or null
const user = await env.DB.prepare(
  'SELECT * FROM users WHERE id = ?'
).bind(userId).first();
// Returns single row object or null

// .first(columnName) - Returns single column value
const email = await env.DB.prepare(
  'SELECT email FROM users WHERE id = ?'
).bind(userId).first('email');
// Returns string | number | null

// .run() - For INSERT/UPDATE/DELETE (no row data returned)
const result = await env.DB.prepare(
  'UPDATE users SET last_login = ? WHERE id = ?'
).bind(new Date().toISOString(), userId).run();
// result.meta: { duration, rows_read, rows_written, last_row_id, changes }

// .raw() - Returns array of arrays (more efficient for large datasets)
const rawResults = await env.DB.prepare('SELECT id, name FROM users').raw();
// [[1, 'Alice'], [2, 'Bob']]
```

#### Batch Operations

```typescript
// Execute multiple queries in single round trip
const results = await env.DB.batch([
  env.DB.prepare('SELECT * FROM users WHERE id = ?').bind(1),
  env.DB.prepare('SELECT * FROM posts WHERE author_id = ?').bind(1),
  env.DB.prepare('UPDATE users SET last_access = ? WHERE id = ?')
    .bind(new Date().toISOString(), 1)
]);

// results is array: [result1, result2, result3]
// Each element has same structure as individual query results

// Batch with same prepared statement, different params
const userIds = [1, 2, 3];
const stmt = env.DB.prepare('SELECT * FROM users WHERE id = ?');
const results = await env.DB.batch(
  userIds.map(id => stmt.bind(id))
);
```

#### Transactions (via batch)

```typescript
// D1 executes batch() as atomic transaction
const results = await env.DB.batch([
  env.DB.prepare('INSERT INTO accounts (id, balance) VALUES (?, ?)').bind(1, 100),
  env.DB.prepare('INSERT INTO accounts (id, balance) VALUES (?, ?)').bind(2, 200),
  env.DB.prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?').bind(50, 1),
  env.DB.prepare('UPDATE accounts SET balance = balance + ? WHERE id = ?').bind(50, 2)
]);

// All succeed or all fail (atomic)
```

### Common Query Patterns

#### Pagination

```typescript
interface PaginationParams {
  page: number;
  pageSize: number;
}

async function getUsers({ page, pageSize }: PaginationParams, env: Env) {
  const offset = (page - 1) * pageSize;
  
  const [countResult, dataResult] = await env.DB.batch([
    env.DB.prepare('SELECT COUNT(*) as total FROM users'),
    env.DB.prepare('SELECT * FROM users ORDER BY created_at DESC LIMIT ? OFFSET ?')
      .bind(pageSize, offset)
  ]);
  
  return {
    data: dataResult.results,
    total: countResult.results[0].total,
    page,
    pageSize,
    totalPages: Math.ceil(countResult.results[0].total / pageSize)
  };
}
```

#### Conditional Queries

```typescript
async function searchUsers(filters: {
  name?: string;
  email?: string;
  active?: boolean;
}, env: Env) {
  const conditions: string[] = [];
  const params: any[] = [];
  
  if (filters.name) {
    conditions.push('name LIKE ?');
    params.push(`%${filters.name}%`);
  }
  
  if (filters.email) {
    conditions.push('email = ?');
    params.push(filters.email);
  }
  
  if (filters.active !== undefined) {
    conditions.push('active = ?');
    params.push(filters.active ? 1 : 0);
  }
  
  const whereClause = conditions.length > 0 
    ? `WHERE ${conditions.join(' AND ')}` 
    : '';
  
  const stmt = env.DB.prepare(`SELECT * FROM users ${whereClause}`);
  return await stmt.bind(...params).all();
}
```

#### Bulk Insert

```typescript
async function bulkInsertUsers(users: Array<{ name: string; email: string }>, env: Env) {
  const stmt = env.DB.prepare(
    'INSERT INTO users (name, email) VALUES (?, ?)'
  );
  
  const batch = users.map(user => stmt.bind(user.name, user.email));
  return await env.DB.batch(batch);
}
```

## Migrations

### Creating Migrations

```bash
# Create migrations directory
mkdir -p migrations

# Create migration file (use sequential numbering)
# migrations/0001_initial_schema.sql
```

Example migration:
```sql
-- migrations/0001_initial_schema.sql
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  updated_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);

CREATE TABLE IF NOT EXISTS posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  title TEXT NOT NULL,
  content TEXT,
  published BOOLEAN DEFAULT 0,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_published ON posts(published);
```

### Running Migrations

```bash
# Local development
wrangler d1 execute <database-name> --local --file=./migrations/0001_initial_schema.sql

# Remote/production
wrangler d1 execute <database-name> --remote --file=./migrations/0001_initial_schema.sql

# Execute SQL command directly
wrangler d1 execute <database-name> --remote --command="SELECT * FROM users LIMIT 10"
```

### Migration Management Pattern

```bash
# Create schema_version table for tracking
wrangler d1 execute <db-name> --remote --command="
CREATE TABLE IF NOT EXISTS schema_version (
  version INTEGER PRIMARY KEY,
  name TEXT NOT NULL,
  applied_at INTEGER NOT NULL DEFAULT (unixepoch()),
  checksum TEXT
);"

# Check applied migrations
wrangler d1 execute <db-name> --remote --command="SELECT * FROM schema_version ORDER BY version"
```

## Local Development

### Using --local Flag

```bash
# Start local dev server with D1
wrangler dev

# Execute queries locally
wrangler d1 execute <database-name> --local --command="SELECT * FROM users"

# Local migrations
wrangler d1 execute <database-name> --local --file=./migrations/0001_initial_schema.sql

# Persist local data across restarts
wrangler dev --persist-to=./.wrangler/state
```

### Local Database Location

```
.wrangler/state/v3/d1/<database-id>.sqlite
```

### Testing Pattern

```typescript
// test-setup.ts
import { unstable_dev } from 'wrangler';

describe('D1 Tests', () => {
  let worker: Awaited<ReturnType<typeof unstable_dev>>;
  
  beforeAll(async () => {
    worker = await unstable_dev('src/index.ts', {
      experimental: { disableExperimentalWarning: true }
    });
  });
  
  afterAll(async () => {
    await worker.stop();
  });
  
  it('should query users', async () => {
    const resp = await worker.fetch('/api/users');
    expect(resp.status).toBe(200);
  });
});
```

## Time Travel & Backups

### Point-in-Time Recovery

```bash
# Restore database to specific timestamp
wrangler d1 time-travel restore <database-name> --timestamp="2024-01-15T14:30:00Z"

# List available restore points (30-day window)
wrangler d1 time-travel info <database-name>
```

### Export & Import

```bash
# Export full database (schema + data)
wrangler d1 export <database-name> --remote --output=./backup.sql

# Export without schema (data only)
wrangler d1 export <database-name> --remote --no-schema --output=./data.sql

# Export specific table
wrangler d1 export <database-name> --remote --table=users --output=./users.sql

# Import
wrangler d1 execute <database-name> --remote --file=./backup.sql
```

## Performance Optimization

### Indexing Strategy

```sql
-- Index frequently queried columns
CREATE INDEX idx_users_email ON users(email);

-- Composite indexes for multi-column queries
CREATE INDEX idx_posts_user_published ON posts(user_id, published);

-- Covering indexes (include queried columns)
CREATE INDEX idx_users_email_name ON users(email, name);

-- Partial indexes for filtered queries
CREATE INDEX idx_active_users ON users(email) WHERE active = 1;

-- Check if query uses index
EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = ?;
```

### Query Optimization

```typescript
// ✅ Use indexes in WHERE clauses
const users = await env.DB.prepare(
  'SELECT * FROM users WHERE email = ?'
).bind(email).all();

// ✅ Limit result sets
const recentPosts = await env.DB.prepare(
  'SELECT * FROM posts ORDER BY created_at DESC LIMIT 100'
).all();

// ✅ Use batch() for multiple independent queries
const [user, posts, comments] = await env.DB.batch([
  env.DB.prepare('SELECT * FROM users WHERE id = ?').bind(userId),
  env.DB.prepare('SELECT * FROM posts WHERE user_id = ?').bind(userId),
  env.DB.prepare('SELECT * FROM comments WHERE user_id = ?').bind(userId)
]);

// ❌ Avoid N+1 queries
for (const post of posts) {
  const author = await env.DB.prepare('SELECT * FROM users WHERE id = ?')
    .bind(post.user_id).first(); // Bad: multiple round trips
}

// ✅ Use JOINs or batch() instead
const postsWithAuthors = await env.DB.prepare(`
  SELECT posts.*, users.name as author_name
  FROM posts
  JOIN users ON posts.user_id = users.id
`).all();
```

### Caching Pattern

```typescript
interface Env {
  DB: D1Database;
  CACHE: KVNamespace; // Optional: Use with Workers KV
}

async function getCachedUser(userId: number, env: Env) {
  const cacheKey = `user:${userId}`;
  
  // Try cache first
  const cached = await env.CACHE?.get(cacheKey, 'json');
  if (cached) return cached;
  
  // Query D1
  const user = await env.DB.prepare('SELECT * FROM users WHERE id = ?')
    .bind(userId).first();
  
  // Cache for 5 minutes
  if (user) {
    await env.CACHE?.put(cacheKey, JSON.stringify(user), {
      expirationTtl: 300
    });
  }
  
  return user;
}
```

## Error Handling

### Proper Error Handling Pattern

```typescript
async function getUser(userId: number, env: Env): Promise<Response> {
  try {
    const result = await env.DB.prepare(
      'SELECT * FROM users WHERE id = ?'
    ).bind(userId).all();
    
    if (!result.success) {
      console.error('D1 query failed:', result.error);
      return new Response('Database error', { status: 500 });
    }
    
    if (result.results.length === 0) {
      return new Response('User not found', { status: 404 });
    }
    
    return Response.json(result.results[0]);
  } catch (error) {
    console.error('Unexpected error:', error);
    return new Response('Internal error', { status: 500 });
  }
}
```

### Common Error Scenarios

```typescript
// Handle constraint violations
try {
  await env.DB.prepare(
    'INSERT INTO users (email, name) VALUES (?, ?)'
  ).bind(email, name).run();
} catch (error) {
  if (error.message?.includes('UNIQUE constraint failed')) {
    return new Response('Email already exists', { status: 409 });
  }
  throw error;
}

// Handle missing tables
const result = await env.DB.prepare('SELECT * FROM users').all();
if (!result.success && result.error?.includes('no such table')) {
  // Run migrations or create table
}
```

## REST API (HTTP) Access

```typescript
// Access D1 via Cloudflare REST API
const CLOUDFLARE_API_TOKEN = 'your-api-token';
const ACCOUNT_ID = 'your-account-id';
const DATABASE_ID = 'your-database-id';

const response = await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/d1/database/${DATABASE_ID}/query`,
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${CLOUDFLARE_API_TOKEN}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      sql: 'SELECT * FROM users WHERE id = ?',
      params: [userId]
    })
  }
);

const data = await response.json();
```

## Integration with ORMs

### Drizzle ORM

```typescript
// drizzle.config.ts
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/schema.ts',
  out: './migrations',
  dialect: 'sqlite',
  driver: 'd1-http',
  dbCredentials: {
    accountId: process.env.CLOUDFLARE_ACCOUNT_ID!,
    databaseId: process.env.D1_DATABASE_ID!,
    token: process.env.CLOUDFLARE_API_TOKEN!
  }
} satisfies Config;
```

```typescript
// schema.ts
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';

export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
  createdAt: text('created_at').default('CURRENT_TIMESTAMP')
});
```

```typescript
// worker.ts
import { drizzle } from 'drizzle-orm/d1';
import { users } from './schema';

export default {
  async fetch(request: Request, env: Env) {
    const db = drizzle(env.DB);
    
    const allUsers = await db.select().from(users);
    return Response.json(allUsers);
  }
}
```

## Limits & Considerations

### Platform Limits
- Database size: 10 GB per database (scale horizontally with multiple DBs)
- Row size: 1 MB maximum
- Query execution: 30 seconds timeout
- Batch size: 10,000 statements per batch
- API rate limits: Check current Cloudflare documentation

### Best Practices
- ✅ Use prepared statements with bind() (never string interpolation)
- ✅ Create indexes on frequently queried columns
- ✅ Use batch() for multiple queries to reduce latency
- ✅ Design for multiple small databases (per-tenant pattern)
- ✅ Use Time Travel for backups/recovery
- ✅ Test migrations locally before applying remotely
- ✅ Monitor query performance via meta.duration
- ❌ Avoid large single databases (use horizontal scaling)
- ❌ Don't store binary data directly (use R2 for blobs)
- ❌ Avoid long-running transactions (30s timeout)

## Common Use Cases

### Multi-Tenant SaaS
```typescript
// Each tenant gets own database
interface Env {
  [key: `TENANT_${string}`]: D1Database;
}

export default {
  async fetch(request: Request, env: Env) {
    const tenantId = request.headers.get('X-Tenant-ID');
    const db = env[`TENANT_${tenantId}`];
    
    const data = await db.prepare('SELECT * FROM records').all();
    return Response.json(data.results);
  }
}
```

### Session Storage
```typescript
async function createSession(userId: number, token: string, env: Env) {
  const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString();
  
  return await env.DB.prepare(`
    INSERT INTO sessions (user_id, token, expires_at, created_at)
    VALUES (?, ?, ?, CURRENT_TIMESTAMP)
  `).bind(userId, token, expiresAt).run();
}

async function validateSession(token: string, env: Env) {
  return await env.DB.prepare(`
    SELECT s.*, u.email, u.name
    FROM sessions s
    JOIN users u ON s.user_id = u.id
    WHERE s.token = ? AND s.expires_at > CURRENT_TIMESTAMP
  `).bind(token).first();
}
```

### Analytics/Events
```typescript
async function logEvent(event: {
  type: string;
  userId?: number;
  metadata: any;
}, env: Env) {
  return await env.DB.prepare(`
    INSERT INTO events (type, user_id, metadata, timestamp)
    VALUES (?, ?, ?, CURRENT_TIMESTAMP)
  `).bind(event.type, event.userId || null, JSON.stringify(event.metadata)).run();
}

async function getEventStats(startDate: string, endDate: string, env: Env) {
  return await env.DB.prepare(`
    SELECT type, COUNT(*) as count
    FROM events
    WHERE timestamp BETWEEN ? AND ?
    GROUP BY type
    ORDER BY count DESC
  `).bind(startDate, endDate).all();
}
```

## Debugging

### Query Inspection
```typescript
// Log query metadata
const result = await env.DB.prepare('SELECT * FROM users').all();
console.log('Query duration:', result.meta.duration, 'ms');
console.log('Rows read:', result.meta.rows_read);
console.log('Rows written:', result.meta.rows_written);

// Use EXPLAIN for query plans
const plan = await env.DB.prepare(
  'EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = ?'
).bind(email).all();
console.log('Query plan:', plan.results);
```

### Local Database Inspection
```bash
# Open local D1 database with sqlite3
sqlite3 .wrangler/state/v3/d1/<database-id>.sqlite

# Common SQLite commands
.tables              # List tables
.schema users        # Show table schema
.indexes users       # Show indexes
PRAGMA table_info(users);  # Column details
```

## Resources

- [Official D1 Documentation](https://developers.cloudflare.com/d1/)
- [D1 Worker API Reference](https://developers.cloudflare.com/d1/worker-api/)
- [SQLite SQL Reference](https://www.sqlite.org/lang.html)
- [Wrangler CLI Reference](https://developers.cloudflare.com/workers/wrangler/)
- [D1 Pricing](https://developers.cloudflare.com/d1/platform/pricing/)
- [D1 Limits](https://developers.cloudflare.com/d1/platform/limits/)
