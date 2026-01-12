### Wrangler Configuration
```jsonc
{
  "name": "browser-worker",
  "main": "src/index.js",
  "compatibility_date": "2025-01-01", // Use current date for new projects
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

### Session Reuse Patt