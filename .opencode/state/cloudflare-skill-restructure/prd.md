# Cloudflare Skill Restructuring Plan

Plan to bring skill into compliance with skill-creator best practices while preserving content quality.

## Review Summary

**Content quality: 4.7/5** — Excellent, needs no rewriting
**Structural compliance: Poor** — Violates line limits, missing frontmatter fields

### Issues by Severity

| Severity | Issue | Impact |
|----------|-------|--------|
| Critical | Reference files 700-1400+ lines (limit: 200) | Defeats progressive disclosure |
| High | Main SKILL.md 232 lines (limit: 200) | Loads unnecessary content |
| High | No `references:` in frontmatter | Poor discoverability |
| High | Activation description too broad | False activations |
| High | No cross-reference guidance | Agents miss related files |
| Medium | Reference files lack frontmatter | No metadata for discovery |
| Medium | Outdated compatibility dates | Agents use old dates |
| Medium | Type safety violation (d1 `any[]`) | Violates standards |
| Medium | Inconsistent error handling | Agents can't debug |
| Low | Inconsistent config format | Minor confusion |
| Low | Decision trees incomplete | Miss some products |

---

## Phase 1: Split Reference Files (Critical)

**Goal:** Split 60 files from 700-1400 lines to <200 lines each

### Standard Structure

```
references/<product>/
├── README.md           # Overview, when to use (<100 lines)
├── configuration.md    # wrangler.jsonc setup (<150 lines)
├── api.md              # Runtime API, types (<150 lines)
├── patterns.md         # Common patterns (<150 lines)
├── gotchas.md          # Pitfalls, limits (<100 lines)
└── advanced.md         # Optional (<150 lines)
```

### Splitting Priority

**Tier 1 (largest, split first):**
- wrangler/ (1470 lines)
- cache-reserve/ (1439 lines)
- stream/ (1340 lines)
- workers/ (1334 lines)
- realtimekit/ (1244 lines)

**Tier 2 (1100-1250 lines):**
terraform, pulumi, sandbox, hyperdrive, pages, ddos, workers-for-platforms, workflows, miniflare, workerd, cron-triggers, secrets-store, pages-functions, network-interconnect

**Tier 3 (700-1100 lines):**
Remaining 40+ files

### Process

For each SKILL.md:
1. Extract overview → `README.md`
2. Extract config sections → `configuration.md`
3. Extract API/types → `api.md`
4. Extract patterns/examples → `patterns.md`
5. Extract gotchas/limits → `gotchas.md`
6. Delete original SKILL.md
7. Add "Related References" section to README.md

**Approach:** Manual with template
**Commit:** `refactor: split reference files into <200 line chunks`

---

## Phase 2: Trim Main SKILL.md (High)

**Goal:** Reduce from 232 to <200 lines

### Changes

| Section | Current | Action | Target |
|---------|---------|--------|--------|
| Frontmatter | 5 | Improve description, add `references:` | 10 |
| Intro | 3 | Keep | 3 |
| Decision trees | 82 | Keep | 82 |
| Product index | 101 | Remove Summary column | 70 |
| Common patterns | 26 | Move to `patterns.md` | 0 |
| Loading references | 12 | Remove | 0 |
| **Total** | **232** | | **~165** |

### New Frontmatter

```yaml
---
name: cloudflare
description: Build and deploy on Cloudflare. Use when creating Workers/Pages apps, configuring KV/D1/R2 storage, implementing AI with Workers AI/Agents SDK, setting up Tunnels, or managing infrastructure via Terraform/Pulumi. NOT for DNS-only config or general web hosting.
references:
  - references/workers/README.md
  - references/d1/README.md
  - references/pages/README.md
  - references/agents-sdk/README.md
  - references/wrangler/README.md
---
```

**Commit:** `refactor: trim main SKILL.md to <200 lines`

---

## Phase 3: Add Cross-References (High)

**Goal:** Guide agents to related files

### Template for README.md

```markdown
## In This Reference
- [Configuration](configuration.md) — wrangler.jsonc setup
- [API Reference](api.md) — runtime APIs, types
- [Patterns](patterns.md) — common examples
- [Gotchas](gotchas.md) — pitfalls, limits

## See Also
- [Related Product](../related-product/) — when to use instead
```

### Cross-Reference Map

| Product | Related Products |
|---------|-----------------|
| workers | kv, d1, r2, durable-objects, queues |
| pages | pages-functions, d1, kv |
| agents-sdk | durable-objects, d1, workers-ai, vectorize |
| d1 | workers, hyperdrive |
| durable-objects | workers, do-storage |

**Commit:** `docs: add cross-references between split files`

---

## Phase 4: Fix Content Issues (Medium)

### 4.1 Type Safety

**File:** d1 api.md or patterns.md (after split)

```typescript
// Before
const params: any[] = [];

// After
const params: (string | number | boolean | null)[] = [];
```

### 4.2 Compatibility Dates

Add note to all config examples:
```jsonc
{
  "compatibility_date": "2025-01-01", // Use current date for new projects
}
```

### 4.3 Config Format

Convert all wrangler.toml to wrangler.jsonc

### 4.4 Missing Imports

Add imports to code examples, particularly:
- agents-sdk: `import { tool } from "ai"`

**Commit:** `fix: type safety, config format, imports`

---

## Phase 5: Low Priority Fixes

### 5.1 Complete Decision Trees

Add to "I need to run code":
```
├─ Lightweight request mods → snippets/
├─ Observability/logging → tail-workers/
```

Add to "I need networking":
```
├─ Optimize Worker placement → smart-placement/
```

Add to "I need to store data":
```
├─ Extended edge cache → cache-reserve/
```

### 5.2 Error Handling Template

Standardize `gotchas.md` structure:
```markdown
## Common Errors

### `<error message>`
**Cause:** ...
**Solution:** ...

## Limits
| Limit | Value |
|-------|-------|
```

**Commit:** `docs: complete decision trees, standardize error handling`

---

## Verification Checklist

After all phases:
- [ ] Main SKILL.md <200 lines
- [ ] All reference files <200 lines
- [ ] Frontmatter has `references:` field
- [ ] Each README.md has cross-references
- [ ] No `any` types in TypeScript
- [ ] All config uses wrangler.jsonc
- [ ] Decision trees complete
- [ ] Cross-reference paths correct

---

## Estimates

| Phase | Effort | Files Changed |
|-------|--------|---------------|
| 1 | 6-8 hours | ~300 new, 60 deleted |
| 2 | 30 min | 2 files |
| 3 | 2 hours | ~60 files |
| 4 | 1-2 hours | ~30 files |
| 5 | 1 hour | ~10 files |
| **Total** | **~12 hours** | |

---

## Open Questions

1. Should smaller reference files (<400 lines) be split, or only trimmed?
2. Should `docs/products.md` be integrated into SKILL.md or kept separate?
3. Priority for Phase 1: all files, or just Tier 1 first?
