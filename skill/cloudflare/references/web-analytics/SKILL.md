# Cloudflare Web Analytics Skill

Comprehensive guide for implementing Cloudflare Web Analytics - a privacy-first web analytics solution that provides performance metrics and user insights without compromising visitor privacy.

## Overview

Cloudflare Web Analytics is a free, privacy-focused analytics service that:
- Measures Core Web Vitals (LCP, INP, CLS)
- Tracks page views, visits, and page load times
- Provides visitor segmentation without cookies or fingerprinting
- Works with both proxied and non-proxied sites
- Requires no personally identifiable information (PII)

## Setup Methods

### 1. Sites Proxied Through Cloudflare (Automatic)

For sites already on Cloudflare's proxy:

1. **Dashboard setup:**
   - Navigate to Web Analytics in Cloudflare dashboard
   - Click "Add a site"
   - Select hostname from dropdown → "Done"
   - Analytics enabled by default (beacon auto-injected)

2. **Configuration options:**
   - **Enable** - Full auto-injection for all visitors
   - **Enable, excluding visitor data in the EU** - No injection for EU visitors
   - **Enable with JS Snippet installation** - Manual snippet required
   - **Disable** - Turn off analytics

**Important:** If `Cache-Control: public, no-transform` header is set, auto-injection fails. Beacon must be added manually.

### 2. Sites Not Proxied Through Cloudflare (Manual)

For non-proxied sites (limit: 10 sites):

1. Dashboard: Web Analytics → "Add a site"
2. Enter hostname manually → "Done"
3. Copy JS snippet from "Manage site"
4. Add snippet before closing `</body>` tag:

```html
<!-- Cloudflare Web Analytics -->
<script defer src='https://static.cloudflareinsights.com/beacon.min.js' 
        data-cf-beacon='{"token": "YOUR_TOKEN_HERE"}'></script>
<!-- End Cloudflare Web Analytics -->
```

### 3. Cloudflare Pages Projects (One-Click)

For Pages projects:

1. Dashboard: Workers & Pages → Select project
2. Go to "Metrics" → Click "Enable" under Web Analytics
3. Beacon automatically added on next deployment

## JavaScript Beacon Implementation

### Basic HTML Integration

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>My Site</title>
</head>
<body>
    <!-- Your content -->
    
    <!-- Place before closing body tag -->
    <script defer src='https://static.cloudflareinsights.com/beacon.min.js' 
            data-cf-beacon='{"token": "abc123def456"}'></script>
</body>
</html>
```

### SPA (Single Page Application) Support

```html
<script defer src='https://static.cloudflareinsights.com/beacon.min.js' 
        data-cf-beacon='{"token": "YOUR_TOKEN", "spa": true}'></script>
```

The `spa: true` flag enables automatic route tracking in single-page applications.

### Framework-Specific Patterns

#### React/Next.js

```tsx
// components/Analytics.tsx
import Script from 'next/script'

export default function Analytics() {
  return (
    <Script
      defer
      src="https://static.cloudflareinsights.com/beacon.min.js"
      data-cf-beacon={JSON.stringify({ 
        token: process.env.NEXT_PUBLIC_CF_ANALYTICS_TOKEN,
        spa: true 
      })}
    />
  )
}
```

```tsx
// pages/_app.tsx or app/layout.tsx
import Analytics from '@/components/Analytics'

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  )
}
```

#### Vue.js/Nuxt

```typescript
// Using Nuxt Scripts module
export default defineNuxtConfig({
  modules: ['@nuxt/scripts'],
  scripts: {
    registry: {
      cloudflareWebAnalytics: {
        token: process.env.CLOUDFLARE_ANALYTICS_TOKEN
      }
    }
  }
})
```

Or manually:

```typescript
// plugins/analytics.client.ts
export default defineNuxtPlugin(() => {
  const script = document.createElement('script')
  script.defer = true
  script.src = 'https://static.cloudflareinsights.com/beacon.min.js'
  script.setAttribute('data-cf-beacon', JSON.stringify({
    token: 'YOUR_TOKEN',
    spa: true
  }))
  document.body.appendChild(script)
})
```

#### Astro

```javascript
// astro.config.mjs
export default defineConfig({
  integrations: [
    starlight({
      head: [
        {
          tag: 'script',
          attrs: {
            defer: true,
            src: 'https://static.cloudflareinsights.com/beacon.min.js',
            'data-cf-beacon': JSON.stringify({ 
              token: 'YOUR_TOKEN' 
            })
          }
        }
      ]
    })
  ]
})
```

#### Docusaurus

```javascript
// docusaurus.config.js
module.exports = {
  scripts: [
    {
      src: 'https://static.cloudflareinsights.com/beacon.min.js',
      defer: true,
      'data-cf-beacon': `{"token": "${process.env.CLOUDFLARE_ANALYTICS_TOKEN}"}`
    }
  ]
}
```

#### VitePress

```typescript
// .vitepress/config.ts
export default defineConfig({
  head: [
    [
      'script',
      {
        defer: '',
        src: 'https://static.cloudflareinsights.com/beacon.min.js',
        'data-cf-beacon': '{"token": "YOUR_TOKEN"}'
      }
    ]
  ]
})
```

## Configuration: Rules

**Rules** control where analytics tracking occurs (proxied sites only).

### Limits by Plan
- Free: 0 rules (all pages tracked)
- Pro: 5 rules
- Business: 20 rules
- Enterprise: 100 rules

### Creating Rules

Dashboard: Web Analytics → "Manage site" → "Advanced options" → "Add rule"

**Actions:**
- **Enable** - Track analytics for matching requests
- **Disable** - Exclude matching requests

**Match conditions:**
- Hostname
- Path patterns

### Rule Examples

```
Action: Disable
Hostname: example.com
Path: /admin/*
→ Exclude admin pages from tracking

Action: Enable
Hostname: blog.example.com
Path: /posts/*
→ Only track blog posts

Action: Disable
Hostname: example.com
Path: /api/*
→ Exclude API endpoints
```

**Warning:** Configuration Rules (via Cloudflare Rules engine) override Web Analytics rules. If both match a request, Configuration Rule takes precedence.

## Data & Metrics

### High-Level Metrics

- **Visits** - Unique visitor sessions (30-min timeout)
- **Page views** - Total pages loaded
- **Page load time** - Average time to load pages
- **Core Web Vitals** - LCP, INP, CLS scores

### Core Web Vitals Details

#### Largest Contentful Paint (LCP)
Measures perceived load speed - time for main content to load.

**Ratings:**
- Good: ≤2.5s
- Needs Improvement: 2.5s - 4.0s
- Poor: >4.0s

**Data collected:**
- Element (CSS selector)
- Path (URL)
- Value (milliseconds)
- URL (source - image, font, etc.)
- Size (bytes)

#### Interaction to Next Paint (INP)
Measures responsiveness - how quickly site responds to interactions.

**Ratings:**
- Good: ≤200ms
- Needs Improvement: 200ms - 500ms
- Poor: >500ms

**Data collected:**
- Element (CSS selector)
- Path (URL)
- Value (milliseconds)

#### Cumulative Layout Shift (CLS)
Measures visual stability - unexpected layout shifts.

**Ratings:**
- Good: ≤0.1
- Needs Improvement: 0.1 - 0.25
- Poor: >0.25

**Data collected:**
- Element (CSS selector)
- Path (URL)
- Value (score)
- CurrentRect (layout after shift)
- PreviousRect (layout before shift)

### Dimensions (Filters)

Available filtering dimensions:

- **Country** - Visitor's country
- **Host** - Site's domain
- **Path** - Internal page paths
- **Referer** - External referring links (host + path via GraphQL)
- **Device type** - Desktop, mobile, tablet
- **Browser** - Chrome, Safari, Firefox, etc.
- **Operating system** - Windows, macOS, iOS, Android, etc.
- **Site** - Domain name (for multi-site segmentation)
- **Exclude Bots** - Filter bot traffic for real user data

## Data Collection & Privacy

### Privacy Features

- **No cookies** - No client-side state storage
- **No localStorage** - No persistent browser storage
- **No fingerprinting** - No IP/User-Agent tracking
- **No PII** - No personally identifiable information
- **GDPR compliant** - Privacy-first by design

### What Gets Collected

- Page URLs (paths)
- Referrer information (external links)
- Performance metrics (timings, Core Web Vitals)
- Device/browser/OS metadata (general categories only)
- Geographic location (country level only)

### Browser Support

- **Full support:** Chromium-based (Chrome, Edge, Brave, etc.)
- **Limited:** Safari, Firefox (Core Web Vitals not yet supported)

## Common Use Cases

### 1. Performance Monitoring

Track Core Web Vitals to identify slow-loading elements:

```typescript
// Debug poor LCP scores
// 1. Enable Web Analytics
// 2. Dashboard → Core Web Vitals → LCP section
// 3. Debug View shows top 5 problematic elements
// 4. Use element CSS selector in browser console:
document.querySelector('.hero-image') // Example element
// 5. Optimize identified elements (lazy loading, compression, etc.)
```

### 2. Bot Traffic Filtering

Exclude bots to see real user metrics:

```
Dashboard filters:
- Exclude Bots: Yes
→ Shows human traffic only
```

### 3. Multi-Site Analytics

Track multiple properties under one account:

```
Proxied sites: Unlimited
Non-proxied: Up to 10 sites

View by dimension:
- Site: example.com
- Site: blog.example.com
→ Compare traffic across properties
```

### 4. Geographic Analysis

Understand visitor distribution:

```
Filter by:
- Country: United States
- Device type: Mobile
→ Mobile traffic from US
```

### 5. Referral Tracking

Identify top traffic sources:

```
Dashboard view: Referers
Shows external domains driving traffic

Via GraphQL API: Access full referrer paths
```

### 6. Path-Based Tracking

Monitor specific sections:

```
Filter by Path:
- /blog/* → Blog performance
- /products/* → Product pages
- /docs/* → Documentation traffic
```

### 7. A/B Testing Validation

Verify performance impact of changes:

```
1. Baseline metrics before deployment
2. Deploy changes
3. Compare Core Web Vitals
4. Filter by Path to isolate tested pages
```

## Cloudflare Pages Integration

Automatic setup for Pages projects:

```bash
# After enabling in dashboard, analytics auto-injected on deploy
wrangler pages deploy ./dist

# Environment-specific tokens (optional)
# wrangler.toml
[env.production.vars]
CF_ANALYTICS_TOKEN = "prod-token-123"

[env.preview.vars]
CF_ANALYTICS_TOKEN = "preview-token-456"
```

Pages automatically injects beacon when enabled, no code changes needed.

## Troubleshooting

### Beacon Not Loading

**Issue:** Script not appearing on page

**Solutions:**
1. Check `Cache-Control` header - remove `no-transform` if present
2. Verify beacon placement before `</body>`
3. Check CSP (Content Security Policy) allows `static.cloudflareinsights.com`
4. For proxied sites: Ensure auto-injection enabled in dashboard

**CSP Fix:**
```html
<meta http-equiv="Content-Security-Policy" 
      content="script-src 'self' 'unsafe-inline' https://static.cloudflareinsights.com;">
```

### No Data Appearing

**Issue:** Dashboard shows no analytics

**Solutions:**
1. Wait 5-10 minutes after setup (data ingestion delay)
2. Verify token is correct in `data-cf-beacon`
3. Check browser console for beacon errors
4. Confirm site has real traffic (test by visiting pages)
5. For non-proxied sites: Verify site count under limit (10)

### Core Web Vitals Missing

**Issue:** Metrics showing "N/A"

**Reason:** Safari/Firefox don't support Web Vitals APIs yet (Chromium only)

**Solution:** Test with Chrome/Edge, or wait for browser support

### SPA Routes Not Tracking

**Issue:** Only initial page load tracked

**Solution:** Enable SPA mode:
```html
data-cf-beacon='{"token": "YOUR_TOKEN", "spa": true}'
```

### EU Visitors Excluded

**Issue:** Missing traffic from Europe

**Solution:** Check dashboard setting isn't "Enable, excluding visitor data in the EU"

## Advanced Patterns

### Dynamic Token Loading

```typescript
// Load token from API/config at runtime
async function initAnalytics() {
  const config = await fetch('/api/config').then(r => r.json())
  
  const script = document.createElement('script')
  script.defer = true
  script.src = 'https://static.cloudflareinsights.com/beacon.min.js'
  script.setAttribute('data-cf-beacon', JSON.stringify({
    token: config.analyticsToken,
    spa: true
  }))
  document.body.appendChild(script)
}

// Call after DOM ready
if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', initAnalytics)
} else {
  initAnalytics()
}
```

### Conditional Loading

```typescript
// Only load analytics in production
const shouldLoadAnalytics = 
  process.env.NODE_ENV === 'production' && 
  !window.location.hostname.includes('localhost')

if (shouldLoadAnalytics) {
  const script = document.createElement('script')
  script.defer = true
  script.src = 'https://static.cloudflareinsights.com/beacon.min.js'
  script.setAttribute('data-cf-beacon', JSON.stringify({
    token: process.env.CF_ANALYTICS_TOKEN
  }))
  document.body.appendChild(script)
}
```

### Type-Safe Configuration (TypeScript)

```typescript
interface CloudflareBeaconConfig {
  token: string
  spa?: boolean
}

interface CloudflareWebAnalyticsAPI {
  __cfBeacon: {
    token: string
    spa?: boolean
  }
}

declare global {
  interface Window {
    __cfBeacon?: CloudflareWebAnalyticsAPI['__cfBeacon']
  }
}

function loadAnalytics(config: CloudflareBeaconConfig): void {
  const script = document.createElement('script')
  script.defer = true
  script.src = 'https://static.cloudflareinsights.com/beacon.min.js'
  script.setAttribute('data-cf-beacon', JSON.stringify(config))
  document.body.appendChild(script)
}

// Usage
loadAnalytics({ 
  token: import.meta.env.VITE_CF_ANALYTICS_TOKEN,
  spa: true 
})
```

### Environment-Based Tokens

```typescript
// .env.production
VITE_CF_ANALYTICS_TOKEN=abc123prod

// .env.development
VITE_CF_ANALYTICS_TOKEN=xyz789dev

// Usage
const analyticsToken = import.meta.env.VITE_CF_ANALYTICS_TOKEN

if (analyticsToken) {
  loadAnalytics({ token: analyticsToken, spa: true })
}
```

## Best Practices

### 1. Script Placement
- Always place beacon before `</body>` (not in `<head>`)
- Use `defer` attribute to avoid blocking rendering
- Load after critical content scripts

### 2. SPA Configuration
- Enable `spa: true` for React, Vue, Angular, Svelte apps
- Ensures route changes tracked without page reloads

### 3. Environment Isolation
- Use different tokens for dev/staging/production
- Never commit tokens to version control
- Load from environment variables

### 4. Performance
- Beacon is ~4KB gzipped (minimal impact)
- Loads asynchronously (non-blocking)
- Use `defer` not `async` for proper sequencing

### 5. Privacy Compliance
- No additional consent required (no PII collected)
- Complies with GDPR, CCPA by default
- Disclose analytics usage in privacy policy

### 6. Bot Filtering
- Enable "Exclude Bots: Yes" for accurate metrics
- Bots skew Core Web Vitals scores
- Real user data more actionable

### 7. Rules Management
- Start with default (track all pages)
- Add exclusion rules for admin/internal paths
- Use Pro+ plans for granular control

### 8. Core Web Vitals Optimization
- Monitor P75 (75th percentile) values
- Focus on "Poor" rated elements first
- Test fixes in Chrome/Edge (full Vitals support)
- Use Debug View to identify specific elements

## API Access (GraphQL)

While Web Analytics is primarily dashboard-based, Cloudflare provides GraphQL API access for advanced queries.

### Authentication

```bash
# API Token with "Web Analytics Read" permission
curl -X POST https://api.cloudflare.com/client/v4/graphql \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"query": "..."}'
```

### Example Queries

**Get page views by path:**

```graphql
query {
  viewer {
    accounts(filter: {accountTag: "YOUR_ACCOUNT_ID"}) {
      rumPageloadEventsAdaptiveGroups(
        filter: {
          date_geq: "2024-01-01"
          date_lt: "2024-01-31"
        }
        limit: 100
      ) {
        dimensions {
          blob1 # Page path
        }
        sum {
          visits
          pageViews
        }
        avg {
          sampleInterval
        }
      }
    }
  }
}
```

**Get Core Web Vitals:**

```graphql
query {
  viewer {
    accounts(filter: {accountTag: "YOUR_ACCOUNT_ID"}) {
      rumPerformanceEventsAdaptiveGroups(
        filter: {
          date_geq: "2024-01-01"
          date_lt: "2024-01-31"
        }
        limit: 100
      ) {
        dimensions {
          blob1 # Page path
        }
        quantiles {
          largestContentfulPaintP75
          interactionToNextPaintP75
          cumulativeLayoutShiftP75
        }
      }
    }
  }
}
```

**Note:** GraphQL schemas evolve. Refer to [Cloudflare GraphQL Analytics API docs](https://developers.cloudflare.com/analytics/graphql-api/) for current field names.

## Wrangler Integration

Wrangler doesn't directly manage Web Analytics, but can deploy sites with analytics:

```bash
# Deploy Pages site (analytics auto-enabled if configured)
wrangler pages deploy ./dist

# Deploy Workers site (manual beacon in HTML)
wrangler deploy

# View project analytics in dashboard
# No wrangler command for Web Analytics data access
```

For programmatic access, use Cloudflare API or GraphQL (not Wrangler).

## Limits & Quotas

### Site Limits
- **Non-proxied sites:** 10 per account
- **Proxied sites:** Unlimited

### Rules Limits
- **Free:** 0 rules (all pages tracked)
- **Pro:** 5 rules
- **Business:** 20 rules
- **Enterprise:** 100 rules

### Data Retention
- **Dashboard:** 6 months historical data
- **GraphQL API:** Varies by plan (consult Cloudflare docs)

### Rate Limits
- No published beacon request limits
- GraphQL API: Standard Cloudflare API rate limits apply

## Comparison: Web Analytics vs Analytics Engine

| Feature | Web Analytics | Analytics Engine |
|---------|---------------|------------------|
| **Purpose** | Frontend website analytics | Custom backend event tracking |
| **Setup** | JS beacon injection | Workers API calls |
| **Cost** | Free | Paid (usage-based) |
| **Use case** | User behavior, Core Web Vitals | Custom metrics, logs, events |
| **Data source** | Browser | Workers, APIs |
| **Privacy** | No PII, cookie-less | Developer-controlled |

**Use Web Analytics for:** Public website traffic, performance monitoring, user insights

**Use Analytics Engine for:** Custom application metrics, API monitoring, worker telemetry

## Resources

### Documentation
- [Cloudflare Web Analytics Docs](https://developers.cloudflare.com/web-analytics/)
- [Core Web Vitals Guide](https://developers.cloudflare.com/web-analytics/data-metrics/core-web-vitals/)
- [Rules Configuration](https://developers.cloudflare.com/web-analytics/configuration-options/rules/)
- [Dimensions Reference](https://developers.cloudflare.com/web-analytics/data-metrics/dimensions/)

### Tools
- [Cloudflare Dashboard](https://dash.cloudflare.com/?to=/:account/web-analytics)
- [Core Web Vitals Explainer (web.dev)](https://web.dev/vitals/)
- [GraphQL Analytics API](https://developers.cloudflare.com/analytics/graphql-api/)

### Community Examples
- [Nuxt Scripts - Cloudflare Web Analytics](https://github.com/nuxt/scripts/blob/main/src/runtime/registry/cloudflare-web-analytics.ts)
- [Workers SDK - Pages Analytics](https://github.com/cloudflare/workers-sdk/blob/main/packages/pages-shared/__tests__/asset-server/handler.test.ts)

## Common Errors & Solutions

### "Token not found or invalid"
**Cause:** Incorrect token in `data-cf-beacon`
**Fix:** Copy token from dashboard "Manage site" section

### "Script blocked by CSP"
**Cause:** Content-Security-Policy restricts beacon domain
**Fix:** Add `https://static.cloudflareinsights.com` to `script-src` directive

### "Data not updating"
**Cause:** Cache serving old dashboard data
**Fix:** Wait 10-15 minutes, clear browser cache, try incognito mode

### "Beacon not injected on proxied site"
**Cause:** `Cache-Control: no-transform` header
**Fix:** Remove `no-transform` or manually add beacon to HTML

### "SPA navigation not tracked"
**Cause:** Default beacon tracks initial load only
**Fix:** Add `"spa": true` to `data-cf-beacon` config

## Summary

Cloudflare Web Analytics provides privacy-first website analytics with:

✅ **Zero configuration** for proxied sites (auto-injection)
✅ **Manual beacon** for non-proxied sites (10 site limit)
✅ **Core Web Vitals** tracking (LCP, INP, CLS)
✅ **No cookies/PII** (GDPR compliant by default)
✅ **Free** for all users
✅ **SPA support** with `spa: true` flag
✅ **Rules engine** for granular tracking control (Pro+)
✅ **GraphQL API** for programmatic data access

**Quick Start:**
1. Dashboard → Web Analytics → Add site
2. Copy beacon snippet OR enable auto-injection
3. Deploy → Wait 10 minutes → View analytics

For advanced use cases, combine with Analytics Engine for custom backend metrics.
