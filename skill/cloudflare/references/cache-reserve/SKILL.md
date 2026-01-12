# Cloudflare Cache Reserve

**Expert guidance for implementing and optimizing Cloudflare Cache Reserve - persistent cache storage built on R2**

## Overview

Cache Reserve is Cloudflare's persistent, large-scale cache storage layer built on R2. It acts as the ultimate upper-tier cache, storing cacheable content for extended periods (30+ days) to maximize cache hits, reduce origin egress fees, and shield origins from repeated requests for long-tail content.

## Core Concepts

### What is Cache Reserve?

- **Persistent storage layer**: Built on R2, sits above tiered cache hierarchy
- **Long-term retention**: 30-day default retention, extended on each access
- **Automatic operation**: Works seamlessly with existing CDN, no code changes required
- **Origin shielding**: Dramatically reduces origin egress by serving cached content longer
- **Usage-based pricing**: Pay only for storage + read/write operations

### Cache Hierarchy

```
Visitor Request
    ↓
Lower-Tier Cache (closest to visitor)
    ↓ (on miss)
Upper-Tier Cache (closest to origin)
    ↓ (on miss)
Cache Reserve (R2 persistent storage)
    ↓ (on miss)
Origin Server
```

### How It Works

1. **On cache miss**: Content fetched from origin → written to Cache Reserve + edge caches simultaneously
2. **On edge eviction**: Content may be evicted from edge cache but remains in Cache Reserve
3. **On subsequent request**: If edge cache misses but Cache Reserve hits → content restored to edge caches
4. **Retention**: Assets remain in Cache Reserve for 30 days since last access (configurable via TTL)

## Asset Eligibility

Cache Reserve only stores assets meeting **ALL** criteria:

```typescript
interface CacheReserveEligibility {
  // Must be cacheable per Cloudflare's standard rules
  cacheable: boolean;
  
  // Minimum 10-hour TTL (36000 seconds) via:
  // - Cache-Control headers
  // - Edge Cache TTL
  // - Cache Rules
  // - Cache TTL By Status
  minimumTTL: 36000;
  
  // Must have Content-Length header
  hasContentLength: boolean;
  
  // Original files only (not transformed images)
  isOriginalAsset: boolean;
}
```

### Not Eligible

- Assets with TTL < 10 hours
- Responses without `Content-Length` header
- Image transformation variants (original images are eligible)
- Responses with `Set-Cookie` headers
- Responses with `Vary: *` header
- Assets from R2 public buckets on same zone
- O2O (Orange-to-Orange) setup requests

## Configuration

### Dashboard Setup

**Minimum steps to enable:**

```bash
# Navigate to dashboard
https://dash.cloudflare.com/caching/cache-reserve

# Click "Enable Storage Sync" or "Purchase" button
```

**Prerequisites:**
- Paid Cache Reserve plan required
- Tiered Cache strongly recommended (Cache Reserve checks for this)

### API Configuration

```typescript
// Enable Cache Reserve
const enableCacheReserve = async (zoneId: string, apiToken: string) => {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/cache/cache_reserve`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        value: 'on'
      })
    }
  );
  
  return await response.json();
};

// Check Cache Reserve status
const getCacheReserveStatus = async (zoneId: string, apiToken: string) => {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/cache/cache_reserve`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
      }
    }
  );
  
  return await response.json();
};

// Disable Cache Reserve (to clear data, must be off first)
const disableCacheReserve = async (zoneId: string, apiToken: string) => {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/cache/cache_reserve`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        value: 'off'
      })
    }
  );
  
  return await response.json();
};
```

### Required API Token Permissions

- `Zone Settings Read`
- `Zone Settings Write`
- `Zone Read`
- `Zone Write`

## Cache Rules Integration

Control Cache Reserve eligibility via Cache Rules:

```typescript
// Example Cache Rule with Cache Reserve configuration
interface CacheRuleWithReserve {
  action: 'set_cache_settings';
  action_parameters: {
    // Enable/disable Cache Reserve for matching requests
    cache_reserve?: {
      // Whether to write to Cache Reserve
      eligible: boolean;
      
      // Minimum TTL for Cache Reserve (must be >= 10 hours)
      minimum_file_ttl?: number;
    };
    
    // Set TTL to meet 10-hour minimum
    edge_ttl?: {
      mode: 'override_origin';
      default: number; // e.g., 36000 for 10 hours
    };
    
    // Ensure content is cacheable
    cache: boolean;
  };
  
  // Match conditions
  expression: string;
}
```

### Example Cache Rules

```typescript
// Enable Cache Reserve for static assets with long TTL
const staticAssetRule = {
  action: 'set_cache_settings',
  action_parameters: {
    cache_reserve: {
      eligible: true,
      minimum_file_ttl: 86400 // 24 hours
    },
    edge_ttl: {
      mode: 'override_origin',
      default: 86400
    },
    cache: true
  },
  expression: '(http.request.uri.path matches "\\.(jpg|jpeg|png|gif|webp|pdf|zip)$")'
};

// Disable Cache Reserve for frequently updated content
const dynamicContentRule = {
  action: 'set_cache_settings',
  action_parameters: {
    cache_reserve: {
      eligible: false
    }
  },
  expression: '(http.request.uri.path matches "^/api/")'
};

// Cache Reserve for specific origin with minimum 12-hour TTL
const specificOriginRule = {
  action: 'set_cache_settings',
  action_parameters: {
    cache_reserve: {
      eligible: true,
      minimum_file_ttl: 43200 // 12 hours
    },
    edge_ttl: {
      mode: 'override_origin',
      default: 43200
    },
    cache: true
  },
  expression: '(http.host eq "cdn.example.com")'
};
```

### Creating Rules via API

```typescript
const createCacheRule = async (
  zoneId: string,
  apiToken: string,
  rule: CacheRuleWithReserve
) => {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/http_request_cache_settings/entrypoint`,
    {
      method: 'PUT',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        rules: [rule]
      })
    }
  );
  
  return await response.json();
};
```

## Workers Integration

Cache Reserve works automatically with Workers Cache API, but requires specific patterns:

### Standard Fetch (Recommended)

```typescript
// Cache Reserve works automatically via standard fetch
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Standard fetch uses Cache Reserve automatically
    const response = await fetch(request);
    
    return response;
  }
};
```

### Cache API Limitations

**IMPORTANT**: `cache.put()` is **NOT compatible** with Cache Reserve or Tiered Cache.

```typescript
// ❌ WRONG: cache.put() bypasses Cache Reserve
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const cache = caches.default;
    
    // This will NOT write to Cache Reserve
    let response = await cache.match(request);
    if (!response) {
      response = await fetch(request);
      await cache.put(request, response.clone()); // Bypasses Cache Reserve!
    }
    
    return response;
  }
};

// ✅ CORRECT: Use standard fetch for Cache Reserve compatibility
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Let Cloudflare's standard caching handle it
    // Cache Reserve works automatically
    return await fetch(request);
  }
};

// ✅ CORRECT: Use Cache API only for custom cache namespaces
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const customCache = await caches.open('my-custom-cache');
    
    let response = await customCache.match(request);
    if (!response) {
      response = await fetch(request);
      // Custom cache doesn't interfere with default caching
      await customCache.put(request, response.clone());
    }
    
    return response;
  }
};
```

### Ensuring Cache Reserve Eligibility in Workers

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const response = await fetch(request);
    
    // Ensure response meets Cache Reserve requirements
    if (response.ok) {
      const headers = new Headers(response.headers);
      
      // Set minimum 10-hour cache
      headers.set('Cache-Control', 'public, max-age=36000');
      
      // Remove Set-Cookie if present (prevents caching)
      headers.delete('Set-Cookie');
      
      // Ensure Content-Length is present
      if (!headers.has('Content-Length')) {
        const blob = await response.blob();
        headers.set('Content-Length', blob.size.toString());
        
        return new Response(blob, {
          status: response.status,
          statusText: response.statusText,
          headers
        });
      }
      
      return new Response(response.body, {
        status: response.status,
        statusText: response.statusText,
        headers
      });
    }
    
    return response;
  }
};
```

### Hostname Best Practices

```typescript
// ✅ CORRECT: Use Worker's hostname for efficient caching
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Keep the Worker's hostname
    const response = await fetch(request);
    return response;
  }
};

// ❌ WRONG: Overriding hostname causes unnecessary DNS lookups
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    url.hostname = 'different-host.com'; // Avoid this!
    
    const response = await fetch(url.toString());
    return response;
  }
};
```

## Purging and Cache Management

### Purge by URL (Instant)

```typescript
// Purge specific URL from Cache Reserve immediately
const purgeCacheReserveByURL = async (
  zoneId: string,
  apiToken: string,
  urls: string[]
) => {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/purge_cache`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        files: urls
      })
    }
  );
  
  return await response.json();
};

// Example usage
await purgeCacheReserveByURL(
  'zone123',
  'token456',
  [
    'https://example.com/image.jpg',
    'https://example.com/video.mp4'
  ]
);
```

### Purge by Tag/Host/Prefix (Revalidation)

```typescript
// Purge by cache tag - forces revalidation, not immediate removal
const purgeCacheReserveByTag = async (
  zoneId: string,
  apiToken: string,
  tags: string[]
) => {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/purge_cache`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        tags: tags
      })
    }
  );
  
  return await response.json();
};

// Note: Assets remain in Cache Reserve but will revalidate on next request
// Storage costs continue until retention TTL expires
```

**Important purge behavior differences:**

- **Purge by URL**: Immediately removes from Cache Reserve + edge cache
- **Purge by tag/host/prefix**: Forces revalidation but assets remain in storage
- **Storage costs**: Continue for tag/host/prefix purges until TTL expiry

### Clear All Cache Reserve Data

```typescript
// Clear ALL Cache Reserve data (requires Cache Reserve to be OFF)
const clearAllCacheReserve = async (zoneId: string, apiToken: string) => {
  // Step 1: Verify Cache Reserve is off
  const statusResponse = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/cache/cache_reserve`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
      }
    }
  );
  
  const status = await statusResponse.json();
  
  if (status.result.value !== 'off') {
    throw new Error('Cache Reserve must be OFF before clearing data');
  }
  
  // Step 2: Clear Cache Reserve data
  const clearResponse = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/cache/cache_reserve_clear`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
      }
    }
  );
  
  return await clearResponse.json();
};

// Check clear status
const getClearStatus = async (zoneId: string, apiToken: string) => {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/cache/cache_reserve_clear`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
      }
    }
  );
  
  return await response.json();
  // Returns: { state: "In-progress" | "Completed", start_ts: "..." }
};
```

**Clear process:**
1. Disable Cache Reserve (`value: 'off'`)
2. Call cache_reserve_clear endpoint
3. Deletion takes up to 24 hours
4. Re-enable Cache Reserve when ready

## Monitoring and Analytics

### Dashboard Analytics

Available in Cache Reserve section:

```typescript
interface CacheReserveAnalytics {
  // Estimated bandwidth saved from origin
  egressSavings: {
    bytes: number;
    percentage: number;
  };
  
  // Total requests served by Cache Reserve
  requestsServed: {
    total: number;
    hits: number;
    misses: number;
  };
  
  // Storage metrics
  storage: {
    currentDataStored: number; // GB
    aggregateUsage: number;    // GB-months
  };
  
  // Operations over time
  operations: {
    classA: number; // Writes
    classB: number; // Reads
  };
}
```

### Logpush Integration

```typescript
// Enable Logpush to track Cache Reserve usage
interface HTTPRequestLog {
  // Standard fields
  ClientRequestHost: string;
  ClientRequestPath: string;
  EdgeResponseStatus: number;
  CacheResponseStatus: 'hit' | 'miss' | 'expired' | 'revalidated';
  
  // Cache Reserve specific
  CacheReserveUsed: boolean; // true if served from Cache Reserve
  
  // Additional useful fields
  EdgeStartTimestamp: number;
  EdgeEndTimestamp: number;
  EdgeResponseBytes: number;
}

// Query example for Cache Reserve hits
const queryCacheReserveHits = `
  SELECT 
    ClientRequestHost,
    ClientRequestPath,
    COUNT(*) as requests,
    SUM(EdgeResponseBytes) as total_bytes
  FROM http_requests
  WHERE CacheReserveUsed = true
  GROUP BY ClientRequestHost, ClientRequestPath
  ORDER BY requests DESC
`;
```

### Custom Monitoring

```typescript
// Track Cache Reserve effectiveness
const monitorCacheReserve = async (zoneId: string, apiToken: string) => {
  const analytics = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/analytics/dashboard`,
    {
      headers: {
        'Authorization': `Bearer ${apiToken}`,
      }
    }
  );
  
  const data = await analytics.json();
  
  return {
    cacheHitRatio: data.result.totals.cacheHitRatio,
    bandwidthSaved: data.result.totals.bandwidthSaved,
    requests: {
      cached: data.result.totals.requests.cached,
      uncached: data.result.totals.requests.uncached
    }
  };
};
```

## Pricing and Cost Optimization

### Pricing Structure

```typescript
interface CacheReservePricing {
  storage: {
    rate: 0.015;        // $0.015 per GB-month
    unit: 'GB-month';
  };
  
  operations: {
    classA: {
      rate: 4.50;       // $4.50 per million writes
      description: 'Writes (cache misses from origin)';
    };
    classB: {
      rate: 0.36;       // $0.36 per million reads
      description: 'Reads (serving from Cache Reserve)';
    };
  };
}

// Typical operation costs:
// - Cache miss: 1 Class A + 1 Class B operation
// - Cache hit: 1 Class B operation
// - Assets > 1GB: Proportionally more operations
```

### Cost Calculation Examples

```typescript
// Example 1: 1,000 assets × 1GB each, read 1,000 times
const example1 = {
  assets: 1000,
  sizeGB: 1,
  readsPerAsset: 1000,
  writesPerAsset: 1,
  
  calculate() {
    const classB = (this.assets * this.readsPerAsset) / 1_000_000 * 0.36;
    const classA = (this.assets * this.writesPerAsset) / 1_000_000 * 4.50;
    const storage = (this.assets * this.sizeGB) * 0.015;
    
    return {
      classB: classB,      // $0.36
      classA: classA,      // $0.0045 (rounded to $4.50 minimum)
      storage: storage,    // $15.00
      total: classB + classA + storage // $19.86
    };
  }
};

// Example 2: 1M assets × 1MB each, rewritten daily, read 2× daily
const example2 = {
  assets: 1_000_000,
  sizeMB: 1,
  readsPerDay: 2,
  writesPerDay: 1,
  days: 30,
  
  calculate() {
    const totalReads = this.assets * this.readsPerDay * this.days;
    const totalWrites = this.assets * this.writesPerDay * this.days;
    
    const classB = (totalReads / 1_000_000) * 0.36;
    const classA = (totalWrites / 1_000_000) * 4.50;
    const storage = (this.assets * this.sizeMB / 1024) * 0.015;
    
    return {
      classB: classB,      // $21.60
      classA: classA,      // $135.00
      storage: storage,    // $15.00
      total: classB + classA + storage // $171.60
    };
  }
};
```

### Cost Optimization Strategies

```typescript
// 1. Enable Tiered Cache (reduces Cache Reserve operations)
// Fewer Cache Reserve reads = lower Class B costs

// 2. Set appropriate TTLs
const optimizeTTL = {
  // Too short: More Class A operations (origin fetches)
  tooShort: 3600,  // 1 hour - not eligible for Cache Reserve
  
  // Optimal: Balance freshness vs. cost
  optimal: 86400,  // 24 hours - reduces rewrites
  
  // Too long: May cache stale content
  tooLong: 2592000 // 30 days - use cautiously
};

// 3. Use selective Cache Rules
const selectiveCaching = {
  // Only cache high-value, stable assets
  eligiblePatterns: [
    /\.(jpg|jpeg|png|gif|webp|svg)$/,  // Images
    /\.(mp4|webm|mp3|wav)$/,            // Media
    /\.(pdf|zip|tar\.gz)$/,             // Documents/archives
    /\.(woff|woff2|ttf|eot)$/,          // Fonts
  ],
  
  // Exclude frequently changing content
  excludePatterns: [
    /^\/api\//,           // API responses
    /^\/user\//,          // User-specific content
    /\.json$/,            // JSON data
  ]
};

// 4. Monitor and adjust
const monitorCosts = async () => {
  // Track operations per asset type
  // Identify high-cost, low-value assets
  // Adjust Cache Rules to exclude expensive assets
};
```

### Cost vs. Egress Savings

```typescript
// Calculate ROI: Origin egress cost vs. Cache Reserve cost
interface ROICalculation {
  originEgressCostPerGB: number;    // e.g., AWS: $0.09, GCP: $0.12
  assetSizeGB: number;
  requestsPerMonth: number;
  cacheMissRate: number;            // Without Cache Reserve
  cacheReserveMissRate: number;     // With Cache Reserve
  
  calculate() {
    // Without Cache Reserve
    const withoutCR = {
      originRequests: this.requestsPerMonth * this.cacheMissRate,
      egressGB: (this.requestsPerMonth * this.cacheMissRate) * this.assetSizeGB,
      cost: ((this.requestsPerMonth * this.cacheMissRate) * this.assetSizeGB) 
            * this.originEgressCostPerGB
    };
    
    // With Cache Reserve
    const withCR = {
      originRequests: this.requestsPerMonth * this.cacheReserveMissRate,
      egressGB: (this.requestsPerMonth * this.cacheReserveMissRate) * this.assetSizeGB,
      egressCost: ((this.requestsPerMonth * this.cacheReserveMissRate) * this.assetSizeGB)
                  * this.originEgressCostPerGB,
      
      // Cache Reserve costs
      classB: (this.requestsPerMonth / 1_000_000) * 0.36,
      classA: ((this.requestsPerMonth * this.cacheReserveMissRate) / 1_000_000) * 4.50,
      storage: this.assetSizeGB * 0.015,
      
      totalCRCost: function() {
        return this.classB + this.classA + this.storage + this.egressCost;
      }
    };
    
    return {
      without: withoutCR,
      with: withCR,
      savings: withoutCR.cost - withCR.totalCRCost(),
      roi: ((withoutCR.cost - withCR.totalCRCost()) / withCR.totalCRCost()) * 100
    };
  }
}

// Example: 100GB asset, 100K requests/month
const roi = {
  originEgressCostPerGB: 0.09,  // AWS pricing
  assetSizeGB: 100,
  requestsPerMonth: 100_000,
  cacheMissRate: 0.20,          // 20% miss rate without CR
  cacheReserveMissRate: 0.02,   // 2% miss rate with CR
} as ROICalculation;

// Typical savings: 50-80% reduction in origin egress
```

## Best Practices

### 1. Always Enable Tiered Cache

```typescript
// Cache Reserve is designed for use WITH Tiered Cache
const configuration = {
  tieredCache: 'enabled',        // Required for optimal performance
  cacheReserve: 'enabled',       // Works best with Tiered Cache
  
  hierarchy: [
    'Lower-Tier Cache (visitor)',
    'Upper-Tier Cache (origin region)',
    'Cache Reserve (persistent)',
    'Origin'
  ]
};
```

### 2. Set Appropriate Cache-Control Headers

```typescript
// Origin response headers for Cache Reserve eligibility
const originHeaders = {
  // Minimum 10 hours for Cache Reserve
  'Cache-Control': 'public, max-age=86400', // 24 hours
  
  // Required for eligibility
  'Content-Length': '1024000',
  
  // Optional: Cache tags for purging
  'Cache-Tag': 'images,product-123',
  
  // Optional: Support revalidation
  'ETag': '"abc123"',
  'Last-Modified': 'Wed, 21 Oct 2025 07:28:00 GMT',
  
  // Avoid: Prevents caching
  // 'Set-Cookie': 'session=xyz',  // Remove or use private directive
  // 'Vary': '*',                  // Not compatible
};
```

### 3. Use Cache Rules for Fine-Grained Control

```typescript
// Example: Different TTLs for different content types
const cacheRules = [
  {
    description: 'Long-term cache for immutable assets',
    expression: '(http.request.uri.path matches "^/static/.*\\.[a-f0-9]{8}\\.")',
    action_parameters: {
      cache_reserve: { eligible: true },
      edge_ttl: { mode: 'override_origin', default: 2592000 }, // 30 days
      cache: true
    }
  },
  {
    description: 'Moderate cache for regular images',
    expression: '(http.request.uri.path matches "\\.(jpg|png|webp)$")',
    action_parameters: {
      cache_reserve: { eligible: true },
      edge_ttl: { mode: 'override_origin', default: 86400 }, // 24 hours
      cache: true
    }
  },
  {
    description: 'Exclude API from Cache Reserve',
    expression: '(http.request.uri.path matches "^/api/")',
    action_parameters: {
      cache_reserve: { eligible: false },
      cache: false
    }
  }
];
```

### 4. Monitor Cache Reserve Usage

```typescript
// Regular monitoring script
const monitoringChecklist = {
  daily: [
    'Check egress savings percentage',
    'Review operation counts (Class A/B)',
    'Identify high-cost assets'
  ],
  
  weekly: [
    'Analyze cache hit ratio trends',
    'Review storage growth',
    'Optimize Cache Rules based on usage'
  ],
  
  monthly: [
    'Calculate ROI vs. origin egress',
    'Review asset eligibility criteria',
    'Audit and purge unused assets'
  ]
};
```

### 5. Handle Set-Cookie Appropriately

```typescript
// Origin response with Set-Cookie
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const response = await fetch(request);
    
    // Option 1: Remove Set-Cookie to enable caching
    if (shouldCache(request)) {
      const headers = new Headers(response.headers);
      headers.delete('Set-Cookie');
      
      return new Response(response.body, {
        status: response.status,
        headers
      });
    }
    
    // Option 2: Use Cache-Control to cache without Set-Cookie
    if (shouldCacheWithCookie(request)) {
      const headers = new Headers(response.headers);
      headers.set('Cache-Control', 'public, max-age=86400, private=Set-Cookie');
      
      return new Response(response.body, {
        status: response.status,
        headers
      });
    }
    
    return response;
  }
};
```

### 6. Compression Considerations

```typescript
// Cache Reserve requests uncompressed content from origin
// This differs from standard CDN behavior

const compressionStrategy = {
  origin: {
    // Origin should serve uncompressed to Cache Reserve
    // Cloudflare compresses for visitor delivery
    behavior: 'uncompressed',
    
    // Do NOT add Accept-Encoding: gzip to origin requests
    note: 'Cache Reserve does not include Accept-Encoding header'
  },
  
  edge: {
    // Edge automatically compresses for visitors
    behavior: 'automatic compression',
    
    // Supports: gzip, brotli, zstd
    compression: ['br', 'gzip']
  }
};
```

### 7. Large File Handling

```typescript
// Assets > 1GB incur proportionally more operations
const largeFileStrategy = {
  // Consider chunking or segmentation
  maxSize: 1024 * 1024 * 1024, // 1GB
  
  approach: {
    // Option 1: Use R2 directly for very large files
    veryLarge: {
      threshold: '> 5GB',
      solution: 'Store directly in R2, bypass Cache Reserve'
    },
    
    // Option 2: Use chunked delivery
    large: {
      threshold: '1GB - 5GB',
      solution: 'Segment files, serve via Range requests'
    },
    
    // Option 3: Standard Cache Reserve
    moderate: {
      threshold: '< 1GB',
      solution: 'Standard Cache Reserve works well'
    }
  }
};
```

## Common Issues and Solutions

### Issue: Assets Not Being Cached in Cache Reserve

**Diagnostics:**

```typescript
const debugEligibility = {
  checks: [
    'Verify asset is cacheable (cf-cache-status header)',
    'Check TTL >= 10 hours',
    'Confirm Content-Length header present',
    'Review Cache Rules configuration',
    'Check for Set-Cookie or Vary: * headers'
  ],
  
  tools: [
    'curl -I https://example.com/asset.jpg',
    'Check cf-cache-status header',
    'Review Cloudflare Trace output',
    'Check Logpush CacheReserveUsed field'
  ]
};
```

**Solutions:**

```typescript
// 1. Ensure minimum TTL
// Origin:
response.headers.set('Cache-Control', 'public, max-age=36000'); // 10+ hours

// Or via Cache Rule:
const rule = {
  action_parameters: {
    edge_ttl: { mode: 'override_origin', default: 36000 }
  }
};

// 2. Add Content-Length
response.headers.set('Content-Length', bodySize.toString());

// 3. Remove blocking headers
response.headers.delete('Set-Cookie');
response.headers.set('Vary', 'Accept-Encoding'); // Not *
```

### Issue: High Class A Operations Costs

**Cause**: Frequent cache misses, short TTLs, or frequent revalidation

**Solutions:**

```typescript
// 1. Increase TTL for stable content
const optimizedTTL = {
  before: 3600,  // 1 hour (not eligible)
  after: 86400   // 24 hours (eligible + fewer rewrites)
};

// 2. Enable Tiered Cache
// Reduces direct Cache Reserve misses

// 3. Review and extend retention
// Longer TTLs = fewer origin fetches = lower Class A costs

// 4. Use stale-while-revalidate (via fetch, not cache.put)
response.headers.set(
  'Cache-Control',
  'public, max-age=86400, stale-while-revalidate=86400'
);
```

### Issue: Purge Not Working as Expected

**Understanding purge behavior:**

```typescript
const purgeBehavior = {
  byURL: {
    cacheReserve: 'Immediately removed',
    edgeCache: 'Immediately removed',
    cost: 'Free'
  },
  
  byTag: {
    cacheReserve: 'Revalidation triggered, NOT removed',
    edgeCache: 'Immediately removed',
    storage: 'Continues until TTL expires',
    cost: 'Storage costs continue'
  },
  
  byHost: {
    cacheReserve: 'Revalidation triggered, NOT removed',
    edgeCache: 'Immediately removed',
    storage: 'Continues until TTL expires',
    cost: 'Storage costs continue'
  }
};

// Solution: Use purge by URL for immediate removal
await purgeByURL(['https://example.com/asset.jpg']);

// Or: Disable + clear for complete removal
await disableCacheReserve(zoneId, token);
await clearAllCacheReserve(zoneId, token);
```

### Issue: Cache Reserve Disabled But Can't Clear

**Error**: "Cache Reserve must be OFF before clearing data"

**Solution:**

```typescript
const clearProcess = async (zoneId: string, token: string) => {
  // Step 1: Check current state
  const status = await getCacheReserveStatus(zoneId, token);
  console.log('Current state:', status.result.value);
  
  // Step 2: Disable if enabled
  if (status.result.value !== 'off') {
    await disableCacheReserve(zoneId, token);
    console.log('Cache Reserve disabled');
  }
  
  // Step 3: Wait briefly for propagation
  await new Promise(resolve => setTimeout(resolve, 5000));
  
  // Step 4: Clear data
  const clearResult = await clearAllCacheReserve(zoneId, token);
  console.log('Clear initiated:', clearResult);
  
  // Step 5: Monitor clear progress
  let clearStatus;
  do {
    await new Promise(resolve => setTimeout(resolve, 60000)); // Check every minute
    clearStatus = await getClearStatus(zoneId, token);
    console.log('Clear status:', clearStatus.result.state);
  } while (clearStatus.result.state === 'In-progress');
  
  console.log('Clear completed');
};
```

## Architecture Patterns

### Pattern 1: Multi-Tier Caching Strategy

```typescript
// Optimal architecture: Tiered Cache + Cache Reserve
interface CachingArchitecture {
  layers: {
    l1: 'Lower-Tier Cache (visitor region)';
    l2: 'Upper-Tier Cache (major region)';
    l3: 'Cache Reserve (R2 persistent)';
    origin: 'Origin Server';
  };
  
  flow: {
    hit: 'L1 → Visitor',
    l1Miss: 'L1 → L2 → Visitor (L1 backfill)',
    l2Miss: 'L1 → L2 → L3 → Visitor (L1+L2 backfill)',
    l3Miss: 'L1 → L2 → L3 → Origin → Visitor (all layers backfill)'
  };
}
```

### Pattern 2: Immutable Asset Optimization

```typescript
// Use content hashing for maximum cache efficiency
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    
    // Check for immutable assets (content-hashed filenames)
    const isImmutable = /\.[a-f0-9]{8,}\.(js|css|jpg|png|woff2)$/.test(url.pathname);
    
    if (isImmutable) {
      const response = await fetch(request);
      
      // Set very long TTL for immutable assets
      const headers = new Headers(response.headers);
      headers.set('Cache-Control', 'public, max-age=31536000, immutable'); // 1 year
      
      return new Response(response.body, {
        status: response.status,
        headers
      });
    }
    
    return fetch(request);
  }
};
```

### Pattern 3: Conditional Request Optimization

```typescript
// Leverage ETags and Last-Modified for efficient revalidation
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const response = await fetch(request);
    
    // Ensure revalidation headers are present
    if (response.ok && !response.headers.has('ETag')) {
      const body = await response.arrayBuffer();
      const hash = await crypto.subtle.digest('SHA-256', body);
      const etag = Array.from(new Uint8Array(hash))
        .map(b => b.toString(16).padStart(2, '0'))
        .join('');
      
      const headers = new Headers(response.headers);
      headers.set('ETag', `"${etag}"`);
      headers.set('Cache-Control', 'public, max-age=86400, must-revalidate');
      
      return new Response(body, {
        status: response.status,
        headers
      });
    }
    
    return response;
  }
};
```

### Pattern 4: Smart Shield Integration

**Note**: Cache Reserve functionality is now part of Smart Shield for Enhanced tier customers.

```typescript
interface SmartShieldIntegration {
  cacheReserve: {
    status: 'Integrated into Smart Shield',
    tier: 'Enhanced',
    features: [
      'Persistent cache via Cache Reserve',
      'Tiered Cache optimization',
      'Origin shielding',
      'Advanced purging'
    ]
  };
  
  migration: {
    from: 'Standalone Cache Reserve',
    to: 'Smart Shield Enhanced',
    benefits: [
      'Unified origin protection',
      'Enhanced analytics',
      'Simplified configuration',
      'Additional security features'
    ]
  };
}
```

## Wrangler Integration

Cache Reserve works automatically with Workers deployed via Wrangler. No special configuration needed.

### wrangler.toml Configuration

```toml
name = "cache-reserve-worker"
main = "src/index.ts"
compatibility_date = "2025-01-11"

# Cache Reserve works automatically with standard routes
routes = [
  { pattern = "example.com/*", zone_name = "example.com" }
]

# Or with custom domains
[env.production]
routes = [
  { pattern = "cdn.example.com/*", custom_domain = true }
]

# No special Cache Reserve configuration needed in wrangler.toml
# Enable via Dashboard or API
```

### Development and Testing

```bash
# Local development (Cache Reserve not active locally)
npx wrangler dev

# Deploy to production (Cache Reserve active if enabled for zone)
npx wrangler deploy

# View logs (including cache behavior)
npx wrangler tail

# Check deployment
npx wrangler deployments list
```

### Testing Cache Reserve

```typescript
// Add debug headers in development
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const response = await fetch(request);
    
    // In development, add cache debugging
    if (env.ENVIRONMENT === 'development') {
      const headers = new Headers(response.headers);
      headers.set('X-Cache-Status', response.headers.get('cf-cache-status') || 'unknown');
      headers.set('X-Cache-Reserve-Eligible', 
        shouldBeInCacheReserve(response) ? 'yes' : 'no'
      );
      
      return new Response(response.body, {
        status: response.status,
        headers
      });
    }
    
    return response;
  }
};

function shouldBeInCacheReserve(response: Response): boolean {
  const cacheControl = response.headers.get('cache-control') || '';
  const maxAge = parseInt(cacheControl.match(/max-age=(\d+)/)?.[1] || '0');
  const hasContentLength = response.headers.has('content-length');
  const hasSetCookie = response.headers.has('set-cookie');
  
  return maxAge >= 36000 && hasContentLength && !hasSetCookie;
}
```

## Troubleshooting Guide

### Diagnostic Commands

```bash
# Check Cache Reserve status
curl -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/cache/cache_reserve" \
  -H "Authorization: Bearer $API_TOKEN" | jq

# Check asset cache status
curl -I https://example.com/asset.jpg | grep -i cache

# View Cloudflare Trace
curl -H "CF-RAY: trace" https://example.com/asset.jpg

# Check Logpush for Cache Reserve usage
# (requires Logpush configured with CacheReserveUsed field)
```

### Common Header Patterns

```typescript
// Successful Cache Reserve eligible response
const successHeaders = {
  'cf-cache-status': 'HIT',           // Served from cache
  'cache-control': 'public, max-age=86400',
  'content-length': '1024000',
  'etag': '"abc123"',
  // CacheReserveUsed: true (in logs)
};

// Not eligible: TTL too short
const tooShortTTL = {
  'cf-cache-status': 'MISS',
  'cache-control': 'public, max-age=3600', // < 10 hours
  'content-length': '1024000',
};

// Not eligible: Missing Content-Length
const missingLength = {
  'cf-cache-status': 'MISS',
  'cache-control': 'public, max-age=86400',
  // Missing: content-length
};

// Not eligible: Has Set-Cookie
const hasSetCookie = {
  'cf-cache-status': 'DYNAMIC',
  'cache-control': 'public, max-age=86400',
  'content-length': '1024000',
  'set-cookie': 'session=xyz', // Blocks caching
};
```

## Quick Reference

### Minimum Requirements Checklist

- [ ] Paid Cache Reserve plan active
- [ ] Tiered Cache enabled (strongly recommended)
- [ ] Assets cacheable per standard rules
- [ ] TTL >= 10 hours (36000 seconds)
- [ ] Content-Length header present
- [ ] No Set-Cookie header (or using private directive)
- [ ] No Vary: * header
- [ ] Not an image transformation variant

### API Endpoints

```typescript
const endpoints = {
  status: 'GET /zones/:zone_id/cache/cache_reserve',
  enable: 'PATCH /zones/:zone_id/cache/cache_reserve',
  disable: 'PATCH /zones/:zone_id/cache/cache_reserve',
  clear: 'POST /zones/:zone_id/cache/cache_reserve_clear',
  clearStatus: 'GET /zones/:zone_id/cache/cache_reserve_clear',
  purge: 'POST /zones/:zone_id/purge_cache',
  cacheRules: 'PUT /zones/:zone_id/rulesets/phases/http_request_cache_settings/entrypoint'
};
```

### Key Limits

```typescript
const limits = {
  minTTL: 36000,              // 10 hours in seconds
  retentionDefault: 2592000,  // 30 days in seconds
  maxFileSize: Infinity,      // Same as R2 limits
  purgeClearTime: 86400000,   // Up to 24 hours in milliseconds
};
```

## Additional Resources

- **Official Docs**: https://developers.cloudflare.com/cache/advanced-configuration/cache-reserve/
- **API Reference**: https://developers.cloudflare.com/api/resources/cache/subresources/cache_reserve/
- **Cache Rules**: https://developers.cloudflare.com/cache/how-to/cache-rules/
- **Workers Cache API**: https://developers.cloudflare.com/workers/runtime-apis/cache/
- **R2 Documentation**: https://developers.cloudflare.com/r2/
- **Smart Shield**: https://developers.cloudflare.com/smart-shield/
- **Tiered Cache**: https://developers.cloudflare.com/cache/how-to/tiered-cache/

## Summary

Cache Reserve provides persistent, long-term caching using R2 storage, positioned as the ultimate upper-tier in Cloudflare's cache hierarchy. Enable it with one click, optimize with Cache Rules, monitor via analytics, and save significantly on origin egress costs while improving cache hit ratios.

**Key Takeaway**: Cache Reserve is most effective when used with Tiered Cache, for assets with TTL >= 10 hours, and for content that benefits from long-term storage to reduce origin load.
