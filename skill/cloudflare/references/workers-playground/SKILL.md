# Cloudflare Workers Playground Skill

## Overview

Cloudflare Workers Playground is a browser-based sandbox for instantly experimenting with, testing, and deploying Cloudflare Workers without authentication or setup. This skill provides patterns, APIs, and best practices specifically for Workers Playground development.

## Core Concepts

### What is Workers Playground?

Workers Playground is:
- **Browser-based IDE**: No local setup, authentication, or tools required
- **Instant preview**: Real-time code execution with live HTTP testing
- **Shareable**: Generate unique URLs to share code examples
- **Deployable**: One-click deployment to Cloudflare's global network
- **DevTools integrated**: Built-in console, network inspector, and debugger

Browser Support: Firefox and Chrome desktop only (Safari shows `PreviewRequestFailed` error).

### Entry Point: fetch Handler

Every Worker in the Playground uses the ES modules format with a `fetch` handler:

```javascript
export default {
  async fetch(request, env, ctx) {
    return new Response('Hello World!');
  },
};
```

**Parameters:**
- `request` (Request): Incoming HTTP request object
- `env` (Object): Bindings to KV, D1, Durable Objects, secrets, etc.
- `ctx` (ExecutionContext): Context methods like `waitUntil()` and `passThroughOnException()`

## Code Patterns

### 1. Basic Response Patterns

#### Return Plain Text
```javascript
export default {
  fetch(request, env, ctx) {
    return new Response('Hello Cloudflare Workers!');
  },
};
```

#### Return JSON
```javascript
export default {
  fetch(request, env, ctx) {
    const data = {
      status: 'success',
      message: 'API response',
      timestamp: Date.now()
    };
    
    return new Response(JSON.stringify(data), {
      headers: {
        'Content-Type': 'application/json',
      },
    });
  },
};
```

#### Return HTML
```javascript
export default {
  fetch(request, env, ctx) {
    const html = `
      <!DOCTYPE html>
      <html>
        <head><title>Workers Playground</title></head>
        <body>
          <h1>Hello from Workers!</h1>
        </body>
      </html>
    `;
    
    return new Response(html, {
      headers: {
        'Content-Type': 'text/html',
      },
    });
  },
};
```

### 2. Multi-Module Workers

Import HTML, CSS, or other modules:

```javascript
// worker.js
import welcome from './welcome.html';

export default {
  fetch(request, env, ctx) {
    console.log('Request received!');
    
    return new Response(welcome, {
      headers: {
        'content-type': 'text/html',
      },
    });
  },
};
```

### 3. Request Processing

#### URL Routing
```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    switch (url.pathname) {
      case '/':
        return new Response('Home page');
      case '/api':
        return new Response(JSON.stringify({ message: 'API endpoint' }), {
          headers: { 'Content-Type': 'application/json' },
        });
      case '/about':
        return new Response('About page');
      default:
        return new Response('Not Found', { status: 404 });
    }
  },
};
```

#### Handle Different HTTP Methods
```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    if (url.pathname === '/api/data') {
      switch (request.method) {
        case 'GET':
          return new Response(JSON.stringify({ data: 'example' }), {
            headers: { 'Content-Type': 'application/json' },
          });
        
        case 'POST':
          const body = await request.json();
          return new Response(JSON.stringify({ received: body }), {
            headers: { 'Content-Type': 'application/json' },
          });
        
        case 'PUT':
        case 'DELETE':
          return new Response('Method supported', { status: 200 });
        
        default:
          return new Response('Method not allowed', { status: 405 });
      }
    }
    
    return new Response('Not Found', { status: 404 });
  },
};
```

#### Parse Request Body
```javascript
export default {
  async fetch(request, env, ctx) {
    if (request.method === 'POST') {
      const contentType = request.headers.get('Content-Type');
      
      if (contentType?.includes('application/json')) {
        const json = await request.json();
        return new Response(`Received: ${JSON.stringify(json)}`);
      }
      
      if (contentType?.includes('application/x-www-form-urlencoded')) {
        const formData = await request.formData();
        const name = formData.get('name');
        return new Response(`Hello, ${name}!`);
      }
      
      const text = await request.text();
      return new Response(`Received text: ${text}`);
    }
    
    return new Response('Send a POST request');
  },
};
```

### 4. Fetch and Proxy Patterns

#### Fetch External API
```javascript
export default {
  async fetch(request, env, ctx) {
    const apiUrl = 'https://api.example.com/data';
    const response = await fetch(apiUrl);
    const data = await response.json();
    
    return new Response(JSON.stringify(data), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

#### Proxy with Modified Headers
```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // Proxy to another origin
    const targetUrl = `https://example.com${url.pathname}`;
    
    // Clone and modify request
    const modifiedRequest = new Request(targetUrl, {
      method: request.method,
      headers: request.headers,
    });
    
    const response = await fetch(modifiedRequest);
    
    // Modify response headers
    const newResponse = new Response(response.body, response);
    newResponse.headers.set('X-Proxied-By', 'Cloudflare Workers');
    
    return newResponse;
  },
};
```

#### Add CORS Headers
```javascript
export default {
  async fetch(request, env, ctx) {
    // Handle preflight
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type',
        },
      });
    }
    
    const response = await fetch('https://api.example.com/data');
    const newResponse = new Response(response.body, response);
    
    newResponse.headers.set('Access-Control-Allow-Origin', '*');
    
    return newResponse;
  },
};
```

### 5. HTMLRewriter Patterns

Transform HTML on-the-fly using streaming HTML parser:

#### Basic Element Manipulation
```javascript
class ElementHandler {
  element(element) {
    // Modify element attributes
    element.setAttribute('data-modified', 'true');
    
    // Change tag name
    if (element.tagName === 'h1') {
      element.tagName = 'h2';
    }
  }
  
  text(text) {
    // Modify text content
    if (text.text.includes('old')) {
      text.replace(text.text.replace('old', 'new'));
    }
  }
}

export default {
  async fetch(request, env, ctx) {
    const response = await fetch('https://example.com');
    
    return new HTMLRewriter()
      .on('h1', new ElementHandler())
      .transform(response);
  },
};
```

#### Inject Content
```javascript
export default {
  async fetch(request, env, ctx) {
    const response = await fetch('https://example.com');
    
    return new HTMLRewriter()
      .on('head', {
        element(element) {
          element.append('<style>body { background: #f0f0f0; }</style>', { html: true });
        },
      })
      .on('body', {
        element(element) {
          element.prepend('<div class="banner">Powered by Workers</div>', { html: true });
        },
      })
      .transform(response);
  },
};
```

#### Remove Elements
```javascript
export default {
  async fetch(request, env, ctx) {
    const response = await fetch('https://example.com');
    
    return new HTMLRewriter()
      .on('script[src*="analytics"]', {
        element(element) {
          element.remove();
        },
      })
      .on('.advertisement', {
        element(element) {
          element.remove();
        },
      })
      .transform(response);
  },
};
```

### 6. Context Methods

#### ctx.waitUntil() - Background Tasks
```javascript
export default {
  async fetch(request, env, ctx) {
    // Return response immediately, log asynchronously
    const response = new Response('Request processed');
    
    ctx.waitUntil(
      fetch('https://analytics.example.com/log', {
        method: 'POST',
        body: JSON.stringify({
          timestamp: Date.now(),
          url: request.url,
          method: request.method,
        }),
      })
    );
    
    return response;
  },
};
```

#### ctx.passThroughOnException()
```javascript
export default {
  async fetch(request, env, ctx) {
    ctx.passThroughOnException();
    
    try {
      // Your worker logic
      return new Response('Success');
    } catch (error) {
      // Request passes through to origin on exception
      throw error;
    }
  },
};
```

### 7. Cache API Patterns

#### Cache API Usage
```javascript
export default {
  async fetch(request, env, ctx) {
    const cache = caches.default;
    const cacheKey = new Request(request.url, { method: 'GET' });
    
    // Check cache
    let response = await cache.match(cacheKey);
    
    if (!response) {
      // Cache miss, fetch from origin
      response = await fetch(request);
      
      // Cache for 1 hour
      const cacheResponse = response.clone();
      ctx.waitUntil(cache.put(cacheKey, cacheResponse));
    }
    
    return response;
  },
};
```

#### Custom Cache Keys
```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // Create cache key without query params
    const cacheKey = new Request(
      `${url.origin}${url.pathname}`,
      { method: 'GET' }
    );
    
    const cache = caches.default;
    let response = await cache.match(cacheKey);
    
    if (!response) {
      response = await fetch(request);
      ctx.waitUntil(cache.put(cacheKey, response.clone()));
    }
    
    return response;
  },
};
```

### 8. Headers Manipulation

#### Modify Request Headers
```javascript
export default {
  async fetch(request, env, ctx) {
    const modifiedRequest = new Request(request);
    modifiedRequest.headers.set('User-Agent', 'Custom Worker Agent');
    modifiedRequest.headers.set('X-Custom-Header', 'value');
    modifiedRequest.headers.delete('Cookie');
    
    return fetch(modifiedRequest);
  },
};
```

#### Set Security Headers
```javascript
export default {
  async fetch(request, env, ctx) {
    const response = await fetch(request);
    const newResponse = new Response(response.body, response);
    
    newResponse.headers.set('X-Content-Type-Options', 'nosniff');
    newResponse.headers.set('X-Frame-Options', 'DENY');
    newResponse.headers.set('X-XSS-Protection', '1; mode=block');
    newResponse.headers.set('Strict-Transport-Security', 'max-age=31536000');
    newResponse.headers.set(
      'Content-Security-Policy',
      "default-src 'self'; script-src 'self' 'unsafe-inline'"
    );
    
    return newResponse;
  },
};
```

### 9. Request Object Properties

Access Cloudflare-specific request properties:

```javascript
export default {
  async fetch(request, env, ctx) {
    const cf = request.cf;
    
    const info = {
      country: cf?.country,
      city: cf?.city,
      continent: cf?.continent,
      latitude: cf?.latitude,
      longitude: cf?.longitude,
      timezone: cf?.timezone,
      postalCode: cf?.postalCode,
      region: cf?.region,
      regionCode: cf?.regionCode,
      asn: cf?.asn,
      asOrganization: cf?.asOrganization,
      colo: cf?.colo, // Edge datacenter
      tlsVersion: cf?.tlsVersion,
      tlsCipher: cf?.tlsCipher,
      httpProtocol: cf?.httpProtocol,
    };
    
    return new Response(JSON.stringify(info, null, 2), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

### 10. Redirects

#### Simple Redirect
```javascript
export default {
  fetch(request, env, ctx) {
    return Response.redirect('https://example.com', 302);
  },
};
```

#### Conditional Redirect
```javascript
export default {
  fetch(request, env, ctx) {
    const url = new URL(request.url);
    const country = request.cf?.country;
    
    if (country === 'US') {
      return Response.redirect('https://us.example.com', 302);
    }
    
    if (country === 'GB') {
      return Response.redirect('https://uk.example.com', 302);
    }
    
    return Response.redirect('https://example.com', 302);
  },
};
```

### 11. Error Handling

```javascript
export default {
  async fetch(request, env, ctx) {
    try {
      const url = new URL(request.url);
      
      if (url.pathname === '/error') {
        throw new Error('Intentional error');
      }
      
      const response = await fetch('https://api.example.com/data');
      
      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }
      
      return response;
    } catch (error) {
      console.error('Error:', error.message);
      
      return new Response(
        JSON.stringify({
          error: error.message,
          status: 'error',
        }),
        {
          status: 500,
          headers: { 'Content-Type': 'application/json' },
        }
      );
    }
  },
};
```

## Playground-Specific Features

### DevTools Panel

The Playground includes built-in DevTools at the bottom of preview:

**Network Tab**: Shows outgoing `fetch()` requests from Worker
**Console Tab**: Displays `console.log()`, `console.error()`, etc.
**Sources Tab**: View Worker modules and bindings (KV, secrets require auth)

### HTTP Test Panel

Switch to **HTTP** tab to test raw HTTP requests:
- Change HTTP method (GET, POST, PUT, DELETE, etc.)
- Add/edit headers
- Modify request body
- Send request to Worker

### Sharing Code

Click **Copy Link** to generate shareable URL with embedded code. Links never expire and can be bookmarked.

### Deploy from Playground

Click **Deploy** button to:
1. Log in to Cloudflare account (if not already)
2. Review Worker code
3. Deploy to Cloudflare's global network
4. Get unique `*.workers.dev` URL
5. Add custom domains, bindings, etc. from dashboard

## Type Safety with JSDoc

Playground provides type-checking via JSDoc comments:

```javascript
/**
 * @typedef {Object} Env
 * @property {KVNamespace} MY_KV
 * @property {string} API_KEY
 */

export default {
  /**
   * @param {Request} request
   * @param {Env} env
   * @param {ExecutionContext} ctx
   * @returns {Promise<Response>}
   */
  async fetch(request, env, ctx) {
    // TypeScript-like autocomplete and validation
    const value = await env.MY_KV.get('key');
    return new Response(value);
  },
};
```

## Common Use Cases

### 1. API Gateway
```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // Route based on path
    if (url.pathname.startsWith('/v1/users')) {
      return fetch('https://users-api.example.com' + url.pathname);
    }
    
    if (url.pathname.startsWith('/v1/posts')) {
      return fetch('https://posts-api.example.com' + url.pathname);
    }
    
    return new Response('API Gateway', { status: 404 });
  },
};
```

### 2. A/B Testing
```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const cookie = request.headers.get('Cookie');
    
    // Check existing variant
    let variant = cookie?.includes('ab_test=B') ? 'B' : null;
    
    if (!variant) {
      // Assign 50/50 split
      variant = Math.random() < 0.5 ? 'A' : 'B';
    }
    
    const targetUrl = variant === 'A' 
      ? 'https://a.example.com' 
      : 'https://b.example.com';
    
    const response = await fetch(targetUrl);
    const newResponse = new Response(response.body, response);
    
    newResponse.headers.append('Set-Cookie', `ab_test=${variant}; Path=/; Max-Age=86400`);
    
    return newResponse;
  },
};
```

### 3. Authentication Proxy
```javascript
export default {
  async fetch(request, env, ctx) {
    const authHeader = request.headers.get('Authorization');
    
    if (!authHeader || authHeader !== 'Bearer secret-token') {
      return new Response('Unauthorized', { status: 401 });
    }
    
    // Forward to protected API
    return fetch('https://api.example.com' + new URL(request.url).pathname);
  },
};
```

### 4. Content Transformation
```javascript
export default {
  async fetch(request, env, ctx) {
    const response = await fetch('https://example.com');
    
    return new HTMLRewriter()
      .on('img', {
        element(element) {
          // Rewrite image URLs to use Cloudflare Images
          const src = element.getAttribute('src');
          if (src) {
            element.setAttribute('src', `/cdn-cgi/image/format=auto/${src}`);
          }
        },
      })
      .transform(response);
  },
};
```

### 5. Rate Limiting (Conceptual)
```javascript
// Note: Full rate limiting requires KV or Durable Objects
export default {
  async fetch(request, env, ctx) {
    const ip = request.headers.get('CF-Connecting-IP');
    
    // Simplified rate limit check
    // In production, use KV or Durable Objects to track counts
    const rateLimitExceeded = false; // Replace with actual logic
    
    if (rateLimitExceeded) {
      return new Response('Too Many Requests', { status: 429 });
    }
    
    return new Response('OK');
  },
};
```

## Best Practices

### 1. Use Async/Await
Always use `async/await` for cleaner asynchronous code:

```javascript
// Good
export default {
  async fetch(request, env, ctx) {
    const response = await fetch('https://api.example.com');
    const data = await response.json();
    return new Response(JSON.stringify(data));
  },
};

// Avoid
export default {
  fetch(request, env, ctx) {
    return fetch('https://api.example.com')
      .then(response => response.json())
      .then(data => new Response(JSON.stringify(data)));
  },
};
```

### 2. Clone Responses Before Reading
Response bodies can only be read once:

```javascript
export default {
  async fetch(request, env, ctx) {
    const response = await fetch('https://api.example.com');
    
    // Clone before caching
    ctx.waitUntil(caches.default.put(request, response.clone()));
    
    return response;
  },
};
```

### 3. Handle Errors Gracefully
Always wrap risky operations in try/catch:

```javascript
export default {
  async fetch(request, env, ctx) {
    try {
      const response = await fetch('https://api.example.com/data');
      return response;
    } catch (error) {
      console.error('Fetch failed:', error);
      return new Response('Service Unavailable', { status: 503 });
    }
  },
};
```

### 4. Use console.log for Debugging
View output in Playground DevTools Console:

```javascript
export default {
  async fetch(request, env, ctx) {
    console.log('Request URL:', request.url);
    console.log('Request method:', request.method);
    console.log('Headers:', Object.fromEntries(request.headers));
    
    const response = new Response('OK');
    return response;
  },
};
```

### 5. Leverage ctx.waitUntil()
Defer work without delaying response:

```javascript
export default {
  async fetch(request, env, ctx) {
    const response = new Response('Request received');
    
    // Log analytics without blocking response
    ctx.waitUntil(
      fetch('https://analytics.example.com/track', {
        method: 'POST',
        body: JSON.stringify({ url: request.url }),
      })
    );
    
    return response;
  },
};
```

## Limitations in Playground

1. **No bindings without auth**: KV, D1, Durable Objects require login
2. **Browser-only**: Firefox/Chrome desktop required
3. **No secrets**: Environment variables require deployment
4. **Limited CPU time**: 50ms CPU time in free tier
5. **No persistent state**: Playground sessions don't persist across refreshes

## Testing in Playground

### Test Different HTTP Methods
Use HTTP tab to send POST, PUT, DELETE requests with custom headers and body.

### Test URL Routing
Enter different paths in preview URL bar:
```
https://playground.workers.dev/
https://playground.workers.dev/api
https://playground.workers.dev/about
```

### Test Request Headers
Add custom headers in HTTP tab to verify header-based logic.

### View Console Output
Check Console tab for `console.log()` statements and errors.

### Monitor Network Requests
Network tab shows all outgoing `fetch()` calls from Worker.

## Common Patterns Summary

| Pattern | Use Case |
|---------|----------|
| `fetch()` | Proxy requests, call APIs |
| `new Response()` | Return data (HTML, JSON, text) |
| `new Request()` | Modify requests before forwarding |
| `HTMLRewriter` | Transform HTML on-the-fly |
| `ctx.waitUntil()` | Background tasks (logging, analytics) |
| `request.cf` | Geolocation, TLS info, device type |
| `Response.redirect()` | Redirect users |
| Cache API | Cache responses at edge |

## Related Resources

- **Workers Playground**: https://workers.cloudflare.com/playground
- **Workers Docs**: https://developers.cloudflare.com/workers/
- **Runtime APIs**: https://developers.cloudflare.com/workers/runtime-apis/
- **Examples Library**: https://developers.cloudflare.com/workers/examples/
- **Wrangler CLI**: https://developers.cloudflare.com/workers/wrangler/

## When to Use This Skill

Use this skill when:
- Writing Workers code for Playground experimentation
- Testing Workers features without local setup
- Sharing Workers code snippets
- Learning Workers APIs interactively
- Prototyping serverless functions
- Teaching/demonstrating Workers capabilities

This skill focuses ONLY on Workers Playground. For production Workers development with Wrangler, bindings setup, or full platform features, consult broader Cloudflare Workers documentation.
