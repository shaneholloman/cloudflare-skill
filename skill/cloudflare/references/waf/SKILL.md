# Cloudflare WAF Expert Skill

**Expertise**: Cloudflare Web Application Firewall (WAF) configuration, custom rules, managed rulesets, rate limiting, attack detection, and API integration

## Core Capabilities

### WAF Architecture & Components

**Managed Rulesets** (Pre-configured protection):
- **Cloudflare Managed Ruleset** (efb7b8c949ac4650a09736fc376e9aee): Fast, effective protection updated frequently for new vulnerabilities
- **Cloudflare OWASP Core Ruleset** (4814384a9e5d4991b9815dcfc25d2f1f): Implementation of OWASP ModSecurity Core Rule Set
- **Cloudflare Exposed Credentials Check** (c2e184081120413c86c3ab7e14069605): Automated credential checks (deprecated - use leaked credentials detection)
- **Cloudflare Free Managed Ruleset** (77454fe2d30c4220b5701f6fdfb893ba): Available on all plans, high-impact vulnerability mitigation
- **Cloudflare Sensitive Data Detection** (e22d83c647c64a3eae91b71b499d988e): Response phase ruleset for data loss prevention (Enterprise paid add-on)

**Custom Rules**: User-defined WAF rules using Rules language for specific traffic patterns and actions

**Rate Limiting Rules**: Define rate limits based on request characteristics with configurable actions and mitigation strategies

**WAF Tools**: Browser Integrity Check, Security Level, IP Access Rules, Zone Lockdown, User Agent Blocking, Validation Checks

### Rules Language (Wirefilter Syntax)

**Expression Components**:
- **Fields**: Request properties (http.request.uri.path, http.request.method, ip.src, cf.threat_score, etc.)
- **Operators**: eq, ne, lt, le, gt, ge, contains, matches (regex), in
- **Functions**: lower(), upper(), len(), concat(), any(), all(), ends_with(), starts_with()
- **Logical operators**: and, or, not
- **Grouping**: Parentheses for expression precedence

**Common Field Categories**:
- General request: http.request.uri, http.request.method, http.request.headers, http.request.body
- IP/Network: ip.src, ip.geoip.country, ip.geoip.asnum
- Security: cf.threat_score, cf.bot_management.score, cf.waf.score, cf.waf.score.sqli, cf.waf.score.xss, cf.waf.score.rce
- SSL/TLS: ssl, cf.tls_client_auth.cert_verified
- Response: http.response.code, http.response.headers

**Example Expressions**:
```wirefilter
# Block specific path
http.request.uri.path eq "/admin"

# Block countries
ip.geoip.country in {"CN" "RU"}

# SQL injection protection
cf.waf.score.sqli lt 20

# Rate limit API endpoints
http.request.uri.path matches "^/api/" and http.request.method eq "POST"

# Bot detection
cf.bot_management.score lt 30

# Complex expression
(http.request.uri.path contains "/login" and http.request.method eq "POST") or cf.threat_score gt 50
```

### Actions

**Standard Actions**:
- `block`: Terminate request, return 403
- `challenge`: Present CAPTCHA (deprecated for managed_challenge)
- `managed_challenge`: Smart challenge based on client characteristics
- `js_challenge`: JavaScript challenge
- `log`: Log matching requests without action (Enterprise only for custom rules)
- `skip`: Skip specific security features
- `allow`: Explicitly allow traffic (bypass subsequent rules)

**Rate Limiting Actions**:
- `block`: Block requests exceeding rate
- `challenge`: Challenge requests exceeding rate
- `managed_challenge`: Smart challenge for rate limit violations
- `log`: Log rate limit violations (Enterprise)
- Action behaviors: Standard (apply action for duration) or Throttle (block excess, allow within limit - Enterprise with Advanced Rate Limiting)

### WAF Attack Score

**Attack Score Fields** (Enterprise):
- `cf.waf.score`: Global attack score (1-99, lower = more likely malicious)
- `cf.waf.score.sqli`: SQL injection score
- `cf.waf.score.xss`: Cross-site scripting score
- `cf.waf.score.rce`: Remote code execution score
- `cf.waf.score.class`: Attack class (Business+): "attack", "likely_attack", "likely_clean", "clean"

**Score Interpretation**:
- 1-20: Attack (high confidence malicious)
- 21-50: Likely attack (may have false positives)
- 51-80: Likely clean
- 81-99: Clean
- 100: Unscored (system decided not to score)

**Recommended Thresholds**:
- Start with `cf.waf.score le 20` for blocking (attack only)
- Combine higher thresholds with additional filters to reduce false positives
- Monitor traffic before enforcing stricter thresholds
- Business plan: Use `cf.waf.score.class eq "attack"` to block

### Rate Limiting Configuration

**Core Parameters**:
- **Characteristics**: Define how to track rate (IP, headers, country, ASN, path, query, cookie, JA3/JA4 fingerprint, JSON field, body, form input, custom expression)
- **Period**: Time window in seconds (10s, 60s, 300s, 3600s, etc. - varies by plan)
- **Requests per period**: Threshold to trigger rate limit
- **Mitigation timeout/Duration**: How long to apply action after rate exceeded
- **Counting expression**: Optional filter to count specific requests only

**Characteristics by Plan**:
- Free/Pro: IP only
- Business: IP, IP with NAT support
- Enterprise: IP, IP with NAT, Query, Host, Headers, Cookie, ASN, Country, Path, JA3/JA4, JSON field, Body, Form input, Custom expression
- Enterprise with Advanced Rate Limiting: Complexity-based counting (not just request count)

**Example Configurations**:
```json
// API rate limit by IP
{
  "expression": "http.request.uri.path matches \"^/api/\"",
  "action": "block",
  "characteristics": [{"name": "ip.src"}],
  "period": 60,
  "requests_per_period": 100,
  "mitigation_timeout": 600
}

// Login rate limit by IP with counting expression
{
  "expression": "http.request.uri.path eq \"/login\" and http.request.method eq \"POST\"",
  "action": "managed_challenge",
  "characteristics": [{"name": "ip.src"}],
  "period": 300,
  "requests_per_period": 5,
  "mitigation_timeout": 900,
  "counting_expression": "http.response.code eq 401"
}

// Advanced: Rate limit by custom header
{
  "expression": "true",
  "action": "block",
  "characteristics": [
    {"name": "cf.colo.id"},
    {"name": "http.request.headers[\"x-api-key\"]"}
  ],
  "period": 60,
  "requests_per_period": 1000,
  "mitigation_timeout": 60
}
```

### Custom Rules Patterns

**Common Use Cases**:

1. **Path-based protection**:
```wirefilter
# Block admin access except from specific IPs
http.request.uri.path contains "/admin" and not ip.src in {1.2.3.4 5.6.7.8}
```

2. **Geographic restrictions**:
```wirefilter
# Allow only specific countries
not ip.geoip.country in {"US" "CA" "GB"}
```

3. **User-Agent filtering**:
```wirefilter
# Block bad bots
http.user_agent contains "scrapy" or http.user_agent contains "curl"
```

4. **Attack score-based protection**:
```wirefilter
# Block likely attacks on sensitive endpoints
http.request.uri.path contains "/checkout" and cf.waf.score le 30
```

5. **Method restrictions**:
```wirefilter
# Only allow GET/POST
http.request.method in {"DELETE" "PUT" "PATCH"}
```

6. **Header validation**:
```wirefilter
# Require specific header
not any(http.request.headers.names[*] == "x-api-key")
```

7. **Skip rules for known good traffic**:
```wirefilter
# Skip WAF for verified bots
cf.bot_management.verified_bot
```
Action: `skip` with products: `waf`, `rateLimit`, `securityLevel`

8. **Content-Type validation**:
```wirefilter
# Ensure POST requests have Content-Type
http.request.method eq "POST" and not any(http.request.headers["content-type"][*] contains "application/")
```

### Ruleset Engine Integration

**Phases** (where rulesets execute):
- `http_request_firewall_custom`: Custom rules phase
- `http_request_firewall_managed`: Managed rules phase
- `http_ratelimit`: Rate limiting phase
- `http_request_late_transform`: Late transform phase
- `http_response_firewall_managed`: Response phase (e.g., sensitive data detection)

**Ruleset Types**:
- `root`: Entry point ruleset for a phase
- `custom`: Custom rulesets (reusable across zones at account level)
- `managed`: Pre-built managed rulesets

**Ruleset Structure**:
```json
{
  "id": "ruleset-id",
  "name": "My Custom WAF Rules",
  "description": "Custom protection rules",
  "kind": "root",
  "phase": "http_request_firewall_custom",
  "rules": [
    {
      "id": "rule-id",
      "action": "block",
      "expression": "cf.waf.score lt 20",
      "description": "Block high-confidence attacks",
      "enabled": true
    }
  ]
}
```

### WAF Configuration via API

**Base URL**: `https://api.cloudflare.com/client/v4`

**Authentication**:
```bash
# API Token (preferred)
Authorization: Bearer <token>

# API Key (legacy)
X-Auth-Email: <email>
X-Auth-Key: <key>
```

**Common Endpoints**:

1. **List Zone Rulesets**:
```bash
GET /zones/{zone_id}/rulesets
```

2. **Get Zone Entry Point Ruleset**:
```bash
GET /zones/{zone_id}/rulesets/phases/{phase}/entrypoint
```

3. **Update Zone Ruleset**:
```bash
PUT /zones/{zone_id}/rulesets/{ruleset_id}
Content-Type: application/json

{
  "rules": [
    {
      "action": "block",
      "expression": "cf.waf.score lt 20",
      "description": "Block attacks"
    }
  ]
}
```

4. **Create Custom Rule** (via ruleset update):
```bash
PUT /zones/{zone_id}/rulesets/phases/http_request_firewall_custom/entrypoint
Content-Type: application/json

{
  "rules": [
    {
      "action": "managed_challenge",
      "expression": "http.request.uri.path contains \"/api/\" and cf.threat_score gt 10",
      "description": "Challenge suspicious API traffic",
      "enabled": true
    }
  ]
}
```

5. **Create Rate Limiting Rule**:
```bash
POST /zones/{zone_id}/rulesets/phases/http_ratelimit/entrypoint/rules
Content-Type: application/json

{
  "action": "block",
  "expression": "http.request.uri.path eq \"/api/submit\"",
  "description": "Rate limit submissions",
  "ratelimit": {
    "characteristics": ["ip.src"],
    "period": 60,
    "requests_per_period": 10,
    "mitigation_timeout": 600
  }
}
```

6. **Deploy Managed Ruleset**:
```bash
PUT /zones/{zone_id}/rulesets/phases/http_request_firewall_managed/entrypoint
Content-Type: application/json

{
  "rules": [
    {
      "action": "execute",
      "action_parameters": {
        "id": "efb7b8c949ac4650a09736fc376e9aee",
        "overrides": {
          "enabled": true
        }
      },
      "expression": "true",
      "description": "Deploy Cloudflare Managed Ruleset"
    }
  ]
}
```

7. **Configure Managed Ruleset Overrides**:
```json
{
  "action": "execute",
  "action_parameters": {
    "id": "efb7b8c949ac4650a09736fc376e9aee",
    "overrides": {
      "enabled": true,
      "action": "log",  // Override default action for all rules
      "categories": [
        {
          "category": "wordpress",
          "action": "block",
          "enabled": true
        }
      ],
      "rules": [
        {
          "id": "rule-id",
          "action": "managed_challenge",
          "enabled": true
        }
      ]
    }
  },
  "expression": "http.host eq \"example.com\"",
  "description": "Managed ruleset with overrides"
}
```

8. **List WAF Packages** (Legacy API - deprecated):
```bash
GET /zones/{zone_id}/firewall/waf/packages
```

9. **Create WAF Exception**:
```bash
POST /zones/{zone_id}/rulesets/phases/http_request_firewall_managed/entrypoint/rules
Content-Type: application/json

{
  "action": "skip",
  "action_parameters": {
    "rulesets": ["efb7b8c949ac4650a09736fc376e9aee"]
  },
  "expression": "http.request.uri.path contains \"/api/webhook\"",
  "description": "Skip WAF for webhook endpoint"
}
```

### Wrangler Integration

While Wrangler primarily manages Workers, WAF configuration is typically done via:
- Dashboard UI
- Cloudflare API directly
- Terraform provider
- Pulumi provider

**Related Wrangler Operations**:
- Deploy Workers that benefit from WAF protection
- Configure zone settings that complement WAF
- Use `wrangler.toml` for environment-specific configurations

**Example: Using Cloudflare API from Worker**:
```typescript
// Worker that calls WAF API
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const response = await fetch(
      `https://api.cloudflare.com/client/v4/zones/${env.ZONE_ID}/rulesets`,
      {
        headers: {
          'Authorization': `Bearer ${env.CF_API_TOKEN}`,
          'Content-Type': 'application/json',
        },
      }
    );
    return response;
  },
};
```

### TypeScript SDK Usage

**Installation**:
```bash
npm install cloudflare
# or
pnpm add cloudflare
```

**Basic Setup**:
```typescript
import Cloudflare from 'cloudflare';

const client = new Cloudflare({
  apiToken: process.env.CLOUDFLARE_API_TOKEN,
});

// Alternative with API key
const client = new Cloudflare({
  apiEmail: process.env.CLOUDFLARE_EMAIL,
  apiKey: process.env.CLOUDFLARE_API_KEY,
});
```

**Working with Rulesets**:
```typescript
// List zone rulesets
const rulesets = await client.rulesets.list({
  zone_id: 'zone-id',
});

// Get specific ruleset
const ruleset = await client.rulesets.get('ruleset-id', {
  zone_id: 'zone-id',
});

// Create custom ruleset
const newRuleset = await client.rulesets.create({
  kind: 'root',
  name: 'My Custom WAF Rules',
  phase: 'http_request_firewall_custom',
  zone_id: 'zone-id',
  rules: [
    {
      action: 'block',
      expression: 'cf.waf.score lt 20',
      description: 'Block high-confidence attacks',
      enabled: true,
    },
  ],
});

// Update ruleset
const updated = await client.rulesets.update('ruleset-id', {
  zone_id: 'zone-id',
  rules: [
    {
      action: 'managed_challenge',
      expression: 'http.request.uri.path contains "/api/"',
      description: 'Challenge API traffic',
      enabled: true,
    },
  ],
});

// Add rule to ruleset
const rule = await client.rulesets.rules.create('ruleset-id', {
  zone_id: 'zone-id',
  action: 'block',
  expression: 'ip.geoip.country eq "XX"',
  description: 'Block specific country',
});

// Delete rule
await client.rulesets.rules.delete('ruleset-id', 'rule-id', {
  zone_id: 'zone-id',
});
```

**Get Phase Entry Point**:
```typescript
const entrypoint = await client.rulesets.phases.get(
  'http_request_firewall_custom',
  { zone_id: 'zone-id' }
);
```

**Update Phase Entry Point**:
```typescript
const updated = await client.rulesets.phases.update(
  'http_request_firewall_custom',
  {
    zone_id: 'zone-id',
    rules: [
      {
        action: 'execute',
        action_parameters: {
          id: 'efb7b8c949ac4650a09736fc376e9aee', // Managed ruleset ID
          overrides: {
            enabled: true,
          },
        },
        expression: 'true',
        description: 'Deploy Cloudflare Managed Ruleset',
      },
    ],
  }
);
```

### Security Best Practices

1. **Layered Defense**:
   - Use both Managed Rulesets AND Custom Rules
   - Combine WAF with Rate Limiting
   - Enable attack score detection for unknown attack variants
   - Use Bot Management for bot-specific threats

2. **Testing & Rollout**:
   - Start with `log` action to monitor before enforcing
   - Use Security Analytics to analyze traffic patterns
   - Test rules on staging zones before production
   - Monitor false positive rates during initial deployment

3. **Rule Optimization**:
   - Order rules by specificity (specific to general)
   - Use skip actions to bypass unnecessary checks for trusted traffic
   - Leverage exceptions instead of disabling entire rulesets
   - Use rule priority to control evaluation order

4. **Performance Considerations**:
   - Minimize regex usage in high-traffic expressions
   - Use field equality checks over contains when possible
   - Combine related conditions in single rules
   - Avoid overlapping rules that perform redundant checks

5. **Monitoring & Maintenance**:
   - Review Security Events regularly for attack patterns
   - Update custom rules based on new threat intelligence
   - Keep managed rulesets enabled for automatic updates
   - Set up alerts for rate limit violations and attack score spikes

6. **Rate Limiting Strategy**:
   - Set rate limits appropriate to legitimate user behavior
   - Use counting expressions to filter what counts toward rate
   - Consider NAT characteristics for shared IPs
   - Use graduated response (log → challenge → block)

7. **Attack Score Usage**:
   - Start with conservative threshold (≤20 for blocking)
   - Combine with path/endpoint filters to reduce false positives
   - Monitor individual attack vector scores (SQLi, XSS, RCE) for targeted protection
   - Use attack score class on Business plans for simpler configuration

### Plan-Specific Features

**Free**:
- Custom rules: 5
- Rate limiting rules: 1
- Managed rulesets: Free Managed Ruleset only
- Rate limiting characteristics: IP only
- Fields: Limited (Path, Verified Bot)

**Pro**:
- Custom rules: 20
- Rate limiting rules: 2
- Managed rulesets: All except Sensitive Data Detection
- Rate limiting characteristics: IP only
- Fields: Host, URI, Path, Query, Verified Bot

**Business**:
- Custom rules: 100
- Rate limiting rules: 5
- Managed rulesets: All except Sensitive Data Detection
- Rate limiting characteristics: IP, IP with NAT
- Fields: General request, Method, Source IP, User Agent
- Regex support: Yes

**Enterprise**:
- Custom rules: 1,000+
- Rate limiting rules: 5+ (or 100 with Advanced Rate Limiting)
- Managed rulesets: All (Sensitive Data Detection paid add-on)
- Rate limiting characteristics: All (IP, Headers, Query, ASN, Country, JA3/JA4, JSON field, Body, Form input, Custom)
- Fields: All including Bot Management fields, request body fields
- Actions: All including Log action
- Advanced features: Complexity-based rate limiting, custom mitigation timeout, throttling behavior
- Account-level configuration: Deploy rulesets to multiple zones

### Terraform Configuration

**Provider Setup**:
```hcl
terraform {
  required_providers {
    cloudflare = {
      source = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}
```

**Custom WAF Rule**:
```hcl
resource "cloudflare_ruleset" "waf_custom_rules" {
  zone_id     = var.zone_id
  name        = "Custom WAF Rules"
  description = "Custom protection rules"
  kind        = "zone"
  phase       = "http_request_firewall_custom"

  rules {
    action = "block"
    expression = "cf.waf.score lt 20"
    description = "Block high-confidence attacks"
    enabled = true
  }

  rules {
    action = "managed_challenge"
    expression = "http.request.uri.path contains \"/admin\" and not ip.src in {1.2.3.4}"
    description = "Challenge admin access from unknown IPs"
    enabled = true
  }
}
```

**Rate Limiting Rule**:
```hcl
resource "cloudflare_ruleset" "rate_limiting" {
  zone_id     = var.zone_id
  name        = "Rate Limiting Rules"
  kind        = "zone"
  phase       = "http_ratelimit"

  rules {
    action = "block"
    expression = "http.request.uri.path eq \"/api/submit\""
    description = "Rate limit API submissions"
    
    ratelimit {
      characteristics = ["ip.src"]
      period = 60
      requests_per_period = 10
      mitigation_timeout = 600
    }
  }
}
```

**Deploy Managed Ruleset**:
```hcl
resource "cloudflare_ruleset" "managed_waf" {
  zone_id     = var.zone_id
  name        = "Managed WAF"
  kind        = "zone"
  phase       = "http_request_firewall_managed"

  rules {
    action = "execute"
    expression = "true"
    description = "Execute Cloudflare Managed Ruleset"

    action_parameters {
      id = "efb7b8c949ac4650a09736fc376e9aee"
      overrides {
        enabled = true
        action = "log"
        
        categories {
          category = "wordpress"
          action = "block"
          enabled = true
        }

        rules {
          id = "specific-rule-id"
          action = "managed_challenge"
          enabled = true
        }
      }
    }
  }
}
```

**WAF Exception**:
```hcl
resource "cloudflare_ruleset" "waf_exceptions" {
  zone_id = var.zone_id
  name    = "WAF Exceptions"
  kind    = "zone"
  phase   = "http_request_firewall_managed"

  rules {
    action = "skip"
    expression = "http.request.uri.path contains \"/webhook\""
    description = "Skip WAF for webhook endpoint"
    
    action_parameters {
      rulesets = ["efb7b8c949ac4650a09736fc376e9aee"]
    }
  }
}
```

### Common Issues & Solutions

**Issue**: False positives blocking legitimate traffic
**Solution**: 
- Start with `log` action to monitor
- Use WAF exceptions for specific endpoints
- Override managed ruleset rules to less aggressive actions
- Combine attack score with path filters

**Issue**: Rate limiting blocking legitimate users behind NAT
**Solution**:
- Use "IP with NAT support" characteristic (Business+)
- Add additional characteristics (headers, cookies)
- Increase rate limits for shared IPs
- Use counting expressions to filter what counts

**Issue**: Rules not applying as expected
**Solution**:
- Check rule order and priority
- Verify expression syntax with Security Events
- Ensure ruleset is deployed to correct phase
- Check for conflicting skip or allow rules

**Issue**: Managed ruleset causing performance issues
**Solution**:
- Use exceptions to skip ruleset for static assets
- Override ruleset to disable unused rule categories
- Consider using zone-specific configuration
- Optimize request body size limits

**Issue**: Cannot use certain fields or actions
**Solution**:
- Verify plan-specific feature availability
- Enterprise features require contract add-ons
- Bot Management fields require BM subscription
- Response fields only work in response phase

### Quick Reference

**Key Ruleset IDs**:
- Cloudflare Managed: `efb7b8c949ac4650a09736fc376e9aee`
- OWASP Core: `4814384a9e5d4991b9815dcfc25d2f1f`
- Exposed Credentials: `c2e184081120413c86c3ab7e14069605`
- Free Managed: `77454fe2d30c4220b5701f6fdfb893ba`
- Sensitive Data Detection: `e22d83c647c64a3eae91b71b499d988e`

**Common Phases**:
- Custom rules: `http_request_firewall_custom`
- Managed rules: `http_request_firewall_managed`
- Rate limiting: `http_ratelimit`
- Response: `http_response_firewall_managed`

**Recommended Starting Configuration**:
1. Enable Cloudflare Managed Ruleset (all plans)
2. Enable OWASP Core Ruleset (Pro+)
3. Add custom rule: `cf.waf.score lt 20` with `block` action (Enterprise) or `cf.waf.score.class eq "attack"` (Business)
4. Add rate limiting for login endpoints: 5 requests per 5 minutes
5. Monitor Security Events for 1-2 weeks before adjusting

**Documentation References**:
- WAF Overview: https://developers.cloudflare.com/waf/
- Rules Language: https://developers.cloudflare.com/ruleset-engine/rules-language/
- Custom Rules: https://developers.cloudflare.com/waf/custom-rules/
- Rate Limiting: https://developers.cloudflare.com/waf/rate-limiting-rules/
- Managed Rules: https://developers.cloudflare.com/waf/managed-rules/
- Attack Score: https://developers.cloudflare.com/waf/detections/attack-score/
- API Reference: https://developers.cloudflare.com/api/

## Usage Guidelines

When helping users with Cloudflare WAF:

1. **Assess Requirements**: Understand the security needs, traffic patterns, and plan limitations
2. **Recommend Architecture**: Suggest appropriate combination of managed rules, custom rules, and rate limiting
3. **Provide Expressions**: Write correct Rules language expressions with proper syntax
4. **Show Examples**: Provide working code examples for API, SDK, or Terraform
5. **Consider Performance**: Optimize rule expressions for efficiency
6. **Plan for Testing**: Recommend using `log` action initially and gradual rollout
7. **Monitor & Iterate**: Emphasize importance of Security Events monitoring and iterative tuning

Always check the user's Cloudflare plan to recommend appropriate features and stay within their limits.
