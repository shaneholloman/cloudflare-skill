# Cloudflare AI Search Skill

Expert guidance for implementing Cloudflare AI Search (formerly AutoRAG), Cloudflare's managed semantic search and RAG service.

## Core Concepts

**AI Search** is Cloudflare's fully managed service for building RAG (Retrieval-Augmented Generation) applications. It automatically indexes data sources and enables natural language queries with AI-powered responses.

**Two main operations:**
- **Indexing**: Async background process converting data → vectors
- **Querying**: Sync process retrieving content + generating responses

**Service foundation**: Built on R2, Vectorize, Workers AI, AI Gateway, Browser Rendering

## Architecture Overview

### Indexing Pipeline
```
Data Source (R2/Website)
  ↓
Markdown Conversion (Workers AI)
  ↓
Chunking (configurable strategy)
  ↓
Embedding (Workers AI)
  ↓
Vector Storage (Vectorize)
```

### Query Pipeline
```
Query Input
  ↓
Query Rewriting (optional, Workers AI)
  ↓
Embedding (same model as indexing)
  ↓
Vector Search (Vectorize)
  ↓
Content Retrieval (R2)
  ↓
Response Generation (Workers AI LLM)
```

## Prerequisites

- **Active R2 subscription required** before creating first AI Search instance
- Account must have access to Workers AI

## Common Use Cases

1. **Enterprise search**: Natural language search over company docs
2. **Customer support**: AI-powered chat with product documentation
3. **Knowledge bases**: Semantic search over technical content
4. **Multitenancy SaaS**: Per-tenant data isolation with folder filters
5. **Content discovery**: Finding relevant content across large datasets

## Workers Binding (Recommended)

### Configuration

**wrangler.toml:**
```toml
[ai]
binding = "AI"
```

**wrangler.jsonc:**
```jsonc
{
  "ai": {
    "binding": "AI"
  }
}
```

### Code Patterns

#### AI Search with Generation
```typescript
// Generate AI response with retrieved context
const answer = await env.AI.autorag("my-autorag").aiSearch({
  query: "How do I configure rate limits?",
  model: "@cf/meta/llama-3.3-70b-instruct-fp8-fast",
  rewrite_query: true,
  max_num_results: 10,
  ranking_options: {
    score_threshold: 0.3
  },
  reranking: {
    enabled: true,
    model: "@cf/baai/bge-reranker-base"
  },
  stream: true
});

// Response includes: search_query, response, data[], has_more, next_page
```

#### Search Only (No Generation)
```typescript
// Retrieve relevant chunks without generation
const results = await env.AI.autorag("my-autorag").search({
  query: "rate limiting configuration",
  rewrite_query: true,
  max_num_results: 5,
  ranking_options: {
    score_threshold: 0.4
  },
  reranking: {
    enabled: true,
    model: "@cf/baai/bge-reranker-base"
  }
});

// Response includes: search_query, data[], has_more, next_page
```

#### With Metadata Filtering
```typescript
const answer = await env.AI.autorag("my-autorag").aiSearch({
  query: "recent policy updates",
  filters: {
    type: "and",
    filters: [
      {
        type: "eq",
        key: "folder",
        value: "policies/security/"
      },
      {
        type: "gte",
        key: "timestamp",
        value: "1735689600000" // Unix timestamp (ms)
      }
    ]
  }
});
```

#### Streaming Responses
```typescript
const stream = await env.AI.autorag("my-autorag").aiSearch({
  query: "explain the authentication flow",
  stream: true
});

// Process stream chunks
for await (const chunk of stream) {
  // Handle chunk data
}
```

## REST API

**Base URL:** `https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/autorag/rags/{AUTORAG_NAME}`

**Note:** API endpoints still use `autorag` naming; functionality identical to AI Search

### Authentication

Create API token with permissions:
- `AI Search - Read`
- `AI Search Edit`

```bash
curl -H "Authorization: Bearer {API_TOKEN}" \
  -H "Content-Type: application/json"
```

### AI Search Endpoint

```bash
curl https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/autorag/rags/{AUTORAG_NAME}/ai-search \
-H "Authorization: Bearer {API_TOKEN}" \
-H "Content-Type: application/json" \
-d '{
  "query": "How do I configure caching?",
  "model": "@cf/meta/llama-3.3-70b-instruct-fp8-fast",
  "system_prompt": "You are a technical documentation assistant.",
  "rewrite_query": true,
  "max_num_results": 10,
  "ranking_options": {
    "score_threshold": 0.3
  },
  "reranking": {
    "enabled": true,
    "model": "@cf/baai/bge-reranker-base"
  },
  "stream": false,
  "filters": {...}
}'
```

### Search Endpoint

```bash
curl https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/autorag/rags/{AUTORAG_NAME}/search \
-H "Authorization: Bearer {API_TOKEN}" \
-H "Content-Type: application/json" \
-d '{
  "query": "caching configuration",
  "rewrite_query": true,
  "max_num_results": 10,
  "ranking_options": {
    "score_threshold": 0.3
  },
  "reranking": {
    "enabled": true,
    "model": "@cf/baai/bge-reranker-base"
  },
  "filters": {...}
}'
```

## Query Parameters Reference

### Common Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `query` | string | Yes | - | Input query text |
| `rewrite_query` | boolean | No | false | Use LLM to optimize query for retrieval |
| `max_num_results` | number | No | 10 | Max results (1-50) |
| `ranking_options` | object | No | {} | Result ranking configuration |
| `reranking` | object | No | {} | Reranking configuration |
| `filters` | object | No | - | Metadata filtering |

### AI Search Only

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model` | string | No | default | Text generation model |
| `system_prompt` | string | No | default | System prompt for generation |
| `stream` | boolean | No | false | Stream response chunks |

### Ranking Options

```typescript
ranking_options: {
  score_threshold: 0.3  // Min score 0-1, default 0
}
```

### Reranking

```typescript
reranking: {
  enabled: true,
  model: "@cf/baai/bge-reranker-base"
}
```

## Metadata Filtering

### Available Attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| `filename` | File name | `"doc.pdf"` or `"docs/guide.pdf"` |
| `folder` | Folder/prefix | `"docs/api/"` |
| `timestamp` | Last modified (Unix ms, rounded to seconds) | `1735689600000` |

### Comparison Operators

```typescript
{
  type: "eq" | "ne" | "gt" | "gte" | "lt" | "lte",
  key: "folder" | "filename" | "timestamp",
  value: string
}
```

### Compound Filters

**AND filter:**
```typescript
filters: {
  type: "and",
  filters: [
    { type: "eq", key: "folder", value: "api/" },
    { type: "gte", key: "timestamp", value: "1735689600000" }
  ]
}
```

**OR filter (limitations):**
- Only `eq` operator allowed
- All conditions must use same key
```typescript
filters: {
  type: "or",
  filters: [
    { type: "eq", key: "folder", value: "api/" },
    { type: "eq", key: "folder", value: "guides/" }
  ]
}
```

**No nested compound operators**

### "Starts With" Pattern for Folders

Match folder + all subfolders:
```typescript
// Match "customer-a/" and "customer-a/contracts/" etc.
filters: {
  type: "and",
  filters: [
    { type: "gt", key: "folder", value: "customer-a//" },
    { type: "lte", key: "folder", value: "customer-a/z" }
  ]
}
```

Logic: Uses ASCII comparison where:
- `gt "customer-a//"` includes paths after "/" character
- `lte "customer-a/z"` includes paths up to lowercase "z"

## Multitenancy Pattern

Isolate tenant data using folder-based filtering:

### 1. Organize by Tenant
```
R2 Structure:
customer-a/
  profile.md
  contracts/
    contract-1.pdf
customer-b/
  profile.md
  invoices/
    invoice-1.pdf
```

**AI Search automatically stores folder as metadata**

### 2. Query with Tenant Filter

**Single folder:**
```typescript
const response = await env.AI.autorag("my-autorag").search({
  query: "When did I sign my contract?",
  filters: {
    type: "eq",
    key: "folder",
    value: "customer-a/contracts/"
  }
});
```

**All tenant folders (recursive):**
```typescript
filters: {
  type: "and",
  filters: [
    { type: "gt", key: "folder", value: "customer-a//" },
    { type: "lte", key: "folder", value: "customer-a/z" }
  ]
}
```

### 3. Dynamic Tenant Filtering
```typescript
export default {
  async fetch(request, env) {
    const tenantId = request.headers.get("X-Tenant-ID");
    
    const response = await env.AI.autorag("my-autorag").aiSearch({
      query: await request.text(),
      filters: {
        type: "and",
        filters: [
          { type: "gt", key: "folder", value: `${tenantId}//` },
          { type: "lte", key: "folder", value: `${tenantId}/z` }
        ]
      }
    });
    
    return Response.json(response);
  }
};
```

## Context Metadata

Add custom context to files (passed to LLM, doesn't affect retrieval):

```typescript
// Upload to R2 with context metadata
await env.MY_BUCKET.put("support/guide.pdf", file, {
  customMetadata: {
    context: "This is a customer support guide for enterprise users."
  }
});
```

**In query response:**
```typescript
{
  data: [{
    file_id: "doc001",
    filename: "support/guide.pdf",
    score: 0.8,
    attributes: {
      folder: "support/",
      timestamp: 1735689600000,
      file: {
        context: "This is a customer support guide for enterprise users."
      }
    },
    content: [...]
  }]
}
```

## Response Structure

### AI Search Response
```typescript
{
  object: "vector_store.search_results.page",
  search_query: string,  // May differ if rewrite_query=true
  response: string,      // Generated AI response
  data: [
    {
      file_id: string,
      filename: string,
      score: number,
      attributes: {
        timestamp: number,
        folder: string,
        file?: {
          url?: string,
          context?: string
        }
      },
      content: [{
        id: string,
        type: "text" | "image",
        text: string
      }]
    }
  ],
  has_more: boolean,
  next_page: string | null
}
```

### Search Response
Same structure minus `response` field

## Similarity Caching

Reuse responses for similar queries via MinHash + LSH

### How It Works
1. Check if similar query exists (based on threshold)
2. Return cached response if found (`HIT`)
3. Generate + cache if not found (`MISS`)

**Check cache status:** `cf-aig-cache-status` header

### Thresholds

| Threshold | Match Strictness | Example |
|-----------|------------------|---------|
| `exact` | Near-identical | "What's the weather today?" ↔ "What is the weather today?" |
| `strong` | High similarity (default) | "What's the weather today?" ↔ "How's the weather today?" |
| `broad` | Moderate | "What's the weather today?" ↔ "Tell me today's weather" |
| `loose` | Low (max reuse) | "What's the weather today?" ↔ "Give me the forecast" |

### Cache Behavior
- **30-day TTL** (automatic expiration)
- **Volatile**: Concurrent similar requests may both miss cache
- **Data-dependent**: Cleared if source chunks change/delete

## Indexing

### Automatic Reindexing
- **Frequency**: Every 6 hours
- **Monitors**: New, modified, deleted files
- **Manual sync**: Initiate via dashboard (30s cooldown)

### Controls
- **Sync Index**: Force immediate check + reindex
- **Pause Indexing**: Stop scheduled reindexing (for debugging)

### Performance Factors
- File count and sizes
- File formats (images slower than text)
- Workers AI model latency

### Best Practices
- Keep files under size limits
- Use supported formats
- Maintain valid Service API token
- Clean up outdated content regularly
- Monitor Vectorize index limits

## Configuration via Dashboard

### 1. Create Instance
```
Dashboard → AI Search → Create
→ Choose data source (R2 bucket or Website)
→ Configure settings
→ Create
```

### 2. Monitor Indexing
```
AI Search → Select instance → Overview
View: indexing status, progress, stats
```

### 3. Test Queries
```
AI Search → Select instance → Playground
→ "Search with AI" or "Search"
→ Enter query
```

### 4. Get API Token
```
AI Search → Select instance → Use AI Search → API
→ Create API Token
→ Permissions: "AI Search - Read", "AI Search Edit"
```

### 5. Connect to Application
```
AI Search → Select instance → Connect
Choose: Workers Binding or REST API
```

## Management API

**Base:** `https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/ai-search`

### Instances

**List instances:**
```bash
GET /accounts/{account_id}/ai-search/instances
```

**Create instance:**
```bash
POST /accounts/{account_id}/ai-search/instances
```

**Read instance:**
```bash
GET /accounts/{account_id}/ai-search/instances/{id}
```

**Update instance:**
```bash
PUT /accounts/{account_id}/ai-search/instances/{id}
```

**Delete instance:**
```bash
DELETE /accounts/{account_id}/ai-search/instances/{id}
```

**Get stats:**
```bash
GET /accounts/{account_id}/ai-search/instances/{id}/stats
# Returns: completed, error, file_embed_errors counts
```

### Items

**List items:**
```bash
GET /accounts/{account_id}/ai-search/instances/{id}/items
```

**Get item:**
```bash
GET /accounts/{account_id}/ai-search/instances/{id}/items/{item_id}
```

### Jobs

**List jobs:**
```bash
GET /accounts/{account_id}/ai-search/instances/{id}/jobs
```

**Create job:**
```bash
POST /accounts/{account_id}/ai-search/instances/{id}/jobs
```

**Get job:**
```bash
GET /accounts/{account_id}/ai-search/instances/{id}/jobs/{job_id}
```

**Get job logs:**
```bash
GET /accounts/{account_id}/ai-search/instances/{id}/jobs/{job_id}/logs
```

### Tokens

**List tokens:**
```bash
GET /accounts/{account_id}/ai-search/tokens
```

**Create token:**
```bash
POST /accounts/{account_id}/ai-search/tokens
```

**Delete token:**
```bash
DELETE /accounts/{account_id}/ai-search/tokens/{id}
```

## Limits (Open Beta)

| Limit | Value |
|-------|-------|
| Max instances per account | 10 |
| Max files per instance | 100,000 |
| Max file size | 4 MB |

**Request increase:** [Limit Increase Form](https://forms.gle/wnizxrEUW33Y15CT8)

## Pricing (Open Beta)

**AI Search**: Free during open beta

**Underlying services billed separately:**
- R2: Storage + operations
- Vectorize: Vector storage + queries
- Workers AI: Embeddings + generation + conversions
- AI Gateway: Monitoring/caching
- Browser Rendering: Dynamic website crawling (if enabled)

See respective service pricing pages for details.

## Local Development

**Proxying:** Local dev proxies requests to deployed AI Search instance
- No local emulation
- Queries forwarded to remote instance
- Responses returned as if local

## Integration Patterns

### Basic Worker Example
```typescript
export default {
  async fetch(request, env) {
    const { query } = await request.json();
    
    const answer = await env.AI.autorag("my-autorag").aiSearch({
      query,
      rewrite_query: true,
      max_num_results: 5
    });
    
    return Response.json(answer);
  }
};
```

### With Error Handling
```typescript
export default {
  async fetch(request, env) {
    try {
      const { query } = await request.json();
      
      if (!query || query.trim() === "") {
        return Response.json(
          { error: "Query required" },
          { status: 400 }
        );
      }
      
      const answer = await env.AI.autorag("my-autorag").aiSearch({
        query,
        rewrite_query: true,
        max_num_results: 10,
        ranking_options: {
          score_threshold: 0.3
        }
      });
      
      return Response.json(answer);
    } catch (error) {
      return Response.json(
        { error: error.message },
        { status: 500 }
      );
    }
  }
};
```

### Streaming Response
```typescript
export default {
  async fetch(request, env) {
    const { query } = await request.json();
    
    const stream = await env.AI.autorag("my-autorag").aiSearch({
      query,
      stream: true
    });
    
    return new Response(stream, {
      headers: { "Content-Type": "text/event-stream" }
    });
  }
};
```

### Search-Only with Custom Processing
```typescript
export default {
  async fetch(request, env) {
    const { query } = await request.json();
    
    // Get search results only
    const results = await env.AI.autorag("my-autorag").search({
      query,
      rewrite_query: true,
      max_num_results: 5
    });
    
    // Custom processing
    const enriched = results.data.map(result => ({
      ...result,
      relevance: result.score > 0.5 ? "high" : "medium",
      preview: result.content[0].text.slice(0, 100)
    }));
    
    return Response.json({ results: enriched });
  }
};
```

### Multi-tenant SaaS Pattern
```typescript
export default {
  async fetch(request, env) {
    const tenantId = request.headers.get("X-Tenant-ID");
    const authToken = request.headers.get("Authorization");
    
    // Verify tenant auth
    if (!await verifyTenantAuth(tenantId, authToken)) {
      return Response.json(
        { error: "Unauthorized" },
        { status: 401 }
      );
    }
    
    const { query } = await request.json();
    
    const answer = await env.AI.autorag("my-autorag").aiSearch({
      query,
      filters: {
        type: "and",
        filters: [
          { type: "gt", key: "folder", value: `${tenantId}//` },
          { type: "lte", key: "folder", value: `${tenantId}/z` }
        ]
      },
      rewrite_query: true
    });
    
    return Response.json(answer);
  }
};
```

## Common Patterns & Best Practices

### Query Rewriting
**When to enable:**
- User queries are conversational
- Queries need optimization for retrieval
- Dealing with ambiguous phrasing

**When to skip:**
- Queries already well-structured
- Minimal latency required
- Keyword-based searches

### Reranking
**Enable when:**
- Initial retrieval casts wide net
- Semantic relevance critical
- Acceptable to add latency for accuracy

**Skip when:**
- Results already highly relevant
- Speed prioritized over precision
- Small result sets (< 5 items)

### Score Thresholds
```typescript
// Strict relevance
score_threshold: 0.7  // Only very relevant results

// Balanced (recommended starting point)
score_threshold: 0.3

// Broad recall
score_threshold: 0.1  // Include more results
```

### Result Count Tuning
```typescript
// Chat/QA use cases
max_num_results: 5  // Focused context

// Search/discovery
max_num_results: 20  // More options

// With reranking
max_num_results: 50  // Wide net, rerank to top results
```

### Metadata Organization
**Folder structure:**
```
{category}/{subcategory}/{file}
api/authentication/oauth.md
api/authorization/rbac.md
guides/quickstart/setup.md
```

**Benefits:**
- Easy multitenancy
- Flexible filtering
- Clear organization

## Troubleshooting

### Indexing Issues

**Files not indexing:**
- Check file size < 4MB
- Verify format supported
- Confirm R2 bucket permissions
- Check job logs in dashboard

**Slow indexing:**
- Reduce concurrent file changes
- Optimize file sizes
- Check Workers AI quota

### Query Issues

**Low relevance results:**
- Enable `rewrite_query`
- Lower `score_threshold`
- Enable reranking
- Check query phrasing

**Empty results:**
- Verify indexing complete
- Check metadata filters
- Lower score threshold
- Confirm data in scope

**Slow queries:**
- Reduce `max_num_results`
- Disable reranking
- Disable query rewriting
- Check similarity cache hit rate

### Cache Issues

**Low cache hit rate:**
- Adjust similarity threshold
- Check query variations
- Consider query normalization
- Review 30-day TTL impact

## Models Reference

### Embedding Models
Default: Workers AI embedding model (check dashboard for current)

### Generation Models
Common options:
- `@cf/meta/llama-3.3-70b-instruct-fp8-fast`
- `@cf/meta/llama-3-8b-instruct`
- Other Workers AI text generation models

### Reranking Models
- `@cf/baai/bge-reranker-base`

Check Workers AI docs for latest models

## Resources

- [AI Search Docs](https://developers.cloudflare.com/ai-search/)
- [Workers AI](https://developers.cloudflare.com/workers-ai/)
- [Vectorize](https://developers.cloudflare.com/vectorize/)
- [R2](https://developers.cloudflare.com/r2/)
- [AI Gateway](https://developers.cloudflare.com/ai-gateway/)
- [API Reference](https://developers.cloudflare.com/api/resources/ai_search/)
- [Discord Community](https://discord.cloudflare.com)
- [@CloudflareDev](https://x.com/cloudflaredev)

## Quick Reference

### aiSearch() - Generate AI response with context
```typescript
env.AI.autorag(name).aiSearch({
  query: string,
  model?: string,
  system_prompt?: string,
  rewrite_query?: boolean,
  max_num_results?: number,
  ranking_options?: { score_threshold?: number },
  reranking?: { enabled?: boolean, model?: string },
  stream?: boolean,
  filters?: object
})
```

### search() - Retrieve relevant chunks only
```typescript
env.AI.autorag(name).search({
  query: string,
  rewrite_query?: boolean,
  max_num_results?: number,
  ranking_options?: { score_threshold?: number },
  reranking?: { enabled?: boolean, model?: string },
  filters?: object
})
```

### REST Endpoints
- **AI Search:** `POST /accounts/{id}/autorag/rags/{name}/ai-search`
- **Search:** `POST /accounts/{id}/autorag/rags/{name}/search`

### Key Defaults
- `rewrite_query`: `false`
- `max_num_results`: `10` (range: 1-50)
- `score_threshold`: `0` (range: 0-1)
- `stream`: `false`
- Reindexing: Every 6 hours

### Metadata Keys
- `folder`: Folder/prefix path
- `filename`: File name
- `timestamp`: Unix timestamp (ms, rounded to seconds)

### Filter Operators
- Comparison: `eq`, `ne`, `gt`, `gte`, `lt`, `lte`
- Compound: `and`, `or` (no nesting)
