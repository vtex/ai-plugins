This reference provides guidance for AI agents supporting **VTEX Master Data solution architecture**. Apply these constraints and patterns when deciding whether Master Data v2 is the right store, designing entity boundaries and schema posture (`v-indexed`, `v-cache`, `v-security`, `v-triggers`), planning capacity and lifecycle, or auditing existing usage. Use with `SKILL.md` and `architecture-well-architected.md`; this file does not replace IO implementation guides or API how-tos.

# Master Data — solution architecture

## When this reference applies

Use **before approving a new Master Data entity** or when reviewing storage fit in an architecture or readiness review:

- Is Master Data the **right** store vs Catalog, OMS, CL/AD, VBase, or an external database?
- How should the **entity and schema** be shaped for performance, security, and operability?
- Which fields belong in **`v-indexed`**, **`v-cache`**, and **`v-security`**?
- Are **`v-triggers`** appropriate, or should orchestration live in VTEX IO events / external middleware?
- What are **capacity, rate, and lifecycle** risks (scroll limits, write throughput, schema count)?

Do **not** use this file for MasterDataClient code, `masterdata` builder manifests, LRU/VBase caching in IO services, or step-by-step CRUD implementation — route those to VTEX IO product documentation and engineering teams.

## Storage fit decision

### When Master Data is appropriate

Master Data is a good fit when **all** of the following hold:

1. **Document-oriented access** — JSON documents queried by indexed fields; full or partial document retrieval.
2. **Platform-integrated value** — You need VTEX-native `v-triggers`, `v-security`, `v-indexed`, or schema-as-code in the extensibility layer.
3. **Moderate volume** — Thousands to low millions of documents with disciplined indexing (not high-frequency time-series or OLTP at web scale).
4. **Off the purchase critical path** — Not synchronous in checkout, cart, or payment with sub-10ms expectations.
5. **No better native fit** — Data does not belong in Catalog, OMS, CL/AD, or VBase.

### When not to use Master Data

| Data type                            | Better storage              | Why                                         |
| ------------------------------------ | --------------------------- | ------------------------------------------- |
| Product attributes, specifications   | **Catalog**                 | Native indexing, search, catalog APIs       |
| Orders, order history                | **OMS** (+ BFF if headless) | Single source of truth; MD duplicate drifts |
| Customer profiles, addresses         | **CL / AD**                 | Platform-managed, indexed, cached           |
| App cache or ephemeral state         | **VBase**                   | Per-app transient storage                   |
| Application logs, traces             | **VTEX logging**            | Not a datastore                             |
| High-throughput events / clickstream | **External DB**             | MD not built for millions of writes/day     |
| Relational models with joins         | **External SQL**            | No joins; denormalize or use SQL            |
| Strong consistency requirements      | **External DB**             | Indexed fields are eventually consistent    |

### Platform limits (architecture)

| Limit                                                 | Implication                                                                         |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------- |
| ~**5,000 records** per scroll iteration               | Large exports use scroll batches; do not assume unbounded in-memory loads           |
| ~**10 indexed fields** per entity (practical ceiling) | Prioritize filters used in production queries                                       |
| ~**1 write req/s per entity** at scale                | Queue writes via IO events; avoid synchronous write storms                          |
| **60 schemas per entity**                             | IO `masterdata` builder creates schemas per app version; lifecycle cleanup required |
| **Eventually consistent** indexes                     | Designs must tolerate lag after writes for filtered queries                         |

```text
Storage fit (simplified)
  Catalog / OMS / CL-AD / VBase  →  native commerce data first
  Master Data v2                   →  custom documents, off hot path
  External DB                      →  joins, heavy write volume, strong consistency
```

---

## Schema design principles

- **One entity per business concept** — e.g. reviews, wishlists, B2B org metadata; do not mix unrelated domains.
- **Index only what you query** — Only `v-indexed` fields appear in `where` / `publicFilter`; each index updates on every write.
- **Minimal `v-default-fields`** — Default API responses should not return large unused payloads.
- **`v-cache` matches read/write ratio** — Default `true` for read-heavy; `false` when consumers need immediate read-after-write consistency.
- **`v-security` is explicit** — Default deny; `allowGetAll: false` unless a public catalog of documents is intentional.
- **`additionalProperties: false`** when strict shape is required — otherwise MD may retain undeclared fields without validation.

---

## VTEX schema extensions (`v-*`)

Master Data v2 extends JSON Schema with VTEX-specific properties. Standard JSON Schema validators ignore them; they govern indexing, cache, security, triggers, and inheritance.

### `v-indexed`

| Aspect       | Guidance                                                                     |
| ------------ | ---------------------------------------------------------------------------- |
| Purpose      | Secondary indexes for `searchDocuments` / `scrollDocuments` `where` and sort |
| Index        | Fields used in filters, sorts, or `publicFilter`                             |
| Do not index | Large text never filtered, fields only read by document ID                   |
| Risk         | Non-indexed `where` → full scans → timeouts at 100k+ docs                    |
| Cost         | Every indexed field updated on **every** write                               |

### `v-cache`

| Value            | Use when                                                                 |
| ---------------- | ------------------------------------------------------------------------ |
| `true` (default) | Read-heavy entities; GET latency matters                                 |
| `false`          | High write frequency; consumers need fresh reads immediately after write |

### `v-default-fields`

Fields returned when callers omit `fields` in the API request. Keep **minimal** to reduce bandwidth on list/search paths.

### `v-security`

Controls **unauthenticated** access. All fields require auth by default.

| Property       | Role                                                                    |
| -------------- | ----------------------------------------------------------------------- |
| `allowGetAll`  | Unauthenticated list of all documents — keep `false` unless intentional |
| `publicRead`   | Field names readable without auth                                       |
| `publicWrite`  | Field names writable without auth                                       |
| `publicFilter` | Fields in public `where` (must also be in `v-indexed`)                  |

**Must not** expose PII (email, phone, government IDs, full addresses), internal scores, or org identifiers via `publicRead` / `publicFilter`.

### `v-triggers`

Automated actions on create/update when `condition` (where-style) matches.

| `action.type` | Typical use                    |
| ------------- | ------------------------------ |
| `email`       | Moderation or ops notification |
| `http`        | Webhook to external system     |
| `action`      | Platform action provider       |

Includes `retry.times` and `retry.delay` — suitable for simple, fire-and-forget workflows, not complex sagas.

### `v-canonicalto`

URL to a base schema in the **same** entity for inheritance (properties and constraints merged from target).

---

## Triggers vs event-driven integration

| Use `v-triggers` when                      | Prefer IO events / middleware when                   |
| ------------------------------------------ | ---------------------------------------------------- |
| Simple email or webhook on document change | Complex orchestration, backoff, dead-letter handling |
| Condition expressible as MD `where` filter | Sub-second reaction required                         |
| Fire-and-forget side effects               | Chain updates across multiple entities               |
|                                            | Risk of trigger loops (A → B → A)                    |
|                                            | Logic modifies other MD entities in cascade          |

---

## Query and export patterns

| Mechanism                      | When                                    | Architecture note                                           |
| ------------------------------ | --------------------------------------- | ----------------------------------------------------------- |
| **Search** (`searchDocuments`) | UI pagination, bounded result sets      | Page size up to 100; not for full-table export              |
| **Scroll** (`scrollDocuments`) | Bulk export, migration, large backfills | Batch through scroll; respect 5k iteration limits           |
| **Count**                      | Capacity planning, audits               | Use range/count headers — avoid fetching all IDs for counts |

Architectures exposing MD to storefronts should prefer **BFF** with caching and timeouts, not direct browser access to MD private APIs.

---

## Hard constraints

### Constraint: Index only queried fields

Every `v-indexed` field maintains a secondary index updated on **each** write. Indexing unused fields wastes write throughput and storage.

**Detection** — `v-indexed` lists fields never used in production `where` or sort contracts.

**Correct** — Index email, status, createdAt if those are the only filter/sort dimensions.

**Wrong** — Index every property “for flexibility” including notes/score never queried.

### Constraint: No sensitive data in public security

`publicRead`, `publicWrite`, and `publicFilter` expose data **without authentication**.

**Detection** — PII, payment-adjacent fields, or internal IDs in `v-security` public arrays; `allowGetAll: true` on sensitive entities.

**Correct** — Public display fields only (e.g. status, displayName, rating); `allowGetAll: false`.

**Wrong** — Public read/filter on email, phone, CPF, internal scores; public write on identity fields.

### Constraint: Respect 60 schemas per entity

Hard limit on schemas per entity. VTEX IO **`masterdata` builder** adds schemas per linked/published app version.

**Why this matters** — Frequent link cycles during development accumulate schemas and **block deployment** at the limit.

**Detection** — Long-running IO apps without schema retirement; approaching 60 via Admin or schemas API audit.

**Correct** — Schema lifecycle policy: delete obsolete schemas in lower environments; automate cleanup in release process.

**Wrong** — No cleanup until production deploy fails at the cap.

### Constraint: Master Data off the purchase critical path

Synchronous MD reads/writes in checkout, cart, or payment flows are **not** acceptable without proven latency SLOs and fallbacks.

**Detection** — Order placement or cart mutation awaits MD round-trip; MD chosen before Catalog/Checkout/Profile native fields.

**Correct** — MD for post-purchase, account enrichment, or async enrichment; native stores for hot path.

**Wrong** — Blocking checkout on MD validation or inventory stored only in MD without cache.

### Constraint: Do not use Master Data as a log or OLTP warehouse

Unbounded append-only entities (events, audit at web volume) outgrow MD operational model.

**Correct** — Stream events to warehouse/CDP/logger; MD for bounded business documents.

**Wrong** — “MD for everything” including clickstream or unbounded debug documents.

---

## Common failure modes

- **Over-indexing** — Many indexes on high-write entities; write latency and cost spike.
- **Missing indexes** — Filters on non-indexed fields; works in dev, fails in production at scale.
- **`v-cache: false` by default** — Unnecessary DB load on read-heavy entities.
- **`allowGetAll: true` with sensitive data** — Unauthenticated enumeration of documents.
- **Schema accumulation** — 60-schema limit blocks releases.
- **Trigger chains** — Cross-entity triggers causing loops or unpredictable latency.
- **MD on critical path** — Checkout depends on MD with no timeout, cache, or native alternative.
- **Duplicating OMS/Catalog** — Two sources of truth for orders or product data.
- **Scroll misuse** — Loading entire corpora into memory instead of batched scroll.

---

## Architecture review checklist

- [ ] **Storage fit** documented: why MD vs Catalog / OMS / CL-AD / VBase / external DB
- [ ] Entity is **off** checkout/cart/payment hot path (or exception justified with SLOs)
- [ ] **`v-indexed`** matches real query contracts; no speculative indexes
- [ ] **`v-cache`** aligned with read/write ratio
- [ ] **`v-security`** least privilege; no PII in public arrays; `allowGetAll` justified if true
- [ ] **Triggers** simple, non-chaining; complex flows use IO/events/middleware
- [ ] **Schema lifecycle** plan for 60-schema limit
- [ ] **Volume and rate** plan: scroll for bulk, queued writes at scale, BFF boundary for storefront
- [ ] **`v-default-fields`** minimal
- [ ] Aligns with **Technical Foundation** (data protection) and **Future-proof** (native-first) in Well-Architected review
