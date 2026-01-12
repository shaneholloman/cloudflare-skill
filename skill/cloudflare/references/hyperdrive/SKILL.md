# Cloudflare Hyperdrive Skill

Expert guidance for implementing Cloudflare Hyperdrive - a service that accelerates database queries from Cloudflare Workers by managing connections, pooling, and caching.

## What is Hyperdrive?

Hyperdrive accelerates access to existing databases from Cloudflare Workers, making single-region databases feel globally distributed. It supports PostgreSQL and MySQL (and compatible databases like CockroachDB, Timescale, PlanetScale).

### Core Benefits

1. **Connection Pooling**: Maintains persistent connections to databases, eliminating repeated handshakes (TCP, TLS, auth)
2. **Edge Connection Setup**: Performs connection setup near Workers (low latency), pools connections near database
3. **Query Caching**: Automatically caches non-mutating queries with configurable TTL (default 60s)
4. **Global Performance**: Reduces latency by 7 round-trips per connection

### Architecture

```
Worker → Edge (connection setup) → Connection Pool (near DB) → Origin Database
         ↓ (cached queries)
         Cache
```

## Configuration

### Creating Hyperdrive Configurations

#### Via Wrangler CLI

**PostgreSQL:**
```bash
# Basic configuration
npx wrangler hyperdrive create my-hyperdrive \
  --connection-string="postgres://user:password@host:5432/database"

# With custom caching
npx wrangler hyperdrive create my-hyperdrive \
  --connection-string="postgres://user:password@host:5432/database" \
  --max-age=120 \
  --swr=30

# Disable caching
npx wrangler hyperdrive create my-hyperdrive \
  --connection-string="postgres://user:password@host:5432/database" \
  --caching-disabled=true
```

**MySQL:**
```bash
npx wrangler hyperdrive create my-hyperdrive \
  --connection-string="mysql://user:password@host:3306/database"
```

#### Wrangler Configuration (wrangler.jsonc)

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-09-23",
  "compatibility_flags": ["nodejs_compat"],
  "observability": {
    "enabled": true
  },
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "<HYPERDRIVE_ID>",
      "localConnectionString": "postgres://user:password@localhost:5432/dev_db"
    }
  ]
}
```

**Multiple Hyperdrive Configurations:**
```jsonc
{
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE_CACHED",
      "id": "<HYPERDRIVE_ID_WITH_CACHE>"
    },
    {
      "binding": "HYPERDRIVE_NO_CACHE",
      "id": "<HYPERDRIVE_ID_NO_CACHE>"
    }
  ]
}
```

### Management Commands

```bash
# List configurations
npx wrangler hyperdrive list

# Get configuration details
npx wrangler hyperdrive get <HYPERDRIVE_ID>

# Update configuration
npx wrangler hyperdrive update <HYPERDRIVE_ID> \
  --origin-password=<NEW_PASSWORD> \
  --max-age=180

# Delete configuration
npx wrangler hyperdrive delete <HYPERDRIVE_ID>
```

### Configuration Options Reference

| Option | Description | Default |
|--------|-------------|---------|
| `--connection-string` | Full database connection string | Required |
| `--caching-disabled` | Disable query caching | `false` |
| `--max-age` | Cache TTL in seconds (max 3600) | `60` |
| `--swr` | Stale-while-revalidate duration | `15` |
| `--origin-connection-limit` | Max connections to origin | 20 (free), 100 (paid) |
| `--access-client-id` | Access token for private DB | - |
| `--access-client-secret` | Access secret for private DB | - |
| `--ca-certificate-id` | Custom CA certificate UUID | - |
| `--mtls-certificate-id` | mTLS certificate UUID | - |
| `--sslmode` | PostgreSQL SSL mode | `require` |

## Code Patterns

### PostgreSQL with node-postgres (pg) - RECOMMENDED

```typescript
import { Client } from "pg";

export interface Env {
  HYPERDRIVE: Hyperdrive;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Create new client per request
    const client = new Client({
      connectionString: env.HYPERDRIVE.connectionString,
    });

    try {
      await client.connect();
      
      // Simple query
      const result = await client.query("SELECT * FROM users WHERE id = $1", [123]);
      
      return Response.json({ success: true, data: result.rows });
    } catch (error: any) {
      console.error("Database error:", error.message);
      return new Response("Database error", { status: 500 });
    } finally {
      // Client cleanup happens automatically
      await client.end();
    }
  },
} satisfies ExportedHandler<Env>;
```

**Installation:**
```bash
npm i pg@^8.16.3
npm i -D @types/pg
```

**wrangler.jsonc:**
```jsonc
{
  "compatibility_flags": ["nodejs_compat"],
  "compatibility_date": "2024-09-23",
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "<YOUR_HYPERDRIVE_ID>"
    }
  ]
}
```

### PostgreSQL with Postgres.js

```typescript
import postgres from "postgres";

export interface Env {
  HYPERDRIVE: Hyperdrive;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const sql = postgres(env.HYPERDRIVE.connectionString, {
      max: 5,           // Limit connections per Worker
      fetch_types: false, // Disable if not using array types (reduces latency)
      prepare: true,     // Enable prepared statements for caching (IMPORTANT)
    });

    try {
      // Tagged template query
      const users = await sql`
        SELECT * FROM users 
        WHERE active = ${true}
        ORDER BY created_at DESC
        LIMIT 10
      `;
      
      return Response.json({ users });
    } catch (e: any) {
      console.error("Database error:", e.message);
      return Response.error();
    }
  },
} satisfies ExportedHandler<Env>;
```

**Installation:**
```bash
npm i postgres@^3.4.5
```

**Key Configuration:**
- `prepare: true` - **CRITICAL** for Hyperdrive caching. Setting to `false` disables prepared statement caching
- `fetch_types: false` - Reduces unnecessary round-trip if not using PostgreSQL array types
- `max: 5` - Limit concurrent connections (Workers external connection limits)

### MySQL with mysql2

```typescript
import { createConnection } from "mysql2/promise";

export interface Env {
  HYPERDRIVE: Hyperdrive;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const connection = await createConnection({
      host: env.HYPERDRIVE.host,
      user: env.HYPERDRIVE.user,
      password: env.HYPERDRIVE.password,
      database: env.HYPERDRIVE.database,
      port: env.HYPERDRIVE.port,
      disableEval: true,  // REQUIRED for Workers compatibility
    });

    try {
      const [results, fields] = await connection.query(
        "SELECT * FROM users WHERE active = ? LIMIT ?",
        [true, 10]
      );

      // Clean up connection after response
      ctx.waitUntil(connection.end());

      return Response.json({ results, fields });
    } catch (e: any) {
      console.error("Database error:", e.message);
      return new Response("Database error", { status: 500 });
    }
  },
} satisfies ExportedHandler<Env>;
```

**Installation:**
```bash
npm i mysql2@^3.13.0
```

**Key Configuration:**
- `disableEval: true` - **REQUIRED**: mysql2 uses `eval()` for optimization, not available in Workers

### Using with Drizzle ORM

```typescript
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import { users } from "./schema";

export interface Env {
  HYPERDRIVE: Hyperdrive;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const client = postgres(env.HYPERDRIVE.connectionString, {
      max: 5,
      prepare: true,  // Enable caching
    });
    
    const db = drizzle(client);

    try {
      const activeUsers = await db
        .select()
        .from(users)
        .where(eq(users.active, true))
        .limit(10);

      return Response.json({ users: activeUsers });
    } catch (e: any) {
      console.error("Database error:", e.message);
      return Response.error();
    }
  },
} satisfies ExportedHandler<Env>;
```

**Installation:**
```bash
npm i drizzle-orm@^0.26.2 postgres@^3.4.5
```

### Using with Kysely

```typescript
import { Kysely, PostgresDialect } from "kysely";
import postgres from "postgres";

interface Database {
  users: {
    id: number;
    name: string;
    email: string;
    active: boolean;
  };
}

export interface Env {
  HYPERDRIVE: Hyperdrive;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const db = new Kysely<Database>({
      dialect: new PostgresDialect({
        postgres: postgres(env.HYPERDRIVE.connectionString, {
          max: 5,
          prepare: true,  // Enable caching
        }),
      }),
    });

    try {
      const users = await db
        .selectFrom("users")
        .selectAll()
        .where("active", "=", true)
        .limit(10)
        .execute();

      return Response.json({ users });
    } catch (e: any) {
      console.error("Database error:", e.message);
      return Response.error();
    }
  },
} satisfies ExportedHandler<Env>;
```

**Installation:**
```bash
npm i kysely@^0.26.3 postgres@^3.4.5
```

## Query Caching

### What Gets Cached

**Cacheable (non-mutating reads):**
```sql
-- PostgreSQL
SELECT * FROM articles WHERE published = true;
SELECT COUNT(*) FROM users;
SELECT title, body FROM posts WHERE author_id = 123;

-- MySQL  
SELECT * FROM products WHERE category = 'electronics';
SELECT AVG(price) FROM orders WHERE created_at > NOW() - INTERVAL 1 DAY;
```

**NOT Cacheable:**
```sql
-- Writes
INSERT INTO users (name, email) VALUES ('John', 'john@example.com');
UPDATE users SET last_login = NOW() WHERE id = 123;
DELETE FROM sessions WHERE expired = true;

-- Volatile functions (PostgreSQL)
SELECT LASTVAL();  -- Uses volatile function
SELECT * FROM generate_series(1, 10);
SELECT random();

-- MySQL
SELECT LAST_INSERT_ID();
SELECT UUID();
```

### Cache Configuration

**Default Settings:**
- `max_age`: 60 seconds
- `stale_while_revalidate`: 15 seconds
- Maximum `max_age`: 3600 seconds (1 hour)

**Disable Caching:**
```bash
npx wrangler hyperdrive update <ID> --caching-disabled=true
```

**Multiple Configurations Pattern:**
```typescript
// Use different configs for cached vs non-cached
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Cacheable queries
    const sqlCached = postgres(env.HYPERDRIVE_CACHED.connectionString);
    const popularPosts = await sqlCached`SELECT * FROM posts ORDER BY views DESC LIMIT 10`;
    
    // Non-cacheable or time-sensitive queries
    const sqlNoCache = postgres(env.HYPERDRIVE_NO_CACHE.connectionString);
    const recentOrders = await sqlNoCache`SELECT * FROM orders WHERE created_at > NOW() - INTERVAL 5 MINUTE`;
    
    return Response.json({ popularPosts, recentOrders });
  },
};
```

## Private Database Access (via Cloudflare Tunnel)

### Architecture

```
Worker → Hyperdrive → Cloudflare Access → Cloudflare Tunnel → Private Network → Database
```

### Setup Steps

**1. Create Cloudflare Tunnel:**
```bash
# In your private network
cloudflared tunnel create my-db-tunnel
```

**2. Configure Public Hostname:**
In Cloudflare Zero Trust dashboard:
- **Domain**: `db-tunnel.example.com`
- **Service Type**: `TCP`
- **URL**: `localhost:5432` (or your DB address)

**3. Create Service Token:**
In Cloudflare Zero Trust > Service Auth:
- Set duration: `Non-expiring`
- Save Client ID and Client Secret

**4. Create Access Application:**
- Application: `db-tunnel.example.com`
- Policy: Service Auth with the token from step 3
- Session Duration: `No duration, expires immediately`
- Disable App Launcher visibility

**5. Create Hyperdrive Configuration:**
```bash
npx wrangler hyperdrive create my-private-db \
  --host=db-tunnel.example.com \
  --user=dbuser \
  --password=dbpass \
  --database=production \
  --access-client-id=<CLIENT_ID> \
  --access-client-secret=<CLIENT_SECRET>
```

**Note:** Do NOT specify `--port` when using Cloudflare Tunnel. The port is configured in the tunnel's service configuration.

### Using Private Database

```typescript
import { Client } from "pg";

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Same code as public database - Hyperdrive handles tunnel routing
    const client = new Client({
      connectionString: env.HYPERDRIVE.connectionString,
    });

    await client.connect();
    const result = await client.query("SELECT * FROM private_data");
    await client.end();

    return Response.json(result.rows);
  },
};
```

## Local Development

### Option 1: wrangler dev (Local Execution)

**Using Environment Variable (RECOMMENDED):**
```bash
# Format: CLOUDFLARE_HYPERDRIVE_LOCAL_CONNECTION_STRING_<BINDING_NAME>
export CLOUDFLARE_HYPERDRIVE_LOCAL_CONNECTION_STRING_HYPERDRIVE="postgres://user:pass@localhost:5432/dev_db"
npx wrangler dev
```

**Using wrangler.jsonc:**
```jsonc
{
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "<PRODUCTION_HYPERDRIVE_ID>",
      "localConnectionString": "postgres://user:pass@localhost:5432/dev_db"
    }
  ]
}
```

**Connecting to Remote Database Locally:**
```bash
# PostgreSQL - requires sslmode
export CLOUDFLARE_HYPERDRIVE_LOCAL_CONNECTION_STRING_HYPERDRIVE="postgres://user:pass@remote-host:5432/db?sslmode=require"

# MySQL - requires sslMode
export CLOUDFLARE_HYPERDRIVE_LOCAL_CONNECTION_STRING_HYPERDRIVE="mysql://user:pass@remote-host:3306/db?sslMode=REQUIRED"
```

**Important Notes:**
- `localConnectionString` bypasses Hyperdrive (direct DB connection)
- No connection pooling or caching in local mode
- Environment variable takes precedence over `wrangler.jsonc`
- Unset: `unset CLOUDFLARE_HYPERDRIVE_LOCAL_CONNECTION_STRING_HYPERDRIVE`

### Option 2: wrangler dev --remote (Remote Execution)

```bash
npx wrangler dev --remote
```

**Characteristics:**
- Worker runs in Cloudflare's network (not locally)
- Uses deployed Hyperdrive configuration
- Connection pooling and caching enabled
- **WARNING**: Affects production data

**Use Cases:**
- Testing caching behavior
- Debugging connection pooling
- Performance testing with Hyperdrive features

## Connection Pooling

### How It Works

Hyperdrive operates in **transaction mode**:
1. Client gets connection for duration of transaction
2. Connection returns to pool when transaction completes
3. Connection is `RESET` before reuse (clears session state)

### Key Behaviors

**SET Statements:**
```typescript
// ✅ SET works within transaction
await client.query("BEGIN");
await client.query("SET work_mem = '256MB'");
await client.query("SELECT * FROM large_table");  // Uses SET value
await client.query("COMMIT");  // Connection RESET after commit

// ✅ SET works in single-statement query
await client.query("SET work_mem = '256MB'; SELECT * FROM large_table");

// ❌ SET does NOT persist across separate queries
await client.query("SET work_mem = '256MB'");  // May get different connection
await client.query("SELECT * FROM large_table");  // SET not applied
```

**Best Practices:**
```typescript
// ❌ DON'T: Long-running transactions block connection pooling
await client.query("BEGIN");
await processThousandsOfRows();  // Connection held for entire duration
await client.query("COMMIT");

// ✅ DO: Keep transactions short
await client.query("BEGIN");
await client.query("UPDATE users SET status = $1 WHERE id = $2", [status, userId]);
await client.query("COMMIT");

// ✅ DO: Use SET within transaction boundaries
await client.query("BEGIN");
await client.query("SET LOCAL work_mem = '256MB'");
await client.query("SELECT * FROM large_table");
await client.query("COMMIT");
```

### Prepared Statements

**Supported:**
- `postgres.js` - Full support (when `prepare: true`)
- `node-postgres` (pg) - Full support

**Performance Notes:**
```typescript
// ✅ Prepared statements cached by Hyperdrive
const sql = postgres(connectionString, {
  prepare: true,  // DEFAULT - enables caching
});

// ❌ Unprepared statements bypass cache
const sql = postgres(connectionString, {
  prepare: false,  // Used by Kysely, sql.unsafe() - extra round-trips
});
```

### Connection Limits

| Tier | Max Connections | Idle Timeout | Connection Timeout |
|------|-----------------|--------------|-------------------|
| Free | ~20 | 10 minutes | 15 seconds |
| Paid | ~100 | 10 minutes | 15 seconds |

**Connection Error Handling:**
```typescript
try {
  const result = await client.query("SELECT * FROM users");
} catch (error: any) {
  if (error.message.includes("Failed to acquire a connection")) {
    // Connection pool exhausted - queries/transactions taking too long
    console.error("Connection pool timeout - optimize queries");
  } else if (error.message.includes("connection_refused")) {
    // Database rejecting connections - firewall or connection limit
    console.error("Database connection refused");
  }
  throw error;
}
```

## Common Use Cases

### 1. High-Traffic Read-Heavy Application

```typescript
import postgres from "postgres";

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const sql = postgres(env.HYPERDRIVE.connectionString, { max: 5, prepare: true });

    // Cacheable: Popular content
    if (url.pathname === "/trending") {
      const posts = await sql`
        SELECT * FROM posts 
        WHERE published = true 
        ORDER BY views DESC 
        LIMIT 20
      `;
      return Response.json({ posts });
    }

    // Cacheable: User profile (reads)
    if (url.pathname.startsWith("/users/")) {
      const userId = url.pathname.split("/")[2];
      const [user] = await sql`
        SELECT id, username, bio, avatar 
        FROM users 
        WHERE id = ${userId}
      `;
      return Response.json({ user });
    }

    return new Response("Not found", { status: 404 });
  },
};
```

**Benefits:**
- Trending posts cached (60s default)
- User profiles cached (reduces DB load)
- Connection pooling handles traffic spikes

### 2. Mixed Read/Write with Multiple Configs

```typescript
export interface Env {
  HYPERDRIVE_CACHED: Hyperdrive;    // max_age=120, for reads
  HYPERDRIVE_REALTIME: Hyperdrive;  // caching disabled, for writes
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // Read-heavy: Use cached Hyperdrive
    if (request.method === "GET") {
      const sql = postgres(env.HYPERDRIVE_CACHED.connectionString, { prepare: true });
      const products = await sql`SELECT * FROM products WHERE category = ${url.searchParams.get("category")}`;
      return Response.json({ products });
    }

    // Writes: Use non-cached Hyperdrive for immediate consistency
    if (request.method === "POST") {
      const sql = postgres(env.HYPERDRIVE_REALTIME.connectionString, { prepare: true });
      const data = await request.json();
      await sql`INSERT INTO orders ${sql(data)}`;
      return Response.json({ success: true });
    }

    return new Response("Method not allowed", { status: 405 });
  },
};
```

### 3. Analytics Dashboard with Time-Series Data

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const client = new Client({ connectionString: env.HYPERDRIVE.connectionString });
    await client.connect();

    try {
      // Aggregate queries benefit from caching
      const dailyStats = await client.query(`
        SELECT 
          DATE(created_at) as date,
          COUNT(*) as total_orders,
          SUM(amount) as revenue
        FROM orders
        WHERE created_at >= NOW() - INTERVAL '30 days'
        GROUP BY DATE(created_at)
        ORDER BY date DESC
      `);

      const topProducts = await client.query(`
        SELECT 
          p.name,
          COUNT(oi.id) as order_count,
          SUM(oi.quantity * oi.price) as revenue
        FROM order_items oi
        JOIN products p ON oi.product_id = p.id
        WHERE oi.created_at >= NOW() - INTERVAL '7 days'
        GROUP BY p.id, p.name
        ORDER BY revenue DESC
        LIMIT 10
      `);

      return Response.json({
        dailyStats: dailyStats.rows,
        topProducts: topProducts.rows,
      });
    } finally {
      await client.end();
    }
  },
};
```

**Benefits:**
- Expensive aggregation queries cached
- Dashboard loads instantly (served from cache)
- Reduced load on database for complex analytics

### 4. Multi-Tenant Application

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const tenantId = request.headers.get("X-Tenant-ID");
    if (!tenantId) {
      return new Response("Missing tenant ID", { status: 400 });
    }

    const sql = postgres(env.HYPERDRIVE.connectionString, { prepare: true });

    // Tenant-scoped queries are automatically cached separately
    const data = await sql`
      SELECT * FROM documents 
      WHERE tenant_id = ${tenantId}
      AND deleted_at IS NULL
      ORDER BY updated_at DESC
      LIMIT 50
    `;

    return Response.json({ documents: data });
  },
};
```

**Benefits:**
- Each tenant's queries cached independently
- Connection pooling shared across tenants
- Protects database from multi-tenant load

### 5. Geographically Distributed Application

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Worker runs at edge closest to user
    // Hyperdrive connection setup happens at edge (low latency)
    // Connection pool near database (minimizes DB round-trips)
    
    const sql = postgres(env.HYPERDRIVE.connectionString, { prepare: true });
    
    // Global user accesses single-region DB with minimal latency
    const [user] = await sql`SELECT * FROM users WHERE id = ${request.headers.get("X-User-ID")}`;
    
    return Response.json({ 
      user,
      serverRegion: request.cf?.colo,  // Edge location that processed request
    });
  },
};
```

**Benefits:**
- Edge connection setup (fast)
- Database connection pooled near origin (efficient)
- Global traffic → single-region database (no replication needed)

## Error Handling and Observability

### Common Errors

```typescript
try {
  const result = await client.query("SELECT * FROM users");
} catch (error: any) {
  const message = error.message || "";

  // Connection pool exhausted
  if (message.includes("Failed to acquire a connection from the pool")) {
    console.error("Pool exhausted - check for long-running transactions");
    return new Response("Service temporarily busy", { status: 503 });
  }

  // Database refusing connections
  if (message.includes("connection_refused")) {
    console.error("Database refusing connections - check firewall/limits");
    return new Response("Database unavailable", { status: 503 });
  }

  // Query timeout
  if (message.includes("timeout") || message.includes("deadline exceeded")) {
    console.error("Query timeout - exceeded 60s limit");
    return new Response("Query timeout", { status: 504 });
  }

  // Authentication failure
  if (message.includes("password authentication failed")) {
    console.error("Database authentication failed - check credentials");
    return new Response("Configuration error", { status: 500 });
  }

  // SSL/TLS issues
  if (message.includes("SSL") || message.includes("TLS")) {
    console.error("TLS connection issue - check sslmode configuration");
    return new Response("Connection security error", { status: 500 });
  }

  console.error("Unknown database error:", error);
  return new Response("Internal server error", { status: 500 });
}
```

### Monitoring Connections

**PostgreSQL - Identify Hyperdrive Connections:**
```sql
-- Show Hyperdrive connections
SELECT DISTINCT usename, application_name, client_addr, state
FROM pg_stat_activity 
WHERE application_name = 'Cloudflare Hyperdrive';

-- Count active Hyperdrive connections
SELECT COUNT(*) as hyperdrive_connections
FROM pg_stat_activity 
WHERE application_name = 'Cloudflare Hyperdrive';
```

### Logging and Debugging

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const start = Date.now();
    const sql = postgres(env.HYPERDRIVE.connectionString, { prepare: true });

    try {
      console.log("Executing query:", request.url);
      const result = await sql`SELECT * FROM users LIMIT 10`;
      const duration = Date.now() - start;
      
      console.log(`Query completed in ${duration}ms, rows: ${result.length}`);
      
      return Response.json({ 
        data: result,
        meta: { duration, cached: duration < 10 }  // Likely cached if <10ms
      });
    } catch (error: any) {
      const duration = Date.now() - start;
      console.error(`Query failed after ${duration}ms:`, error.message);
      throw error;
    }
  },
};
```

## Limits

### Configuration Limits
| Limit | Free | Paid |
|-------|------|------|
| Max configured databases | 10 | 25 |
| Max username length | 63 bytes | 63 bytes |
| Max database name length | 63 bytes | 63 bytes |

### Connection Limits
| Limit | Free | Paid |
|-------|------|------|
| Initial connection timeout | 15s | 15s |
| Idle connection timeout | 10min | 10min |
| Max origin connections (per config) | ~20 | ~100 |

### Query Limits
| Limit | Free | Paid |
|-------|------|------|
| Max query duration | 60s | 60s |
| Max cached response size | 50MB | 50MB |

**Note:** Queries exceeding 60s are terminated. Responses >50MB are returned but not cached.

## TypeScript Types

```typescript
// Global Hyperdrive binding interface
interface Hyperdrive {
  connectionString: string;  // PostgreSQL: full connection string
  
  // MySQL: individual properties
  host: string;
  port: number;
  user: string;
  password: string;
  database: string;
}

// Environment bindings
interface Env {
  HYPERDRIVE: Hyperdrive;
  // Multiple bindings
  HYPERDRIVE_CACHED: Hyperdrive;
  HYPERDRIVE_NO_CACHE: Hyperdrive;
}

// Worker handler
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Implementation
  },
} satisfies ExportedHandler<Env>;
```

## Performance Best Practices

### 1. Enable Prepared Statements
```typescript
// ✅ Best performance
const sql = postgres(connectionString, {
  prepare: true,  // Enables prepared statement caching
});

// ❌ Slower - extra round-trips
const sql = postgres(connectionString, {
  prepare: false,
});
```

### 2. Optimize Connection Settings
```typescript
const sql = postgres(connectionString, {
  max: 5,             // Limit per Worker (external connection limits)
  fetch_types: false, // Disable if not using array types
  prepare: true,      // Enable caching
  idle_timeout: 60,   // Match typical Worker lifetime
});
```

### 3. Keep Transactions Short
```typescript
// ❌ BAD: Long transaction blocks connection
await client.query("BEGIN");
for (const item of largeArray) {
  await processItem(item);  // Hundreds of operations
}
await client.query("COMMIT");

// ✅ GOOD: Batch operations
await client.query("BEGIN");
await client.query("INSERT INTO items SELECT * FROM UNNEST($1::int[])", [itemIds]);
await client.query("COMMIT");
```

### 4. Use Connection Pooling Effectively
```typescript
// ❌ DON'T: Create multiple clients per request
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const client1 = new Client({ connectionString: env.HYPERDRIVE.connectionString });
    const client2 = new Client({ connectionString: env.HYPERDRIVE.connectionString });
    // Multiple connections per request is inefficient
  },
};

// ✅ DO: Single client per request
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const client = new Client({ connectionString: env.HYPERDRIVE.connectionString });
    // Reuse single connection for all queries in request
  },
};
```

### 5. Cache-Friendly Query Patterns
```typescript
// ✅ Cacheable: Deterministic queries
await sql`SELECT * FROM products WHERE category = 'electronics' LIMIT 10`;
await sql`SELECT COUNT(*) FROM users WHERE active = true`;

// ❌ Not cacheable: Volatile functions
await sql`SELECT * FROM logs WHERE created_at > NOW()`;  // NOW() is volatile
await sql`SELECT random() * 100`;

// ✅ Make it cacheable: Use parameters instead
const timestamp = Date.now();
await sql`SELECT * FROM logs WHERE created_at > ${timestamp}`;
```

### 6. Monitor Cache Hit Rates
```typescript
// Measure query performance to infer cache hits
const measureQuery = async (sql: any, query: any) => {
  const start = Date.now();
  const result = await query;
  const duration = Date.now() - start;
  
  console.log({
    duration,
    likelyCached: duration < 10,  // Sub-10ms suggests cache hit
    rows: result.length,
  });
  
  return result;
};
```

## Supported Databases

### PostgreSQL
- PostgreSQL 11+
- CockroachDB (PostgreSQL-compatible)
- Timescale (PostgreSQL-compatible)
- Materialize (PostgreSQL-compatible)
- Neon
- Supabase

**Recommended Driver:** `pg` (node-postgres) >= 8.16.3

### MySQL
- MySQL 5.7+
- PlanetScale (MySQL-compatible)

**Recommended Driver:** `mysql2` >= 3.13.0

### SSL/TLS Modes

**PostgreSQL (`sslmode`):**
- `require` - Recommended default
- `verify-ca` - With custom CA certificate
- `verify-full` - Full certificate verification

**MySQL (`sslMode`):**
- `REQUIRED` - Recommended default
- `VERIFY_CA` - With custom CA certificate
- `VERIFY_IDENTITY` - Full certificate verification

## Migration Checklist

Migrating from direct database connections to Hyperdrive:

- [ ] Create Hyperdrive configuration via Wrangler
- [ ] Add Hyperdrive binding to `wrangler.jsonc`
- [ ] Enable `nodejs_compat` compatibility flag
- [ ] Set `compatibility_date` to `2024-09-23` or later
- [ ] Update connection code to use `env.HYPERDRIVE.connectionString` (Postgres) or individual properties (MySQL)
- [ ] Configure `localConnectionString` for local development
- [ ] Set `prepare: true` if using Postgres.js
- [ ] Set `disableEval: true` if using mysql2
- [ ] Test queries locally with `wrangler dev`
- [ ] Deploy and monitor connection pool usage
- [ ] Validate cache behavior with `wrangler dev --remote`
- [ ] Update firewall rules if needed (Cloudflare IP ranges)
- [ ] Configure observability for production monitoring

## Troubleshooting

### Connection Issues

**Problem:** `connection_refused`
```
Solutions:
1. Check database firewall allows Cloudflare IPs
2. Verify database is listening on configured port
3. Confirm database service is running
4. Check connection string credentials
```

**Problem:** `Failed to acquire a connection from the pool`
```
Solutions:
1. Reduce transaction duration
2. Avoid long-running queries (>60s timeout)
3. Don't hold connections during external API calls
4. Consider upgrading to paid plan (more connections)
```

**Problem:** `SSL/TLS connection failed`
```
Solutions:
1. Add sslmode=require (PostgreSQL) or sslMode=REQUIRED (MySQL)
2. Upload custom CA certificate if using self-signed certs
3. Verify database has SSL/TLS enabled
4. Check certificate expiry
```

### Query Issues

**Problem:** Queries not being cached
```
Debugging:
1. Verify query is non-mutating (SELECT, not INSERT/UPDATE)
2. Check for volatile functions (NOW(), RANDOM(), etc.)
3. Confirm caching is not disabled in configuration
4. Use wrangler dev --remote to test caching behavior
5. Check that prepare=true for Postgres.js
```

**Problem:** Query timeout (>60s)
```
Solutions:
1. Optimize query with proper indexes
2. Reduce dataset size (use LIMIT)
3. Break into smaller queries
4. Consider async processing for heavy operations
```

### Local Development Issues

**Problem:** Can't connect to local database
```
Solutions:
1. Verify localConnectionString is set correctly
2. Check local database is running
3. Confirm environment variable name matches binding
4. Try connection string directly with psql/mysql client
```

**Problem:** Environment variable not working
```
Check:
1. Format: CLOUDFLARE_HYPERDRIVE_LOCAL_CONNECTION_STRING_<BINDING_NAME>
2. Binding name matches wrangler.jsonc exactly
3. Variable is exported in current shell session
4. Restart wrangler dev after setting variable
```

## Resources

- [Official Hyperdrive Docs](https://developers.cloudflare.com/hyperdrive/)
- [Getting Started Guide](https://developers.cloudflare.com/hyperdrive/get-started/)
- [Wrangler Commands Reference](https://developers.cloudflare.com/hyperdrive/reference/wrangler-commands/)
- [Supported Databases](https://developers.cloudflare.com/hyperdrive/reference/supported-databases-and-features/)
- [Cloudflare Discord - #hyperdrive](https://discord.cloudflare.com)
- [Limit Increase Request Form](https://forms.gle/ukpeZVLWLnKeixDu7)

## When to Use Hyperdrive

**✅ Great For:**
- Existing single-region databases accessed globally
- High read-to-write ratios (10:1 or higher)
- Applications with popular queries (caching benefits)
- Connection-heavy workloads (connection pooling)
- Latency-sensitive applications
- Private databases (via Cloudflare Tunnel)

**❌ Not Ideal For:**
- Write-heavy workloads (limited caching benefit)
- Queries requiring real-time data (<1s freshness)
- Single-region applications close to database
- Very simple applications (overhead not justified)
- Databases with strict connection limits already exceeded

**Consider Alternatives:**
- D1 - Cloudflare's native distributed SQL database
- Durable Objects - Stateful Workers for coordination
- KV - Key-value storage with global replication
- R2 - Object storage for files/blobs
