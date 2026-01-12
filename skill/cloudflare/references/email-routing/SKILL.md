# Cloudflare Email Routing Skill

## Overview

Cloudflare Email Routing enables custom email addresses for your domain that route to verified destination addresses. It's free, privacy-focused (no storage/access), and includes Email Workers for programmatic email processing.

**Available to all Cloudflare customers using Cloudflare as authoritative nameserver.**

## Core Capabilities

### Email Routing Features
- **Custom Addresses**: Create unlimited custom email addresses (e.g., `info@example.com`)
- **Email Workers**: Process emails with Cloudflare Workers (allowlists, blocklists, custom logic)
- **Catch-all**: Handle typos/variations of email addresses
- **Subaddressing**: Support plus addressing (RFC 5233) like `user+detail@example.com`
- **Drop Action**: Make addresses appear valid without routing (privacy)

### Limits
- **Message size**: Max 25 MiB
- **Rules**: 200 per zone
- **Destination addresses**: 200 per account
- **Email Workers**: Subject to Workers CPU/memory limits (Free: allocation errors possible)

## Email Workers

### Runtime API

Email Workers use the `email` handler with the `ForwardableEmailMessage` interface:

```typescript
export default {
  async email(message, env, ctx) {
    // Process email
    await message.forward("destination@example.com");
  },
};
```

### ForwardableEmailMessage Interface

```typescript
interface ForwardableEmailMessage {
  readonly from: string;           // Envelope From
  readonly to: string;             // Envelope To
  readonly headers: Headers;       // Email headers
  readonly raw: ReadableStream;    // Raw email content stream
  readonly rawSize: number;        // Email size in bytes

  setReject(reason: string): void;
  forward(rcptTo: string, headers?: Headers): Promise<void>;
  reply(message: EmailMessage): Promise<void>;
}
```

### EmailMessage (for sending/replying)

```typescript
interface EmailMessage {
  readonly from: string;
  readonly to: string;
}
```

## Common Patterns

### 1. Allowlist Email Worker

```typescript
export default {
  async email(message, env, ctx) {
    const allowList = ["friend@example.com", "coworker@example.com"];
    if (allowList.indexOf(message.from) == -1) {
      message.setReject("Address not allowed");
    } else {
      await message.forward("inbox@corp");
    }
  },
};
```

### 2. Parse Email with postal-mime

```typescript
import * as PostalMime from 'postal-mime';

export default {
  async email(message, env, ctx) {
    const parser = new PostalMime.default();
    const rawEmail = new Response(message.raw);
    const email = await parser.parse(await rawEmail.arrayBuffer());
    
    console.log({
      from: email.from,
      to: email.to,
      subject: email.subject,
      html: email.html,
      attachments: email.attachments
    });
    
    await message.forward("destination@example.com");
  },
};
```

### 3. Forward with Custom Headers

```typescript
export default {
  async email(message, env, ctx) {
    const customHeaders = new Headers();
    customHeaders.set("X-Custom-Header", "value");
    customHeaders.set("X-Processed-By", "Email Worker");
    
    // Only X-* headers allowed
    await message.forward("destination@example.com", customHeaders);
  },
};
```

### 4. Reply to Sender

```typescript
import { EmailMessage } from 'cloudflare:email';
import { createMimeMessage } from 'mimetext';

export default {
  async email(message, env, ctx) {
    const msg = createMimeMessage();
    msg.setSender({ name: 'Auto Reply', addr: 'noreply@example.com' });
    msg.setRecipient(message.from);
    msg.setHeader('In-Reply-To', message.headers.get('Message-ID'));
    msg.setSubject('Re: Your inquiry');
    msg.addMessage({
      contentType: 'text/plain',
      data: 'Thank you for your message. We will respond shortly.',
    });

    const replyMessage = new EmailMessage(
      'noreply@example.com',
      message.from,
      msg.asRaw()
    );
    
    await message.reply(replyMessage);
    await message.forward("team@example.com");
  },
};
```

### 5. Send Email (via fetch handler)

```typescript
import { EmailMessage } from "cloudflare:email";
import { createMimeMessage } from 'mimetext';

export default {
  async fetch(request, env, ctx) {
    const msg = createMimeMessage();
    msg.setSender({ name: 'Sender Name', addr: 'sender@example.com' });
    msg.setRecipient('recipient@example.com');
    msg.setSubject('Email from Worker');
    msg.addMessage({
      contentType: 'text/plain',
      data: 'Message body content',
    });

    const message = new EmailMessage(
      'sender@example.com',
      'recipient@example.com',
      msg.asRaw()
    );
    
    await env.EMAIL.send(message);
    return Response.json({ ok: true });
  }
};
```

### 6. Conditional Routing

```typescript
export default {
  async email(message, env, ctx) {
    const to = message.to.toLowerCase();
    
    if (to.includes('support@')) {
      await message.forward("support-team@company.com");
    } else if (to.includes('sales@')) {
      await message.forward("sales-team@company.com");
    } else if (to.includes('info@')) {
      await message.forward("general@company.com");
    } else {
      message.setReject("Unknown recipient");
    }
  },
};
```

### 7. Spam Score Checking

```typescript
export default {
  async email(message, env, ctx) {
    const spamScore = message.headers.get('X-Cf-Spamh-Score');
    
    if (spamScore && parseInt(spamScore) >= 4) {
      // High spam score (5 = max)
      message.setReject("Spam detected");
    } else {
      await message.forward("inbox@example.com");
    }
  },
};
```

### 8. Store Email in R2

```typescript
import * as PostalMime from 'postal-mime';

export default {
  async email(message, env, ctx) {
    // Parse email
    const parser = new PostalMime.default();
    const rawEmail = new Response(message.raw);
    const email = await parser.parse(await rawEmail.arrayBuffer());
    
    // Store in R2
    const key = `emails/${Date.now()}-${email.messageId}.json`;
    await env.EMAIL_BUCKET.put(key, JSON.stringify({
      from: email.from,
      to: email.to,
      subject: email.subject,
      date: email.date,
      html: email.html,
      text: email.text,
      attachments: email.attachments.map(a => ({
        filename: a.filename,
        mimeType: a.mimeType,
        size: a.content?.length || 0
      }))
    }));
    
    await message.forward("archive@example.com");
  },
};
```

## Wrangler Configuration

### Basic Email Worker

```toml
name = "email-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[send_email]]
name = "EMAIL"
```

Or in JSON:

```jsonc
{
  "name": "email-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-01-01",
  "send_email": [
    {
      "name": "EMAIL"
    }
  ]
}
```

### With KV/R2 Bindings

```toml
name = "email-processor"
main = "src/index.ts"

[[send_email]]
name = "EMAIL"

[[kv_namespaces]]
binding = "EMAIL_METADATA"
id = "your-kv-namespace-id"

[[r2_buckets]]
binding = "EMAIL_BUCKET"
bucket_name = "email-archive"
```

### Local Development

Run locally with `wrangler dev`:

```bash
npx wrangler dev
```

Test receiving email via curl:

```bash
curl --request POST 'http://localhost:8787/cdn-cgi/handler/email' \
  --url-query 'from=sender@example.com' \
  --url-query 'to=recipient@example.com' \
  --header 'Content-Type: application/json' \
  --data-raw 'From: sender@example.com
To: recipient@example.com
Subject: Test Email

Email body content'
```

## REST API Operations

### Authentication

Use API tokens (preferred) or API keys:

```bash
# API Token (recommended)
curl -H "Authorization: Bearer $API_TOKEN" \
  https://api.cloudflare.com/client/v4/...

# API Key (legacy)
curl -H "X-Auth-Email: $EMAIL" \
     -H "X-Auth-Key: $API_KEY" \
  https://api.cloudflare.com/client/v4/...
```

### Enable Email Routing

```bash
POST /zones/{zone_id}/email/routing/dns

curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/dns" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json"
```

### Disable Email Routing

```bash
DELETE /zones/{zone_id}/email/routing/dns

curl -X DELETE "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/dns" \
  -H "Authorization: Bearer $API_TOKEN"
```

### Get Email Routing Settings

```bash
GET /zones/{zone_id}/email/routing

curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing" \
  -H "Authorization: Bearer $API_TOKEN"
```

Response:

```json
{
  "result": {
    "id": "example.com",
    "enabled": true,
    "name": "example.com",
    "tag": "zone-tag",
    "created": "2024-01-01T00:00:00Z",
    "modified": "2024-01-02T00:00:00Z",
    "status": "ready"
  }
}
```

### List Routing Rules

```bash
GET /zones/{zone_id}/email/routing/rules

curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules" \
  -H "Authorization: Bearer $API_TOKEN"

# Filter by enabled status
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules?enabled=true" \
  -H "Authorization: Bearer $API_TOKEN"

# Pagination
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules?page=1&per_page=20" \
  -H "Authorization: Bearer $API_TOKEN"
```

Response:

```json
{
  "result": [
    {
      "id": "rule-id",
      "tag": "rule-tag",
      "name": "Forward to team inbox",
      "enabled": true,
      "priority": 0,
      "matchers": [
        {
          "type": "literal",
          "field": "to",
          "value": "info@example.com"
        }
      ],
      "actions": [
        {
          "type": "forward",
          "value": ["team@company.com"]
        }
      ]
    }
  ],
  "result_info": {
    "count": 1,
    "page": 1,
    "per_page": 20,
    "total_count": 1
  }
}
```

### Create Routing Rule

```bash
POST /zones/{zone_id}/email/routing/rules

curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support forwarding",
    "enabled": true,
    "matchers": [
      {
        "type": "literal",
        "field": "to",
        "value": "support@example.com"
      }
    ],
    "actions": [
      {
        "type": "forward",
        "value": ["support-team@company.com"]
      }
    ],
    "priority": 0
  }'
```

### Create Rule with Worker Action

```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Process with Worker",
    "enabled": true,
    "matchers": [
      {
        "type": "literal",
        "field": "to",
        "value": "automated@example.com"
      }
    ],
    "actions": [
      {
        "type": "worker",
        "value": ["email-worker"]
      }
    ]
  }'
```

### Create Drop Rule

```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Drop spam trap",
    "enabled": true,
    "matchers": [
      {
        "type": "literal",
        "field": "to",
        "value": "spamtrap@example.com"
      }
    ],
    "actions": [
      {
        "type": "drop",
        "value": []
      }
    ]
  }'
```

### Update Routing Rule

```bash
PUT /zones/{zone_id}/email/routing/rules/{rule_id}

curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules/$RULE_ID" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": false
  }'
```

### Delete Routing Rule

```bash
DELETE /zones/{zone_id}/email/routing/rules/{rule_id}

curl -X DELETE "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules/$RULE_ID" \
  -H "Authorization: Bearer $API_TOKEN"
```

### Get Catch-All Rule

```bash
GET /zones/{zone_id}/email/routing/rules/catch_all

curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules/catch_all" \
  -H "Authorization: Bearer $API_TOKEN"
```

### Update Catch-All Rule

```bash
PUT /zones/{zone_id}/email/routing/rules/catch_all

curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules/catch_all" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "actions": [
      {
        "type": "forward",
        "value": ["catchall@example.com"]
      }
    ]
  }'
```

### List Destination Addresses

```bash
GET /accounts/{account_id}/email/routing/addresses

curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/email/routing/addresses" \
  -H "Authorization: Bearer $API_TOKEN"
```

Response:

```json
{
  "result": [
    {
      "id": "address-id",
      "tag": "address-tag",
      "email": "user@gmail.com",
      "verified": "2024-01-01T00:00:00Z",
      "created": "2024-01-01T00:00:00Z",
      "modified": "2024-01-01T00:00:00Z"
    }
  ]
}
```

### Create Destination Address

```bash
POST /accounts/{account_id}/email/routing/addresses

curl -X POST "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/email/routing/addresses" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "newdest@example.com"
  }'
```

**Note**: Verification email sent automatically. Must verify before use.

### Delete Destination Address

```bash
DELETE /accounts/{account_id}/email/routing/addresses/{address_id}

curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/email/routing/addresses/$ADDRESS_ID" \
  -H "Authorization: Bearer $API_TOKEN"
```

**Warning**: Disables all rules using this destination.

### Get DNS Records for Email Routing

```bash
GET /zones/{zone_id}/email/routing/dns

curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/dns" \
  -H "Authorization: Bearer $API_TOKEN"
```

Response shows required MX and SPF records:

```json
{
  "result": [
    {
      "name": "example.com",
      "type": "MX",
      "content": "amir.mx.cloudflare.net",
      "priority": 13,
      "locked": true
    },
    {
      "name": "example.com",
      "type": "TXT",
      "content": "v=spf1 include:_spf.mx.cloudflare.net ~all"
    }
  ]
}
```

## DNS Configuration

Email Routing automatically adds:

### MX Records

```
example.com. 300 IN MX 13 amir.mx.cloudflare.net.
example.com. 300 IN MX 86 linda.mx.cloudflare.net.
example.com. 300 IN MX 24 isaac.mx.cloudflare.net.
```

### SPF Record

```
example.com. 300 IN TXT "v=spf1 include:_spf.mx.cloudflare.net ~all"
```

### DKIM (automatic)

Cloudflare adds two DKIM signatures:
1. `cf2024-1._domainkey.email.cloudflare.net` (Cloudflare)
2. `cf2024-1._domainkey.example.com` (your domain)

Query your DKIM:

```bash
dig TXT cf2024-1._domainkey.example.com +short
```

## Architecture & Technical Details

### Email Flow

1. **Inbound**: Sender → MX records → Cloudflare Email Routing
2. **Processing**: Email Workers (optional) → Rules evaluation
3. **Outbound**: Cloudflare → Destination address

### Sender Rewriting (SRS)

Email Routing rewrites `MAIL FROM` envelope to avoid SPF issues while preserving original `From:` header for user experience.

### Authentication

- **SPF**: Enforced via `_spf.mx.cloudflare.net`
- **DKIM**: Dual signatures (Cloudflare + custom domain)
- **DMARC**: Enforced (rejects on auth failure)
- **ARC**: Supported (preserves auth results through forwarding)
- **Mail Auth Requirement**: Must pass SPF or DKIM to forward

### Outbound IPs

**IPv4**: `104.30.0.0/19`
**IPv6**: `2405:8100:c000::/38`

**Hostnames** (HELO/EHLO):
- `cloudflare-email.net`
- `cloudflare-email.org`
- `cloudflare-email.com`

### Anti-Spam

- **DNSBL/RBL**: Blocks known abusive IPs
- **Spam Score**: `X-Cf-Spamh-Score` header (0-5, experimental)

## Use Cases

### 1. Email Alias System

Create department aliases forwarding to teams:

```typescript
export default {
  async email(message, env, ctx) {
    const aliases = {
      'support@': 'support-team@company.com',
      'sales@': 'sales-team@company.com',
      'info@': 'general@company.com',
      'abuse@': 'security@company.com'
    };
    
    for (const [alias, destination] of Object.entries(aliases)) {
      if (message.to.includes(alias)) {
        await message.forward(destination);
        return;
      }
    }
    
    message.setReject("Unknown recipient");
  }
};
```

### 2. Support Ticket System

Parse emails and create tickets:

```typescript
import * as PostalMime from 'postal-mime';

export default {
  async email(message, env, ctx) {
    const parser = new PostalMime.default();
    const rawEmail = new Response(message.raw);
    const email = await parser.parse(await rawEmail.arrayBuffer());
    
    // Create ticket in external system
    await fetch('https://tickets.company.com/api/create', {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${env.TICKET_API_KEY}` },
      body: JSON.stringify({
        from: email.from.address,
        subject: email.subject,
        body: email.text || email.html,
        attachments: email.attachments.length
      })
    });
    
    // Auto-reply to sender
    const reply = createAutoReply(message, email);
    await message.reply(reply);
    
    // Forward to support team
    await message.forward('support@company.com');
  }
};
```

### 3. Newsletter/Marketing Integration

Forward to marketing platform:

```typescript
export default {
  async email(message, env, ctx) {
    if (message.to.includes('subscribe@')) {
      // Extract email address from sender
      await fetch('https://api.mailchimp.com/3.0/lists/LIST_ID/members', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${env.MAILCHIMP_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          email_address: message.from,
          status: 'subscribed'
        })
      });
    }
    
    await message.forward('marketing@company.com');
  }
};
```

### 4. Email Archival

Store all emails in R2 for compliance:

```typescript
import * as PostalMime from 'postal-mime';

export default {
  async email(message, env, ctx) {
    // Parse email
    const parser = new PostalMime.default();
    const rawEmail = new Response(message.raw);
    const [email, rawBuffer] = await Promise.all([
      parser.parse(await rawEmail.arrayBuffer()),
      new Response(message.raw).arrayBuffer()
    ]);
    
    // Store raw email
    const timestamp = new Date().toISOString();
    const key = `archive/${timestamp}-${email.messageId}.eml`;
    await env.EMAIL_BUCKET.put(key, rawBuffer);
    
    // Store metadata in KV
    await env.EMAIL_METADATA.put(email.messageId, JSON.stringify({
      from: email.from.address,
      to: email.to.map(t => t.address),
      subject: email.subject,
      date: email.date,
      archived: timestamp,
      s3Key: key
    }));
    
    await message.forward('archive@company.com');
  }
};
```

### 5. Domain-based Routing

Route based on sender domain:

```typescript
export default {
  async email(message, env, ctx) {
    const domain = message.from.split('@')[1];
    
    const routes = {
      'partner1.com': 'partner1-inbox@company.com',
      'partner2.com': 'partner2-inbox@company.com',
      'customer.com': 'customer-success@company.com'
    };
    
    if (routes[domain]) {
      await message.forward(routes[domain]);
    } else {
      await message.forward('general@company.com');
    }
  }
};
```

### 6. Content Filtering

Block emails with specific content:

```typescript
import * as PostalMime from 'postal-mime';

export default {
  async email(message, env, ctx) {
    const parser = new PostalMime.default();
    const rawEmail = new Response(message.raw);
    const email = await parser.parse(await rawEmail.arrayBuffer());
    
    const blockedTerms = ['viagra', 'lottery', 'prize'];
    const content = (email.text || email.html || '').toLowerCase();
    
    for (const term of blockedTerms) {
      if (content.includes(term)) {
        message.setReject('Content policy violation');
        return;
      }
    }
    
    // Check spam score
    const spamScore = parseInt(message.headers.get('X-Cf-Spamh-Score') || '0');
    if (spamScore >= 4) {
      message.setReject('High spam score');
      return;
    }
    
    await message.forward('inbox@company.com');
  }
};
```

## Known Limitations

1. **No internationalized local-parts**: `piñata@example.com` not supported (but `info@piñata.es` works)
2. **No NDRs forwarded**: Sender won't receive delivery failure notifications
3. **DMARC forwarding issues**: Restrictive policies may fail forwarded emails
4. **No send/reply from custom address**: Replies use destination address, not custom address
5. **Dot character special handling**: `.` treated as literal (unlike Gmail)
6. **Message size limit**: 25 MiB maximum

## Best Practices

1. **Verify destinations first**: Rules auto-disabled until destination verified
2. **Use Email Workers for complex logic**: Don't create dozens of rules; use Workers
3. **Monitor spam scores**: Check `X-Cf-Spamh-Score` header for filtering
4. **Handle auth failures**: Enforce SPF/DKIM/DMARC at sender domain
5. **Use subaddressing strategically**: Track where emails come from (`user+service@`)
6. **Consider Worker limits**: Upgrade to Paid plan for heavy processing
7. **Store raw emails carefully**: R2 for archival, KV for metadata
8. **Implement proper error handling**: Always handle `setReject` cases
9. **Test locally with wrangler dev**: Use curl to simulate email delivery
10. **Use priority for rule ordering**: Lower priority = evaluated first

## Debugging

### Check DNS Configuration

```bash
# Verify MX records
dig MX example.com +short

# Verify SPF
dig TXT example.com +short | grep spf

# Verify DKIM
dig TXT cf2024-1._domainkey.example.com +short
```

### Test Email Delivery

```bash
# Test SMTP connection
telnet amir.mx.cloudflare.net 25

# Send test email
swaks --to test@example.com \
      --from sender@test.com \
      --server amir.mx.cloudflare.net
```

### Worker Logs

Use `wrangler tail` for real-time logs:

```bash
wrangler tail email-worker
```

Check for CPU limit errors (`EXCEEDED_CPU`) indicating need for Paid plan.

### Common Issues

**"Address not allowed"**: Sender not in allowlist (if using allowlist pattern)
**"DMARC policy violation"**: Sender domain has strict DMARC; can't forward
**"Found on RBL"**: Sender IP on blocklist; check MxToolbox
**Allocation errors**: Email too large/complex for Free tier; upgrade to Paid

## Resources

- [Email Routing Docs](https://developers.cloudflare.com/email-routing/)
- [Email Workers Runtime API](https://developers.cloudflare.com/email-routing/email-workers/runtime-api/)
- [Local Development Guide](https://developers.cloudflare.com/email-routing/email-workers/local-development/)
- [Postmaster Information](https://developers.cloudflare.com/email-routing/postmaster/)
- [Community Forum](https://community.cloudflare.com/)
- [Discord Server](https://discord.com/invite/cloudflaredev)

## TypeScript Types

```typescript
// Full type definitions for Email Workers

interface ForwardableEmailMessage<Body = unknown> {
  readonly from: string;
  readonly to: string;
  readonly headers: Headers;
  readonly raw: ReadableStream;
  readonly rawSize: number;

  setReject(reason: string): void;
  forward(rcptTo: string, headers?: Headers): Promise<void>;
  reply(message: EmailMessage): Promise<void>;
}

interface EmailMessage {
  readonly from: string;
  readonly to: string;
}

interface EmailWorker {
  email(
    message: ForwardableEmailMessage,
    env: Record<string, unknown>,
    ctx: ExecutionContext
  ): Promise<void>;
}

// API Response Types

interface EmailRoutingSettings {
  id: string;
  enabled: boolean;
  name: string;
  tag: string;
  created: string;
  modified: string;
  status: 'ready' | 'disabled';
}

interface EmailRoutingRule {
  id: string;
  tag: string;
  name: string;
  enabled: boolean;
  priority: number;
  matchers: Array<{
    type: 'literal' | 'all';
    field?: 'to' | 'from';
    value?: string;
  }>;
  actions: Array<{
    type: 'forward' | 'worker' | 'drop';
    value: string[];
  }>;
}

interface DestinationAddress {
  id: string;
  tag: string;
  email: string;
  verified: string | null;
  created: string;
  modified: string;
}

interface DNSRecord {
  name: string;
  type: 'MX' | 'TXT';
  content: string;
  priority?: number;
  locked?: boolean;
}
```

## When to Use This Skill

Use this skill when working on:
- Setting up custom email addresses for domains
- Creating Email Workers for email processing
- Implementing email routing rules via API
- Building email-based automation systems
- Configuring email forwarding and filtering
- Debugging Email Routing DNS/SMTP issues
- Architecting email-to-Worker integrations
- Building support/ticketing systems with email
- Implementing email archival/compliance solutions
