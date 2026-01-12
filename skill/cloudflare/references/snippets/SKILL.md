# Cloudflare Snippets Skill

## Description
Expert guidance for **Cloudflare Snippets ONLY** - a lightweight JavaScript-based edge logic platform for modifying HTTP requests and responses. Snippets run as part of the Ruleset Engine and are included at no additional cost on paid plans (Pro, Business, Enterprise).

## Core Concepts

### What are Snippets?
- **Lightweight edge logic**: Fast, declarative JavaScript code for request/response modifications
- **Part of Ruleset Engine**: Execute based on filter expressions, not HTTP routes
- **Sequential execution**: Multiple snippets can run on the same request, each modifying the result before passing to the next
- **Included in paid plans**: No additional cost for 99% of use cases
- **Not Workers**: Snippets are specifically designed for traffic modifications, not compute-heavy tasks

### Key Differences: Snippets vs Workers

| Capability | Snippets | Workers |
|-----------|----------|---------|
| **Trigger mechanism** | Ruleset Engine filters (bot score, country, cookies, headers, etc.) | HTTP routes only |
| **Execution time** | 5ms max | 30s HTTP / 15min cron |
| **Memory** | 2MB max | 128MB max |
| **Package size** | 32KB max | 5MB max |
| **Subrequests** | 1-5 (plan dependent) | 1000/request |
| **Cost** | Included in paid plans | Usage-based billing |
| **State/Storage** | ❌ No (KV, D1, DO) | ✅ Yes |
| **Environment variables** | 8 per snippet (1KB each) | 64 per worker (5KB each) |
| **Cron triggers** | ❌ | ✅ |
| **Deployment CLI** | ❌ No Wrangler | ✅ Wrangler |

### Plan Limits

| Plan | Snippets | Subrequests |
|------|----------|-------------|
| Free | 0 | 0 |
| Pro | 25 | 2 |
| Business | 50 | 3 |
| Enterprise | 300 | 5 |

## Architecture & Execution

### Execution Flow
```
1. Request arrives → Cloudflare evaluates all snippet rules
2. Matching rules added to Snippets table (by snippet ID)
3. Snippets Internal Worker Service processes table sequentially
4. Each snippet receives modified request from previous snippet
5. Final modified request continues through execution workflow
```

### Execution Order in Rules Pipeline
Snippets run **AFTER** most other rules products:
1. Single Redirects
2. URL Rewrite Rules
3. Configuration Rules
4. Origin Rules
5. Bulk Redirects
6. Managed Transforms
7. Request Header Transform Rules
8. Cache Rules
9. **→ Snippets** ← (you are here)
10. Cloud Connector

**Important**: Snippets run BEFORE origin fetch, can see modifications from earlier rules, and later rules/Workers won't see snippet changes unless they explicitly fetch the modified request.

### Snippet Elements
Every snippet requires:
1. **Code snippet**: JavaScript code (32KB max)
2. **Snippet rule**: Filter expression defining when to run
   - Each snippet = exactly one rule
   - Use Ruleset Engine filter expressions

## Code Patterns

### Basic Structure
```javascript
export default {
  async fetch(request) {
    // Your logic here
    const response = await fetch(request);
    return response;
  },
};
```

### Request Modification Pattern
```javascript
export default {
  async fetch(request) {
    // Clone request to modify headers
    const modifiedRequest = new Request(request, {
      headers: new Headers(request.headers),
    });
    
    // Add/modify headers
    modifiedRequest.headers.set("X-Custom-Header", "value");
    modifiedRequest.headers.delete("X-Remove-Header");
    
    // Pass to origin
    return fetch(modifiedRequest);
  },
};
```

### Response Modification Pattern
```javascript
export default {
  async fetch(request) {
    const response = await fetch(request);
    
    // Clone response to modify it
    const newResponse = new Response(response.body, response);
    
    // Modify headers
    newResponse.headers.set("X-Custom-Header", "value");
    newResponse.headers.delete("X-Remove-Header");
    
    return newResponse;
  },
};
```

### URL Rewriting Pattern
```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    
    // Modify path
    url.pathname = url.pathname.replace(/^\/old/, "/new");
    
    // Modify query params
    const params = new URLSearchParams(url.search);
    params.set("utm_source", "cloudflare");
    url.search = params.toString();
    
    // Create new request with modified URL
    const modifiedRequest = new Request(url, request);
    return fetch(modifiedRequest);
  },
};
```

### Conditional Response Pattern
```javascript
export default {
  async fetch(request) {
    // Check condition (bot score, cookie, header, etc.)
    const botScore = request.cf.botManagement.score;
    
    if (botScore < 30) {
      // Serve custom response
      return new Response("Access denied", { status: 403 });
    }
    
    // Otherwise, continue normally
    return fetch(request);
  },
};
```

### Redirect Pattern
```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    
    if (url.pathname.startsWith("/old-path")) {
      const newUrl = url.origin + "/new-path";
      return Response.redirect(newUrl, 301);
    }
    
    return fetch(request);
  },
};
```

### Custom Caching Pattern
```javascript
const CACHE_DURATION = 30 * 24 * 60 * 60; // 30 days

export default {
  async fetch(request) {
    const cache = caches.default;
    const cacheKey = new Request(request.url, { method: "GET" });
    
    // Try cache first
    let response = await cache.match(cacheKey);
    
    if (!response) {
      // Cache miss - fetch from origin
      response = await fetch(request);
      response = new Response(response.body, response);
      response.headers.set("Cache-Control", `s-maxage=${CACHE_DURATION}`);
      
      // Store in cache
      await cache.put(cacheKey, response.clone());
    }
    
    return response;
  },
};
```

## Common Use Cases

### 1. Header Manipulation
**Use for**: Adding/removing/modifying request or response headers
```javascript
// Add timestamp header
const timestamp = Date.now().toString(16);
modifiedRequest.headers.set("X-Hex-Timestamp", timestamp);

// Remove sensitive headers
newResponse.headers.delete("X-Powered-By");
newResponse.headers.delete("Server");

// Set security headers
newResponse.headers.set("X-Frame-Options", "DENY");
newResponse.headers.set("X-Content-Type-Options", "nosniff");
```

### 2. Cookie Modification
**Use for**: Setting dynamic cookies, expiry times, A/B testing
```javascript
const expiry = new Date(Date.now() + 7 * 86400000).toUTCString();
const group = request.headers.get("userGroup") == "premium" ? "A" : "B";
response.headers.append(
  "Set-Cookie",
  `testGroup=${group}; Expires=${expiry}; path=/`
);
```

### 3. Bot Management
**Use for**: Routing bots, honeypots, rate limiting
```javascript
// Access bot score from request.cf
const botScore = request.cf.botManagement.score;

if (botScore < 30) {
  // Send to honeypot
  const honeypot = "https://honeypot.example.com/";
  return fetch(honeypot, request);
}

// Add bot data to request headers for origin
modifiedRequest.headers.set("X-Bot-Score", botScore.toString());
modifiedRequest.headers.set("X-Bot-Verified", request.cf.botManagement.verifiedBot);
```

### 4. Geographic Routing
**Use for**: Country-based redirects, localization
```javascript
const country = request.cf.country;
const redirectMap = {
  US: "https://example.com/us",
  GB: "https://example.com/uk",
  FR: "https://example.com/fr",
  DE: "https://example.com/de",
};

if (redirectMap[country]) {
  return Response.redirect(redirectMap[country], 302);
}
```

### 5. Authentication
**Use for**: JWT validation, API key verification, pre-shared keys
```javascript
// Simple header-based auth
const apiKey = request.headers.get("X-API-Key");
const validKey = "your-secret-key-here";

if (apiKey !== validKey) {
  return new Response("Unauthorized", { status: 401 });
}

// For JWT validation, see full example in documentation
```

### 6. Origin Failover
**Use for**: Retry logic, multiple origins, error handling
```javascript
const response = await fetch(request);

// If primary origin fails, try backup
if (!response.ok && !response.redirected) {
  const newRequest = new Request(request);
  newRequest.headers.set("X-Rerouted", "1");
  
  const url = new URL(request.url);
  url.hostname = "backup.example.com";
  
  return fetch(url, newRequest);
}

return response;
```

### 7. Query String Management
**Use for**: Adding/removing query parameters, UTM tracking
```javascript
const url = new URL(request.url);
const params = new URLSearchParams(url.search);

// Remove specific params
params.delete("fbclid");
params.delete("gclid");

// Add tracking params
if (request.headers.get("User-Agent").includes("Facebook")) {
  params.set("utm_source", "facebook");
}

url.search = params.toString();
```

### 8. A/B Testing
**Use for**: Traffic splitting, feature flags, variant testing
```javascript
// Check for existing test cookie
let testGroup = request.cookies.get("test_group");

if (!testGroup) {
  // Assign random group
  testGroup = Math.random() < 0.5 ? "A" : "B";
}

const response = await fetch(request);
const newResponse = new Response(response.body, response);

// Set cookie with expiry
const expiry = new Date(Date.now() + 30 * 86400000).toUTCString();
newResponse.headers.append(
  "Set-Cookie",
  `test_group=${testGroup}; Expires=${expiry}; Path=/`
);

return newResponse;
```

### 9. Maintenance Mode
**Use for**: Planned downtime, service unavailable pages
```javascript
export default {
  async fetch(request) {
    return new Response(`
      <!DOCTYPE html>
      <html>
        <head><title>Maintenance</title></head>
        <body>
          <h1>We'll Be Right Back!</h1>
          <p>Site is undergoing maintenance. Check back soon!</p>
        </body>
      </html>
    `, {
      status: 503,
      headers: { "Content-Type": "text/html" }
    });
  },
};
```

### 10. CORS Headers
**Use for**: Cross-origin requests, API endpoints
```javascript
const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type, Authorization",
  "Access-Control-Max-Age": "86400",
};

// Handle preflight
if (request.method === "OPTIONS") {
  return new Response(null, { headers: corsHeaders, status: 200 });
}

const response = await fetch(request);
const newResponse = new Response(response.body, response);

// Add CORS headers to response
Object.keys(corsHeaders).forEach(header => {
  newResponse.headers.set(header, corsHeaders[header]);
});
```

## Available APIs

### Workers Runtime APIs (Subset)
Snippets support essential Workers runtime APIs:
- ✅ **Fetch API**: `fetch(request)`, Request/Response objects
- ✅ **Cache API**: `caches.default` for programmatic caching
- ✅ **HTMLRewriter**: DOM manipulation for HTML responses
- ✅ **request.cf object**: Cloudflare-specific metadata
  - `request.cf.country` - ISO 3166-1 Alpha 2 country code
  - `request.cf.botManagement.score` - Bot score (0-100)
  - `request.cf.botManagement.verifiedBot` - Boolean
  - `request.cf.tlsVersion` - TLS version
  - `request.cf.colo` - Cloudflare data center code
- ✅ **Web Crypto API**: Cryptographic operations
- ✅ **Standard JavaScript**: Date, Math, JSON, etc.

### NOT Available
- ❌ Environment variables (use with limits: 8 vars, 1KB each)
- ❌ KV, D1, Durable Objects, R2
- ❌ Bindings to services
- ❌ Console.log (no observability/logs)
- ❌ Cron triggers
- ❌ WebSockets
- ❌ Wrangler CLI deployment

## Configuration Methods

### 1. Dashboard (GUI)
```
1. Go to zone → Rules → Snippets
2. Create Snippet or select template
3. Enter snippet name (a-z, 0-9, _ only, cannot change later)
4. Write JavaScript code (32KB max)
5. Configure snippet rule:
   - Expression Builder or Expression Editor
   - Use Ruleset Engine filter expressions
6. Deploy or Save as Draft
```

### 2. API
```bash
# Create/update snippet
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/snippets/$SNIPPET_NAME" \
  --request PUT \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --form "files=@example.js" \
  --form "metadata={\"main_module\": \"example.js\"}"

# Create snippet rule
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/snippets/snippet_rules" \
  --request PUT \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "rules": [
      {
        "description": "Trigger snippet on specific cookie",
        "enabled": true,
        "expression": "http.cookie eq \"a=b\"",
        "snippet_name": "snippet_name_01"
      }
    ]
  }'

# List snippets
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/snippets" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"

# Get snippet content
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/snippets/$SNIPPET_NAME/content" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"

# Delete snippet
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/snippets/$SNIPPET_NAME" \
  --request DELETE \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

**API Endpoints**:
- `GET /zones/{zone_id}/snippets` - List all snippets
- `PUT /zones/{zone_id}/snippets/{snippet_name}` - Create/update
- `GET /zones/{zone_id}/snippets/{snippet_name}` - Get details
- `GET /zones/{zone_id}/snippets/{snippet_name}/content` - Get code
- `DELETE /zones/{zone_id}/snippets/{snippet_name}` - Delete
- `GET /zones/{zone_id}/snippets/snippet_rules` - List rules
- `PUT /zones/{zone_id}/snippets/snippet_rules` - Update rules
- `DELETE /zones/{zone_id}/snippets/snippet_rules` - Delete all rules

**Required permissions**: `Zone > Snippets > Edit`

### 3. Terraform
```hcl
resource "cloudflare_snippet" "my_snippet" {
  zone_id     = "<ZONE_ID>"
  name        = "my_test_snippet_1"
  main_module = "file1.js"
  
  files {
    name    = "file1.js"
    content = file("file1.js")
  }
}

resource "cloudflare_snippet_rules" "cookie_snippet_rule" {
  zone_id = "<ZONE_ID>"
  
  rules {
    enabled      = true
    expression   = "http.cookie eq \"a=b\""
    description  = "Trigger snippet on specific cookie"
    snippet_name = "my_test_snippet_1"
  }
  
  depends_on = [cloudflare_snippet.my_snippet]
}
```

## Filter Expressions

### Common Expression Patterns
```javascript
// Cookie-based
http.cookie contains "session_id"
http.cookie eq "test=value"

// Header-based
http.request.headers["user-agent"] contains "Mobile"
http.request.headers["x-api-key"] eq "secret"

// Bot score
cf.bot_management.score lt 30
cf.bot_management.verified_bot

// Geographic
ip.geoip.country eq "US"
ip.geoip.country in {"US" "CA" "MX"}

// Path-based
http.request.uri.path matches "^/api/"
http.request.uri.path contains "/admin"

// Combination
(cf.bot_management.score lt 30) and (ip.geoip.country eq "US")

// Hostname
http.host eq "example.com"
http.host in {"example.com" "www.example.com"}
```

### Available Fields
- `http.request.uri` - Full URI
- `http.request.uri.path` - Path component
- `http.request.uri.query` - Query string
- `http.host` - Hostname
- `http.request.method` - HTTP method
- `http.request.headers` - Request headers
- `http.cookie` - Cookie values
- `ip.geoip.country` - Country code
- `cf.bot_management.score` - Bot score (0-100)
- `cf.threat_score` - Threat score
- `cf.edge.server_port` - Server port (80, 443)

## Migration Guides

### From VCL (Varnish)
Cloudflare provides VCL converter tools:
- Upload VCL code → auto-generates Snippets + Rules
- Convert operational logic to modern JavaScript
- No need to maintain legacy VCL infrastructure

**Common VCL → Snippet patterns**:
```vcl
# VCL: Set header
set req.http.X-Custom = "value";

# Snippet equivalent:
modifiedRequest.headers.set("X-Custom", "value");
```

### From Apache .htaccess
Upload .htaccess files → converter generates Snippets

**Common .htaccess → Snippet patterns**:
```apache
# .htaccess: Redirect
RedirectPermanent /old-path /new-path

# Snippet equivalent:
if (url.pathname === "/old-path") {
  return Response.redirect("/new-path", 301);
}
```

### From NGINX config
Upload NGINX config → converter generates Snippets

### From EdgeWorkers (Akamai)
Manual migration - map EdgeWorkers code to Snippet patterns

### From Lambda@Edge (AWS)
Manual migration - adapt Lambda functions to Snippet structure

### From Workers to Snippets
**When to migrate**:
- Worker only does header/URL modifications
- No bindings, KV, D1, Durable Objects needed
- Want to eliminate usage-based billing
- Need advanced request matching (Ruleset Engine filters)

**Migration steps**:
1. Extract core logic from Worker
2. Remove bindings, environment variables (or limit to 8)
3. Ensure execution < 5ms, memory < 2MB, code < 32KB
4. Create Snippet rule expression (replaces route pattern)
5. Test with dashboard preview feature

### From Snippets to Workers
**When to migrate**:
- Exceeds 5ms execution / 2MB memory / 32KB code limits
- Need persistent storage (KV, D1, Durable Objects)
- Need compute-intensive operations (AI, image transform)
- Need observability (logs, metrics)
- Need Wrangler CLI deployment
- Need cron triggers, WebSockets, environment variables (>8)

## Best Practices

### Performance
1. **Keep code minimal**: 32KB limit, but aim for <10KB
2. **Avoid blocking operations**: 5ms execution limit
3. **Clone only when needed**: `new Request(request)` creates copy
4. **Cache strategically**: Use `caches.default` for repeated data
5. **Limit subrequests**: Plan-based limits (2-5)

### Security
1. **Validate input**: Never trust user data
2. **Use Web Crypto API**: For cryptographic operations
3. **Sanitize headers**: Remove sensitive information
4. **Check bot scores**: Use `request.cf.botManagement.score`
5. **Rate limit carefully**: Snippets run on every matching request

### Debugging
1. **Test in dashboard**: Use HTTP/Preview tabs
2. **Start simple**: Test with basic logic first
3. **Use custom headers**: Add debug headers to responses
4. **Check execution order**: Remember Snippets run after Cache Rules
5. **Monitor subrequest limits**: Track usage by plan

### Naming & Organization
1. **Descriptive names**: `add_security_headers`, `bot_honeypot`
2. **Use underscores**: Only a-z, 0-9, _ allowed
3. **Cannot rename**: Choose wisely, name is permanent
4. **Group by function**: headers, routing, auth, etc.
5. **Document rules**: Use clear rule descriptions

### Error Handling
```javascript
export default {
  async fetch(request) {
    try {
      const response = await fetch(request);
      
      if (!response.ok) {
        // Handle error response
        return new Response("Service unavailable", { status: 503 });
      }
      
      return response;
    } catch (error) {
      // Handle fetch failure
      return new Response("Error: " + error.message, { status: 500 });
    }
  },
};
```

### Testing Strategies
1. **Use snippet rules**: Test with specific cookies/headers
2. **Progressive rollout**: Start with small traffic percentage
3. **A/B test**: Compare snippet vs no-snippet performance
4. **Monitor origin load**: Check if caching is working
5. **Test edge cases**: Empty values, missing headers, etc.

## Troubleshooting

### Common Issues

#### 1. Snippet not executing
- **Check rule expression**: Verify it matches your traffic
- **Check DNS proxy**: Must be orange-clouded (proxied)
- **Check execution order**: Snippets run after Cache Rules
- **Check plan limits**: Free plans don't have Snippets

#### 2. Execution timeout (5ms)
- **Reduce complexity**: Simplify logic
- **Avoid blocking**: No synchronous operations
- **Migrate to Worker**: If consistently >5ms

#### 3. Subrequest limit exceeded
- **Check plan limits**: Pro=2, Business=3, Enterprise=5
- **Count redirects**: Each redirect counts as subrequest
- **Optimize fetches**: Combine requests where possible

#### 4. Snippet code too large (32KB)
- **Minimize code**: Remove comments, whitespace
- **Split functionality**: Use multiple snippets
- **Migrate to Worker**: If code >32KB required

#### 5. Headers not appearing
- **Check cloning**: Must clone Response to modify
- **Check case**: Header names are case-insensitive
- **Check origin**: Origin may override headers

#### 6. Caching not working
- **Check Cache-Control**: Must set appropriate directives
- **Check cache key**: Use consistent keys
- **Check Cloudflare caching**: May need Cache Rules

#### 7. Cannot modify immutable response
```javascript
// ❌ Wrong - response is immutable
const response = await fetch(request);
response.headers.set("X-Custom", "value"); // ERROR

// ✅ Correct - clone first
const response = await fetch(request);
const newResponse = new Response(response.body, response);
newResponse.headers.set("X-Custom", "value");
```

### Debug Techniques
```javascript
// Add debug headers to response
newResponse.headers.set("X-Debug-Executed", "true");
newResponse.headers.set("X-Debug-Timestamp", Date.now().toString());
newResponse.headers.set("X-Debug-Country", request.cf.country);

// Return debug info as response
return new Response(JSON.stringify({
  url: request.url,
  method: request.method,
  headers: Object.fromEntries(request.headers),
  cf: request.cf,
}, null, 2), {
  headers: { "Content-Type": "application/json" }
});
```

## Advanced Patterns

### HTML Rewriting
```javascript
export default {
  async fetch(request) {
    class AttributeRewriter {
      constructor(attributeName) {
        this.attributeName = attributeName;
      }
      
      element(element) {
        const attr = element.getAttribute(this.attributeName);
        if (attr) {
          element.setAttribute(
            this.attributeName,
            attr.replace("oldsite.com", "newsite.com")
          );
        }
      }
    }
    
    const rewriter = new HTMLRewriter()
      .on("a", new AttributeRewriter("href"))
      .on("img", new AttributeRewriter("src"));
    
    const response = await fetch(request);
    const contentType = response.headers.get("Content-Type");
    
    if (contentType?.startsWith("text/html")) {
      return rewriter.transform(response);
    }
    
    return response;
  },
};
```

### JSON Response Modification
```javascript
export default {
  async fetch(request) {
    const response = await fetch(request);
    
    try {
      const data = await response.json();
      
      // Remove sensitive fields
      delete data.internal_id;
      delete data.api_key;
      
      // Add computed fields
      data.computed_at = Date.now();
      
      return Response.json(data);
    } catch (err) {
      // Not JSON, return original
      return response;
    }
  },
};
```

### Request Signing
```javascript
export default {
  async fetch(request) {
    const secret = "your-secret-key";
    const signature = request.headers.get("X-Signature");
    
    if (!signature) {
      return new Response("Missing signature", { status: 401 });
    }
    
    // Verify signature using Web Crypto API
    const encoder = new TextEncoder();
    const key = await crypto.subtle.importKey(
      "raw",
      encoder.encode(secret),
      { name: "HMAC", hash: "SHA-256" },
      false,
      ["verify"]
    );
    
    const data = await request.clone().text();
    const verified = await crypto.subtle.verify(
      "HMAC",
      key,
      hexToBuffer(signature),
      encoder.encode(data)
    );
    
    if (!verified) {
      return new Response("Invalid signature", { status: 403 });
    }
    
    return fetch(request);
  },
};
```

### Bulk Redirects via Map
```javascript
export default {
  async fetch(request) {
    const redirectMap = {
      "/old-page": "/new-page",
      "/legacy-url": "/modern-url",
      "/deprecated": "/current",
    };
    
    const url = new URL(request.url);
    const newPath = redirectMap[url.pathname];
    
    if (newPath) {
      url.pathname = newPath;
      return Response.redirect(url.toString(), 301);
    }
    
    return fetch(request);
  },
};
```

### Slow Down Requests (Rate Limiting)
```javascript
export default {
  async fetch(request) {
    const botScore = request.cf.botManagement.score;
    
    // Slow down suspicious requests
    if (botScore < 50) {
      const delay = 5; // seconds
      await new Promise(resolve => setTimeout(resolve, delay * 1000));
    }
    
    return fetch(request);
  },
};
```

## Integration with Other Cloudflare Products

### With Workers
- **Snippets run first**: Execute before Workers route
- **Pass data via headers**: Workers can read snippet-added headers
- **Separate paths**: Don't run on same URL to avoid conflicts
- **Use subrequests**: Snippet → Worker → Origin

Example flow:
```
Request → Snippet (add headers) → Worker (process) → Origin
```

### With Cache Rules
- **Cache Rules run BEFORE Snippets**: Snippet sees cached response
- **Snippets can override**: Use Cache API programmatically
- **Custom cache keys**: Snippet can create unique keys

### With Transform Rules
- **Transform Rules run BEFORE Snippets**: Snippet sees transformed request
- **Snippets extend functionality**: Add logic Transform Rules can't do
- **Use both together**: Transform for simple, Snippets for complex

### With WAF
- **WAF runs early**: Before Snippets
- **Blocked requests never reach Snippets**: WAF terminates first
- **Use bot score**: Access WAF data in `request.cf`

### With Zaraz / Page Shield
- **Snippets run before HTML delivery**: Can modify scripts
- **Use HTMLRewriter**: Add/remove/modify script tags
- **CSP headers**: Snippets can set Content-Security-Policy

## Decision Framework: When to Use Snippets

### ✅ Use Snippets when:
- Modifying HTTP headers (request or response)
- Performing URL rewrites/redirects with logic
- Routing based on bot score, country, cookies
- Simple authentication (API keys, pre-shared secrets)
- Custom caching strategies
- A/B testing with cookies
- Setting security headers
- Geographic routing
- Origin failover logic
- Query string manipulation
- Cookie management
- Serving custom responses (maintenance pages)
- HTML content rewriting (links, attributes)
- Migrating from VCL, .htaccess, NGINX configs
- Need advanced request matching (Ruleset Engine)
- Want zero additional cost

### ❌ Don't use Snippets when:
- Need persistent storage (use Workers + KV/D1/DO)
- Compute-intensive tasks (use Workers + Workers AI)
- Need environment variables >8 or >1KB each
- Need observability/logs (use Workers)
- Execution time >5ms (use Workers)
- Memory usage >2MB (use Workers)
- Code size >32KB (use Workers)
- Need cron triggers (use Workers)
- Need WebSockets (use Workers)
- Need TypeScript/Python/Rust (use Workers)
- Need CLI deployment/testing (use Workers)
- Need version management/rollback (use Workers)
- Building full-stack applications (use Workers/Pages)

## Quick Reference

### Snippet Naming Rules
- Characters: `a-z`, `0-9`, `_` only
- Unique per zone
- Cannot change after creation
- Descriptive names recommended

### Execution Limits
- **Time**: 5ms max
- **Memory**: 2MB max
- **Code size**: 32KB max
- **Subrequests**: 1-5 (plan dependent)
- **Environment vars**: 8 max (1KB each)

### HTTP Methods
```javascript
request.method // GET, POST, PUT, DELETE, etc.
```

### Response Constructors
```javascript
// Plain text
new Response("Hello", { status: 200 })

// JSON
Response.json({ key: "value" })

// HTML
new Response("<h1>Hi</h1>", { 
  headers: { "Content-Type": "text/html" }
})

// Redirect
Response.redirect("https://example.com", 301)
```

### Header Operations
```javascript
// Request headers
request.headers.get("X-Header")
request.headers.has("X-Header")
request.headers.set("X-Header", "value")
request.headers.delete("X-Header")

// Response headers (must clone first)
const res = new Response(response.body, response);
res.headers.set("X-Header", "value")
res.headers.append("Set-Cookie", "value")
res.headers.delete("X-Header")
```

### URL Operations
```javascript
const url = new URL(request.url);
url.hostname    // "example.com"
url.pathname    // "/path/to/page"
url.search      // "?query=value"
url.searchParams.get("query") // "value"
```

### Cloudflare Properties
```javascript
request.cf.country              // "US"
request.cf.botManagement.score  // 0-100
request.cf.botManagement.verifiedBot // true/false
request.cf.colo                 // "SFO"
request.cf.tlsVersion           // "TLSv1.3"
```

## Resources

### Official Documentation
- [Snippets Overview](https://developers.cloudflare.com/rules/snippets/)
- [How Snippets Work](https://developers.cloudflare.com/rules/snippets/how-it-works/)
- [Examples Gallery](https://developers.cloudflare.com/rules/snippets/examples/)
- [When to Use Snippets vs Workers](https://developers.cloudflare.com/rules/snippets/when-to-use/)
- [API Reference](https://developers.cloudflare.com/api/resources/snippets/)
- [Terraform Provider](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs)

### Related Documentation
- [Ruleset Engine](https://developers.cloudflare.com/ruleset-engine/)
- [Filter Expressions](https://developers.cloudflare.com/ruleset-engine/rules-language/expressions/)
- [Workers Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)
- [Cache API](https://developers.cloudflare.com/workers/runtime-apis/cache/)
- [HTMLRewriter](https://developers.cloudflare.com/workers/runtime-apis/html-rewriter/)

### Community
- [Cloudflare Community Forums](https://community.cloudflare.com/)
- [Discord](https://discord.gg/cloudflaredev)
- [Blog: Snippets Announcement](https://blog.cloudflare.com/snippets-announcement)

---

## Skill Usage Instructions

When a user asks about Cloudflare Snippets:

1. **Always clarify**: Is this about Snippets specifically, or Workers?
2. **Emphasize constraints**: 5ms, 2MB, 32KB, no persistent storage
3. **Focus on use cases**: Header manipulation, routing, simple auth, caching
4. **Provide code examples**: Always show complete, working code
5. **Mention execution order**: Snippets run after Cache Rules
6. **Consider alternatives**: Suggest Workers if limits are exceeded
7. **Use proper patterns**: Clone requests/responses, handle errors
8. **Include filter expressions**: Every snippet needs a rule
9. **Reference plan limits**: Free=0, Pro=25, Business=50, Enterprise=300
10. **Provide deployment methods**: Dashboard, API, or Terraform

**DO NOT confuse with Workers** - they are different products with different capabilities, pricing, and use cases.
