# Cloudflare Argo Smart Routing Skill

## Overview

Cloudflare Argo Smart Routing is a performance optimization service that detects real-time network issues and routes web traffic across the most efficient network path. This skill provides comprehensive guidance for implementing, configuring, and managing Argo Smart Routing via API, CLI tools, and TypeScript/JavaScript SDKs.

**Key Capabilities:**
- Automatic intelligent routing to avoid network congestion
- Real-time path optimization across Cloudflare's global network
- Reduced time-to-first-byte (TTFB) for origin requests
- Analytics and performance metrics
- Integration with Tiered Cache for enhanced performance

**Important:** Argo Smart Routing is now part of Smart Shield, Cloudflare's origin server safeguard service.

## When to Use Argo Smart Routing

### Ideal Use Cases
1. **Global Applications** - Users distributed worldwide, especially far from origin
2. **High-Performance Requirements** - Applications requiring minimal latency
3. **Origin Optimization** - Reducing load on origin servers by optimizing network transit
4. **E-commerce & SaaS** - Business-critical applications where performance impacts revenue
5. **Media Delivery** - Large file delivery and streaming services
6. **API Services** - Low-latency API endpoints serving global clients

### When NOT to Use
- Applications with users primarily in single region close to origin
- Development/testing environments (due to usage-based billing)
- Applications with minimal traffic where cost may exceed benefit

## Architecture & How It Works

### Core Concepts

**Network Path Optimization:**
- Cloudflare monitors real-time network conditions across its global network
- Dynamically selects optimal routes between edge and origin
- Bypasses congested internet routes
- Uses Cloudflare's private backbone when beneficial

**Time-to-First-Byte (TTFB) Optimization:**
- Measures delay between Cloudflare sending request to origin and receiving first byte
- Argo minimizes this delay by optimizing network transit time
- Does not charge for DDoS mitigation, WAF, or other security-blocked traffic

**Integration with Tiered Cache:**
- Tiered Cache divides data centers into hierarchy (lower-tier and upper-tier)
- Concentrates origin requests through fewer data centers
- Reduces bandwidth and connection overhead
- Argo + Tiered Cache = optimal routing + caching strategy

### Billing Model
- Usage-based billing per GB of traffic
- Recommended: Set up usage-based billing notifications
- Enterprise customers may access as non-contract preview service

## API Reference

### Base Endpoint
```
https://api.cloudflare.com/client/v4
```

### Authentication
Use API tokens with Zone:Argo Smart Routing:Edit permissions:

```bash
# Headers required
X-Auth-Email: user@example.com
Authorization: Bearer YOUR_API_TOKEN
```

### Get Argo Smart Routing Status

**Endpoint:** `GET /zones/{zone_id}/argo/smart_routing`

**Description:** Retrieves current Argo Smart Routing enablement status.

**cURL Example:**
```bash
curl -X GET "https://api.cloudflare.com/client/v4/zones/{zone_id}/argo/smart_routing" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json"
```

**Response:**
```json
{
  "result": {
    "id": "smart_routing",
    "value": "on",
    "editable": true,
    "modified_on": "2024-01-11T12:00:00Z"
  },
  "success": true,
  "errors": [],
  "messages": []
}
```

**TypeScript SDK Example:**
```typescript
import Cloudflare from 'cloudflare';

const client = new Cloudflare({
  apiToken: process.env.CLOUDFLARE_API_TOKEN,
});

const status = await client.argo.smartRouting.get({
  zone_id: '023e105f4ecef8ad9ca31a8372d0c353',
});

console.log(`Argo Smart Routing: ${status.value}`);
```

### Enable/Disable Argo Smart Routing

**Endpoint:** `PATCH /zones/{zone_id}/argo/smart_routing`

**Description:** Configures Argo Smart Routing enablement. Requires existing billing profile.

**Request Body:**
```json
{
  "value": "on"  // or "off"
}
```

**cURL Example:**
```bash
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/{zone_id}/argo/smart_routing" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "on"}'
```

**TypeScript SDK Example:**
```typescript
const response = await client.argo.smartRouting.edit({
  zone_id: '023e105f4ecef8ad9ca31a8372d0c353',
  value: 'on',
});

console.log(`Argo Smart Routing enabled: ${response.value === 'on'}`);
```

**JavaScript SDK (Node.js) Complete Example:**
```javascript
const Cloudflare = require('cloudflare');

async function enableArgoSmartRouting(zoneId) {
  const client = new Cloudflare({
    apiToken: process.env.CLOUDFLARE_API_TOKEN,
  });

  try {
    // Check current status
    const currentStatus = await client.argo.smartRouting.get({
      zone_id: zoneId,
    });

    if (currentStatus.value === 'on') {
      console.log('Argo Smart Routing already enabled');
      return currentStatus;
    }

    // Enable Argo Smart Routing
    const result = await client.argo.smartRouting.edit({
      zone_id: zoneId,
      value: 'on',
    });

    console.log('Argo Smart Routing enabled successfully');
    return result;
  } catch (error) {
    if (error.message.includes('billing')) {
      console.error('Billing profile required. Set up billing first.');
    }
    throw error;
  }
}

// Usage
enableArgoSmartRouting('023e105f4ecef8ad9ca31a8372d0c353');
```

## Code Patterns & Best Practices

### Pattern 1: Conditional Enablement with Validation

```typescript
interface ArgoConfig {
  zoneId: string;
  enableArgo: boolean;
  checkBilling?: boolean;
}

async function configureArgo(
  client: Cloudflare,
  config: ArgoConfig
): Promise<{ enabled: boolean; editable: boolean }> {
  const { zoneId, enableArgo, checkBilling = true } = config;

  // Get current status
  const status = await client.argo.smartRouting.get({
    zone_id: zoneId,
  });

  if (!status.editable) {
    throw new Error('Argo Smart Routing not editable for this zone');
  }

  // Check if change needed
  const targetValue = enableArgo ? 'on' : 'off';
  if (status.value === targetValue) {
    return { enabled: enableArgo, editable: status.editable };
  }

  // Apply change
  const result = await client.argo.smartRouting.edit({
    zone_id: zoneId,
    value: targetValue,
  });

  return {
    enabled: result.value === 'on',
    editable: result.editable,
  };
}
```

### Pattern 2: Bulk Zone Management

```typescript
async function enableArgoForMultipleZones(
  client: Cloudflare,
  zoneIds: string[]
): Promise<Map<string, { success: boolean; error?: string }>> {
  const results = new Map();

  await Promise.allSettled(
    zoneIds.map(async (zoneId) => {
      try {
        await client.argo.smartRouting.edit({
          zone_id: zoneId,
          value: 'on',
        });
        results.set(zoneId, { success: true });
      } catch (error) {
        results.set(zoneId, {
          success: false,
          error: error instanceof Error ? error.message : 'Unknown error',
        });
      }
    })
  );

  return results;
}
```

### Pattern 3: Argo Status Monitoring

```typescript
interface ArgoStatus {
  enabled: boolean;
  editable: boolean;
  lastModified: string;
  zoneId: string;
}

async function monitorArgoStatus(
  client: Cloudflare,
  zoneIds: string[]
): Promise<ArgoStatus[]> {
  const statuses = await Promise.all(
    zoneIds.map(async (zoneId) => {
      const status = await client.argo.smartRouting.get({
        zone_id: zoneId,
      });

      return {
        enabled: status.value === 'on',
        editable: status.editable,
        lastModified: status.modified_on || '',
        zoneId,
      };
    })
  );

  return statuses;
}

// Usage with reporting
async function reportArgoStatus(client: Cloudflare, zoneIds: string[]) {
  const statuses = await monitorArgoStatus(client, zoneIds);

  const summary = {
    total: statuses.length,
    enabled: statuses.filter((s) => s.enabled).length,
    disabled: statuses.filter((s) => !s.enabled).length,
    notEditable: statuses.filter((s) => !s.editable).length,
  };

  console.log('Argo Smart Routing Summary:', summary);
  return { statuses, summary };
}
```

### Pattern 4: Integration with Environment Configuration

```typescript
interface CloudflareConfig {
  apiToken: string;
  zones: {
    production: string;
    staging: string;
  };
  argo: {
    enableProduction: boolean;
    enableStaging: boolean;
  };
}

async function applyArgoConfiguration(config: CloudflareConfig) {
  const client = new Cloudflare({
    apiToken: config.apiToken,
  });

  const updates: Array<Promise<void>> = [];

  if (config.argo.enableProduction) {
    updates.push(
      client.argo.smartRouting.edit({
        zone_id: config.zones.production,
        value: 'on',
      }).then(() => {
        console.log('Production Argo enabled');
      })
    );
  }

  if (config.argo.enableStaging) {
    updates.push(
      client.argo.smartRouting.edit({
        zone_id: config.zones.staging,
        value: 'on',
      }).then(() => {
        console.log('Staging Argo enabled');
      })
    );
  }

  await Promise.all(updates);
}
```

## Analytics & Monitoring

### Performance Metrics

Argo Smart Routing provides analytics via the Cloudflare dashboard:

**Location:** Dashboard → Analytics → Performance → Origin Performance (Argo)

**Requirements:**
- At least 500 origin requests routed by Argo in last 48 hours
- Detailed metrics only show with sufficient traffic

**Available Metrics:**

1. **Origin Response Time (Histogram)**
   - Blue bars: TTFB without Argo
   - Orange bars: TTFB with Argo Smart Route
   - Shows performance improvement quantitatively

2. **Geography View (Map)**
   - Response time improvement per Cloudflare data center
   - Negative values: requests routed directly (Argo would not help)
   - Positive values: improvement achieved by Argo

### Programmatic Analytics Access

While Argo-specific analytics are primarily dashboard-based, you can use GraphQL Analytics API:

```typescript
interface ArgoAnalyticsQuery {
  zoneTag: string;
  since: string;
  until: string;
}

async function queryArgoPerformance(
  client: Cloudflare,
  query: ArgoAnalyticsQuery
) {
  // Note: Use Cloudflare GraphQL Analytics API
  // This requires separate GraphQL client setup
  const graphqlQuery = `
    query ArgoPerformance($zoneTag: String!, $since: String!, $until: String!) {
      viewer {
        zones(filter: { zoneTag: $zoneTag }) {
          httpRequests1dGroups(
            filter: { date_geq: $since, date_leq: $until }
          ) {
            sum {
              bytes
              requests
            }
            dimensions {
              date
            }
          }
        }
      }
    }
  `;

  // Implementation depends on GraphQL client
  // See: https://developers.cloudflare.com/analytics/graphql-api/
}
```

## Spectrum Applications Integration

Argo Smart Routing can be enabled for Spectrum applications (TCP/UDP proxy):

**Requirements:**
- Only available for TCP applications
- `traffic_type` must be set to `"direct"`

**TypeScript Example:**
```typescript
interface SpectrumAppConfig {
  dns: {
    type: string;
    name: string;
  };
  origin_direct: string[];
  protocol: string;
  traffic_type: 'direct' | 'http' | 'https';
  argo_smart_routing?: boolean; // Enable Argo
}

const appConfig: SpectrumAppConfig = {
  dns: {
    type: 'CNAME',
    name: 'ssh.example.com',
  },
  origin_direct: ['tcp://203.0.113.1:22'],
  protocol: 'tcp/22',
  traffic_type: 'direct',
  argo_smart_routing: true, // Enable Argo for this app
};

// Use Spectrum API to create/update app with Argo enabled
```

## Integration with Tiered Cache

Combine Argo Smart Routing with Tiered Cache for maximum performance:

**Endpoint:** `PATCH /zones/{zone_id}/argo/tiered_caching`

**Benefits:**
- Argo optimizes routing between edge and origin
- Tiered Cache reduces origin requests via cache hierarchy
- Combined: optimal network path + reduced origin load

**Enable Both Services:**
```typescript
async function enableArgoWithTieredCache(
  client: Cloudflare,
  zoneId: string
) {
  // Enable Argo Smart Routing
  await client.argo.smartRouting.edit({
    zone_id: zoneId,
    value: 'on',
  });

  // Enable Tiered Caching
  await client.argo.tieredCaching.edit({
    zone_id: zoneId,
    value: 'on',
  });

  console.log('Argo Smart Routing and Tiered Cache enabled');
}
```

**Architecture Flow:**
```
Visitor → Edge Data Center (Lower-Tier)
         ↓ [Cache Miss]
         Upper-Tier Data Center
         ↓ [Cache Miss + Argo Smart Route]
         Origin Server
```

## Usage-Based Billing Management

### Setting Up Billing Notifications

**API Endpoint:** Use Notifications API to configure alerts

**Notification Setup Pattern:**
```typescript
interface BillingNotificationConfig {
  name: string;
  product: 'argo';
  threshold: number; // GB or dollar amount
  email: string;
}

async function setupArgoUsageAlert(
  client: Cloudflare,
  accountId: string,
  config: BillingNotificationConfig
) {
  // Use Cloudflare Notifications API
  // Endpoint: POST /accounts/{account_id}/alerting/v3/policies
  
  const policy = {
    name: config.name,
    alert_type: 'usage_based_billing',
    filters: {
      product: ['argo'],
      limit: config.threshold,
    },
    mechanisms: {
      email: [{ id: config.email }],
    },
    enabled: true,
  };

  // Implementation using Cloudflare client
  // See: https://developers.cloudflare.com/fundamentals/notifications/
}
```

### Cost Monitoring Pattern

```typescript
interface ArgoCostTracker {
  checkInterval: number; // milliseconds
  costThreshold: number; // dollars
  onThresholdExceeded: () => void;
}

class ArgoUsageMonitor {
  private client: Cloudflare;
  private tracker: ArgoCostTracker;

  constructor(client: Cloudflare, tracker: ArgoCostTracker) {
    this.client = client;
    this.tracker = tracker;
  }

  async monitorUsage(zoneId: string) {
    // Use Analytics API to check usage
    // Compare against threshold
    // Trigger callback if exceeded
    
    setInterval(async () => {
      // Implementation: Query analytics for bandwidth usage
      // Calculate cost based on Argo pricing
      // Check against threshold
    }, this.tracker.checkInterval);
  }
}
```

## Common Use Case Examples

### Use Case 1: Global E-commerce Site

**Scenario:** Online store with customers worldwide, origin in US-East

```typescript
async function setupGlobalEcommerce(client: Cloudflare, zoneId: string) {
  // 1. Enable Argo for optimal global routing
  await client.argo.smartRouting.edit({
    zone_id: zoneId,
    value: 'on',
  });

  // 2. Enable Tiered Cache to reduce origin hits
  await client.argo.tieredCaching.edit({
    zone_id: zoneId,
    value: 'on',
  });

  // 3. Set up billing alerts
  // (Use notifications API - see billing section)

  console.log('Global e-commerce optimizations applied');
}
```

### Use Case 2: SaaS Platform with Regional Failover

**Scenario:** Multi-region SaaS, need fast failover and optimal routing

```typescript
interface SaaSRegion {
  zoneId: string;
  region: 'us' | 'eu' | 'asia';
  isPrimary: boolean;
}

async function configureSaaSRouting(
  client: Cloudflare,
  regions: SaaSRegion[]
) {
  for (const region of regions) {
    // Enable Argo for all regions
    await client.argo.smartRouting.edit({
      zone_id: region.zoneId,
      value: 'on',
    });

    // Configure health checks and load balancing
    // (Use Load Balancer API)

    console.log(`Argo enabled for ${region.region} region`);
  }

  // Argo automatically routes to fastest available origin
  // Combined with load balancing for intelligent failover
}
```

### Use Case 3: API Gateway Optimization

**Scenario:** REST API serving global clients, latency-sensitive

```typescript
async function optimizeAPIGateway(client: Cloudflare, zoneId: string) {
  // Enable Argo for API routes
  await client.argo.smartRouting.edit({
    zone_id: zoneId,
    value: 'on',
  });

  // Configure aggressive caching for static responses
  // (Use Cache Rules API)

  // Set up performance monitoring
  // (Use RUM or custom monitoring)

  console.log('API Gateway optimized with Argo Smart Routing');
}
```

## CLI & Automation Scripts

### Shell Script: Quick Enable/Disable

```bash
#!/bin/bash
# argo-toggle.sh - Enable or disable Argo Smart Routing

ZONE_ID="$1"
ACTION="$2" # "on" or "off"
API_TOKEN="${CLOUDFLARE_API_TOKEN}"

if [ -z "$ZONE_ID" ] || [ -z "$ACTION" ]; then
  echo "Usage: $0 <zone_id> <on|off>"
  exit 1
fi

curl -X PATCH "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/argo/smart_routing" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"value\": \"${ACTION}\"}"
```

### Shell Script: Status Check Across Zones

```bash
#!/bin/bash
# argo-status.sh - Check Argo status for multiple zones

ZONES_FILE="$1" # File with one zone ID per line
API_TOKEN="${CLOUDFLARE_API_TOKEN}"

while IFS= read -r zone_id; do
  echo "Checking zone: $zone_id"
  
  response=$(curl -s -X GET \
    "https://api.cloudflare.com/client/v4/zones/${zone_id}/argo/smart_routing" \
    -H "Authorization: Bearer ${API_TOKEN}")
  
  status=$(echo "$response" | jq -r '.result.value')
  echo "  Status: $status"
  echo ""
done < "$ZONES_FILE"
```

### Node.js CLI Tool

```javascript
#!/usr/bin/env node
// argo-cli.js - Command-line tool for Argo management

const Cloudflare = require('cloudflare');
const { program } = require('commander');

program
  .name('argo-cli')
  .description('Manage Cloudflare Argo Smart Routing')
  .version('1.0.0');

program
  .command('enable <zone-id>')
  .description('Enable Argo Smart Routing')
  .action(async (zoneId) => {
    const client = new Cloudflare({
      apiToken: process.env.CLOUDFLARE_API_TOKEN,
    });

    await client.argo.smartRouting.edit({
      zone_id: zoneId,
      value: 'on',
    });

    console.log(`Argo Smart Routing enabled for zone ${zoneId}`);
  });

program
  .command('disable <zone-id>')
  .description('Disable Argo Smart Routing')
  .action(async (zoneId) => {
    const client = new Cloudflare({
      apiToken: process.env.CLOUDFLARE_API_TOKEN,
    });

    await client.argo.smartRouting.edit({
      zone_id: zoneId,
      value: 'off',
    });

    console.log(`Argo Smart Routing disabled for zone ${zoneId}`);
  });

program
  .command('status <zone-id>')
  .description('Check Argo Smart Routing status')
  .action(async (zoneId) => {
    const client = new Cloudflare({
      apiToken: process.env.CLOUDFLARE_API_TOKEN,
    });

    const status = await client.argo.smartRouting.get({
      zone_id: zoneId,
    });

    console.log(`Zone: ${zoneId}`);
    console.log(`Status: ${status.value}`);
    console.log(`Editable: ${status.editable}`);
    console.log(`Last Modified: ${status.modified_on || 'N/A'}`);
  });

program.parse();
```

## Error Handling

### Common Errors & Solutions

**1. Billing Profile Required**
```typescript
try {
  await client.argo.smartRouting.edit({
    zone_id: zoneId,
    value: 'on',
  });
} catch (error) {
  if (error.message.includes('billing')) {
    console.error('Create billing profile first:');
    console.error('https://dash.cloudflare.com/profile/billing');
  }
}
```

**2. Not Editable for Zone**
```typescript
const status = await client.argo.smartRouting.get({
  zone_id: zoneId,
});

if (!status.editable) {
  throw new Error(
    'Argo Smart Routing not available for this zone. ' +
    'Check plan compatibility.'
  );
}
```

**3. Authentication Errors**
```typescript
try {
  await client.argo.smartRouting.get({ zone_id: zoneId });
} catch (error) {
  if (error.status === 403) {
    console.error('API token lacks required permissions.');
    console.error('Required: Zone:Argo Smart Routing:Edit');
  } else if (error.status === 401) {
    console.error('Invalid API token');
  }
}
```

### Robust Error Handling Pattern

```typescript
class ArgoError extends Error {
  constructor(
    message: string,
    public code: string,
    public zoneId?: string
  ) {
    super(message);
    this.name = 'ArgoError';
  }
}

async function safeArgoOperation<T>(
  operation: () => Promise<T>,
  context: { zoneId: string; operation: string }
): Promise<T> {
  try {
    return await operation();
  } catch (error) {
    if (error.status === 403) {
      throw new ArgoError(
        'Insufficient permissions',
        'PERMISSION_DENIED',
        context.zoneId
      );
    } else if (error.status === 400) {
      throw new ArgoError(
        'Invalid request - check zone ID and parameters',
        'INVALID_REQUEST',
        context.zoneId
      );
    } else if (error.message.includes('billing')) {
      throw new ArgoError(
        'Billing profile required',
        'BILLING_REQUIRED',
        context.zoneId
      );
    }
    throw error;
  }
}

// Usage
await safeArgoOperation(
  () => client.argo.smartRouting.edit({ zone_id: zoneId, value: 'on' }),
  { zoneId, operation: 'enable' }
);
```

## Testing & Development

### Mock Client for Testing

```typescript
interface MockArgoClient {
  smartRouting: {
    get: jest.Mock;
    edit: jest.Mock;
  };
}

function createMockArgoClient(): MockArgoClient {
  return {
    smartRouting: {
      get: jest.fn().mockResolvedValue({
        id: 'smart_routing',
        value: 'off',
        editable: true,
        modified_on: new Date().toISOString(),
      }),
      edit: jest.fn().mockResolvedValue({
        id: 'smart_routing',
        value: 'on',
        editable: true,
        modified_on: new Date().toISOString(),
      }),
    },
  };
}

// Test example
describe('Argo Configuration', () => {
  it('should enable Argo when requested', async () => {
    const mockClient = createMockArgoClient();
    
    await configureArgo(mockClient as any, {
      zoneId: 'test-zone',
      enableArgo: true,
    });

    expect(mockClient.smartRouting.edit).toHaveBeenCalledWith({
      zone_id: 'test-zone',
      value: 'on',
    });
  });
});
```

### Integration Testing Pattern

```typescript
describe('Argo Integration Tests', () => {
  const testZoneId = process.env.TEST_ZONE_ID;
  const client = new Cloudflare({
    apiToken: process.env.CLOUDFLARE_API_TOKEN,
  });

  beforeEach(async () => {
    // Reset to known state
    await client.argo.smartRouting.edit({
      zone_id: testZoneId,
      value: 'off',
    });
  });

  it('should toggle Argo on and off', async () => {
    // Enable
    const enableResult = await client.argo.smartRouting.edit({
      zone_id: testZoneId,
      value: 'on',
    });
    expect(enableResult.value).toBe('on');

    // Verify
    const status = await client.argo.smartRouting.get({
      zone_id: testZoneId,
    });
    expect(status.value).toBe('on');

    // Disable
    const disableResult = await client.argo.smartRouting.edit({
      zone_id: testZoneId,
      value: 'off',
    });
    expect(disableResult.value).toBe('off');
  });
});
```

## Configuration Management

### Infrastructure as Code (Terraform)

```hcl
# terraform/argo.tf
# Note: Use Cloudflare Terraform provider

resource "cloudflare_argo" "example" {
  zone_id        = var.zone_id
  smart_routing  = "on"
  tiered_caching = "on"
}

variable "zone_id" {
  description = "Cloudflare Zone ID"
  type        = string
}

output "argo_enabled" {
  value       = cloudflare_argo.example.smart_routing
  description = "Argo Smart Routing status"
}
```

### Environment-Based Configuration

```typescript
// config/argo.ts
interface ArgoEnvironmentConfig {
  enabled: boolean;
  tieredCache: boolean;
  monitoring: {
    usageAlerts: boolean;
    threshold: number;
  };
}

const configs: Record<string, ArgoEnvironmentConfig> = {
  production: {
    enabled: true,
    tieredCache: true,
    monitoring: {
      usageAlerts: true,
      threshold: 1000, // GB
    },
  },
  staging: {
    enabled: true,
    tieredCache: false,
    monitoring: {
      usageAlerts: false,
      threshold: 100,
    },
  },
  development: {
    enabled: false,
    tieredCache: false,
    monitoring: {
      usageAlerts: false,
      threshold: 10,
    },
  },
};

export function getArgoConfig(env: string): ArgoEnvironmentConfig {
  return configs[env] || configs.development;
}
```

## Best Practices Summary

1. **Always check editability** before attempting to enable/disable Argo
2. **Set up billing notifications** to avoid unexpected costs
3. **Combine with Tiered Cache** for maximum performance benefit
4. **Use in production only** - disable for dev/staging to control costs
5. **Monitor analytics** - require 500+ requests in 48h for detailed metrics
6. **Handle errors gracefully** - check for billing, permissions, zone compatibility
7. **Test configuration changes** in staging before production
8. **Use TypeScript SDK** for type safety and better developer experience
9. **Implement retry logic** for API calls in production systems
10. **Document zone-specific settings** for team visibility

## Additional Resources

- [Official Argo Smart Routing Docs](https://developers.cloudflare.com/argo-smart-routing/)
- [Cloudflare API Documentation](https://developers.cloudflare.com/api/)
- [TypeScript SDK](https://github.com/cloudflare/cloudflare-typescript)
- [Smart Shield Documentation](https://developers.cloudflare.com/smart-shield/)
- [Tiered Cache Documentation](https://developers.cloudflare.com/cache/how-to/tiered-cache/)
- [Usage-Based Billing](https://developers.cloudflare.com/billing/usage-based-billing/)

## Quick Reference

### API Endpoints
- `GET /zones/{zone_id}/argo/smart_routing` - Get status
- `PATCH /zones/{zone_id}/argo/smart_routing` - Enable/disable
- `GET /zones/{zone_id}/argo/tiered_caching` - Get Tiered Cache status
- `PATCH /zones/{zone_id}/argo/tiered_caching` - Enable/disable Tiered Cache

### Required Permissions
- `Zone:Argo Smart Routing:Read`
- `Zone:Argo Smart Routing:Edit`

### Key Configuration Values
- `value: "on"` - Enable Argo Smart Routing
- `value: "off"` - Disable Argo Smart Routing

### Minimum Requirements
- Active Cloudflare account with billing profile
- Zone on compatible plan
- API token with appropriate permissions
