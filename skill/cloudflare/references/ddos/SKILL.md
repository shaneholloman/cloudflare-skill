# Cloudflare DDoS Protection Skill

Expert knowledge for implementing and configuring Cloudflare's DDoS Protection, including HTTP (L7) and Network-layer (L3/4) managed rulesets, Adaptive DDoS Protection, API configuration, and integration patterns.

## Overview

Cloudflare DDoS Protection provides autonomous, always-on protection against distributed denial-of-service attacks across layers 3/4 and 7 of the OSI model. This skill covers configuration, API usage, and integration patterns.

### Core Components

1. **HTTP DDoS Attack Protection** (Layer 7)
   - Protects HTTP/HTTPS traffic
   - Phase: `ddos_l7`
   - Configured at zone or account level
   
2. **Network-layer DDoS Attack Protection** (L3/4)
   - Protects against UDP floods, SYN floods, DNS floods
   - Phase: `ddos_l4`
   - Configured at account level only

3. **Adaptive DDoS Protection**
   - Learns traffic patterns over 7 days
   - Adapts to detect deviations from baseline
   - Available on Enterprise plans

## Configuration Patterns

### HTTP DDoS Protection (L7)

#### Zone-level Override via API

```typescript
// Configure HTTP DDoS protection for a zone
interface HTTPDDoSOverride {
  description: string;
  rules: Array<{
    action: "execute";
    expression: string;
    action_parameters: {
      id: string; // Managed ruleset ID
      overrides: {
        sensitivity_level?: "default" | "medium" | "low" | "eoff";
        action?: "block" | "managed_challenge" | "challenge" | "log";
        categories?: Array<{
          category: string; // Tag name
          sensitivity_level?: "default" | "medium" | "low" | "eoff";
        }>;
        rules?: Array<{
          id: string; // Rule ID
          action?: "block" | "managed_challenge" | "challenge" | "log";
          sensitivity_level?: "default" | "medium" | "low" | "eoff";
        }>;
      };
    };
  }>;
}

// Example: Configure zone-level HTTP DDoS protection
async function configureHTTPDDoS(
  zoneId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config: HTTPDDoSOverride = {
    description: "Custom HTTP DDoS protection",
    rules: [
      {
        action: "execute",
        expression: "true", // Apply to all traffic
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "medium",
            action: "managed_challenge",
            categories: [
              {
                category: "http-flood",
                sensitivity_level: "low",
              },
            ],
            rules: [
              {
                id: "rule-id-here",
                action: "block",
              },
            ],
          },
        },
      },
    ],
  };

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );

  return response.json();
}
```

#### Account-level Override (Enterprise with Advanced DDoS)

```typescript
// Account-level override with custom expressions
async function configureAccountHTTPDDoS(
  accountId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config = {
    description: "Account-level HTTP DDoS override for allowlisted IPs",
    rules: [
      {
        // Custom expression for specific traffic
        expression: "ip.src in $allowlisted_ips",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            rules: [
              {
                id: "rule-id",
                action: "log", // Enterprise only
                sensitivity_level: "eoff",
              },
            ],
          },
        },
      },
    ],
  };

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );

  return response.json();
}
```

### Network-layer DDoS Protection (L3/4)

```typescript
// Configure Network-layer DDoS protection (account level only)
async function configureNetworkDDoS(
  accountId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config = {
    description: "Network-layer DDoS protection for specific IP range",
    rules: [
      {
        action: "execute",
        expression: "ip.dst in { 1.1.1.0/24 }", // Target specific IPs
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "medium",
            categories: [
              {
                category: "udp-flood",
                sensitivity_level: "low",
              },
            ],
            rules: [
              {
                id: "rule-id",
                action: "block",
              },
            ],
          },
        },
      },
    ],
  };

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/rulesets/phases/ddos_l4/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );

  return response.json();
}
```

### Adaptive DDoS Protection Configuration

```typescript
// Adaptive rules are part of the managed rulesets
// Configure them by targeting specific rule IDs

interface AdaptiveRuleConfig {
  ruleId: string;
  type: "origins" | "user-agents" | "locations" | "protocols";
  action: "log" | "managed_challenge" | "block";
  sensitivity: "default" | "medium" | "low" | "eoff";
}

// Example adaptive rule IDs (check dashboard for actual IDs)
const ADAPTIVE_RULES = {
  HTTP_ORIGINS: "adaptive-ddos-origins-rule-id",
  HTTP_USER_AGENTS: "adaptive-ddos-ua-rule-id",
  HTTP_LOCATIONS: "adaptive-ddos-location-rule-id",
  L4_PROTOCOLS: "adaptive-ddos-protocols-rule-id",
};

async function enableAdaptiveProtection(
  zoneId: string,
  managedRulesetId: string,
  config: AdaptiveRuleConfig,
  apiToken: string
) {
  const override = {
    description: `Enable adaptive DDoS protection for ${config.type}`,
    rules: [
      {
        action: "execute",
        expression: "true",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            rules: [
              {
                id: config.ruleId,
                action: config.action,
                sensitivity_level: config.sensitivity,
              },
            ],
          },
        },
      },
    ],
  };

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(override),
    }
  );

  return response.json();
}
```

## Using the Cloudflare TypeScript SDK

```typescript
import Cloudflare from "cloudflare";

const client = new Cloudflare({
  apiToken: process.env.CLOUDFLARE_API_TOKEN,
});

// Get current HTTP DDoS ruleset
async function getCurrentHTTPDDoSRuleset(zoneId: string) {
  const rulesets = await client.zones.rulesets.phases.entrypoint.get(
    "ddos_l7",
    {
      zone_id: zoneId,
    }
  );
  return rulesets;
}

// Update HTTP DDoS ruleset
async function updateHTTPDDoSRuleset(
  zoneId: string,
  managedRulesetId: string
) {
  const result = await client.zones.rulesets.phases.entrypoint.update(
    "ddos_l7",
    {
      zone_id: zoneId,
      rules: [
        {
          action: "execute",
          expression: "true",
          action_parameters: {
            id: managedRulesetId,
            overrides: {
              sensitivity_level: "medium",
              action: "managed_challenge",
            },
          },
        },
      ],
    }
  );
  return result;
}

// Get Network-layer DDoS ruleset (account level)
async function getNetworkDDoSRuleset(accountId: string) {
  const rulesets = await client.accounts.rulesets.phases.entrypoint.get(
    "ddos_l4",
    {
      account_id: accountId,
    }
  );
  return rulesets;
}
```

## Wrangler Integration

DDoS protection is configured via API, not wrangler.toml. However, you can create Workers to programmatically manage DDoS settings:

```typescript
// worker.ts - DDoS configuration management worker
interface Env {
  CLOUDFLARE_API_TOKEN: string;
  ZONE_ID: string;
  ACCOUNT_ID: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === "/configure-ddos") {
      // Configure HTTP DDoS protection
      const response = await fetch(
        `https://api.cloudflare.com/client/v4/zones/${env.ZONE_ID}/rulesets/phases/ddos_l7/entrypoint`,
        {
          method: "PUT",
          headers: {
            Authorization: `Bearer ${env.CLOUDFLARE_API_TOKEN}`,
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            description: "Dynamic DDoS configuration",
            rules: [
              {
                action: "execute",
                expression: "true",
                action_parameters: {
                  id: "managed-ruleset-id",
                  overrides: {
                    sensitivity_level: "medium",
                  },
                },
              },
            ],
          }),
        }
      );

      return response;
    }

    if (url.pathname === "/ddos-status") {
      // Get current configuration
      const response = await fetch(
        `https://api.cloudflare.com/client/v4/zones/${env.ZONE_ID}/rulesets/phases/ddos_l7/entrypoint`,
        {
          headers: {
            Authorization: `Bearer ${env.CLOUDFLARE_API_TOKEN}`,
          },
        }
      );

      return response;
    }

    return new Response("Not found", { status: 404 });
  },
};
```

### wrangler.toml for DDoS Management Worker

```toml
name = "ddos-manager"
main = "src/worker.ts"
compatibility_date = "2024-01-01"

[vars]
ZONE_ID = "your-zone-id"
ACCOUNT_ID = "your-account-id"

# API token stored as secret
# Run: wrangler secret put CLOUDFLARE_API_TOKEN
```

## Alert Configuration

```typescript
// Configure DDoS alerts via API
interface DDoSAlertConfig {
  name: string;
  enabled: boolean;
  alert_type:
    | "http_ddos_attack_alert"
    | "layer_3_4_ddos_attack_alert"
    | "advanced_http_ddos_attack_alert"
    | "advanced_layer_3_4_ddos_attack_alert";
  filters?: {
    zones?: string[]; // Zone IDs
    hostnames?: string[]; // Specific hostnames
    requests_per_second?: number; // Min RPS threshold
    packets_per_second?: number; // Min PPS threshold
    megabits_per_second?: number; // Min Mbps threshold
    ip_prefixes?: string[]; // CIDR notation
    ip_addresses?: string[]; // Specific IPs
    protocols?: string[]; // Protocol names
  };
  mechanisms: {
    email?: Array<{ id: string }>;
    webhooks?: Array<{ id: string }>;
    pagerduty?: Array<{ id: string }>;
  };
}

async function createDDoSAlert(
  accountId: string,
  config: DDoSAlertConfig,
  apiToken: string
) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/alerting/v3/policies`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );

  return response.json();
}

// Example: Create advanced HTTP DDoS alert
const alertConfig: DDoSAlertConfig = {
  name: "High RPS DDoS Alert",
  enabled: true,
  alert_type: "advanced_http_ddos_attack_alert",
  filters: {
    zones: ["zone-id-1", "zone-id-2"],
    requests_per_second: 1000, // Alert on attacks > 1000 RPS
  },
  mechanisms: {
    email: [{ id: "email-destination-id" }],
    webhooks: [{ id: "webhook-id" }],
  },
};
```

## Common Use Cases

### 1. Allowlist Trusted IPs from DDoS Mitigation

```typescript
// Use account-level override with custom expression
async function allowlistIPs(
  accountId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config = {
    description: "Allowlist trusted IPs",
    rules: [
      {
        expression: "ip.src in { 203.0.113.0/24 192.0.2.1 }",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "eoff", // Essentially off
          },
        },
      },
    ],
  };

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );

  return response.json();
}
```

### 2. Adjust Sensitivity for Specific Routes

```typescript
// Lower sensitivity for API endpoints with bursty traffic
async function configureBurstyEndpoints(
  zoneId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config = {
    description: "Lower sensitivity for API endpoints",
    rules: [
      {
        // High sensitivity for regular traffic
        expression: "not http.request.uri.path matches \"^/api/\"",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "default",
            action: "block",
          },
        },
      },
      {
        // Lower sensitivity for API traffic
        expression: "http.request.uri.path matches \"^/api/\"",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "low",
            action: "managed_challenge",
          },
        },
      },
    ],
  };

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );

  return response.json();
}
```

### 3. Testing with Log Action (Enterprise)

```typescript
// Enable logging mode before blocking
async function enableLoggingMode(
  zoneId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config = {
    description: "Test DDoS rules with logging",
    rules: [
      {
        expression: "true",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            action: "log", // Enterprise only
            sensitivity_level: "medium",
          },
        },
      },
    ],
  };

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );

  return response.json();
}
```

### 4. Multi-rule Configuration (Enterprise Advanced DDoS)

```typescript
// Create multiple rules with different conditions (up to 10)
async function configureMultiRuleDDoS(
  accountId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config = {
    description: "Multi-rule DDoS configuration",
    rules: [
      {
        // Rule 1: Strictest for unknown traffic
        expression:
          "not ip.src in $known_ips and not cf.bot_management.score gt 30",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "default",
            action: "block",
          },
        },
      },
      {
        // Rule 2: Medium for verified bots
        expression: "cf.bot_management.verified_bot",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "medium",
            action: "managed_challenge",
          },
        },
      },
      {
        // Rule 3: Low for trusted IPs
        expression: "ip.src in $trusted_ips",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "low",
          },
        },
      },
    ],
  };

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );

  return response.json();
}
```

## Architecture Patterns

### Pattern 1: Defense in Depth

```typescript
// Combine DDoS protection with WAF and Rate Limiting
interface DefenseInDepthConfig {
  zoneId: string;
  accountId: string;
  apiToken: string;
}

async function setupDefenseInDepth(config: DefenseInDepthConfig) {
  const { zoneId, accountId, apiToken } = config;

  // 1. Configure HTTP DDoS Protection
  await configureHTTPDDoS(zoneId, "http-ddos-ruleset-id", apiToken);

  // 2. Configure Network DDoS Protection
  await configureNetworkDDoS(accountId, "network-ddos-ruleset-id", apiToken);

  // 3. Enable adaptive protection (gradual rollout)
  const adaptiveRules: AdaptiveRuleConfig[] = [
    {
      ruleId: ADAPTIVE_RULES.HTTP_ORIGINS,
      type: "origins",
      action: "log", // Start with logging
      sensitivity: "medium",
    },
    {
      ruleId: ADAPTIVE_RULES.HTTP_USER_AGENTS,
      type: "user-agents",
      action: "log",
      sensitivity: "medium",
    },
  ];

  for (const rule of adaptiveRules) {
    await enableAdaptiveProtection(
      zoneId,
      "http-ddos-ruleset-id",
      rule,
      apiToken
    );
  }

  // 4. Set up alerts
  await createDDoSAlert(
    accountId,
    {
      name: "Critical DDoS Alert",
      enabled: true,
      alert_type: "advanced_http_ddos_attack_alert",
      filters: {
        zones: [zoneId],
        requests_per_second: 5000,
      },
      mechanisms: {
        email: [{ id: "email-id" }],
        pagerduty: [{ id: "pd-id" }],
      },
    },
    apiToken
  );

  return { success: true };
}
```

### Pattern 2: Progressive Enhancement

```typescript
// Gradually increase DDoS protection strictness
enum ProtectionLevel {
  MONITORING = "monitoring",
  LOW = "low",
  MEDIUM = "medium",
  HIGH = "high",
}

async function setProtectionLevel(
  zoneId: string,
  level: ProtectionLevel,
  managedRulesetId: string,
  apiToken: string
) {
  const levelConfig = {
    [ProtectionLevel.MONITORING]: {
      action: "log",
      sensitivity: "eoff",
    },
    [ProtectionLevel.LOW]: {
      action: "managed_challenge",
      sensitivity: "low",
    },
    [ProtectionLevel.MEDIUM]: {
      action: "managed_challenge",
      sensitivity: "medium",
    },
    [ProtectionLevel.HIGH]: {
      action: "block",
      sensitivity: "default",
    },
  } as const;

  const settings = levelConfig[level];

  const config = {
    description: `DDoS protection level: ${level}`,
    rules: [
      {
        expression: "true",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            action: settings.action,
            sensitivity_level: settings.sensitivity,
          },
        },
      },
    ],
  };

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );

  return response.json();
}

// Usage: Start with monitoring, gradually increase
async function progressiveRollout(
  zoneId: string,
  managedRulesetId: string,
  apiToken: string
) {
  // Week 1: Monitor
  await setProtectionLevel(
    zoneId,
    ProtectionLevel.MONITORING,
    managedRulesetId,
    apiToken
  );
  console.log("Week 1: Monitoring enabled");

  // Week 2: Low sensitivity
  // await setProtectionLevel(zoneId, ProtectionLevel.LOW, managedRulesetId, apiToken);

  // Week 3: Medium sensitivity
  // await setProtectionLevel(zoneId, ProtectionLevel.MEDIUM, managedRulesetId, apiToken);

  // Week 4: High sensitivity
  // await setProtectionLevel(zoneId, ProtectionLevel.HIGH, managedRulesetId, apiToken);
}
```

### Pattern 3: Dynamic Response to Attacks

```typescript
// Worker that adjusts DDoS settings based on attack patterns
interface Env {
  CLOUDFLARE_API_TOKEN: string;
  ZONE_ID: string;
  ATTACK_THRESHOLD: string; // RPS threshold
  KV_NAMESPACE: KVNamespace; // Store attack history
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === "/attack-detected") {
      // Called by webhook when attack detected
      const attackData = await request.json<{
        rps: number;
        target: string;
        attackType: string;
      }>();

      // Store attack history
      await env.KV_NAMESPACE.put(
        `attack:${Date.now()}`,
        JSON.stringify(attackData),
        { expirationTtl: 86400 } // 24 hours
      );

      // Check recent attack frequency
      const recentAttacks = await getRecentAttacks(env.KV_NAMESPACE);

      if (recentAttacks.length > 5) {
        // Multiple attacks: increase protection
        await increaseProtection(
          env.ZONE_ID,
          "managed-ruleset-id",
          env.CLOUDFLARE_API_TOKEN
        );
        return new Response("Protection increased", { status: 200 });
      }

      return new Response("Attack logged", { status: 200 });
    }

    return new Response("OK", { status: 200 });
  },

  // Scheduled handler to auto-decrease protection after quiet period
  async scheduled(event: ScheduledEvent, env: Env): Promise<void> {
    const recentAttacks = await getRecentAttacks(env.KV_NAMESPACE);

    if (recentAttacks.length === 0) {
      // No recent attacks: normalize protection
      await normalizeProtection(
        env.ZONE_ID,
        "managed-ruleset-id",
        env.CLOUDFLARE_API_TOKEN
      );
    }
  },
};

async function getRecentAttacks(kv: KVNamespace): Promise<any[]> {
  const list = await kv.list({ prefix: "attack:" });
  const attacks = await Promise.all(
    list.keys.map((key) => kv.get(key.name, "json"))
  );
  return attacks.filter(Boolean);
}

async function increaseProtection(
  zoneId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config = {
    description: "Increased protection due to attacks",
    rules: [
      {
        expression: "true",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "default", // High
            action: "block",
          },
        },
      },
    ],
  };

  return fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );
}

async function normalizeProtection(
  zoneId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config = {
    description: "Normal protection level",
    rules: [
      {
        expression: "true",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "medium",
            action: "managed_challenge",
          },
        },
      },
    ],
  };

  return fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );
}
```

## Key Parameters Reference

### Sensitivity Levels

| UI Value | API Value | Description |
|----------|-----------|-------------|
| High | `default` | Lowest threshold, most aggressive |
| Medium | `medium` | Balanced protection |
| Low | `low` | Higher threshold, less aggressive |
| Essentially Off | `eoff` | Very high threshold, minimal mitigation |

### Actions (HTTP DDoS)

| Action | API Value | Description | Plan |
|--------|-----------|-------------|------|
| Block | `block` | Block requests | All |
| Managed Challenge | `managed_challenge` | Dynamic challenge | All |
| Interactive Challenge | `challenge` | CAPTCHA challenge | All |
| Log | `log` | Log only, no mitigation | Enterprise Advanced |

### Actions (Network DDoS)

| Action | API Value | Description | Plan |
|--------|-----------|-------------|------|
| Block | `block` | Drop packets | All |
| Log | `log` | Log only, no mitigation | Enterprise |

### Phase Types

- `ddos_l7`: HTTP DDoS Attack Protection
- `ddos_l4`: Network-layer DDoS Attack Protection

### Common Rule Categories (Tags)

- `http-flood`
- `http-anomaly`
- `udp-flood`
- `syn-flood`
- `dns-flood`

## Important Constraints

1. **Always Enabled**: DDoS managed rulesets cannot be disabled completely
2. **Read-only Rules**: Some managed rules cannot be overridden
3. **Rule Limits**: 
   - Free/Pro/Business: 1 override rule
   - Enterprise Advanced: Up to 10 rules
4. **Scope**:
   - HTTP DDoS: Zone or account level
   - Network DDoS: Account level only
5. **Account vs Zone**: If zone has overrides, account overrides are ignored
6. **Adaptive Protection**: Requires 7 days of traffic history
7. **Log Action**: Enterprise Advanced DDoS only

## Best Practices

1. **Start with Logging**: Use `log` action to validate rules before blocking
2. **Monitor Adaptive Rules**: Check flagged traffic before enabling mitigation
3. **Gradual Rollout**: Increase sensitivity incrementally
4. **Zone-specific Tuning**: Use zone-level overrides for custom needs
5. **Alert Configuration**: Set up alerts with appropriate thresholds
6. **Document Changes**: Keep track of sensitivity adjustments
7. **Test During Quiet Periods**: Validate changes when traffic is normal
8. **Combine with WAF**: Layer DDoS protection with WAF rules
9. **Use IP Lists**: Reference IP lists in expressions for easier management
10. **Avoid Over-tuning**: Too many rules can be harder to maintain

## Troubleshooting

### False Positives

```typescript
// Check events to identify false positives
async function analyzeHTTPDDoSEvents(
  zoneId: string,
  ruleId: string,
  apiToken: string
) {
  // Query via GraphQL API
  const query = `
    query {
      viewer {
        zones(filter: { zoneTag: "${zoneId}" }) {
          httpRequestsAdaptiveGroups(
            filter: { ruleId: "${ruleId}", action: "log" }
            limit: 100
            orderBy: [datetime_DESC]
          ) {
            dimensions {
              clientCountryName
              clientRequestHTTPHost
              clientRequestPath
              userAgent
            }
            count
          }
        }
      }
    }
  `;

  const response = await fetch("https://api.cloudflare.com/client/v4/graphql", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiToken}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ query }),
  });

  return response.json();
}

// If legitimate traffic flagged, lower sensitivity or add exception
async function fixFalsePositive(
  zoneId: string,
  managedRulesetId: string,
  ruleId: string,
  apiToken: string
) {
  const config = {
    description: "Reduce false positives",
    rules: [
      {
        expression: "true",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            rules: [
              {
                id: ruleId,
                sensitivity_level: "low", // Lower sensitivity
              },
            ],
          },
        },
      },
    ],
  };

  return fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );
}
```

### Missing Attack Mitigation

```typescript
// Increase sensitivity if attacks are getting through
async function increaseSensitivity(
  zoneId: string,
  managedRulesetId: string,
  apiToken: string
) {
  const config = {
    description: "Increase DDoS sensitivity",
    rules: [
      {
        expression: "true",
        action: "execute",
        action_parameters: {
          id: managedRulesetId,
          overrides: {
            sensitivity_level: "default", // High
            action: "block",
          },
        },
      },
    ],
  };

  return fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/rulesets/phases/ddos_l7/entrypoint`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(config),
    }
  );
}
```

## References

- [Cloudflare DDoS Protection Docs](https://developers.cloudflare.com/ddos-protection/)
- [HTTP DDoS Configuration API](https://developers.cloudflare.com/ddos-protection/managed-rulesets/http/configure-api/)
- [Network DDoS Configuration API](https://developers.cloudflare.com/ddos-protection/managed-rulesets/network/configure-api/)
- [Adaptive DDoS Protection](https://developers.cloudflare.com/ddos-protection/managed-rulesets/adaptive-protection/)
- [DDoS Alerts](https://developers.cloudflare.com/ddos-protection/reference/alerts/)
- [Rulesets API](https://developers.cloudflare.com/ruleset-engine/rulesets-api/)
