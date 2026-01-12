# Cloudflare Bot Management Skill

Expert guidance for implementing and using Cloudflare Bot Management across all tiers (Bot Fight Mode, Super Bot Fight Mode, and Bot Management for Enterprise).

## Core Concepts

### Bot Scores
- **Range**: 1-99 (1 = definitely automated, 99 = definitely human)
- **Threshold**: Scores below 30 typically indicate bot traffic
- **Detection engines**: Heuristics, Machine Learning (ML), Anomaly Detection (AD), JavaScript Detections (JSD)
- **Access**: Granular scores (1-99) only available with Enterprise Bot Management

### Bot Score Groupings
| Category | Score Range | Description |
|----------|-------------|-------------|
| Not computed | 0 | Bot Management did not run |
| Automated | 1 | Definitely automated (heuristic match) |
| Likely automated | 2-29 | Probably bot traffic |
| Likely human | 30-99 | Probably human traffic |
| Verified bot | N/A | Allowlisted good bots (search engines, etc.) |

### Detection Engines

#### Heuristics
- Processes all requests
- Identifies known malicious fingerprints
- Immediately assigns score of 1 to automated traffic

#### Machine Learning (ML)
- Majority of detections
- Supervised learning on billions of requests
- Predicts probability that client is human
- **Enable auto-updates** for latest models (Enterprise only)

#### Anomaly Detection (AD)
- Optional, unsupervised learning
- Records baseline of domain traffic
- User agent-agnostic
- Not recommended for Cloudflare for SaaS or high API traffic

#### JavaScript Detections (JSD)
- Identifies headless browsers and malicious fingerprints
- Lightweight, invisible JS injection
- Optional, enabled by default
- Honors strict privacy standards
- 15-minute lifespan, auto-refreshed

### Verified Bots
- Allowlisted good bots (Google, Bing, etc.)
- Verified via reverse DNS or Web Bot Auth
- Access via `cf.bot_management.verified_bot` or `cf.verified_bot_category`
- Categories: Search Engine Crawler, AI Crawler, Monitoring & Analytics, etc.

## Product Tiers

### Bot Fight Mode (Free)
- Basic bot detection and mitigation
- Auto-blocks definite bots (score = 1)
- Excludes verified bots by default
- JavaScript Detections always enabled
- No configuration options

### Super Bot Fight Mode (Pro/Business)
- Configurable actions for bot categories
- Static resource protection (optional)
- Bot Analytics with groupings
- Actions: Allow, Block, Challenge
- JavaScript Detections optional

### Bot Management for Enterprise
- Granular bot scores (1-99)
- Full WAF custom rules integration
- Workers API access
- Advanced analytics
- JavaScript Detections optional
- JA3/JA4 fingerprinting
- Corporate proxy detection
- Anomaly Detection option

## Configuration Patterns

### Enterprise Setup (Recommended)

```txt
# 1. Enable auto-updates to ML model
Dashboard > Security > Bots > Configure > Auto-updates ON

# 2. Deploy default template for definite bots (score = 1)
(cf.bot_management.score eq 1 
  and not cf.bot_management.verified_bot 
  and not cf.bot_management.static_resource)
Action: Block

# 3. Deploy template for likely bots (score 2-29)
(cf.bot_management.score ge 2 
  and cf.bot_management.score le 29 
  and not cf.bot_management.verified_bot 
  and not cf.bot_management.static_resource)
Action: Managed Challenge

# 4. Optional: JavaScript Detections for JS-only endpoints
(not cf.bot_management.js_detection.passed 
  and http.request.method eq "POST" 
  and http.request.uri.path eq "/api/v1/submit")
Action: Managed Challenge
```

### Super Bot Fight Mode Setup

```txt
# Dashboard > Security > Bots > Configure
- Definitely automated: Block or Challenge
- Likely automated: Challenge or Allow  
- Verified bots: Allow (recommended)
- Static resource protection: ON (careful with mail clients)
- JavaScript Detections: ON (optional)
```

## WAF Custom Rules

### Basic Bot Protection
```txt
# Block definite bots, allow verified bots
(cf.bot_management.score eq 1 
  and not cf.bot_management.verified_bot)
```

### Protect Sensitive Endpoints
```txt
# Higher threshold for login/checkout
(cf.bot_management.score lt 50 
  and http.request.uri.path in {"/login" "/checkout"}
  and not cf.bot_management.verified_bot)
```

### Allow Good Bots & Corporate Proxies
```txt
# Exempt from bot rules
(not cf.bot_management.verified_bot
  and not cf.bot_management.static_resource
  and not cf.bot_management.corporate_proxy
  and cf.bot_management.score lt 30)
```

### JavaScript Detections Enforcement
```txt
# Require JS for API endpoints (not first page visit!)
(not cf.bot_management.js_detection.passed
  and http.request.uri.path matches "^/api/"
  and not cf.bot_management.verified_bot)
Action: Managed Challenge
```

**CRITICAL**: Never use `cf.bot_management.js_detection.passed` on first HTML page visit. Needs at least one HTML request to inject JS.

### JA3/JA4 Fingerprinting
```txt
# Block specific attack fingerprint
(cf.bot_management.ja3_hash eq "8b8e3d5e3e8b3d5e")

# Allow mobile app by fingerprint
(cf.bot_management.ja4 eq "your_mobile_app_fingerprint")
```

### Verified Bot Categories
```txt
# Allow search engines only
(cf.verified_bot_category eq "Search Engine Crawler")

# Block AI crawlers
(cf.verified_bot_category eq "AI Crawler")
Action: Block
```

## Cloudflare Workers Integration

### Basic Bot Score Access
```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const botScore = request.cf?.botManagement?.score;
    
    if (botScore && botScore < 30) {
      return new Response('Bot detected', { status: 403 });
    }
    
    return fetch(request);
  }
};
```

### Complete Bot Management Object
```typescript
interface BotManagement {
  score: number;              // 1-99
  verifiedBot: boolean;       // Is verified bot
  staticResource: boolean;    // Serves static resource
  ja3Hash: string;            // JA3 fingerprint
  ja4: string;                // JA4 fingerprint
  jsDetection?: {
    passed: boolean;          // Passed JS detection
  };
  detectionIds: number[];     // Heuristic detection IDs
  corporateProxy?: boolean;   // From corporate proxy
}

export default {
  async fetch(request: Request): Promise<Response> {
    const cf = request.cf as any;
    const botMgmt = cf?.botManagement;
    
    if (!botMgmt) {
      return fetch(request);
    }
    
    // Allow verified bots
    if (botMgmt.verifiedBot) {
      return fetch(request);
    }
    
    // Block definite bots
    if (botMgmt.score === 1) {
      return new Response('Blocked', { 
        status: 403,
        headers: { 'X-Bot-Score': botMgmt.score.toString() }
      });
    }
    
    // Challenge likely bots
    if (botMgmt.score < 30) {
      return new Response('Challenge required', { status: 429 });
    }
    
    return fetch(request);
  }
};
```

### Advanced Pattern: Score + JS Detection
```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const cf = request.cf as any;
    const botMgmt = cf?.botManagement;
    const url = new URL(request.url);
    
    // Skip bot checks for static resources
    if (botMgmt?.staticResource) {
      return fetch(request);
    }
    
    // API endpoints: require JS detection + good score
    if (url.pathname.startsWith('/api/')) {
      const jsDetectionPassed = botMgmt?.jsDetection?.passed ?? false;
      const score = botMgmt?.score ?? 100;
      
      if (!jsDetectionPassed || score < 30) {
        return new Response('Unauthorized', { status: 401 });
      }
    }
    
    return fetch(request);
  }
};
```

### JA4 Signals Pattern
```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const cf = request.cf as any;
    const ja4Signals = cf?.ja4Signals;
    
    if (!ja4Signals) {
      // JA4 signals not available (HTTP traffic or Worker routing)
      return fetch(request);
    }
    
    // Check for anomalous behavior
    const heuristicRatio = ja4Signals.heuristic_ratio_1h ?? 0;
    const browserRatio = ja4Signals.browser_ratio_1h ?? 0;
    
    if (heuristicRatio > 0.5 || browserRatio < 0.3) {
      return new Response('Suspicious traffic', { status: 403 });
    }
    
    return fetch(request);
  }
};
```

### Mobile App Pattern
```typescript
// Allow mobile app by JA3/JA4 fingerprint
const MOBILE_APP_JA4 = 'your_mobile_app_ja4_fingerprint';

export default {
  async fetch(request: Request): Promise<Response> {
    const cf = request.cf as any;
    const botMgmt = cf?.botManagement;
    
    // Allow mobile app regardless of bot score
    if (botMgmt?.ja4 === MOBILE_APP_JA4) {
      return fetch(request);
    }
    
    // Apply normal bot rules
    if (botMgmt?.score && botMgmt.score < 30) {
      return new Response('Bot detected', { status: 403 });
    }
    
    return fetch(request);
  }
};
```

### Corporate Proxy Exemption
```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const cf = request.cf as any;
    const botMgmt = cf?.botManagement;
    
    // Exempt corporate proxy traffic from bot rules
    if (botMgmt?.corporateProxy) {
      return fetch(request);
    }
    
    // Apply bot rules to other traffic
    if (botMgmt?.score && botMgmt.score < 30 && !botMgmt.verifiedBot) {
      return new Response('Bot detected', { status: 403 });
    }
    
    return fetch(request);
  }
};
```

### Log Bot Data
```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const cf = request.cf as any;
    const botMgmt = cf?.botManagement;
    
    // Log bot data for analysis
    console.log({
      score: botMgmt?.score,
      verifiedBot: botMgmt?.verifiedBot,
      ja3Hash: botMgmt?.ja3Hash,
      ja4: botMgmt?.ja4,
      detectionIds: botMgmt?.detectionIds,
      jsDetection: botMgmt?.jsDetection?.passed,
      corporateProxy: botMgmt?.corporateProxy,
    });
    
    return fetch(request);
  }
};
```

## JavaScript Detections

### Enabling JSD
```txt
# Dashboard path
Dashboard > Security > Bots > Configure Bot Management > JS Detections: ON

# Prerequisite: Update CSP headers
Content-Security-Policy: 
  script-src 'self' /cdn-cgi/challenge-platform/;
```

### Manual JS Injection (API)
```html
<script>
function jsdOnload() {
  window.cloudflare.jsd.executeOnce({
    callback: function(result) {
      console.log('JSD outcome:', result); // 'success' or 'error'
    }
  });
}
</script>
<script src="/cdn-cgi/challenge-platform/scripts/jsd/api.js?onload=jsdOnload" async></script>
```

**Use API for**: Selective deployment on specific pages

**Don't combine**: Zone-wide toggle + manual injection

### JS Detection Rules
```txt
# WAF rule - NEVER use on first page visit
(not cf.bot_management.js_detection.passed
  and http.request.uri.path eq "/api/user/create"
  and http.request.method eq "POST"
  and not cf.bot_management.verified_bot)
Action: Managed Challenge (always use Managed Challenge, not Block)
```

### CSP Requirements
- Allow `/cdn-cgi/challenge-platform/` 
- Use `script-src 'self'` or `nonce` (not `unsafe-inline`)
- Nonces parsed from CSP response header (not `<meta>` tags)

### Limitations
- First request won't have JSD data (needs HTML page first)
- Strips ETags from HTML responses
- Not supported with CSP via `<meta>` tags
- Websocket endpoints not supported
- Native mobile apps won't pass

### cf_clearance Cookie
- Contains JSD outcome
- 15-minute lifespan
- Max 4096 bytes
- `js_detection.passed` = true/false

## Static Resource Protection

### File Extensions Protected
```txt
ico|jpg|png|jpeg|gif|css|js|tif|tiff|bmp|pict|webp|svg|svgz|
class|jar|txt|csv|doc|docx|xls|xlsx|pdf|ps|pls|ppt|pptx|
ttf|otf|woff|woff2|eot|eps|ejs|swf|torrent|midi|mid|
m3u8|m4a|mp3|ogg|ts
```

Plus `/.well-known/` path (all files)

### Access via Field
```txt
# Exclude static resources from bot rules
(cf.bot_management.score lt 30 
  and not cf.bot_management.static_resource)
```

**WARNING**: May block mail clients fetching static images

## Bot Analytics

### Access Locations
- Dashboard > Security > Bots (old) or Security > Analytics > Bot analysis (new)
- GraphQL API for programmatic access
- Security Events & Security Analytics
- Logpush/Logpull

### Available Data
- **Enterprise BM**: Bot scores (1-99), bot score source, distribution
- **Pro/Business**: Bot groupings (automated, likely automated, likely human)
- Top attributes: IPs, paths, user agents, countries
- Detection sources: Heuristics, ML, AD, JSD
- Verified bot categories

### Time Ranges
- **Enterprise BM**: Up to 1 week at a time, 30 days history
- **Pro/Business**: Up to 72 hours at a time, 30 days history
- Real-time in most cases
- Adaptive sampling (1-10% depending on volume)

## Logs & Fields

### HTTP Request Log Fields
```txt
BotScore              # 1-99 or 0 if not computed
BotScoreSrc           # Detection engine (ML, Heuristics, etc.)
BotTags               # Classification tags
BotDetectionIDs       # Heuristic detection IDs
```

### Access via
- Logpush: Stream to cloud storage/SIEM
- Logpull: API to fetch logs
- GraphQL API: Query analytics data

## Common Use Cases

### E-commerce Protection
```txt
# High security for checkout
(cf.bot_management.score lt 50 
  and http.request.uri.path in {"/checkout" "/cart/add"}
  and not cf.bot_management.verified_bot
  and not cf.bot_management.corporate_proxy)
Action: Managed Challenge
```

### API Protection
```txt
# Protect API with JS detection + score
(http.request.uri.path matches "^/api/"
  and (cf.bot_management.score lt 30 
    or not cf.bot_management.js_detection.passed)
  and not cf.bot_management.verified_bot)
Action: Block
```

### SEO-Friendly Bot Handling
```txt
# Allow search engine crawlers
(cf.bot_management.score lt 30
  and not cf.verified_bot_category in {"Search Engine Crawler"})
Action: Managed Challenge
```

### Block AI Scrapers
```txt
# Block AI training bots
(cf.verified_bot_category eq "AI Crawler")
Action: Block

# Or use dashboard: Security > Settings > Bot Management > Block AI Bots
```

### Rate Limiting by Bot Score
```txt
# Stricter limits for suspicious traffic
(cf.bot_management.score lt 50)
Rate: 10 requests per 10 seconds

(cf.bot_management.score ge 50)
Rate: 100 requests per 10 seconds
```

### Mobile App Allowlisting
```txt
# Identify mobile app by JA3/JA4
(cf.bot_management.ja4 in {"fingerprint1" "fingerprint2"})
Action: Skip (all remaining rules)
```

## Troubleshooting

### False Positives
1. Check Bot Analytics for affected IPs/paths
2. Identify detection source (ML, Heuristics, etc.)
3. Create exception rule:
   ```txt
   (cf.bot_management.score lt 30 
     and http.request.uri.path eq "/problematic-path")
   Action: Skip (Bot Management)
   ```
4. Or allowlist by IP/ASN/country

### False Negatives (Bots Not Caught)
1. Lower score threshold (30 → 50)
2. Enable JavaScript Detections
3. Add JA3/JA4 fingerprinting rules
4. Use rate limiting as fallback

### Bot Score = 0
- Bot Management didn't run
- Internal Cloudflare request
- Worker routing to zone (Orange-to-Orange)
- Request handled before BM (Redirect Rules, etc.)

### JavaScript Detections Not Working
1. Check CSP headers allow `/cdn-cgi/challenge-platform/`
2. Verify not using on first page visit
3. Check for ad blockers or disabled JS
4. Confirm JSD enabled in dashboard
5. Use Managed Challenge action, not Block

### Verified Bot Blocked
- Check WAF Managed Rules (not just Bot Management)
- Yandex bot: May need exception during IP update (48h)
- Create WAF exception for specific rule ID
- Verify bot via reverse DNS

### JA3/JA4 Missing
- Non-HTTPS traffic (scores are TLS-based)
- Worker routing traffic
- Orange-to-Orange traffic via Worker
- Bot Management skipped

## Best Practices

### General
- Start with Managed Challenge, not Block (test first)
- Always exclude verified bots (unless specific reason)
- Use static resource exception for performance
- Enable ML auto-updates (Enterprise)
- Monitor Bot Analytics regularly
- Log bot data for analysis

### WAF Rules
- Use compound expressions (score + verified bot + static resource)
- Lower thresholds for sensitive endpoints
- Higher thresholds for public content
- Always use `not cf.bot_management.verified_bot`
- Exempt corporate proxies for B2B traffic

### JavaScript Detections
- Don't use on first page visit
- Always Managed Challenge, never Block
- Update CSP headers first
- Consider impact on native apps
- Use API for selective deployment

### Workers
- Check for null/undefined bot management object
- Handle missing JA4 signals gracefully
- Log bot data for debugging
- Use typed interfaces for safety
- Optimize performance (avoid unnecessary checks)

### Performance
- Static resource exception reduces overhead
- JavaScript Detections adds minimal latency
- Cache bot decisions where possible
- Use Workers only when WAF rules insufficient

### Security
- Defense in depth: BM + rate limiting + WAF
- Monitor for new attack patterns
- Adjust thresholds based on traffic
- Block AI bots if not needed
- Use JA3/JA4 for attack response

## Architecture Patterns

### Layered Defense
```txt
1. Bot Management (score-based)
2. JavaScript Detections (for JS-capable clients)
3. Rate Limiting (fallback protection)
4. WAF Managed Rules (OWASP, etc.)
```

### Progressive Enhancement
```txt
Public content: High threshold (score < 10)
Authenticated: Medium threshold (score < 30)
Sensitive: Low threshold (score < 50) + JSD
```

### Zero Trust for Bots
```txt
1. Default deny (all scores < 30)
2. Allowlist verified bots
3. Allowlist mobile apps (JA3/JA4)
4. Allowlist corporate proxies
5. Allowlist static resources
```

## Integration Points

### WAF Custom Rules
- Primary enforcement mechanism
- Expression builder with bot variables
- Actions: Allow, Block, Managed Challenge, JS Challenge, Skip

### Rate Limiting Rules
- Bot score as dimension
- Stricter limits for low scores
- Bypass for verified bots

### Transform Rules
- Modify headers based on bot score
- Pass score to origin
- Custom logging headers

### Workers
- Programmatic bot logic
- Custom scoring algorithms
- Integration with external APIs
- A/B testing bot rules

### Page Rules / Configuration Rules
- Zone-level overrides
- Path-specific settings
- Cache behavior based on bot score

### Logpush
- Stream to Datadog, Splunk, etc.
- Bot score analysis
- Threat intelligence feeds

## Limitations & Caveats

### Bot Score Limitations
- Score = 0 means not computed (not score = 100)
- First request may not have JSD data
- Score doesn't guarantee 100% accuracy
- False positives/negatives possible

### JavaScript Detections
- Doesn't work on first HTML page visit
- Requires JavaScript-enabled browser
- Strips ETags from HTML responses
- Not compatible with some CSP configurations
- Not supported via `<meta>` CSP tags

### JA3/JA4 Fingerprints
- Only available for HTTPS/TLS traffic
- Missing for Worker-routed traffic
- Not unique per user (shared by clients)
- Can change with browser/library updates

### Plan Restrictions
- Granular scores: Enterprise only
- JA3/JA4: Enterprise only
- Anomaly Detection: Enterprise only
- Corporate Proxy detection: Enterprise only
- Verified bot categories: All plans (limited actions on Free)

### Technical Constraints
- Max 25 WAF custom rules (varies by plan)
- Workers CPU time limits apply
- Bot Analytics sampling (1-10%)
- 30-day maximum history
- CSP requirements for JSD

## References

### Official Documentation
- Bot Management: https://developers.cloudflare.com/bots/
- Bot Scores: https://developers.cloudflare.com/bots/concepts/bot-score/
- WAF Custom Rules: https://developers.cloudflare.com/waf/custom-rules/
- Workers API: https://developers.cloudflare.com/workers/runtime-apis/request/#incomingrequestcfproperties
- Bot Analytics: https://developers.cloudflare.com/bots/bot-analytics/

### Key Fields Reference
```txt
cf.bot_management.score                    # 0-99
cf.bot_management.verified_bot             # boolean
cf.bot_management.static_resource          # boolean
cf.bot_management.ja3_hash                 # string
cf.bot_management.ja4                      # string
cf.bot_management.detection_ids            # array
cf.bot_management.js_detection.passed      # boolean
cf.bot_management.corporate_proxy          # boolean
cf.verified_bot_category                   # string
```

### Workers Fields
```typescript
request.cf.botManagement.score
request.cf.botManagement.verifiedBot
request.cf.botManagement.staticResource
request.cf.botManagement.ja3Hash
request.cf.botManagement.ja4
request.cf.botManagement.detectionIds
request.cf.botManagement.jsDetection.passed
request.cf.botManagement.corporateProxy
request.cf.verifiedBotCategory
```

## Skill Usage Triggers

Use this skill when:
- Implementing bot detection/mitigation
- Creating WAF rules with bot variables
- Integrating Bot Management with Workers
- Configuring JavaScript Detections
- Analyzing bot traffic patterns
- Troubleshooting false positives/negatives
- Setting up JA3/JA4 fingerprinting
- Handling verified bots
- Optimizing bot protection strategy
- Questions about bot score thresholds
- Integration with rate limiting/WAF
