---
name: cloudflare
description: Comprehensive Cloudflare platform skill covering Workers, Pages, storage (KV, D1, R2), AI (Workers AI, Vectorize, Agents SDK), networking (Tunnel, Spectrum), security (WAF, DDoS), and infrastructure-as-code (Terraform, Pulumi). Use for any Cloudflare development task.
---

# Cloudflare Platform Skill

Consolidated skill for building on the Cloudflare platform. Use decision trees below to find the right product, then load detailed references.

## Quick Decision Trees

### "I need to run code"

```
Need to run code?
├─ Serverless functions at the edge → workers/
├─ Full-stack web app with Git deploys → pages/
├─ Stateful coordination/real-time → durable-objects/
├─ Long-running multi-step jobs → workflows/
├─ Run containers → containers/
├─ Multi-tenant (customers deploy code) → workers-for-platforms/
└─ Scheduled tasks (cron) → cron-triggers/
```

### "I need to store data"

```
Need storage?
├─ Key-value (config, sessions, cache) → kv/
├─ Relational SQL → d1/ (SQLite) or hyperdrive/ (existing Postgres/MySQL)
├─ Object/file storage (S3-compatible) → r2/
├─ Message queue (async processing) → queues/
├─ Vector embeddings (AI/semantic search) → vectorize/
├─ Strongly-consistent per-entity state → durable-objects/ (DO storage)
├─ Secrets management → secrets-store/
└─ Streaming ETL to R2 → pipelines/
```

### "I need AI/ML"

```
Need AI?
├─ Run inference (LLMs, embeddings, images) → workers-ai/
├─ Vector database for RAG/search → vectorize/
├─ Build stateful AI agents → agents-sdk/
├─ Gateway for any AI provider (caching, routing) → ai-gateway/
└─ AI-powered search widget → ai-search/
```

### "I need networking/connectivity"

```
Need networking?
├─ Expose local service to internet → tunnel/
├─ TCP/UDP proxy (non-HTTP) → spectrum/
├─ WebRTC TURN server → turn/
├─ Private network connectivity → network-interconnect/
├─ Optimize routing → argo-smart-routing/
└─ Real-time video/audio → realtimekit/ or realtime-sfu/
```

### "I need security"

```
Need security?
├─ Web Application Firewall → waf/
├─ DDoS protection → ddos/
├─ Bot detection/management → bot-management/
├─ API protection → api-shield/
├─ CAPTCHA alternative → turnstile/
└─ Credential leak detection → waf/ (managed ruleset)
```

### "I need media/content"

```
Need media?
├─ Image optimization/transformation → images/
├─ Video streaming/encoding → stream/
├─ Browser automation/screenshots → browser-rendering/
└─ Third-party script management → zaraz/
```

### "I need infrastructure-as-code"

```
Need IaC?
├─ Pulumi → pulumi/
├─ Terraform → terraform/
└─ Direct API → api/
```

## Product Index

### Compute & Runtime
| Product | Summary | Reference |
|---------|---------|-----------|
| Workers | Serverless functions on V8 isolates, global edge deployment | `references/workers/` |
| Pages | JAMstack platform with Git integration, preview deploys | `references/pages/` |
| Pages Functions | File-based routing for serverless functions in Pages | `references/pages-functions/` |
| Durable Objects | Stateful coordination with co-located storage | `references/durable-objects/` |
| Workflows | Durable multi-step apps with retries and state persistence | `references/workflows/` |
| Containers | Run containers alongside Durable Objects | `references/containers/` |
| Workers for Platforms | Multi-tenant: let customers deploy Workers | `references/workers-for-platforms/` |
| Cron Triggers | Scheduled Worker execution | `references/cron-triggers/` |
| Tail Workers | Observability workers for logging/tracing | `references/tail-workers/` |
| Snippets | Lightweight request/response modifications | `references/snippets/` |
| Smart Placement | Optimize Worker placement near data | `references/smart-placement/` |

### Storage & Data
| Product | Summary | Reference |
|---------|---------|-----------|
| KV | Global key-value store, eventually consistent, read-heavy | `references/kv/` |
| D1 | Serverless SQLite, horizontal scale-out, Time Travel | `references/d1/` |
| R2 | S3-compatible object storage, zero egress fees | `references/r2/` |
| Queues | Distributed message queue with guaranteed delivery | `references/queues/` |
| Hyperdrive | Accelerate existing Postgres/MySQL from Workers | `references/hyperdrive/` |
| DO Storage | Strongly-consistent storage within Durable Objects | `references/do-storage/` |
| Secrets Store | Centralized secrets management | `references/secrets-store/` |
| Pipelines | ETL streaming to R2 as Iceberg/Parquet | `references/pipelines/` |
| R2 Data Catalog | Apache Iceberg catalog for R2 | `references/r2-data-catalog/` |
| R2 SQL | Query R2 objects with SQL | `references/r2-sql/` |

### AI & Machine Learning
| Product | Summary | Reference |
|---------|---------|-----------|
| Workers AI | Serverless AI inference (LLMs, embeddings, images) | `references/workers-ai/` |
| Vectorize | Vector database for embeddings and semantic search | `references/vectorize/` |
| Agents SDK | Framework for building stateful AI agents | `references/agents-sdk/` |
| AI Gateway | Universal gateway with caching, rate limiting, routing | `references/ai-gateway/` |
| AI Search | Embeddable AI-powered search widget | `references/ai-search/` |

### Networking & Connectivity
| Product | Summary | Reference |
|---------|---------|-----------|
| Tunnel | Secure outbound-only connections to Cloudflare | `references/tunnel/` |
| Spectrum | TCP/UDP proxy for non-HTTP protocols | `references/spectrum/` |
| TURN | WebRTC TURN server for NAT traversal | `references/turn/` |
| Network Interconnect | Private network connectivity to Cloudflare | `references/network-interconnect/` |
| Argo Smart Routing | Optimized routing across Cloudflare network | `references/argo-smart-routing/` |
| Workers VPC | Private networking for Workers | `references/workers-vpc/` |

### Security
| Product | Summary | Reference |
|---------|---------|-----------|
| WAF | Web Application Firewall with managed/custom rules | `references/waf/` |
| DDoS Protection | Automatic DDoS mitigation | `references/ddos/` |
| Bot Management | Bot detection and mitigation | `references/bot-management/` |
| API Shield | API discovery, schema validation, abuse detection | `references/api-shield/` |
| Turnstile | Privacy-preserving CAPTCHA alternative | `references/turnstile/` |

### Media & Content
| Product | Summary | Reference |
|---------|---------|-----------|
| Images | Image optimization, transformation, delivery | `references/images/` |
| Stream | Video streaming, encoding, live streaming | `references/stream/` |
| Browser Rendering | Headless browser for screenshots, PDFs, scraping | `references/browser-rendering/` |
| Zaraz | Third-party script management | `references/zaraz/` |

### Real-Time Communication
| Product | Summary | Reference |
|---------|---------|-----------|
| RealtimeKit | Real-time audio/video SDK | `references/realtimekit/` |
| Realtime SFU | Selective Forwarding Unit for WebRTC | `references/realtime-sfu/` |

### Developer Tools
| Product | Summary | Reference |
|---------|---------|-----------|
| Wrangler | CLI for Workers development and deployment | `references/wrangler/` |
| Miniflare | Local Workers simulator for testing | `references/miniflare/` |
| C3 | Create Cloudflare CLI for project scaffolding | `references/c3/` |
| Observability | Logs, metrics, analytics for Workers | `references/observability/` |
| Analytics Engine | Write and query custom analytics data | `references/analytics-engine/` |
| Web Analytics | Privacy-first web analytics | `references/web-analytics/` |
| Sandbox | Isolated execution environment | `references/sandbox/` |
| Workerd | Open-source Workers runtime | `references/workerd/` |
| Workers Playground | Browser-based Workers development | `references/workers-playground/` |

### Infrastructure as Code
| Product | Summary | Reference |
|---------|---------|-----------|
| Pulumi | Pulumi provider for Cloudflare | `references/pulumi/` |
| Terraform | Terraform provider for Cloudflare | `references/terraform/` |
| API | Direct Cloudflare API access | `references/api/` |

### Other Services
| Product | Summary | Reference |
|---------|---------|-----------|
| Email Routing | Email forwarding and routing | `references/email-routing/` |
| Email Workers | Process emails with Workers | `references/email-workers/` |
| Static Assets | Serve static files from Workers | `references/static-assets/` |
| Bindings | Cross-service bindings reference | `references/bindings/` |
| Cache Reserve | Persistent edge cache | `references/cache-reserve/` |

## Common Patterns

### Full-Stack Web App
```
Pages (frontend) + Pages Functions (API) + D1 (database)
```

### Real-Time Collaboration
```
Workers (entry) + Durable Objects (state) + WebSockets
```

### RAG / AI Search
```
Workers AI (embeddings + LLM) + Vectorize (vector store) + D1/KV (metadata)
```

### Background Processing
```
Workers (producer) + Queues (buffer) + Workers (consumer) + R2 (results)
```

### Multi-Region Database Access
```
Hyperdrive (pooling/caching) → Existing Postgres/MySQL
```

## Loading References

Each reference contains detailed implementation guidance. Load as needed:

```
Read references/workers/SKILL.md for Workers development
Read references/d1/SKILL.md for D1 database patterns
Read references/agents-sdk/SKILL.md for AI agents
```

References preserve full content from original skills with ~50k total lines of documentation.
