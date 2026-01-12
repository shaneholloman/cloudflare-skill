# Cloudflare API Shield

Expert guidance for Cloudflare API Shield - comprehensive API security suite for discovery, protection, and monitoring of API endpoints.

## Overview

API Shield is Cloudflare's Enterprise-grade API security platform providing:
- **Discovery & Management**: Automatic API endpoint discovery via ML and session identifiers
- **Posture Management**: Detect vulnerabilities, abuse patterns, authentication issues
- **Runtime Protection**: Schema validation, JWT validation, mTLS, sequence enforcement

Available as Enterprise add-on (preview as non-contract service with full access, no metered fees).

## Core Features

### 1. API Discovery
Auto-discovers API endpoints via ML + session identifier analysis, normalizes paths to map attack surface.

**Path normalization example:**
```
api.example.com/profile/238 → api.example.com/profile/{var1}
api.example.com/profile/392 → api.example.com/profile/{var1}
```

**Subdomain consolidation:**
```
us-api.example.com/api/v1/users/{var1}
de-api.example.com/api/v1/users/{var1}
→ {hostVar1}.example.com/api/v1/users/{var1}
```

**Requirements for discovery:**
- 500+ requests within 10-day period
- 2xx response codes from Cloudflare edge
- Not directly from Workers

**Workflow:**
1. Endpoints cataloged in inbox view (needs review/ignored)
2. Save to Endpoint Management to unlock security features
3. Search `var1` to bulk-save endpoints with variables
4. Review non-variable endpoints for accuracy

### 2. Schema Validation
Validate requests against OpenAPI v3.0 schemas (YAML/JSON).

**Upload schema (Dashboard):**
```
Security > API Shield > Schema validation > Add validation
- Upload .yml/.yaml/.json file
- Endpoints auto-added to Endpoint Management
- Choose action: Log/Block/None
```

**Apply learned schema:**
```
Schema validation > Add validation > Apply learned schema
- Single endpoint or entire hostname
- Overwrites if learned schema exists
```

**Actions:**
- `Log`: logs to Firewall Events
- `Block`: blocks + logs non-compliant requests
- `None`: no action

**Change default action:**
```
Security > API Shield > Settings > Schema validation
Or per-endpoint: Filter endpoint > ellipses > Change action
```

**Fallthrough rule** (catch-all for unknown endpoints):
```
Security > API Shield > Settings > Fallthrough settings > Use Template
- Select hostnames
- Create custom rule with cf.api_gateway.fallthrough_triggered
```

**Body inspection:**
- Supports `application/json` content-type validation
- Matches media-ranges: `*/*`, `application/*`, `application/json`
- Accepts `charset=utf-8` parameter
- Disable origin MIME sniffing to prevent bypasses

**API operations:**
```bash
# List operations
curl -X GET "https://api.cloudflare.com/client/v4/zones/{zone_id}/api_gateway/operations" \
  -H "Authorization: Bearer {token}"

# Add operation
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/api_gateway/operations/item" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "endpoint": "/api/users/{user_id}",
    "host": "api.example.com",
    "method": "GET"
  }'

# Delete operation
curl -X DELETE "https://api.cloudflare.com/client/v4/zones/{zone_id}/api_gateway/operations/{operation_id}" \
  -H "Authorization: Bearer {token}"
```

**Limitations:**
- OpenAPI v3.0.x only (not v3.1 or v2.0)
- No external references, non-basic path templating, unique items
- 10K total operations for Enterprise (contact team for higher)
- Required fields: `type` in schema, `schema` in parameter
- Only default `style`/`explode` values supported
- No `content` field in parameters (use `schema`)
- No object type parameter validation
- `anyOf` not supported in parameter schemas

**Validated formats:**
date-time, time, date, email, hostname, ipv4, ipv6, uri, uri-reference, iri, iri-reference, int32, int64, float, double, password, uuid, byte, uint64

**Troubleshooting oneOf errors:**
- **Matches Zero**: payload missing required fields for discriminator type
- **Matches Multiple**: payload ambiguous, matches multiple subschemas

### 3. JWT Validation
Cryptographically verify JWTs, stop replay attacks/tampering, reject expired tokens.

**Setup token configuration:**
```
Security > API Shield > Settings > JSON Web Token Settings > Add configuration
- Name: "Auth0 JWT Config"
- Location: Header/Cookie + name (e.g., "Authorization")
- JWKS: Paste public keys from identity provider
```

**Auto-update JWKS via Worker:**
```javascript
// Use Worker to fetch and cache JWKS from identity provider
// Updates when keys rotate
export default {
  async scheduled(event, env, ctx) {
    const jwks = await fetch('https://auth.example.com/.well-known/jwks.json');
    await env.KV.put('jwks', await jwks.text());
  }
}
```

**Create validation rule:**
```
Security > API Shield > API Rules > Add rule
- Hostname: api.example.com
- Deselect endpoints to ignore (e.g., /token endpoint)
- Token config: "Auth0 JWT Config"
- Enforce presence: Ignore (if not all clients send JWTs) or Mark as non-compliant
- Action: Log/Block/Challenge
```

**Rate limit by JWT claim:**
```
Rate limiting > Custom rules
- Field: lookup_json_string(http.request.jwt.claims["{config_id}"][0], "sub")
- Matches: user ID claim
```

**Rate limit by user tier:**
```
(http.request.method eq "GET" and
http.host eq "api.example.com" and
http.request.uri.path matches "/premium" and
lookup_json_string(http.request.jwt.claims["{config_id}"][0], "aud") eq "free-tier")
- Action: Block if > 5 req/min
```

**Special cases:**
- **Two JWTs, different IdPs**: Create 2 configs, select both in rule, choose "Validate all configurations"
- **IdP migration**: Create 2 configs + 2 rules, adjust actions per migration state
- **Bearer prefix**: API Shield handles with/without `Bearer` prefix
- **Nested claims**: Access via dot notation: `user.email`

**Limitations:**
- JWTs in headers/cookies only (not POST body)
- Only validates endpoints in Endpoint Management

### 4. Mutual TLS (mTLS)
Bidirectional authentication via client certificates (IoT devices, machine-to-machine).

**Setup:**
```
SSL/TLS > Client Certificates > Create Certificate
- Generate Cloudflare-managed CA (all plans)
- Or upload custom CA (Enterprise, max 5, contact for higher limits)
```

**Configure mTLS rule:**
```
Security > API Shield > mTLS
- Select hostname(s)
- Choose certificate(s)
- Action: Block/Log/Challenge
```

**Test mTLS:**
```bash
# Generate client cert
openssl req -x509 -newkey rsa:4096 -keyout client-key.pem -out client-cert.pem -days 365

# Request with mTLS
curl https://api.example.com/endpoint \
  --cert client-cert.pem \
  --key client-key.pem
```

**Supports gRPC** with binary protocol buffers.

**Yubikey note:** Browser may prompt for unlock due to PKCS#11 library issue.

### 5. Sequence Mitigation (Closed Beta)
Enforce request patterns over time, detect out-of-order API abuse.

**Example: Bank transfer sequence**
```
1. GET /api/v1/users/{user_id}/accounts
2. GET /api/v1/accounts/{account_id}/balance
3. GET /api/v1/accounts/{account_id}/balance (another account)
4. POST /api/v1/transferFunds
```

**Positive security model** (enforce expected behavior):
```
Security > API Shield > Sequence mitigation > Create rule
- Require: GET /api/v1/accounts/{account_id}/balance
- Before: POST /api/v1/transferFunds
- Lookback: 10 requests, 10 minutes
- Action: Block if violated
```

**Negative security model** (block known abuse):
```
- Block sequence: GET /api/v1/users/{var1}/profile → POST /api/v1/transferFunds
- Use for authorization bug exploitation
```

**Lookback window:** 10 requests (to Endpoint Management endpoints), 10 minutes. Contact team for adjustment.

**De-duplication:** Consecutive requests to same endpoint stored once.

**Requirements:**
- Endpoints in Endpoint Management
- Session identifier configured (unique per user)
- Contact account team for beta access

### 6. Volumetric Abuse Detection
Auto-detect unusual traffic patterns, credential stuffing, endpoint scanning.

**Monitors:**
- Request volume spikes per endpoint
- Failed auth attempts
- Sequential endpoint probing

**Dashboard:** Security > API Shield > Volumetric Abuse

### 7. Authentication Posture
Analyze auth methods across API surface, identify unprotected endpoints.

**Visibility:**
- Endpoints with/without authentication
- Auth method distribution (JWT, mTLS, API key, OAuth)

### 8. BOLA Vulnerability Detection
Detect Broken Object Level Authorization via ML analysis of access patterns.

**Identifies:**
- Users accessing objects outside normal scope
- Unauthorized resource enumeration

### 9. Sequence Analytics
Visualize common request sequences, understand user flows, spot anomalies.

**View:** Security > API Shield > Sequence Analytics

### 10. GraphQL Query Protection
Mitigate deep/complex queries that cause resource exhaustion.

**Limits:**
- Query depth
- Aliases
- Array sizes

## API Operations via Cloudflare API

### Endpoint Management
```bash
# Get all operations
GET /zones/{zone_id}/api_gateway/operations

# Get single operation
GET /zones/{zone_id}/api_gateway/operations/{operation_id}

# Create operation
POST /zones/{zone_id}/api_gateway/operations/item
{
  "endpoint": "/api/v1/resource/{id}",
  "host": "api.example.com",
  "method": "POST"
}

# Bulk create
POST /zones/{zone_id}/api_gateway/operations
{
  "operations": [
    {"endpoint": "/api/users", "host": "api.example.com", "method": "GET"},
    {"endpoint": "/api/users/{id}", "host": "api.example.com", "method": "GET"}
  ]
}

# Delete operation
DELETE /zones/{zone_id}/api_gateway/operations/{operation_id}

# Delete multiple
DELETE /zones/{zone_id}/api_gateway/operations
{
  "operation_ids": ["op_id_1", "op_id_2"]
}
```

### Discovery Operations
```bash
# Get discovered operations
GET /zones/{zone_id}/api_gateway/discovery/operations

# Patch operation state
PATCH /zones/{zone_id}/api_gateway/discovery/operations/{operation_id}
{
  "state": "saved"  # or "ignored"
}

# Bulk patch
PATCH /zones/{zone_id}/api_gateway/discovery/operations
{
  "operation_ids": {
    "op_id_1": {"state": "saved"},
    "op_id_2": {"state": "ignored"}
  }
}

# Get as OpenAPI schemas
GET /zones/{zone_id}/api_gateway/discovery
```

### Configuration
```bash
# Get session identifier config
GET /zones/{zone_id}/api_gateway/configuration

# Update session identifier
PUT /zones/{zone_id}/api_gateway/configuration
{
  "auth_id_characteristics": [
    {
      "name": "Authorization",
      "type": "header"
    }
  ]
}
```

### Token Validation API
```bash
# List token configs
GET /zones/{zone_id}/api_gateway/token_validation

# Create token config
POST /zones/{zone_id}/api_gateway/token_validation
{
  "name": "Auth0 Config",
  "location": {
    "header": "Authorization"
  },
  "jwks": "{...json web key set...}"
}

# Create validation rule
POST /zones/{zone_id}/api_gateway/jwt_validation_rules
{
  "name": "Validate Auth0 tokens",
  "hostname": "api.example.com",
  "token_validation_id": "{config_id}",
  "action": "block"
}
```

## Session Identifiers

Critical for Sequence Mitigation and analytics. Configure header/cookie that uniquely IDs API users.

**Examples:**
- JWT sub claim
- Session token
- API key
- Custom user ID header

**Configure:**
```
Security > API Shield > Settings > Session Identifiers
- Type: Header/Cookie
- Name: "X-User-ID" or "Authorization"
```

## OWASP API Security Top 10 Mapping

| OWASP Issue | API Shield Solutions |
|-------------|---------------------|
| Broken Object Level Authorization | BOLA detection, Sequence mitigation, Schema validation, JWT validation, Rate Limiting |
| Broken Authentication | Authentication Posture, mTLS, JWT validation, Exposed Credential Checks, Bot Management |
| Broken Object Property Level Auth | Schema validation, JWT validation |
| Unrestricted Resource Consumption | Rate Limiting, Sequence mitigation, Bot Management, GraphQL protection |
| Broken Function Level Authorization | Schema validation, JWT validation |
| Unrestricted Sensitive Business Flows | Sequence mitigation, Bot Management, GraphQL protection |
| SSRF | Schema validation, WAF managed rules, WAF custom rules |
| Security Misconfiguration | Sequence mitigation, Schema validation, WAF managed rules, GraphQL protection |
| Improper Inventory Management | Discovery, Schema learning |
| Unsafe Consumption of APIs | JWT validation, WAF managed rules |

## Common Patterns

### Protect API with Schema + JWT
```bash
# 1. Upload OpenAPI schema
POST /zones/{zone_id}/api_gateway/user_schemas
# Endpoints auto-added to Endpoint Management

# 2. Configure JWT validation
POST /zones/{zone_id}/api_gateway/token_validation
{
  "name": "Auth0",
  "location": {"header": "Authorization"},
  "jwks": "{...}"
}

# 3. Create JWT rule
POST /zones/{zone_id}/api_gateway/jwt_validation_rules
{
  "hostname": "api.example.com",
  "token_validation_id": "{config_id}",
  "action": "block"
}

# 4. Set schema validation action
PUT /zones/{zone_id}/api_gateway/settings/schema_validation
{
  "validation_default_mitigation_action": "block"
}
```

### Progressive Rollout
```
1. Log mode: Observe false positives
   - Schema validation: Action = Log
   - JWT validation: Action = Log

2. Block mode on subset: Protect critical endpoints
   - Change specific endpoint actions to Block
   - Monitor firewall events

3. Full enforcement: Block all violations
   - Change default action to Block
   - Handle fallthrough with custom rule
```

### Fallthrough Detection (Zombie APIs)
```javascript
// WAF Custom Rule
(cf.api_gateway.fallthrough_triggered and
 http.host eq "api.example.com")

// Action: Log (to discover unknown endpoints) or Block (strict mode)
```

### Rate Limiting by User
```javascript
// Rate Limiting Rule
(http.host eq "api.example.com" and
 lookup_json_string(http.request.jwt.claims["{config_id}"][0], "sub") ne "")

// Rate: 100 requests per 60 seconds
// Counting expression: lookup_json_string(http.request.jwt.claims["{config_id}"][0], "sub")
```

## Wrangler Integration

API Shield configured via dashboard/API, but Workers can interact with protected APIs.

**Workers as API clients:**
```javascript
// Worker making authenticated API calls
export default {
  async fetch(request, env) {
    const jwt = await generateJWT(env.JWT_SECRET);
    
    const response = await fetch('https://api.example.com/data', {
      headers: {
        'Authorization': `Bearer ${jwt}`,
        'Content-Type': 'application/json'
      }
    });
    
    return response;
  }
}
```

**Workers with mTLS:**
```javascript
// Use mTLS certificate for Worker requests
export default {
  async fetch(request, env) {
    const response = await fetch('https://api.example.com/secure', {
      cf: {
        // mTLS handled by Cloudflare infrastructure
      }
    });
    
    return response;
  }
}
```

**Dynamic JWKS update Worker:**
```javascript
export default {
  async scheduled(event, env, ctx) {
    // Fetch JWKS from IdP
    const jwksResponse = await fetch('https://auth.example.com/.well-known/jwks.json');
    const jwks = await jwksResponse.json();
    
    // Update Cloudflare API Shield config via API
    await fetch(
      `https://api.cloudflare.com/client/v4/zones/${env.ZONE_ID}/api_gateway/token_validation/${env.CONFIG_ID}`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${env.CF_API_TOKEN}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({jwks: JSON.stringify(jwks)})
      }
    );
  }
}
```

## Terraform Configuration

```hcl
# Configure session identifier
resource "cloudflare_api_shield" "main" {
  zone_id = var.zone_id
  
  auth_id_characteristics {
    type = "header"
    name = "Authorization"
  }
}

# Add endpoint to management
resource "cloudflare_api_shield_operation" "users_get" {
  zone_id  = var.zone_id
  method   = "GET"
  host     = "api.example.com"
  endpoint = "/api/users/{id}"
}

# Schema validation (via API, not native Terraform resource yet)
resource "null_resource" "upload_schema" {
  provisioner "local-exec" {
    command = <<EOT
      curl -X POST "https://api.cloudflare.com/client/v4/zones/${var.zone_id}/api_gateway/user_schemas" \
        -H "Authorization: Bearer ${var.cf_api_token}" \
        -F "file=@openapi.yaml"
    EOT
  }
}

# JWT validation rule
resource "cloudflare_ruleset" "jwt_validation" {
  zone_id = var.zone_id
  name    = "API JWT Validation"
  kind    = "zone"
  phase   = "http_request_firewall_custom"

  rules {
    action = "block"
    expression = "(http.host eq \"api.example.com\" and not cf.api_gateway.jwt_claims_valid)"
    description = "Block invalid JWTs"
  }
}

# mTLS rule
resource "cloudflare_mtls_certificate" "api_clients" {
  zone_id     = var.zone_id
  name        = "API Client Cert"
  certificates = file("client-ca.pem")
}

resource "cloudflare_ruleset" "mtls_enforcement" {
  zone_id = var.zone_id
  name    = "API mTLS Enforcement"
  kind    = "zone"
  phase   = "http_request_firewall_custom"

  rules {
    action = "block"
    expression = "(http.host eq \"api.example.com\" and not cf.tls_client_auth.cert_verified)"
    description = "Require mTLS for API"
  }
}
```

## Monitoring & Analytics

**Security Events:**
```
Security > Events
- Filter: Action = block, Service = API Shield
- Shows schema/JWT validation failures
```

**Firewall Analytics:**
```
Analytics > Security
- Filter by cf.api_gateway.* fields
- Track schema violations, JWT failures, fallthrough triggers
```

**API Shield Analytics:**
```
API Shield > Analytics
- Request volume per endpoint
- Top endpoints by traffic
- Error rates
- Schema compliance trends
```

**Logpush fields:**
```json
{
  "APIGatewayAuthIDPresent": true,
  "APIGatewayRequestViolatesSchema": false,
  "APIGatewayFallthroughDetected": false,
  "JWTValidationResult": "valid",
  "ClientCertFingerprint": "abc123..."
}
```

## Architecture Patterns

### Public API (High Security)
```
Cloudflare Edge
├── API Discovery (identify all endpoints)
├── Schema Validation (enforce OpenAPI spec)
├── JWT Validation (verify tokens)
├── Rate Limiting (per-user limits)
├── Bot Management (filter automated abuse)
└── Origin → API server
```

### Partner API (mTLS + Schema)
```
Cloudflare Edge
├── mTLS (verify client certificates)
├── Schema Validation (validate payloads)
├── Sequence Mitigation (enforce order)
└── Origin → API server
```

### Internal API (Discovery + Monitoring)
```
Cloudflare Edge
├── API Discovery (map shadow APIs)
├── Schema Learning (auto-generate specs)
├── Authentication Posture (audit coverage)
└── Origin → API server
```

### Multi-Region API
```
{hostVar1}.api.example.com/v1/resource/{id}

- API Discovery consolidates subdomains
- Single schema applies to all regions
- JWT config shared across regions
- Per-region rate limits via custom rules
```

## Troubleshooting

**Schema validation blocking valid requests:**
1. Check Firewall Events for violation details
2. Review schema: `Security > API Shield > Schema validation > Settings > Download schema`
3. Test locally: Use Swagger Editor to validate request against schema
4. Temporarily set action to Log, collect samples
5. Update schema or adjust validation

**JWT validation failing:**
1. Verify JWKS public keys match IdP
2. Check JWT expiration (`exp` claim)
3. Confirm header/cookie name in config
4. Test JWT at jwt.io
5. Check for clock skew (grace period in IdP config)

**API Discovery not finding endpoints:**
1. Ensure 500+ requests in 10 days
2. Check response codes (must be 2xx)
3. Verify not coming from Workers directly
4. Review session identifier config
5. Wait for ML model to process (daily updates)

**False positives in sequence mitigation:**
1. Adjust lookback window (contact team)
2. Review sequence definition
3. Check session identifier uniqueness
4. Consider negative vs. positive security model

**Performance impact:**
- Schema validation: ~1-2ms latency
- JWT validation: ~0.5-1ms latency
- mTLS: ~2-5ms latency (TLS handshake)
- Sequence tracking: ~0.5ms latency

## Best Practices

1. **Start with Discovery**: Map surface before applying enforcement
2. **Progressive rollout**: Log → Block on critical endpoints → Full enforcement
3. **Session identifiers**: Must be unique per user for sequences/analytics
4. **Schema quality**: Use Swagger Editor to validate OpenAPI compliance
5. **JWKS rotation**: Automate with Worker scheduled tasks
6. **Fallthrough rules**: Catch zombie APIs, legacy endpoints
7. **Monitoring**: Set up Logpush, alert on validation failures
8. **Rate limiting**: Combine with JWT claims for per-user/tier limits
9. **Bot Management**: Layer with API Shield for comprehensive protection
10. **Test in staging**: Validate configs before production deployment

## Field Reference (Firewall Rules)

```javascript
// API Gateway Fields
cf.api_gateway.auth_id_present          // Boolean: Session ID present
cf.api_gateway.request_violates_schema  // Boolean: Schema violation
cf.api_gateway.fallthrough_triggered    // Boolean: No matching endpoint

// JWT Fields (requires token validation config)
cf.api_gateway.jwt_claims_valid         // Boolean: JWT validated
lookup_json_string(http.request.jwt.claims["{config_id}"][0], "claim_name")  // Extract claim

// mTLS Fields
cf.tls_client_auth.cert_verified        // Boolean: Client cert valid
cf.tls_client_auth.cert_fingerprint_sha256  // String: Cert fingerprint
```

## Resources

- **Docs**: https://developers.cloudflare.com/api-shield/
- **API Reference**: https://developers.cloudflare.com/api/resources/api_gateway/
- **Blog**: https://blog.cloudflare.com/api-shield/
- **Support**: Enterprise customers contact account team

## Availability Summary

| Feature | Availability |
|---------|-------------|
| mTLS (CF-managed CA) | All plans |
| Endpoint Management | All plans (limited operations) |
| Schema Validation | All plans (limited operations) |
| API Discovery | Enterprise only |
| JWT Validation | Enterprise (API Shield add-on) |
| Sequence Mitigation | Enterprise (closed beta) |
| BOLA Detection | Enterprise (API Shield add-on) |
| Volumetric Abuse | Enterprise (API Shield add-on) |
| Full API Shield Suite | Enterprise add-on |

Enterprise limits: 10K operations (contact for higher), non-contract preview available.
