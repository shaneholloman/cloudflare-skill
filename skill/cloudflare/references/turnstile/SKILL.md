# Cloudflare Turnstile Implementation Skill

Expert guidance for implementing Cloudflare Turnstile - a smart CAPTCHA alternative that protects websites from bots without showing traditional CAPTCHA puzzles.

## Core Concepts

### What is Turnstile?
- Smart CAPTCHA alternative using non-interactive JavaScript challenges
- Works without requiring Cloudflare CDN - can be embedded on any website
- Issues challenges through proof-of-work, proof-of-space, web API probing, and browser behavior detection
- Adapts challenge difficulty based on visitor/browser signals
- WCAG 2.1 AA compliant

### Widget Types
1. **Managed** (recommended): Auto-chooses between non-interactive and interactive checkbox
2. **Non-Interactive**: Shows loading spinner, never requires user interaction
3. **Invisible**: Completely hidden, no visual indication

## Architecture

### Client-Server Flow
```
1. Page loads → Turnstile script loads
2. Widget renders → JavaScript challenges run in browser
3. Challenge completes → Token generated
4. Form submits → Token sent to your server
5. Server validates → Calls Siteverify API
6. Cloudflare responds → success/failure
7. Server acts → Allow/reject request
```

### Key Components
- **Sitekey**: Public key for client-side widget (safe to expose)
- **Secret key**: Private key for server-side validation (NEVER expose in client code)
- **Token**: Single-use verification token, expires after 300 seconds (5 minutes)
- **Siteverify API**: `https://challenges.cloudflare.com/turnstile/v0/siteverify`

## Implementation Patterns

### Client-Side: Implicit Rendering
**Use for**: Static forms, simple implementations, immediate loading

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
  <!-- CRITICAL: Never proxy or cache api.js - must fetch from exact URL -->
</head>
<body>
  <form action="/submit" method="POST">
    <input type="email" name="email" required />
    
    <!-- Basic widget -->
    <div class="cf-turnstile" data-sitekey="YOUR_SITEKEY"></div>
    
    <!-- With configuration -->
    <div 
      class="cf-turnstile"
      data-sitekey="YOUR_SITEKEY"
      data-theme="auto"
      data-size="normal"
      data-callback="onSuccess"
      data-error-callback="onError"
      data-expired-callback="onExpired"
      data-action="form_submit"
    ></div>
    
    <button type="submit">Submit</button>
  </form>
  
  <script>
    function onSuccess(token) {
      console.log('Success:', token);
      document.querySelector('button[type="submit"]').disabled = false;
    }
    
    function onError(errorCode) {
      console.error('Error:', errorCode);
    }
    
    function onExpired() {
      console.warn('Token expired');
    }
  </script>
</body>
</html>
```

**Auto-integration**: When widget is inside a `<form>`, a hidden input `cf-turnstile-response` is automatically created containing the token.

### Client-Side: Explicit Rendering
**Use for**: SPAs, React/Vue/Svelte, conditional rendering, dynamic forms, multiple widgets

```html
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js?render=explicit" defer></script>
<div id="turnstile-container"></div>

<script>
  // Basic explicit render
  const widgetId = turnstile.render('#turnstile-container', {
    sitekey: 'YOUR_SITEKEY',
    theme: 'auto',
    size: 'normal',
    callback: (token) => console.log('Success:', token),
    'error-callback': (error) => console.error('Error:', error),
    'expired-callback': () => console.warn('Expired'),
    execution: 'render', // or 'execute' for manual trigger
    appearance: 'always', // or 'execute', 'interaction-only'
    action: 'form_submit',
    cData: 'user_metadata'
  });
  
  // Widget lifecycle methods
  turnstile.getResponse(widgetId);  // Get current token
  turnstile.reset(widgetId);        // Reset widget (on error/expiry)
  turnstile.remove(widgetId);       // Remove widget from DOM
  turnstile.execute('#container');  // Manually trigger challenge (execution: 'execute')
  turnstile.isExpired(widgetId);    // Check if expired
</script>
```

### React/TypeScript Pattern
```typescript
import { useEffect, useRef, useState } from 'react';

interface TurnstileProps {
  sitekey: string;
  onSuccess: (token: string) => void;
  onError?: (error: string) => void;
  theme?: 'light' | 'dark' | 'auto';
  size?: 'normal' | 'compact' | 'flexible';
}

export function Turnstile({ sitekey, onSuccess, onError, theme = 'auto', size = 'normal' }: TurnstileProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const widgetIdRef = useRef<string | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const loadTurnstile = () => {
      if (window.turnstile && containerRef.current) {
        widgetIdRef.current = window.turnstile.render(containerRef.current, {
          sitekey,
          theme,
          size,
          callback: onSuccess,
          'error-callback': onError || (() => {}),
        });
      }
    };

    if (window.turnstile) {
      loadTurnstile();
    } else {
      window.onloadTurnstileCallback = loadTurnstile;
    }

    return () => {
      if (widgetIdRef.current && window.turnstile) {
        window.turnstile.remove(widgetIdRef.current);
      }
    };
  }, [sitekey, theme, size, onSuccess, onError]);

  return <div ref={containerRef} />;
}

// Type declarations
declare global {
  interface Window {
    turnstile?: {
      render: (container: HTMLElement | string, options: any) => string;
      remove: (widgetId: string) => void;
      reset: (widgetId: string) => void;
      getResponse: (widgetId: string) => string;
      isExpired: (widgetId: string) => boolean;
      execute: (container: HTMLElement | string) => void;
    };
    onloadTurnstileCallback?: () => void;
  }
}
```

### Server-Side Validation (MANDATORY)

**CRITICAL SECURITY REQUIREMENTS**:
- Server-side validation is NOT optional - client-side alone is insecure
- Tokens can be forged, expire after 5 min, and are single-use
- Secret key must NEVER be in client code
- Always validate before processing requests

#### Node.js/Express Pattern
```javascript
const SECRET_KEY = process.env.TURNSTILE_SECRET_KEY;

async function validateTurnstile(token, remoteip) {
  const formData = new FormData();
  formData.append('secret', SECRET_KEY);
  formData.append('response', token);
  formData.append('remoteip', remoteip);
  
  try {
    const response = await fetch(
      'https://challenges.cloudflare.com/turnstile/v0/siteverify',
      { method: 'POST', body: formData }
    );
    return await response.json();
  } catch (error) {
    console.error('Validation error:', error);
    return { success: false, 'error-codes': ['internal-error'] };
  }
}

// Express route example
app.post('/submit-form', async (req, res) => {
  const token = req.body['cf-turnstile-response'];
  const ip = req.headers['cf-connecting-ip'] || 
             req.headers['x-forwarded-for'] || 
             req.ip;
  
  const validation = await validateTurnstile(token, ip);
  
  if (validation.success) {
    // Optionally validate action and hostname
    if (validation.action !== 'form_submit') {
      return res.status(400).json({ error: 'Invalid action' });
    }
    if (validation.hostname !== 'example.com') {
      return res.status(400).json({ error: 'Invalid hostname' });
    }
    
    // Process form
    return res.json({ success: true });
  } else {
    return res.status(400).json({ 
      error: 'Verification failed', 
      codes: validation['error-codes'] 
    });
  }
});
```

#### Cloudflare Workers Pattern
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method === 'POST') {
      const formData = await request.formData();
      const token = formData.get('cf-turnstile-response');
      
      // Validate with Turnstile
      const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
      const validation = await validateTurnstile(
        token as string,
        env.TURNSTILE_SECRET_KEY,
        ip
      );
      
      if (!validation.success) {
        return new Response('Invalid verification', { status: 400 });
      }
      
      // Process request
      return new Response('Success');
    }
    
    return new Response('Method not allowed', { status: 405 });
  }
};

async function validateTurnstile(
  token: string,
  secret: string,
  remoteip: string
): Promise<TurnstileResponse> {
  const formData = new FormData();
  formData.append('secret', secret);
  formData.append('response', token);
  formData.append('remoteip', remoteip);
  
  const result = await fetch(
    'https://challenges.cloudflare.com/turnstile/v0/siteverify',
    { method: 'POST', body: formData }
  );
  
  return await result.json();
}

interface TurnstileResponse {
  success: boolean;
  'error-codes'?: string[];
  challenge_ts?: string;
  hostname?: string;
  action?: string;
  cdata?: string;
}
```

#### Python/Flask Pattern
```python
import requests
from flask import Flask, request, jsonify

app = Flask(__name__)
SECRET_KEY = os.environ['TURNSTILE_SECRET_KEY']

def validate_turnstile(token, remoteip=None):
    url = 'https://challenges.cloudflare.com/turnstile/v0/siteverify'
    data = {'secret': SECRET_KEY, 'response': token}
    if remoteip:
        data['remoteip'] = remoteip
    
    try:
        response = requests.post(url, data=data, timeout=10)
        response.raise_for_status()
        return response.json()
    except requests.RequestException as e:
        return {'success': False, 'error-codes': ['internal-error']}

@app.route('/submit', methods=['POST'])
def submit_form():
    token = request.form.get('cf-turnstile-response')
    ip = request.headers.get('CF-Connecting-IP') or \
         request.headers.get('X-Forwarded-For') or \
         request.remote_addr
    
    validation = validate_turnstile(token, ip)
    
    if validation['success']:
        # Process form
        return jsonify({'status': 'success'})
    else:
        return jsonify({
            'status': 'error',
            'errors': validation['error-codes']
        }), 400
```

### Siteverify API Response

**Success Response**:
```json
{
  "success": true,
  "challenge_ts": "2022-02-28T15:14:30.096Z",
  "hostname": "example.com",
  "error-codes": [],
  "action": "login",
  "cdata": "session-123",
  "metadata": {
    "ephemeral_id": "x:9f78e0ed210960d7693b167e"
  }
}
```

**Failure Response**:
```json
{
  "success": false,
  "error-codes": ["invalid-input-response"]
}
```

**Error Codes**:
- `missing-input-secret`: Secret key not provided
- `invalid-input-secret`: Secret key invalid/expired
- `missing-input-response`: Token not provided
- `invalid-input-response`: Token invalid/malformed/expired
- `bad-request`: Malformed request
- `timeout-or-duplicate`: Token already validated or expired
- `internal-error`: Internal error (retry)

## Configuration Options

### Widget Configurations
```javascript
{
  sitekey: 'required',              // Your widget sitekey
  action: 'optional-string',        // Custom action identifier for analytics
  cData: 'optional-string',         // Custom data passed back in validation
  callback: (token) => {},          // Success callback with token
  'error-callback': (code) => {},   // Error callback
  'expired-callback': () => {},     // Token expiry callback
  'timeout-callback': () => {},     // Challenge timeout callback
  'before-interactive-callback': () => {}, // Before showing checkbox
  'after-interactive-callback': () => {},  // After checkbox interaction
  theme: 'auto',                    // 'light', 'dark', 'auto'
  size: 'normal',                   // 'normal', 'flexible', 'compact'
  tabindex: 0,                      // Tab index for accessibility
  'response-field': true,           // Create hidden input (default: true)
  'response-field-name': 'cf-turnstile-response', // Hidden input name
  retry: 'auto',                    // 'auto', 'never'
  'retry-interval': 8000,           // Retry interval in ms
  'refresh-expired': 'auto',        // 'auto', 'manual', 'never'
  language: 'auto',                 // Language code or 'auto'
  execution: 'render',              // 'render' (auto), 'execute' (manual)
  appearance: 'always',             // 'always', 'execute', 'interaction-only'
}
```

### Themes
- `'auto'`: Matches system dark mode preference
- `'light'`: Light theme
- `'dark'`: Dark theme

### Sizes
- `'normal'`: 300x65px - Standard size
- `'flexible'`: 100% width, responsive (max 100vw)
- `'compact'`: 150x140px - Smaller footprint

### Execution Modes
- `'render'`: Auto-execute challenge immediately
- `'execute'`: Manual execution via `turnstile.execute()`

### Appearance Modes
- `'always'`: Widget always visible
- `'execute'`: Widget appears when challenge runs
- `'interaction-only'`: Only visible for interactive challenges (managed mode)

## Testing

### Test Sitekeys
```javascript
// Always passes - visible widget
'1x00000000000000000000AA'

// Always fails - visible widget
'2x00000000000000000000AB'

// Always passes - invisible widget
'1x00000000000000000000BB'

// Always fails - invisible widget
'2x00000000000000000000BB'

// Forces interactive challenge
'3x00000000000000000000FF'
```

### Test Secret Keys
```javascript
// Always passes validation
'1x0000000000000000000000000000000AA'

// Always fails validation
'2x0000000000000000000000000000000AA'

// Returns "token already spent" error
'3x0000000000000000000000000000000AA'
```

### Environment-Based Config
```bash
# .env.development
TURNSTILE_SITEKEY=1x00000000000000000000AA
TURNSTILE_SECRET_KEY=1x0000000000000000000000000000000AA

# .env.production
TURNSTILE_SITEKEY=your-real-sitekey
TURNSTILE_SECRET_KEY=your-real-secret-key
```

```javascript
const config = {
  sitekey: process.env.NODE_ENV === 'production'
    ? process.env.TURNSTILE_SITEKEY
    : '1x00000000000000000000AA',
  secretKey: process.env.NODE_ENV === 'production'
    ? process.env.TURNSTILE_SECRET_KEY
    : '1x0000000000000000000000000000000AA'
};
```

**Test Token**: Test sitekeys generate dummy token `XXXX.DUMMY.TOKEN.XXXX`
- Test secret keys only accept dummy tokens
- Production secret keys reject dummy tokens

## Widget Management

### Creating Widgets

**Via Dashboard**:
1. Log into Cloudflare Dashboard
2. Navigate to Turnstile
3. Create widget → Get sitekey + secret key
4. Configure mode, hostnames, domain restrictions

**Via API**:
```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/challenges/widgets" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "Login Form Widget",
    "mode": "managed",
    "domains": ["example.com", "www.example.com"]
  }'
```

**Via Terraform**:
```hcl
resource "cloudflare_turnstile_widget" "login_form" {
  account_id = var.cloudflare_account_id
  name       = "Login Form Widget"
  mode       = "managed"
  domains    = ["example.com", "www.example.com"]
  
  # Optional
  region     = "world"
  offlabel   = false
}

output "turnstile_sitekey" {
  value = cloudflare_turnstile_widget.login_form.sitekey
}

output "turnstile_secret" {
  value     = cloudflare_turnstile_widget.login_form.secret
  sensitive = true
}
```

### Widget Modes
- `managed`: Adaptive (shows checkbox if needed) - **RECOMMENDED**
- `non-interactive`: Never requires user interaction
- `invisible`: Completely hidden

### Hostname Management
- Restrict widgets to specific domains
- Supports wildcards: `*.example.com`
- Test keys work on `localhost`, `127.0.0.1`, any domain
- Production widgets should NOT allow local domains

## Advanced Patterns

### Enhanced Validation with Retry & Idempotency
```javascript
const crypto = require('crypto');

async function validateWithRetry(token, remoteip, maxRetries = 3) {
  const idempotencyKey = crypto.randomUUID();
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const formData = new FormData();
      formData.append('secret', SECRET_KEY);
      formData.append('response', token);
      formData.append('remoteip', remoteip);
      formData.append('idempotency_key', idempotencyKey);
      
      const response = await fetch(
        'https://challenges.cloudflare.com/turnstile/v0/siteverify',
        { method: 'POST', body: formData }
      );
      
      const result = await response.json();
      if (response.ok) return result;
      
      if (attempt === maxRetries) return result;
      
      // Exponential backoff
      await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 1000));
    } catch (error) {
      if (attempt === maxRetries) {
        return { success: false, 'error-codes': ['internal-error'] };
      }
    }
  }
}
```

### Action & Hostname Validation
```javascript
async function validateEnhanced(token, remoteip, expectedAction, expectedHostname) {
  const validation = await validateTurnstile(token, remoteip);
  
  if (!validation.success) {
    return { valid: false, reason: 'turnstile_failed', errors: validation['error-codes'] };
  }
  
  if (expectedAction && validation.action !== expectedAction) {
    return { valid: false, reason: 'action_mismatch', expected: expectedAction, received: validation.action };
  }
  
  if (expectedHostname && validation.hostname !== expectedHostname) {
    return { valid: false, reason: 'hostname_mismatch', expected: expectedHostname, received: validation.hostname };
  }
  
  const challengeTime = new Date(validation.challenge_ts);
  const ageMinutes = (new Date() - challengeTime) / (1000 * 60);
  
  if (ageMinutes > 4) {
    console.warn(`Token is ${ageMinutes.toFixed(1)} minutes old`);
  }
  
  return { valid: true, data: validation, tokenAge: ageMinutes };
}

// Usage
const result = await validateEnhanced(token, ip, 'login', 'example.com');
if (result.valid) {
  // Process request
} else {
  console.log('Validation failed:', result.reason);
}
```

### SPA Widget Manager Class
```javascript
class TurnstileManager {
  constructor() {
    this.widgets = new Map();
  }
  
  createWidget(containerId, config) {
    return new Promise((resolve) => {
      if (window.turnstile) {
        const widgetId = window.turnstile.render(containerId, {
          sitekey: config.sitekey,
          theme: config.theme || 'auto',
          callback: (token) => {
            if (config.onSuccess) config.onSuccess(token, widgetId);
          },
          'error-callback': (error) => {
            if (config.onError) config.onError(error, widgetId);
          }
        });
        this.widgets.set(containerId, widgetId);
        resolve(widgetId);
      }
    });
  }
  
  removeWidget(containerId) {
    const widgetId = this.widgets.get(containerId);
    if (widgetId && window.turnstile) {
      window.turnstile.remove(widgetId);
      this.widgets.delete(containerId);
    }
  }
  
  resetWidget(containerId) {
    const widgetId = this.widgets.get(containerId);
    if (widgetId && window.turnstile) {
      window.turnstile.reset(widgetId);
    }
  }
  
  getToken(containerId) {
    const widgetId = this.widgets.get(containerId);
    return widgetId && window.turnstile ? window.turnstile.getResponse(widgetId) : null;
  }
}

// Usage
const manager = new TurnstileManager();

document.getElementById('show-form').addEventListener('click', async () => {
  await manager.createWidget('#turnstile-widget', {
    sitekey: 'YOUR_SITEKEY',
    onSuccess: (token) => console.log('Ready to submit'),
    onError: (error) => console.error('Error:', error)
  });
});
```

## Security Best Practices

### Secret Key Protection
- ✅ Store in environment variables
- ✅ Use secret management (HashiCorp Vault, AWS Secrets Manager)
- ✅ Rotate keys regularly via API/dashboard
- ✅ Use different widgets for dev/staging/prod
- ❌ NEVER expose in client-side code
- ❌ NEVER commit to git
- ❌ NEVER log secret keys

### Validation Requirements
- ✅ Always validate server-side
- ✅ Validate before processing any requests
- ✅ Check action and hostname when specified
- ✅ Handle expired tokens (redirect to form)
- ✅ Log failed validations for abuse monitoring
- ✅ Use HTTPS for all validation requests
- ❌ NEVER trust client-side validation alone
- ❌ NEVER skip server validation

### Token Handling
- Tokens are single-use (one validation only)
- Tokens expire after 300 seconds (5 minutes)
- Store tokens temporarily if needed for multi-step flows
- Reset widget on form errors requiring resubmission

### Hostname Restrictions
- Restrict production widgets to actual domains
- Use wildcards carefully: `*.example.com`
- Never allow `localhost` in production widgets
- Monitor unexpected hostnames in validation responses

## Common Pitfalls

### ❌ Skipping Server-Side Validation
```javascript
// WRONG - Client-only, easily bypassed
<form onsubmit="return window.turnstileToken ? true : false">
```

```javascript
// CORRECT - Always validate server-side
app.post('/submit', async (req, res) => {
  const validation = await validateTurnstile(token, ip);
  if (!validation.success) {
    return res.status(400).json({ error: 'Invalid' });
  }
  // Process request
});
```

### ❌ Exposing Secret Key
```javascript
// WRONG - Secret in client code
const SECRET = 'your-secret-key';
fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
  body: JSON.stringify({ secret: SECRET, response: token })
});
```

```javascript
// CORRECT - Validation on server only
// Client sends token to your backend
// Backend validates with secret key
```

### ❌ Not Handling Token Expiry
```javascript
// WRONG - No expiry handling
form.addEventListener('submit', (e) => {
  e.preventDefault();
  submitForm(turnstileToken);
});
```

```javascript
// CORRECT - Reset on expiry
turnstile.render('#widget', {
  sitekey: 'YOUR_SITEKEY',
  'expired-callback': () => {
    turnstile.reset(widgetId);
    submitButton.disabled = true;
  },
  callback: (token) => {
    submitButton.disabled = false;
  }
});
```

### ❌ Proxying/Caching api.js
```html
<!-- WRONG - Will break on updates -->
<script src="/cached/turnstile-api.js"></script>
```

```html
<!-- CORRECT - Always load from Cloudflare -->
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
```

### ❌ Ignoring Validation Response Fields
```javascript
// WRONG - Only checking success
if (validation.success) { processForm(); }
```

```javascript
// CORRECT - Validate action/hostname
if (validation.success && 
    validation.action === 'login' &&
    validation.hostname === 'example.com') {
  processForm();
} else {
  logSecurityEvent(validation);
}
```

## Performance Optimization

### Preconnect Hint
```html
<head>
  <link rel="preconnect" href="https://challenges.cloudflare.com">
  <script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
</head>
```

### Load Early
Execute Turnstile as early as possible so verification completes before user submits form.

### Defer Non-Critical Scripts
```html
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
```

### Use Invisible Mode for Best UX
```javascript
turnstile.render('#container', {
  sitekey: 'YOUR_SITEKEY',
  size: 'invisible',
  execution: 'execute' // Trigger when user attempts action
});

// Later, when user clicks submit
turnstile.execute('#container');
```

## Monitoring & Analytics

### Turnstile Analytics Dashboard
- Challenge solve rate
- Number of challenges issued
- Widget performance metrics
- Error rates by widget

### Custom Logging
```javascript
async function validateWithLogging(token, remoteip, userId) {
  const startTime = Date.now();
  const validation = await validateTurnstile(token, remoteip);
  const duration = Date.now() - startTime;
  
  await logValidation({
    userId,
    success: validation.success,
    errorCodes: validation['error-codes'] || [],
    hostname: validation.hostname,
    action: validation.action,
    duration,
    ip: remoteip
  });
  
  return validation;
}
```

## Migration from Other CAPTCHAs

### From reCAPTCHA
```html
<!-- Before: reCAPTCHA -->
<script src="https://www.google.com/recaptcha/api.js" async defer></script>
<div class="g-recaptcha" data-sitekey="OLD_SITEKEY"></div>

<!-- After: Turnstile -->
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
<div class="cf-turnstile" data-sitekey="NEW_SITEKEY"></div>
```

Server-side changes:
```javascript
// Before: reCAPTCHA verification
const response = await fetch(
  'https://www.google.com/recaptcha/api/siteverify',
  { method: 'POST', body: formData }
);

// After: Turnstile verification
const response = await fetch(
  'https://challenges.cloudflare.com/turnstile/v0/siteverify',
  { method: 'POST', body: formData }
);
```

### From hCaptcha
Similar to reCAPTCHA - just update script URL, widget class, and validation endpoint.

## Mobile Implementation

### WebView Considerations
- Turnstile works in mobile WebViews
- Ensure JavaScript is enabled
- Test on both iOS WKWebView and Android WebView
- Consider invisible mode for better mobile UX

### React Native Example
```javascript
import { WebView } from 'react-native-webview';

<WebView
  source={{ uri: 'https://your-site.com/form' }}
  javaScriptEnabled={true}
  onMessage={(event) => {
    const token = event.nativeEvent.data;
    // Handle token from WebView
  }}
/>
```

## Troubleshooting

### Widget Not Rendering
- Check browser console for errors
- Verify sitekey is correct
- Ensure JavaScript is enabled
- Check for CSP (Content Security Policy) blocking script
- Verify not using `file://` protocol (only `http://` and `https://` work)

### Validation Failing
- Check secret key is correct
- Verify token not expired (>5 min old)
- Ensure token not already validated (single-use)
- Check server can reach Siteverify API
- Verify not using test secret key in production

### CSP Configuration
```html
<meta http-equiv="Content-Security-Policy" 
      content="script-src 'self' https://challenges.cloudflare.com; 
               frame-src https://challenges.cloudflare.com;">
```

## Reference

### Official Docs
- [Turnstile Overview](https://developers.cloudflare.com/turnstile/)
- [Get Started Guide](https://developers.cloudflare.com/turnstile/get-started/)
- [Client-Side Rendering](https://developers.cloudflare.com/turnstile/get-started/client-side-rendering/)
- [Server-Side Validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)
- [Widget Concepts](https://developers.cloudflare.com/turnstile/concepts/widget/)
- [Testing Guide](https://developers.cloudflare.com/turnstile/troubleshooting/testing/)

### Key URLs
- **API Script**: `https://challenges.cloudflare.com/turnstile/v0/api.js`
- **Siteverify API**: `https://challenges.cloudflare.com/turnstile/v0/siteverify`
- **Dashboard**: `https://dash.cloudflare.com/`

### Token Limits
- **Length**: Max 2048 characters
- **Validity**: 300 seconds (5 minutes)
- **Usage**: Single-use only

### Widget Limits
- Multiple widgets per page supported
- Different configurations per widget
- Each widget needs unique container element
