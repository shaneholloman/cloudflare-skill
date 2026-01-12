# Cloudflare API Integration Skill

Guide for working with Cloudflare's REST API - covering authentication, SDK usage, common patterns, and API categories.

## When to Use This Skill

Use when working with:
- Cloudflare API authentication and tokens
- Official Cloudflare SDKs (TypeScript, Python, Go)
- Zone management, DNS, Workers, or storage APIs
- API requests via curl or HTTP clients
- Wrangler CLI API interactions
- Error handling and pagination
- Rate limiting and best practices

## Authentication

### API Token (Recommended)

**Create token**: Dashboard → My Profile → API Tokens → Create Token

```bash
# Environment variable
export CLOUDFLARE_API_TOKEN='your-token-here'

# curl
curl "https://api.cloudflare.com/client/v4/zones" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

**Token scopes**: Always use minimal permissions
- Zone-specific vs account-level
- Read vs edit permissions
- Time-limited tokens for CI/CD

### API Key (Legacy - Not Recommended)

```bash
curl "https://api.cloudflare.com/client/v4/zones" \
  --header "X-Auth-Email: user@example.com" \
  --header "X-Auth-Key: $CLOUDFLARE_API_KEY"
```

**Limitations**:
- Full account access (insecure)
- Cannot scope permissions
- No expiration
- Use tokens instead

## Official SDKs

### TypeScript SDK

```typescript
import Cloudflare from 'cloudflare';

const client = new Cloudflare({
  apiToken: process.env.CLOUDFLARE_API_TOKEN,
});

// List zones
const zones = await client.zones.list();

// Get zone details
const zone = await client.zones.get({ zone_id: 'zone-id' });

// Create DNS record
const record = await client.dns.records.create({
  zone_id: 'zone-id',
  name: 'example.com',
  type: 'A',
  content: '192.0.2.1',
  ttl: 3600,
  proxied: true,
});

// Update zone settings
await client.zones.settings.edit('zone-id', 'ssl', {
  value: 'full',
});
```

**Common patterns**:

```typescript
// Error handling
try {
  const zone = await client.zones.get({ zone_id: 'invalid-id' });
} catch (error) {
  if (error instanceof Cloudflare.APIError) {
    console.error('API error:', error.status, error.message);
  }
  throw error;
}

// Pagination
const allRecords = [];
for await (const record of client.dns.records.list({
  zone_id: 'zone-id',
})) {
  allRecords.push(record);
}

// Batch operations
const promises = zoneIds.map(id =>
  client.zones.settings.edit(id, 'ssl', { value: 'full' })
);
await Promise.all(promises);
```

### Python SDK

```python
from cloudflare import Cloudflare

client = Cloudflare(api_token=os.environ["CLOUDFLARE_API_TOKEN"])

# List zones
zones = client.zones.list()

# Create DNS record
record = client.dns.records.create(
    zone_id="zone-id",
    name="example.com",
    type="A",
    content="192.0.2.1",
    ttl=3600,
    proxied=True,
)
```

### Go SDK

```go
import (
    "github.com/cloudflare/cloudflare-go"
)

api, err := cloudflare.NewWithAPIToken(os.Getenv("CLOUDFLARE_API_TOKEN"))
if err != nil {
    log.Fatal(err)
}

zones, err := api.ListZones(context.Background())
```

## API Base URL and Versioning

```
https://api.cloudflare.com/client/v4/
```

All endpoints use v4. Structure:
- `/zones/{zone_id}` - Zone-level resources
- `/accounts/{account_id}` - Account-level resources
- `/user` - User-level resources

## Common API Categories

### Zone Management

```typescript
// List zones (with filtering)
const zones = await client.zones.list({
  account: { id: 'account-id' },
  status: 'active',
});

// Create zone
const zone = await client.zones.create({
  account: { id: 'account-id' },
  name: 'example.com',
  type: 'full', // or 'partial'
});

// Update zone
await client.zones.edit('zone-id', {
  paused: false,
  vanity_name_servers: ['ns1.example.com', 'ns2.example.com'],
});

// Delete zone
await client.zones.delete('zone-id');
```

### DNS Management

```typescript
// Create record
await client.dns.records.create({
  zone_id: 'zone-id',
  type: 'A' | 'AAAA' | 'CNAME' | 'TXT' | 'MX' | 'SRV',
  name: 'subdomain.example.com',
  content: '192.0.2.1',
  ttl: 1, // 1 = auto, or specific seconds
  proxied: true, // Orange cloud
  priority: 10, // For MX/SRV records
});

// List records (with filtering)
const records = await client.dns.records.list({
  zone_id: 'zone-id',
  type: 'A',
  name: 'example.com',
});

// Update record
await client.dns.records.update('zone-id', 'record-id', {
  content: '192.0.2.2',
  proxied: false,
});

// Bulk import (BIND format)
await client.dns.records.import('zone-id', {
  file: bindFileContent,
});
```

### Workers API

```typescript
// Upload worker script
await client.workers.scripts.update('worker-name', {
  account_id: 'account-id',
  script: workerCode,
  bindings: [
    { type: 'kv_namespace', name: 'MY_KV', namespace_id: 'kv-id' },
    { type: 'plain_text', name: 'API_KEY', text: 'secret' },
  ],
});

// List workers
const workers = await client.workers.scripts.list({
  account_id: 'account-id',
});

// Deploy to route
await client.workers.routes.create({
  zone_id: 'zone-id',
  pattern: 'api.example.com/*',
  script: 'worker-name',
});
```

### KV Storage

```typescript
// Create namespace
const namespace = await client.kv.namespaces.create({
  account_id: 'account-id',
  title: 'my-namespace',
});

// Write key
await client.kv.namespaces.values.update(
  'account-id',
  'namespace-id',
  'key-name',
  { value: 'value-data' }
);

// Read key
const value = await client.kv.namespaces.values.get(
  'account-id',
  'namespace-id',
  'key-name'
);

// List keys
const keys = await client.kv.namespaces.keys.list({
  account_id: 'account-id',
  namespace_id: 'namespace-id',
  prefix: 'user:',
});
```

### R2 Storage

```typescript
// Create bucket
await client.r2.buckets.create({
  account_id: 'account-id',
  name: 'my-bucket',
});

// List buckets
const buckets = await client.r2.buckets.list({
  account_id: 'account-id',
});

// Use with S3-compatible API
import { S3Client } from '@aws-sdk/client-s3';

const s3 = new S3Client({
  region: 'auto',
  endpoint: `https://${accountId}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: r2AccessKeyId,
    secretAccessKey: r2SecretAccessKey,
  },
});
```

### D1 Database

```typescript
// Create database
const db = await client.d1.databases.create({
  account_id: 'account-id',
  name: 'my-database',
});

// Query database
const result = await client.d1.databases.query('account-id', 'database-id', {
  sql: 'SELECT * FROM users WHERE id = ?',
  params: ['123'],
});
```

### Pages API

```typescript
// List projects
const projects = await client.pages.projects.list({
  account_id: 'account-id',
});

// Create deployment
const deployment = await client.pages.projects.deployments.create({
  account_id: 'account-id',
  project_name: 'my-project',
  branch: 'main',
});

// Get deployment logs
const logs = await client.pages.projects.deployments.logs.list({
  account_id: 'account-id',
  project_name: 'my-project',
  deployment_id: 'deployment-id',
});
```

### Cache Management

```typescript
// Purge all cache
await client.cache.purge.create({
  zone_id: 'zone-id',
  purge_everything: true,
});

// Purge by URL
await client.cache.purge.create({
  zone_id: 'zone-id',
  files: ['https://example.com/image.jpg'],
});

// Purge by tags
await client.cache.purge.create({
  zone_id: 'zone-id',
  tags: ['product', 'homepage'],
});

// Purge by prefix
await client.cache.purge.create({
  zone_id: 'zone-id',
  prefixes: ['https://example.com/api/'],
});
```

### Firewall Rules & WAF

```typescript
// Create firewall rule
await client.rulesets.create({
  zone_id: 'zone-id',
  kind: 'zone',
  phase: 'http_request_firewall_custom',
  rules: [
    {
      action: 'block',
      expression: '(ip.src eq 192.0.2.1)',
      description: 'Block specific IP',
    },
  ],
});

// Update WAF settings
await client.zones.settings.edit('zone-id', 'waf', {
  value: 'on',
});
```

### SSL/TLS Certificates

```typescript
// Get SSL settings
const ssl = await client.zones.settings.get('zone-id', 'ssl');

// Update SSL mode
await client.zones.settings.edit('zone-id', 'ssl', {
  value: 'full' | 'flexible' | 'full_strict' | 'off',
});

// Order universal SSL certificate
await client.ssl.certificatePacks.create({
  zone_id: 'zone-id',
  type: 'advanced',
  hosts: ['example.com', '*.example.com'],
  validation_method: 'txt',
  validity_days: 90,
});

// List certificates
const certs = await client.ssl.certificatePacks.list({
  zone_id: 'zone-id',
});
```

### Load Balancers

```typescript
// Create load balancer
const lb = await client.loadBalancers.create({
  zone_id: 'zone-id',
  name: 'api.example.com',
  default_pools: ['pool-id-1', 'pool-id-2'],
  fallback_pool: 'pool-id-3',
  ttl: 30,
  steering_policy: 'dynamic_latency',
});

// Create pool
const pool = await client.loadBalancers.pools.create({
  account_id: 'account-id',
  name: 'us-east-pool',
  origins: [
    {
      name: 'origin-1',
      address: '192.0.2.1',
      enabled: true,
      weight: 1,
    },
  ],
  check_regions: ['WNAM', 'ENAM'],
});
```

### Analytics & Logs

```typescript
// Get zone analytics
const analytics = await client.zones.analytics.dashboard.get({
  zone_id: 'zone-id',
  since: '2024-01-01T00:00:00Z',
  until: '2024-01-31T23:59:59Z',
});

// Create logpush job
const job = await client.logpush.jobs.create({
  zone_id: 'zone-id',
  destination_conf: 's3://bucket/path?region=us-east-1',
  dataset: 'http_requests',
  logpull_options: 'fields=ClientIP,ClientRequestHost,ClientRequestMethod',
});
```

## curl Usage Patterns

### Basic request structure

```bash
# GET request
curl "https://api.cloudflare.com/client/v4/zones" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"

# POST request
curl --request POST \
  "https://api.cloudflare.com/client/v4/zones" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "name": "example.com",
    "account": {"id": "account-id"},
    "type": "full"
  }'

# PATCH request
curl --request PATCH \
  "https://api.cloudflare.com/client/v4/zones/zone-id" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --data '{"paused": false}'

# DELETE request
curl --request DELETE \
  "https://api.cloudflare.com/client/v4/zones/zone-id" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

### Query parameters

```bash
# Filtering
curl "https://api.cloudflare.com/client/v4/zones?status=active&account.id=$ACCOUNT_ID" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"

# Pagination
curl "https://api.cloudflare.com/client/v4/dns_records?per_page=100&page=2" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"

# Ordering
curl "https://api.cloudflare.com/client/v4/zones?order=name&direction=asc" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

### Response formatting

```bash
# Format JSON with jq
curl "https://api.cloudflare.com/client/v4/zones" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | jq .

# Extract specific fields
curl "https://api.cloudflare.com/client/v4/zones" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | \
  jq '.result[] | {id, name, status}'

# Get first zone ID
ZONE_ID=$(curl "https://api.cloudflare.com/client/v4/zones" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | \
  jq -r '.result[0].id')
```

## Response Structure

All API responses follow this structure:

```json
{
  "success": true,
  "errors": [],
  "messages": [],
  "result": {
    // Response data
  },
  "result_info": {
    "page": 1,
    "per_page": 20,
    "total_pages": 5,
    "count": 20,
    "total_count": 95
  }
}
```

### Error responses

```json
{
  "success": false,
  "errors": [
    {
      "code": 1003,
      "message": "Invalid zone identifier"
    }
  ],
  "messages": [],
  "result": null
}
```

## Pagination

### Manual pagination

```typescript
let page = 1;
const perPage = 100;
const allRecords = [];

while (true) {
  const response = await client.dns.records.list({
    zone_id: 'zone-id',
    per_page: perPage,
    page: page,
  });
  
  allRecords.push(...response.result);
  
  if (page >= response.result_info.total_pages) break;
  page++;
}
```

### Auto-pagination

```typescript
// SDK handles pagination automatically
const allRecords = [];
for await (const record of client.dns.records.list({
  zone_id: 'zone-id',
})) {
  allRecords.push(record);
}
```

## Rate Limiting

**Limits**:
- Enterprise: 1200 requests/5 minutes per API token
- Non-Enterprise: Variable based on endpoint

**Headers**:
```
X-RateLimit-Limit: 1200
X-RateLimit-Remaining: 1150
X-RateLimit-Reset: 1672531200
```

**Handling**:
```typescript
async function rateLimitedRequest<T>(
  fn: () => Promise<T>
): Promise<T> {
  try {
    return await fn();
  } catch (error) {
    if (error instanceof Cloudflare.RateLimitError) {
      const resetTime = error.headers['x-ratelimit-reset'];
      const waitMs = (resetTime * 1000) - Date.now();
      await new Promise(resolve => setTimeout(resolve, waitMs));
      return rateLimitedRequest(fn);
    }
    throw error;
  }
}
```

## Error Handling

### Common error codes

- `1001`: Invalid request
- `1003`: Invalid identifier
- `1004`: Access denied
- `1006`: Invalid email/token
- `1010`: Unauthorized
- `1012`: Invalid API token
- `9103`: Missing required parameter
- `9106`: Missing required fields

### SDK error handling

```typescript
try {
  await client.zones.get({ zone_id: 'invalid' });
} catch (error) {
  if (error instanceof Cloudflare.APIError) {
    console.error(`API Error ${error.status}:`, error.message);
    console.error('Errors:', error.errors);
    
    // Specific error handling
    if (error.status === 403) {
      // Permission denied
    } else if (error.status === 404) {
      // Resource not found
    } else if (error.status === 429) {
      // Rate limited
    }
  }
  throw error;
}
```

## Environment Variables

### Shell (bash/zsh)

```bash
# Current session
export CLOUDFLARE_API_TOKEN='token'
export ZONE_ID='zone-id'
export ACCOUNT_ID='account-id'

# Persistent (add to ~/.bashrc or ~/.zshrc)
echo 'export CLOUDFLARE_API_TOKEN="token"' >> ~/.bashrc
```

### PowerShell

```powershell
# Current session
$Env:CLOUDFLARE_API_TOKEN='token'

# Persistent (user)
[Environment]::SetEnvironmentVariable(
  "CLOUDFLARE_API_TOKEN",
  "token",
  "User"
)
```

### Node.js (.env file)

```bash
CLOUDFLARE_API_TOKEN=token
CLOUDFLARE_ACCOUNT_ID=account-id
CLOUDFLARE_ZONE_ID=zone-id
```

```typescript
import 'dotenv/config';

const client = new Cloudflare({
  apiToken: process.env.CLOUDFLARE_API_TOKEN,
});
```

## Wrangler CLI Integration

Wrangler uses Cloudflare API internally:

```bash
# Configure authentication
wrangler login
# Or
export CLOUDFLARE_API_TOKEN='token'

# Common commands that use API
wrangler deploy              # Uploads worker via API
wrangler kv:key put          # KV operations
wrangler r2 bucket create    # R2 operations
wrangler d1 execute          # D1 operations
wrangler pages deploy        # Pages operations

# Get API configuration
wrangler whoami              # Shows authenticated user
```

## Best Practices

### Security

- **Never commit tokens**: Use environment variables
- **Minimal permissions**: Create scoped API tokens
- **Rotate tokens**: Regularly refresh tokens
- **Use token expiration**: Set expiry dates
- **Audit token usage**: Monitor API logs

### Performance

- **Batch operations**: Group related API calls
- **Use pagination wisely**: Don't fetch all data if unnecessary
- **Cache responses**: Store rarely-changing data locally
- **Parallel requests**: Use `Promise.all()` for independent operations
- **Handle rate limits**: Implement exponential backoff

### Code organization

```typescript
// Create reusable client instance
export const cfClient = new Cloudflare({
  apiToken: process.env.CLOUDFLARE_API_TOKEN,
});

// Wrap common operations
export async function getDNSRecords(zoneId: string, type?: string) {
  const records = [];
  for await (const record of cfClient.dns.records.list({
    zone_id: zoneId,
    ...(type && { type }),
  })) {
    records.push(record);
  }
  return records;
}

// Error handling wrapper
export async function withErrorHandling<T>(
  operation: () => Promise<T>
): Promise<T> {
  try {
    return await operation();
  } catch (error) {
    if (error instanceof Cloudflare.APIError) {
      console.error(`Cloudflare API error: ${error.message}`);
      throw new Error(`Cloudflare operation failed: ${error.message}`);
    }
    throw error;
  }
}
```

## Common Patterns

### Get zone ID by domain name

```typescript
async function getZoneId(domain: string): Promise<string> {
  const zones = await client.zones.list({ name: domain });
  if (zones.result.length === 0) {
    throw new Error(`Zone not found: ${domain}`);
  }
  return zones.result[0].id;
}
```

### Update multiple DNS records

```typescript
async function updateRecords(
  zoneId: string,
  updates: Array<{ name: string; content: string }>
): Promise<void> {
  const records = await getDNSRecords(zoneId);
  
  const promises = updates.map(async ({ name, content }) => {
    const record = records.find(r => r.name === name);
    if (record) {
      return client.dns.records.update(zoneId, record.id, { content });
    }
  });
  
  await Promise.all(promises);
}
```

### Check zone activation status

```typescript
async function waitForActivation(
  zoneId: string,
  maxAttempts = 30
): Promise<void> {
  for (let i = 0; i < maxAttempts; i++) {
    const zone = await client.zones.get({ zone_id: zoneId });
    
    if (zone.status === 'active') {
      return;
    }
    
    await new Promise(resolve => setTimeout(resolve, 10000)); // Wait 10s
  }
  
  throw new Error('Zone activation timeout');
}
```

### Purge cache with retry

```typescript
async function purgeCacheWithRetry(
  zoneId: string,
  files: string[],
  maxRetries = 3
): Promise<void> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await client.cache.purge.create({
        zone_id: zoneId,
        files,
      });
      return;
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
}
```

## Testing

### Mock client for tests

```typescript
import { vi } from 'vitest';
import Cloudflare from 'cloudflare';

const mockClient = {
  zones: {
    list: vi.fn().mockResolvedValue({
      result: [{ id: 'zone-1', name: 'example.com' }],
    }),
    get: vi.fn().mockResolvedValue({
      id: 'zone-1',
      name: 'example.com',
      status: 'active',
    }),
  },
} as unknown as Cloudflare;

// Use in tests
test('fetches zone', async () => {
  const zones = await mockClient.zones.list();
  expect(zones.result).toHaveLength(1);
});
```

## Documentation References

- **API Docs**: https://developers.cloudflare.com/api/
- **TypeScript SDK**: https://github.com/cloudflare/cloudflare-typescript
- **Python SDK**: https://github.com/cloudflare/cloudflare-python
- **Go SDK**: https://github.com/cloudflare/cloudflare-go
- **API Fundamentals**: https://developers.cloudflare.com/fundamentals/api/
- **API Token Guide**: https://developers.cloudflare.com/fundamentals/api/get-started/create-token/

## Quick Reference

### Most Common Operations

```typescript
// Auth
const client = new Cloudflare({ apiToken: process.env.CLOUDFLARE_API_TOKEN });

// List zones
await client.zones.list();

// Get zone
await client.zones.get({ zone_id: 'id' });

// DNS record CRUD
await client.dns.records.create({ zone_id, type, name, content });
await client.dns.records.list({ zone_id });
await client.dns.records.update(zone_id, record_id, { content });
await client.dns.records.delete(zone_id, record_id);

// Purge cache
await client.cache.purge.create({ zone_id, purge_everything: true });

// Update settings
await client.zones.settings.edit(zone_id, 'ssl', { value: 'full' });

// Workers
await client.workers.scripts.update(name, { account_id, script });

// KV
await client.kv.namespaces.values.update(account_id, ns_id, key, { value });

// R2 (use S3 SDK)
// D1
await client.d1.databases.query(account_id, db_id, { sql, params });
```
