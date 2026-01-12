# Cloudflare Developer Platform (2026)

## Compute

- **Workers** — Serverless JS/TS/Python/Rust/WASM execution at the edge with sub-ms cold starts
- **Durable Objects** — Stateful serverless primitives with strongly-consistent transactional storage
- **Workflows** — Durable multi-step execution with automatic retries, sleep/scheduling, state persistence
- **Containers** — OCI containers as part of Workers apps, managed via Durable Objects bindings
- **Sandbox SDK** — Isolated execution environments for untrusted code (AI agents, user scripts)
- **Pages** — Full-stack deployment with Git integration, preview deployments, SSR
- **Pages Functions** — Server-side Workers code for Pages projects
- **Workers for Platforms** — Multi-tenant platform for running customer code in isolated Workers
- **Cron Triggers** — Scheduled Worker execution without HTTP requests
- **Snippets** — Lightweight JavaScript code executed at the edge for request customization (header modification, JWT validation, redirects). Limited to 5ms execution, 2MB memory, 32KB package size. Configured via filter expressions in Rules.

## Storage & Databases

- **KV** — Global eventually-consistent key-value store with edge caching
- **R2** — S3-compatible object storage, zero egress fees
- **D1** — Serverless SQLite with global read replication and Time Travel backups
- **Durable Objects Storage** — Co-located transactional SQLite/KV per Durable Object
- **Hyperdrive** — Connection pooling and query caching for external Postgres/MySQL
- **Queues** — Message queues with guaranteed delivery, batching, DLQs, pull consumers
- **R2 Data Catalog** — Managed Apache Iceberg catalog, turns R2 into data lakehouse
- **Analytics Engine** — Time-series database for unlimited-cardinality metrics
- **Cache Reserve** — Persistent cache for extended TTLs
- **Secrets Store** — Centralized encrypted secrets, reusable across Workers/AI Gateway

## Data Pipelines

- **Pipelines** — Streaming ingestion, SQL transforms, delivery to R2 as Iceberg/Parquet/JSON
- **R2 SQL** — Serverless distributed query engine for Iceberg tables in R2

## AI/ML

- **Workers AI** — 50+ open-source models (LLMs, image gen, embeddings, STT) on serverless GPUs
- **AI Gateway** — Proxy with caching, rate limiting, retries, fallback, analytics for AI APIs
- **Vectorize** — Vector database for embeddings, semantic search, RAG
- **AI Search** — Managed RAG service with auto-indexing of websites/R2
- **Agents SDK** — Framework for AI agents with state management, WebSockets, scheduling

## Media

- **Images** — Store, transform, optimize, deliver images at edge
- **Stream** — Live and on-demand video platform with adaptive bitrate
- **Browser Rendering** — Headless Chrome at edge for screenshots, PDFs, Puppeteer/Playwright

## Real-Time

- **RealtimeKit** — High-level SDKs for live video/voice with pre-built UI
- **Realtime SFU** — Low-level WebRTC selective forwarding unit
- **TURN Service** — Managed relay for WebRTC through restrictive firewalls

## Networking

- **Workers VPC** — Connect Workers to private networks (AWS/Azure/GCP/on-prem) via Tunnel
- **Smart Placement** — Auto-run Workers near backend services to minimize latency
- **Argo Smart Routing** — Fastest network path routing
- **Cloudflare Tunnel** — Secure outbound-only connections from private networks
- **Spectrum** — Proxy any TCP/UDP application through Cloudflare
- **Network Interconnect** — Direct physical/virtual connections to Cloudflare network

## Security

- **Turnstile** — Invisible CAPTCHA alternative, WCAG 2.1 AA compliant
- **WAF** — Web application firewall with managed/custom rules
- **Bot Management** — ML-based bot detection and mitigation
- **DDoS Protection** — Automatic always-on DDoS protection
- **API Shield** — API discovery, schema validation, protection

## Email

- **Email Routing** — Custom domain email addresses routed to existing mailboxes
- **Email Workers** — Process incoming emails with Workers code

## Third-Party & Analytics

- **Zaraz** — Server-side tag manager for third-party tools at edge
- **Web Analytics** — Privacy-first analytics without cookies

## Developer Tools

- **Wrangler** — Official CLI for Workers/Pages development and deployment
- **C3 (create-cloudflare)** — Project scaffolding CLI with templates
- **Miniflare** — Local Workers runtime simulator
- **Workerd** — Open-source Workers runtime for local/self-hosted execution
- **Workers Playground** — Browser-based IDE for testing

## Infrastructure as Code

- **Terraform Provider** — Manage Cloudflare via Terraform
- **Pulumi Provider** — Define Cloudflare infra in TS/Python/Go/C#
- **Cloudflare API** — RESTful API with OpenAPI spec and SDKs

## Platform Features

- **Bindings** — Typed APIs connecting Workers to storage/compute/services
- **Static Assets** — Host static files with Workers, CDN-served
- **Tail Workers** — Receive logs from other Workers for custom processing
- **Observability** — Built-in logging, real-time logs, metrics, tracing
