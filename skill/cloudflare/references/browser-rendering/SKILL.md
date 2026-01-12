# Cloudflare Browser Rendering Skill

**Description**: Expert knowledge for Cloudflare Browser Rendering - control headless Chrome on Cloudflare's global network for browser automation, screenshots, PDFs, web scraping, testing, and content generation.

**When to use**: Any task involving Cloudflare Browser Rendering including: taking screenshots, generating PDFs, web scraping, automated testing, extracting HTML/data, converting pages to markdown, session management, configuration, pricing questions, or troubleshooting rate limits.

---

## Overview

Cloudflare Browser Rendering runs headless Chrome instances on Cloudflare's global edge network. Available on Free and Paid plans.

**Key capabilities**:
- Screenshots, PDFs, HTML extraction, markdown conversion
- Structured data extraction (scraping)
- Browser automation and testing
- Global distribution with low latency
- Scale to thousands of concurrent browsers
- Session reuse for performance

---

## Integration Methods

### 1. REST API
**Best for**: Simple, stateless, one-off tasks

**Authentication**: Requires API Token with `Browser Rendering - Edit` permissions

**Base URL**: `https://api.cloudflare.com/client/v4/accounts/<accountId>/browser-rendering/`

**Available endpoints**:
- `/content` - Fetch rendered HTML
- `/screenshot` - Capture screenshots (PNG/JPEG)
- `/pdf` - Generate PDFs
- `/snapshot` - Webpage snapshots
- `/scrape` - Extract HTML elements with selectors
- `/json` - Extract structured data using AI
- `/links` - Retrieve all links from page
- `/markdown` - Convert page to markdown

**Usage monitoring**: Response header `X-Browser-Ms-Used` reports browser time (milliseconds)

**Example - Take screenshot**:
```bash
curl -X POST 'https://api.cloudflare.com/client/v4/accounts/<accountId>/browser-rendering/screenshot' \
  -H 'Authorization: Bearer <apiToken>' \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://example.com",
    "screenshotOptions": {
      "omitBackground": true,
      "fullPage": true
    }
  }' \
  --output "screenshot.png"
```

### 2. Workers Bindings
**Best for**: Complex multi-step automation, custom workflows, persistent sessions

**Available libraries**:
- **@cloudflare/puppeteer** (v22.13.1) - Industry-standard Chrome automation
- **@cloudflare/playwright** (v1.57.0) - Modern automation with dev-friendly API
- **Stagehand** - AI-native with natural language selectors

---

## Puppeteer Setup & Patterns

### Installation
```bash
npm i -D @cloudflare/puppeteer
```

### Wrangler Configuration
```jsonc
{
  "name": "browser-worker",
  "main": "src/index.js",
  "compatibility_date": "2023-03-14",
  "compatibility_flags": ["nodejs_compat"],
  "browser": {
    "binding": "MYBROWSER"
  }
}
```

### Basic Pattern
```typescript
import puppeteer from "@cloudflare/puppeteer";

interface Env {
  MYBROWSER: Fetcher;
}

export default {
  async fetch(request, env): Promise<Response> {
    const browser = await puppeteer.launch(env.MYBROWSER);
    const page = await browser.newPage();
    
    try {
      await page.goto("https://example.com");
      const metrics = await page.metrics();
      return Response.json(metrics);
    } finally {
      await browser.close(); // ALWAYS close in finally block
    }
  },
} satisfies ExportedHandler<Env>;
```

### Keep-Alive Sessions
```javascript
// Default: 60 seconds idle timeout
// Max: 10 minutes (600000 ms)
const browser = await puppeteer.launch(env.MYBROWSER, { 
  keep_alive: 600000 
});
```

### Session Reuse Pattern
```javascript
// Reuse existing session for performance
const { sessionId } = await puppeteer.sessions();
const browser = await puppeteer.connect(env.MYBROWSER, sessionId);
// ... do work ...
await browser.close(); // Disconnects, doesn't destroy session
```

### Custom User Agent
```javascript
await page.setUserAgent(
  "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
);
// Note: Does NOT bypass bot detection
```

### Element Selection (No XPath)
XPath selectors NOT supported. Use CSS selectors or `page.evaluate()`:

```typescript
// Instead of XPath, use page.evaluate()
const innerHtml = await page.evaluate(() => {
  return (
    // @ts-ignore runs in browser context
    new XPathEvaluator()
      .createExpression("/html/body/div/h1")
      // @ts-ignore runs in browser context
      .evaluate(document, XPathResult.FIRST_ORDERED_NODE_TYPE)
      .singleNodeValue.innerHTML
  );
});
```

### Session Management API
```javascript
// List open sessions
const sessions = await puppeteer.sessions();
// Returns: [{ sessionId, startTime, connectionId?, connectionStartTime? }]

// View session history
const history = await puppeteer.history();
// Returns: [{ sessionId, startTime, endTime?, closeReason?, closeReasonText? }]

// Check current limits
const limits = await puppeteer.limits();
// Returns: { 
//   activeSessions, 
//   maxConcurrentSessions, 
//   allowedBrowserAcquisitions,
//   timeUntilNextAllowedBrowserAcquisition 
// }
```

---

## Playwright Setup & Patterns

### Installation
```bash
npm i -D @cloudflare/playwright
```

### Wrangler Configuration
```jsonc
{
  "name": "playwright-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-09-17",
  "compatibility_flags": ["nodejs_compat"],
  "browser": {
    "binding": "MYBROWSER"
  }
}
```

### Basic Pattern
```typescript
import { launch } from "@cloudflare/playwright";

export default {
  async fetch(request: Request, env: Env) {
    const browser = await launch(env.MYBROWSER);
    const page = await browser.newPage();
    
    try {
      await page.goto("https://example.com");
      return new Response(await page.content());
    } finally {
      await browser.close();
    }
  },
};
```

### Screenshots
```typescript
const img = await page.screenshot();
return new Response(img, {
  headers: { "Content-Type": "image/png" }
});
```

### Tracing for Debugging
```typescript
import fs from "fs";

// Start trace
await page.context().tracing.start({ 
  screenshots: true, 
  snapshots: true 
});

// ... perform actions ...

// Save trace
await page.context().tracing.stop({ path: "trace.zip" });
const file = await fs.promises.readFile("trace.zip");

return new Response(file, {
  headers: { "Content-Type": "application/zip" }
});
// Upload to https://trace.playwright.dev/ for analysis
```

### Test Assertions
```typescript
import { expect } from "@cloudflare/playwright/test";

await expect(page.getByTestId("todo-title")).toHaveCount(3);
await expect(page.getByTestId("todo-title").nth(0))
  .toHaveText("buy cheese");
```

### Storage State (Persist Cookies/Storage)
```typescript
// Get storage state from KV
const storageStateJson = await env.KV.get('storageState');
const storageState = storageStateJson ? 
  JSON.parse(storageStateJson) : undefined;

// Launch with stored state
const browser = await launch(env.MYBROWSER);
const context = await browser.newContext({ storageState });
const page = await context.newPage();

// ... perform actions ...

// Save updated state
const updatedStorageState = await context.storageState({ 
  indexedDB: true 
});
await env.KV.put('storageState', JSON.stringify(updatedStorageState));
```

### Session Reuse
```typescript
import { acquire, connect } from "@cloudflare/playwright";

// Acquire new session
const { sessionId } = await acquire(env.MYBROWSER);

// Connect multiple times to same session
for (let i = 0; i < 5; i++) {
  const browser = await connect(env.MYBROWSER, sessionId);
  // ... do work ...
  await browser.close(); // Disconnects, keeps session alive
}
```

### Custom User Agent
```typescript
const context = await browser.newContext({
  userAgent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."
});
```

---

## Common Integration Patterns

### With KV (Caching Screenshots)
```typescript
import puppeteer from "@cloudflare/puppeteer";

export default {
  async fetch(request, env) {
    const { searchParams } = new URL(request.url);
    const url = searchParams.get("url");
    
    if (!url) {
      return new Response("Missing ?url= parameter", { status: 400 });
    }
    
    const normalized = new URL(url).toString();
    
    // Check cache
    let img = await env.BROWSER_KV.get(normalized, { 
      type: "arrayBuffer" 
    });
    
    if (!img) {
      // Generate screenshot
      const browser = await puppeteer.launch(env.MYBROWSER);
      try {
        const page = await browser.newPage();
        await page.goto(normalized);
        img = await page.screenshot();
        
        // Cache for 24 hours
        await env.BROWSER_KV.put(normalized, img, {
          expirationTtl: 60 * 60 * 24,
        });
      } finally {
        await browser.close();
      }
    }
    
    return new Response(img, {
      headers: { "content-type": "image/jpeg" }
    });
  },
};
```

### With Durable Objects (Long-lived Sessions)
```typescript
// Use Durable Objects to persist browser sessions across requests
// and improve performance by eliminating cold starts

export class BrowserDO {
  private browser: Browser | null = null;
  
  async fetch(request: Request) {
    if (!this.browser) {
      this.browser = await puppeteer.launch(this.env.MYBROWSER, {
        keep_alive: 600000 // 10 minutes
      });
    }
    
    // Reuse existing browser
    const page = await this.browser.newPage();
    // ... perform work ...
    await page.close(); // Close page, keep browser
    
    return new Response("Done");
  }
}
```

---

## Wrangler Commands

```bash
# Create new Worker project
npm create cloudflare@latest browser-worker

# Create KV namespace
npx wrangler kv namespace create BROWSER_KV
npx wrangler kv namespace create BROWSER_KV --preview

# Test locally with remote browser
npx wrangler dev
# Or use local bindings (mock)
npx wrangler dev --local

# Deploy to production
npx wrangler deploy

# View logs
npx wrangler tail

# Monitor in dashboard
# Go to: Cloudflare Dashboard > Workers & Pages > Browser Rendering
```

### Remote vs Local Development
```jsonc
// For real browser during dev (uses browser hours):
{
  "browser": {
    "binding": "MYBROWSER",
    "remote": true
  }
}

// For mock browser (no browser hours, limited functionality):
{
  "browser": {
    "binding": "MYBROWSER"
    // remote defaults to false
  }
}
```

---

## Limits & Quotas

### Workers Free Plan
- **Browser time**: 10 minutes per day
- **Concurrent browsers**: 3
- **New browsers**: 3 per minute
- **Browser timeout**: 60 seconds (default)
- **REST API requests**: 6 per minute

### Workers Paid Plan
- **Browser time**: 10 hours/month included, then $0.09/hour
- **Concurrent browsers**: 10 (avg monthly), then $2.00/browser
- **New browsers**: 30 per minute
- **Browser timeout**: 60 seconds (default, up to 10 min with keep_alive)
- **REST API requests**: 180 per minute

### Rate Limit Notes
- Enforced as per-second fill rate (not burst)
- Example: 180/min = 3 requests per second evenly distributed
- Retry-After header provided on 429 responses
- Concurrent browsers calculated as monthly average of daily peaks

### Handling 429 Errors
```javascript
// REST API
const response = await fetch('https://api.cloudflare.com/...');

if (response.status === 429) {
  const retryAfter = response.headers.get('Retry-After');
  await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
  // Retry request
}

// Workers Bindings
try {
  const browser = await puppeteer.launch(env.MYBROWSER);
} catch (error) {
  if (error.status === 429) {
    const retryAfter = error.headers.get("Retry-After");
    await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
    // Retry
  }
}
```

---

## Best Practices

### Always Close Browsers
```typescript
// ❌ BAD - Session stays open until timeout
const browser = await puppeteer.launch(env.MYBROWSER);
const page = await browser.newPage();
await page.goto("https://example.com");
return new Response(await page.content());

// ✅ GOOD - Always use try/finally
const browser = await puppeteer.launch(env.MYBROWSER);
try {
  const page = await browser.newPage();
  await page.goto("https://example.com");
  return new Response(await page.content());
} finally {
  await browser.close(); // Ensures cleanup even on errors
}
```

### Optimize Concurrency
Instead of launching multiple browsers:
- Use multiple tabs in single browser
- Reuse sessions with session IDs
- Use incognito contexts for isolation without new browsers

```typescript
// ❌ BAD - Uses 3 concurrent browser slots
const browser1 = await puppeteer.launch(env.MYBROWSER);
const browser2 = await puppeteer.launch(env.MYBROWSER);
const browser3 = await puppeteer.launch(env.MYBROWSER);

// ✅ GOOD - Uses 1 browser slot
const browser = await puppeteer.launch(env.MYBROWSER);
const [page1, page2, page3] = await Promise.all([
  browser.newPage(),
  browser.newPage(),
  browser.newPage(),
]);
```

### Monitor Usage
```typescript
// Check session close reasons to identify issues
const history = await puppeteer.history();
history.forEach(session => {
  if (session.closeReasonText === "BrowserIdle") {
    // Browser wasn't closed explicitly - indicates missing browser.close()
  }
});

// Dashboard: Workers & Pages > Browser Rendering
// Shows: total browser hours, REST API requests, session close reasons
```

### Session Isolation for Testing
```typescript
// Use incognito contexts instead of new browsers
const browser = await puppeteer.launch(env.MYBROWSER);

const context1 = await browser.createIncognitoBrowserContext();
const context2 = await browser.createIncognitoBrowserContext();

// Each context has isolated cookies/cache but shares browser
const page1 = await context1.newPage();
const page2 = await context2.newPage();
```

---

## Pricing

### Browser Hours
- **Charged for**: Time browser is actively open
- **Both methods**: REST API and Workers Bindings share pool
- **Billing**: Daily totals summed monthly, rounded to nearest hour
- **Free tier**: 10 mins/day (Free), 10 hours/month (Paid)
- **Paid rate**: $0.09 per hour after free tier

### Concurrent Browsers (Workers Bindings only)
- **Calculation**: Monthly average of daily peak usage
- **Example**: 10 browsers for 15 days + 20 browsers for 15 days = ((10×15)+(20×15))/30 = 15 avg
- **Free tier**: 10 browsers (avg monthly, Paid plan only)
- **Paid rate**: $2.00 per browser after free tier

### Not Charged
- Failed requests (e.g., `waitForTimeout` errors)
- REST API call processing time
- Workers CPU time (billed separately under Workers pricing)

---

## Troubleshooting

### "429 Browser time limit exceeded for today"
**Cause**: Hit 10 min/day limit on Free plan

**Solutions**:
- Upgrade to Workers Paid plan
- After upgrading, redeploy Worker to apply new plan
- Ensure browsers are closed with `browser.close()`

### Higher usage than expected
**Cause**: Browsers not closed, stay open until timeout

**Debug**:
```typescript
const history = await puppeteer.history();
// Look for closeReasonText: "BrowserIdle" instead of "NormalClosure"
```

**Fix**:
- Always call `browser.close()`
- Use try/finally blocks
- Reduce `keep_alive` duration if too high

### "429 Too many requests"
**Cause**: Rate limit exceeded (not distributed evenly over time)

**Solutions**:
- Implement Retry-After header logic
- Distribute requests evenly (not bursts)
- Request limit increase for Paid plan

### Upgrade doesn't apply
**Cause**: Worker deployment cached old plan

**Fix**: Redeploy Worker with `npx wrangler deploy`

---

## Architecture Patterns

### Stateless REST API Pattern
```
User Request → REST API Endpoint → Render → Return Result
- Simple, no session management
- Good for: screenshots, PDFs, HTML extraction
- Charged: browser hours only
```

### Stateful Worker Pattern
```
User Request → Worker → Launch Browser → Multi-step automation → Close
- Full control, custom logic
- Good for: testing, complex scraping, interactions
- Charged: browser hours + concurrent browsers
```

### Session Reuse Pattern
```
Request 1 → Launch Browser → [Keep Open]
Request 2 → Connect to Same Browser → [Reuse]
Request N → Connect to Same Browser → Close when done
- Best performance, no cold starts
- Good for: high-frequency requests, same site
- Use: Durable Objects or session IDs
```

### Durable Objects Pattern
```
User Requests → DO Instance → Persistent Browser Session
- Single browser serves multiple requests
- Good for: long-lived sessions, state preservation
- Benefit: No cold start after first launch
```

---

## Configuration Reference

### Browser Binding (wrangler.jsonc)
```jsonc
{
  "browser": {
    "binding": "MYBROWSER",      // Env variable name
    "remote": true               // Optional: use real browser in dev
  }
}
```

### Compatibility Requirements
- **Flag**: `nodejs_compat` (required)
- **Date**: `2023-03-14` or later for Puppeteer
- **Date**: `2025-09-17` or later for Playwright (requires node.fs)

### Common Bindings Together
```jsonc
{
  "compatibility_flags": ["nodejs_compat"],
  "compatibility_date": "2025-09-17",
  "browser": {
    "binding": "MYBROWSER"
  },
  "kv_namespaces": [
    { "binding": "CACHE", "id": "..." }
  ],
  "durable_objects": {
    "bindings": [
      { "name": "BROWSER_DO", "class_name": "BrowserDO" }
    ]
  }
}
```

---

## Use Case Examples

### Web Scraping
```typescript
const page = await browser.newPage();
await page.goto("https://example.com/products");

// Extract data
const products = await page.evaluate(() => {
  return Array.from(document.querySelectorAll('.product')).map(el => ({
    name: el.querySelector('.name')?.textContent,
    price: el.querySelector('.price')?.textContent,
  }));
});
```

### PDF Generation
```typescript
// REST API
curl -X POST '.../browser-rendering/pdf' \
  -H 'Authorization: Bearer <token>' \
  -d '{"url": "https://example.com"}' \
  --output doc.pdf

// Puppeteer
const page = await browser.newPage();
await page.goto("https://example.com");
const pdf = await page.pdf({ format: 'A4' });
```

### Automated Testing
```typescript
import { expect } from "@cloudflare/playwright/test";

const page = await browser.newPage();
await page.goto("https://app.example.com");

await page.fill('#username', 'testuser');
await page.fill('#password', 'testpass');
await page.click('button[type="submit"]');

await expect(page.locator('.welcome')).toBeVisible();
```

### Form Automation
```typescript
const page = await browser.newPage();
await page.goto("https://example.com/form");

await page.type('input[name="email"]', 'user@example.com');
await page.select('select[name="country"]', 'US');
await page.check('input[type="checkbox"]');
await page.click('button[type="submit"]');

await page.waitForNavigation();
```

---

## API Quick Reference

### Puppeteer
```typescript
// Browser
puppeteer.launch(env.MYBROWSER, opts?)
puppeteer.connect(env.MYBROWSER, sessionId)
puppeteer.sessions() // List open
puppeteer.history()  // View closed
puppeteer.limits()   // Check quotas

browser.newPage()
browser.close()
browser.createIncognitoBrowserContext()

// Page
page.goto(url)
page.content()
page.screenshot()
page.pdf()
page.evaluate(fn)
page.metrics()
page.setUserAgent(ua)
page.type(selector, text)
page.click(selector)
page.select(selector, value)
```

### Playwright
```typescript
// Browser
launch(env.MYBROWSER, opts?)
connect(env.MYBROWSER, sessionId)
acquire(env.MYBROWSER) // Get new sessionId
playwright.sessions()
playwright.history()
playwright.limits()

browser.newPage()
browser.newContext(opts?)
browser.close()

// Page
page.goto(url)
page.content()
page.screenshot()
page.getByTestId(id)
page.getByPlaceholder(text)
page.locator(selector)
page.fill(selector, value)
page.press(selector, key)

// Context
context.storageState(opts?)
context.tracing.start(opts?)
context.tracing.stop(opts?)

// Assertions
expect(locator).toHaveCount(n)
expect(locator).toHaveText(text)
expect(locator).toBeVisible()
```

---

## Related Resources

- **Docs**: https://developers.cloudflare.com/browser-rendering/
- **Puppeteer API**: https://github.com/cloudflare/puppeteer/blob/main/docs/api/index.md
- **Playwright API**: https://playwright.dev/docs/api/class-playwright
- **Trace Viewer**: https://trace.playwright.dev/
- **Discord**: https://discord.cloudflare.com
- **REST API Reference**: https://developers.cloudflare.com/api/resources/browser_rendering/
- **Dashboard**: Cloudflare Dashboard > Workers & Pages > Browser Rendering

---

## Code Quality Standards

- ✅ Always close browsers in `finally` blocks
- ✅ Type environment interfaces (`interface Env { MYBROWSER: Fetcher }`)
- ✅ Handle 429 errors with Retry-After header
- ✅ Reuse sessions for performance when possible
- ✅ Use incognito contexts instead of multiple browsers
- ✅ Monitor usage with `history()` to catch unclosed sessions
- ❌ Never leave browsers open without explicit close
- ❌ Never assume bot detection bypass
- ❌ Never use XPath selectors (use CSS or evaluate())
- ❌ Never ignore rate limits or burst requests
