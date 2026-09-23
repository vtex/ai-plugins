This reference provides guidance for AI agents supporting **VTEX headless solution architecture**. Apply these constraints and patterns when scoping, reviewing, or documenting headless storefronts — BFF boundaries, credential handling, caching posture, checkout proxy design, and Intelligent Search integration. Use with `SKILL.md` decision frameworks; this file does not replace implementation guides or developer how-tos.

# Headless commerce — solution architecture

## When this reference applies

Use when the task is **decision-oriented** across a headless VTEX experience layer:

- Choosing or reviewing **BFF** placement, API exposure, and authentication boundaries
- Classifying which VTEX APIs are **public vs private**, **cacheable vs never-cacheable**
- Designing **checkout** flows that proxy through server-side layers (OrderForm, cookies, order placement)
- Planning **Intelligent Search** integration (direct frontend vs BFF, analytics, pagination contracts)

Do **not** use this file for line-by-line implementation, framework setup, or library-specific code — route execution to product teams and VTEX Developer documentation.

## Platform boundaries

Headless VTEX splits **Experience** (custom frontend) from **Commerce** (Checkout, OMS, Catalog, IS) with a mandatory **BFF** for everything except Intelligent Search reads.

```text
Experience (browser / app)
    │
    ├── Direct (allowed): Intelligent Search — public, read-only, no credentials
    │
    └── BFF (required): Checkout, Profile, OMS, Payments, Catalog (when proxied), all /pvt/
            │
            ├── Server session: VtexIdclientAutCookie, orderFormId, checkout cookies
            ├── Server secrets: VTEX_APP_KEY / VTEX_APP_TOKEN (per-module keys, least privilege)
            └── VTEX Commerce APIs
```

---

## BFF layer design & security

### Decision rules

- A **BFF is mandatory** for every headless VTEX program. There is no safe headless model without a server-side layer.
- Route **all** VTEX API traffic through the BFF **except** Intelligent Search (public, read-only, designed for direct frontend use).
- Use **`VtexIdclientAutCookie`** (server-side session) for shopper-scoped calls; use **`X-VTEX-API-AppKey` / `X-VTEX-API-AppToken`** for machine-to-machine calls.
- Treat exposure of app keys, app tokens, `VtexIdclientAutCookie`, checkout cookies, or session tokens in the browser as a **security violation** — use “must not” / “never”, not “prefer” / “ideally”.
- Classify by path: `/pub/` is public but most still need BFF for session and data protection; `/pvt/` **must** use BFF.
- **Checkout** (`/api/checkout/`) must be proxied even when labeled public — it carries PII and cart state.
- Use **separate API keys** with minimal permissions per BFF domain (OMS, checkout, catalog), not one Owner-scoped key.

### Hard constraints

#### Constraint: BFF is mandatory — no exceptions

Every headless storefront **must** have a server-side BFF. Client code **must not** call private VTEX endpoints (`/pvt/`) or embed app credentials.

**Why this matters** — Private APIs require app keys. Keys in the browser are extractable via DevTools and enable order, pricing, and admin abuse.

**Detection** — Browser bundles or SPA code calling `vtexcommercestable.com.br` paths under `/api/checkout`, `/api/oms`, `/api/profile`, or any `/pvt/` endpoint.

**Correct** — Frontend calls only same-origin BFF routes; BFF holds credentials and forwards to VTEX.

**Wrong** — SPA calls VTEX OMS/Checkout/Profile directly with `X-VTEX-API-AppKey` / `X-VTEX-API-AppToken` or shopper cookies managed in the client.

#### Constraint: VtexIdclientAutCookie is server-side only

The shopper token **must** live in a secure server session (encrypted cookie, Redis, etc.). **Must not** store it in `localStorage`, `sessionStorage`, or client-accessible variables.

**Why this matters** — Bearer token impersonates the shopper (orders, profile, payment context). XSS or shared devices enable account takeover.

**Detection** — `VtexIdclientAutCookie` in client storage, query params passed to the SPA, or Cookie headers built in frontend code.

**Correct** — BFF captures token at login callback, stores in session, clears VTEX cookie from browser; BFF attaches token on upstream calls.

**Wrong** — Frontend reads token from URL/localStorage and sends it to VTEX APIs.

#### Constraint: API keys never in client scope

`VTEX_APP_KEY` / `VTEX_APP_TOKEN` **only** in server environment variables. **Must not** appear in browser bundles or `NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*` variables.

**Why this matters** — Keys inherit role permissions; leaked keys enable programmatic abuse across the account.

**Detection** — Key names or header values in `src/`, `app/`, `public/`, or client-prefixed env vars.

**Correct** — Keys injected only in BFF/middleware process environment.

**Wrong** — Keys in frontend env or hardcoded in client-accessible files.

### Architecture pattern

```text
Frontend
    ├── Intelligent Search (direct, CDN-cacheable)
    └── BFF
            ├── Session: VtexIdclientAutCookie
            ├── M2M: scoped app keys per domain
            ├── Validate/sanitize inputs
            └── Proxy → Checkout, OMS, Profile, other VTEX APIs
```

Authentication flow (architecture): redirect to VTEX login → callback to BFF → token in server session → frontend uses only BFF session cookie.

### Common failure modes

- **Proxying Intelligent Search through BFF** without SSR or custom enrichment — adds latency and cost on a high-frequency path; IS is the intentional frontend exception.
- **Single API key with broad role** — compromise grants full account access; split keys by domain with least privilege.
- **Logging credentials or cookies** — request logging must redact app keys, `Cookie`, and `Authorization` headers.

### Architecture review checklist

- [ ] BFF present for all non-IS VTEX integration
- [ ] No `/pvt/` or private Checkout/OMS/Profile calls from the browser
- [ ] App keys server-only; no client-prefixed env exposure
- [ ] `VtexIdclientAutCookie` in server session only
- [ ] Intelligent Search not proxied without documented justification
- [ ] Scoped API keys per BFF module; sensitive headers redacted in logs

---

## Caching & performance

### Decision rules

- Classify every VTEX API as **cacheable** or **non-cacheable** before designing layers.
- **Cacheable** (public, read-only, non-personalized): Intelligent Search, Catalog public endpoints, top searches, autocomplete.
- **Non-cacheable** (transactional, personalized, sensitive): Checkout, Profile, OMS, Payments, private Pricing — **never** at CDN, BFF, Redis, or browser.
- Prefer **`stale-while-revalidate`** for freshness vs latency on catalog/search data.
- Use **moderate TTLs (2–15 minutes)** plus invalidation; avoid hour/day TTLs without purge/webhooks.
- Cache keys by **URL/params** (and trade policy if needed), **not** by user identity.
- **Layering**: CDN edge for direct IS calls; BFF cache (Redis/in-memory) for proxied catalog; `no-store` on checkout/profile/orders.

| API                     | Recommended TTL | SWR   |
| ----------------------- | --------------- | ----- |
| IS `product_search`     | 2–5 min         | 60s   |
| Catalog category tree   | 5–15 min        | 5 min |
| IS autocomplete         | 1–2 min         | 30s   |
| IS top searches         | 5–10 min        | 2 min |
| Catalog product details | 5 min           | 60s   |

| API             | Must never cache                    | Why |
| --------------- | ----------------------------------- | --- |
| Checkout        | Per-user cart; changes every action |
| Profile         | PII; GDPR/LGPD                      |
| OMS orders      | Status and ownership per user       |
| Payments        | Financial; real-time                |
| Pricing private | May be segment-specific             |

### Hard constraints

#### Constraint: Cache public read-only data aggressively

Search and catalog responses **must** use CDN and/or BFF caching. Uncached headless traffic hits rate limits (429), adds hundreds of ms per view, and degrades conversion at scale.

**Detection** — IS/Catalog called on every page view with no `Cache-Control`, CDN policy, or BFF cache.

**Correct** — Bounded TTL + SWR on IS/catalog; observability (`X-Cache` or equivalent hit/miss/stale metrics).

**Wrong** — Every request forwarded live to VTEX with no cache layer.

#### Constraint: Never cache transactional or personal data

Checkout, Profile, OMS, Payments responses **must not** be cached at any layer.

**Why this matters** — Stale carts, wrong prices, cross-user profile leakage, and compliance violations.

**Detection** — `Cache-Control` with `max-age > 0`, Redis/memory cache, or CDN cache on checkout/order/profile/payment routes.

**Correct** — `no-store` / `must-revalidate` on all checkout and account routes.

**Wrong** — Shared cache keyed by session or caching OrderForm/profile payloads.

#### Constraint: Invalidation strategy required

Every cache **must** have bounded TTL and a path to purge when catalog/pricing changes (webhook, admin purge, or conservative TTL + SWR).

**Detection** — In-memory maps with no expiry, or multi-hour TTL with no catalog-change hook.

**Correct** — TTL per data class + event-driven or secured manual invalidation for launches.

**Wrong** — “Cache forever until restart” for product or price data.

### Architecture pattern

```text
Frontend
    ├── IS → CDN edge (TTL 2–5 min, SWR)
    └── BFF
            ├── Catalog routes → BFF/Redis cache → Catalog API
            └── Checkout / Profile / Orders → no cache → VTEX
```

### Common failure modes

- **Per-user catalog cache keys** — multiplies storage and removes benefit; key by query/facets/trade policy only.
- **Very long TTL without invalidation** — stale price, stock, and assortment until TTL expires.
- **No cache observability** — cannot tune TTLs or prove hit rate without metrics/headers.

### Architecture review checklist

- [ ] IS and catalog cached at CDN and/or BFF with bounded TTL
- [ ] Checkout, profile, OMS, payments explicitly non-cacheable
- [ ] Invalidation defined (TTL + webhook or purge for catalog events)
- [ ] Cache keys not tied to shopper identity
- [ ] SWR used where appropriate; TTLs moderate (minutes, not days)
- [ ] Cache hit/miss/stale observable in operations

---

## Checkout proxy & OrderForm

### Decision rules

- **All** Checkout API operations go through the BFF — cart, attachments, placement.
- **`orderFormId`** in server-side session only — not `localStorage` / `sessionStorage`.
- Forward **`CheckoutOrderFormOwnership`** and **`checkout.vtex.com`** cookies on every BFF ↔ VTEX checkout call; persist from responses.
- **Validate inputs** in BFF before forwarding — never pass raw client bodies through.
- **Order placement**: place → pay → process in one **synchronous BFF operation** within VTEX’s **5-minute** window after place.
- **Reuse** existing `orderFormId` from session; create new cart only when none exists.

| Attachment        | Purpose                      |
| ----------------- | ---------------------------- |
| items             | Add/update/remove line items |
| clientProfileData | Shopper profile              |
| shippingData      | Address and delivery         |
| paymentData       | Payment method selection     |
| marketingData     | Coupons, UTM                 |

### Hard constraints

#### Constraint: All checkout via BFF

Browser **must not** call `/api/checkout/` directly.

**Why this matters** — PII, payment context, and cart manipulation; cookies and validation belong server-side.

**Detection** — Client `fetch`/HTTP to `/api/checkout/` on any storefront origin.

**Correct** — Frontend → BFF cart/checkout routes only.

**Wrong** — SPA posts items or profile to VTEX checkout URLs with client-held `orderFormId`.

#### Constraint: orderFormId server-side

Cart identifier **must not** be exposed for direct VTEX API use from the client.

**Why this matters** — `orderFormId` unlocks cart contents including profile and shipping; enables tampering if used without BFF validation.

**Detection** — `orderFormId` in browser storage or returned to client for direct VTEX calls.

**Correct** — Session holds id; BFF returns sanitized OrderForm without internal cookies/ids needed for bypass.

**Wrong** — Client stores `orderFormId` and calls checkout APIs directly.

#### Constraint: Server-side validation before VTEX

All cart/checkout payloads **must** be validated in BFF (types, ranges, allowed sellers/SKUs) before proxying.

**Detection** — BFF forwards `req.body` unchanged.

**Correct** — Schema/contract validation and safe error mapping to the client.

**Wrong** — Trust client prices, quantities, or seller IDs without server checks.

### Architecture pattern

```text
Frontend → BFF /cart/*
    1. Validate input
    2. Read orderFormId + checkout cookies from session
    3. Call VTEX Checkout API
    4. Update session; return sanitized OrderForm
```

Order placement: single BFF transaction chaining place → payment → process; clear session on success; map VTEX errors to safe client messages (log details server-side only).

### Common failure modes

- **New cart every page load** — abandons carts; always retrieve by session `orderFormId` first.
- **Splitting placement across async client steps** — exceeds 5-minute window → incomplete/canceled orders.
- **Raw VTEX errors to the client** — leaks paths and account internals.

### Architecture review checklist

- [ ] No direct browser calls to `/api/checkout/`
- [ ] `orderFormId` and checkout cookies server-managed
- [ ] Input validation before VTEX proxy
- [ ] Place → pay → process in one server flow within 5 minutes
- [ ] Session reuses cart; errors sanitized for the client

---

## Intelligent Search

### Decision rules

- Call Intelligent Search **from the frontend** by default — no auth required; BFF proxy only for SSR, enrichment, or compliance-driven aggregation.
- Use API **`sort`** and **facet paths** for filtering/sorting — not client-side re-rank of one page.
- Include **`page`** and **`locale`** on every request.
- On pagination after page 0, **reuse `operator` and `fuzzy`** from the first response (first request sends null/absent; API sets values).
- Send **analytics events** to Intelligent Search Events API (headless) — minimum search impressions and product clicks; ranking degrades without them.
- Fetch facets from **`/facets`** — `/product_search` does not return filter metadata.
- Use **`hideUnavailableItems`** when out-of-stock products should not appear (API-level, not client filter of one page).

| Endpoint                    | Purpose                         |
| --------------------------- | ------------------------------- |
| `/product_search/{facets}`  | Product listing by query/facets |
| `/facets/{facets}`          | Available filters               |
| `/autocomplete_suggestions` | Typeahead                       |
| `/top_searches`             | Popular terms                   |
| `/correction_search`        | Spelling correction             |
| `/search_suggestions`       | Related terms                   |
| `/banners/{facets}`         | Merchandising banners           |

**Facet logic**: same facet type = OR; different types = AND.

### Hard constraints

#### Constraint: Analytics events required

Implementations **must** send events to Intelligent Search Events API (headless) when results render and when users click products from search.

**Detection** — Search UI with no `sp.vtex.com/event-api` (or documented equivalent) integration.

**Correct** — Events include query text, `operator`, `correction.misspelled`, locale, match count, product positions; failures must not break UX.

**Wrong** — Product search only, no behavioral feedback to IS.

#### Constraint: Persist operator and fuzzy across pages

Page 2+ **must** send the `operator` and `fuzzy` values returned on page 0.

**Why this matters** — API tunes matching per query; fixed or missing values cause inconsistent pagination.

**Detection** — Pagination requests with hardcoded `operator`/`fuzzy` or omitting them after first page.

**Correct** — Architecture stores search session state (operator, fuzzy) server- or client-side for the search session only.

**Wrong** — Independent page requests with default operator/fuzzy.

#### Constraint: Do not proxy IS without justification

Default is **direct frontend → IS** with CDN caching.

**Detection** — BFF pass-through search with no SSR, auth, or enrichment.

**Correct** — Document reason if BFF or edge worker sits in front of IS.

**Wrong** — Mandatory BFF hop “for consistency” on all search calls.

### Architecture pattern

```text
Frontend
    ├── GET …/intelligent-search/product_search|facets|autocomplete…
    └── POST https://sp.vtex.com/event-api/v1/{account}/event
```

**Response contracts (architecture)**:

- **`operator` / `fuzzy`**: persist from first page; pass to analytics.
- **`correction.misspelled`**: must match analytics payload.
- **`recordsFiltered`**: total matches; **`products`**: current page only.
- **`pagination`**: use API metadata for next/previous pages.

### Common failure modes

- **Missing locale** — wrong language and relevance in multi-language stores.
- **Client-side re-sort/filter** — breaks IS ranking and only affects current page.
- **Assuming facets on product_search** — need parallel `/facets` call.
- **No debounce strategy for autocomplete** — rate limit and UX risk (architecture: throttle typeahead requests).
- **Proxying IS through BFF** without SSR or added value.

### Architecture review checklist

- [ ] IS called directly unless SSR/enrichment documented
- [ ] `page` and `locale` on all search requests
- [ ] `operator` / `fuzzy` persisted for pagination
- [ ] Analytics events for impressions and clicks with correct `misspelled` / `operator`
- [ ] Sort/filter via API facets and `sort`, not client re-processing
- [ ] Facets from `/facets`; unavailable items hidden at API when required

---

## Cross-cutting architecture review

- [ ] Experience layer respects BFF boundary (IS exception documented)
- [ ] Technical Foundation: secrets, tokens, PCI-adjacent checkout data server-side
- [ ] Caching classified; transactional paths non-cacheable
- [ ] Checkout and search contracts aligned with VTEX time windows and IS pagination rules
- [ ] Operational metrics for cache and checkout error rates defined
