This reference provides guidance for AI agents supporting **VTEX payments solution architecture**. Apply these constraints and patterns when choosing payment integration models, reviewing custom Payment Provider Protocol (PPP) designs, PCI scope, async flows, idempotency, and connector hosting. Use with `SKILL.md` payment frameworks; this file does not replace PPP implementation guides or connector code reviews.

# Payments — solution architecture

## When this reference applies

Use when the task is **decision-oriented** across VTEX payments:

- **Native vs custom PPP** vs external orchestration (Adyen, Stripe, Spreedly)
- **PCI scope** and Secure Proxy requirements for card flows
- **Async methods** (Pix, Boleto, redirects) and callback/retry semantics
- **Idempotency**, payment state, and Gateway retry behavior (up to 7 days for `undefined`)
- **Connector topology**: VTEX IO Payment Provider Framework (PPF) vs self-hosted middleware
- **Homologation readiness**: endpoint coverage, latency, manifest accuracy

Do **not** use this file for Express route samples, `manifest.json` / `tsconfig` wiring, or TypeScript implementation detail — route execution to payment engineering and VTEX Developer documentation.

## Integration model (decision)

| Approach                           | When                                                              | Trade-offs                                            |
| ---------------------------------- | ----------------------------------------------------------------- | ----------------------------------------------------- |
| **VTEX native payment conditions** | Default; connector exists in catalog                              | Lowest custom surface; PCI largely on VTEX            |
| **Custom PPP connector**           | Acquirer not in catalog; non-standard authorize/capture lifecycle | Full homologation; you own SLA, idempotency, PCI path |
| **External orchestration**         | Multi-market acquirer routing, advanced 3DS/fraud                 | Extra hop; reconcile with Checkout and promotions     |

**PCI (architecture):** VTEX Checkout tokenizes card data — merchants often **SAQ A / SAQ A-EP**. Custom paths that touch raw PAN/CVV widen scope to **SAQ D** or full QSA. **Never** route raw card data through merchant BFF/Checkout API.

**Anti-pattern:** Checkout API used to pass raw card data through your backend — use Gateway tokenization, Secure Proxy, or PSP hosted fields.

```text
Shopper → VTEX Checkout → Payment Gateway → Connector (PPP) → PSP/Acquirer
                              ↑
                    Configuration flow (optional, Admin onboarding)
```

---

## Payment Provider Protocol (PPP) surface

The **Gateway initiates** all calls. The connector calls the Gateway only via **`callbackUrl`** (async status) and **Secure Proxy** (card authorization forwarding).

| Endpoint                                  | Method | Required | Role                                                |
| ----------------------------------------- | ------ | -------- | --------------------------------------------------- |
| `/manifest`                               | GET    | Yes      | Declares supported `paymentMethods` and split rules |
| `/payments`                               | POST   | Yes      | Create Payment (authorize)                          |
| `/payments/{id}/cancellations`            | POST   | Yes      | Cancel                                              |
| `/payments/{id}/settlements`              | POST   | Yes      | Capture / settle                                    |
| `/payments/{id}/refunds`                  | POST   | Yes      | Refund                                              |
| `/payments/{id}/inbound-request/{action}` | POST   | Yes      | Server-to-server inbound from Gateway               |
| `/authorization/token`                    | POST   | Optional | Merchant onboarding — auth token                    |
| `/authorization/redirect`                 | GET    | Optional | OAuth / PSP login redirect                          |
| `/authorization/credentials`              | GET    | Optional | Exchange code for credentials                       |

**Operational requirements:** HTTPS on port 443, TLS 1.2; homologation latency typically under 5 seconds, production under 20 seconds per Gateway call.

### Create Payment response (contract)

Architectures must ensure implementations return complete PPP shapes. Minimum fields the Gateway expects:

| Field                                                                   | Purpose                                           |
| ----------------------------------------------------------------------- | ------------------------------------------------- |
| `paymentId`, `status`                                                   | Identity and `approved` / `denied` / `undefined`  |
| `authorizationId`, `tid`, `nsu`, `acquirer`, `code`, `message`          | Reconciliation and tracking                       |
| `delayToAutoSettle`, `delayToAutoSettleAfterAntifraud`, `delayToCancel` | Auto-settle/cancel timers (seconds)               |
| `paymentUrl`                                                            | Optional — redirect for async / external checkout |

### Manifest

Must list **every** method the PSP actually supports, with correct `allowsSplit` (`onCapture`, `onAuthorize`, `disabled`). Empty or overstated manifests break Admin configuration and homologation.

---

## Async payment flows & callbacks

### Decision rules

- If the PSP cannot return a **final status synchronously**, the method is **async** → Create Payment returns **`status: "undefined"`** until confirmed.
- **Async examples:** Boleto (`BankInvoice`), Pix, bank transfer, redirect-based auth (PayPal, 3DS).
- **Sync examples:** card/debit with immediate authorization response.
- Store and use the exact **`callbackUrl`** from the request — including **`X-VTEX-signature`** query params; never hardcode or strip.
- **`inboundRequestsUrl` is server-to-server POST only** — browser GET redirects return 400. Redirect flows require a **custom public route** on IO (or equivalent on self-hosted), then POST to stored `callbackUrl` to trigger Gateway retry.

### Callback semantics (IO vs non-IO)

| Hosting               | After PSP confirms payment                                                                                                                         |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Non-IO middleware** | POST `callbackUrl` with body `{ paymentId, status }` and **`X-VTEX-API-AppKey` / `AppToken`** headers                                              |
| **VTEX IO (PPF)**     | POST `callbackUrl` (retry endpoint, often no payload) → Gateway **re-calls** Create Payment; connector returns updated status from persisted state |

```text
Non-IO notification:
  Gateway → Create (undefined) → PSP webhook → connector → POST callbackUrl → Gateway updates

IO retry:
  Gateway → Create (undefined) → PSP webhook → connector → POST callbackUrl
  Gateway → Create (retry) → connector returns approved/denied from store

Redirect (IO):
  Create (undefined + paymentUrl) → user on PSP → browser GET custom /_v/.../callback
  → update state → POST callbackUrl → Gateway retry → approved/denied → redirect to checkout
```

### `delayToCancel` (async)

Must reflect **real payment validity**, not the Gateway’s internal ~7-day retry window for `undefined`.

| Method          | Guidance                                                           |
| --------------- | ------------------------------------------------------------------ |
| **Pix**         | **900–3600 s** (15–60 min), aligned with QR TTL                    |
| **Boleto**      | Seconds until due date / provider deadline                         |
| **Other async** | Provider expiry SLA — never longer than actual instrument validity |

**Wrong:** `delayToCancel = 604800` (7 days) for Pix — orders stay “Authorizing” with expired QR codes.

### Hard constraints

#### Constraint: Async methods return `undefined` until paid

**Detection** — Create Payment returns `approved`/`denied` for Pix/Boleto/redirect when charge is only _initiated_.

**Correct** — `undefined` until PSP confirms; then callback/retry path updates to `approved` or `denied`.

**Wrong** — `approved` when QR/slip/redirect URL was generated but no funds captured.

#### Constraint: Use provided `callbackUrl` verbatim

**Detection** — Hardcoded `vtexpayments.com.br` URLs; missing `X-VTEX-signature`.

**Correct** — Persist exact URL from Create Payment; notify/retry with that URL.

**Wrong** — Manual callback URL construction.

#### Constraint: Redirect flows use custom public route (IO)

**Detection** — `inboundRequestsUrl` used as PayPal/3DS `return_url`.

**Correct** — Public `/_v/{connector}/callback` (or equivalent) → update state → POST `callbackUrl` → redirect shopper to checkout.

**Wrong** — Browser redirect to `inboundRequestsUrl`.

#### Constraint: Idempotent Create with evolving status

Gateway may call Create Payment with the same `paymentId` for up to **7 days** while `undefined`.

- **First call:** one PSP charge creation.
- **Retries:** no second charge; response from **persisted state** (status may move `undefined` → `approved`/`denied` after webhook).

**Wrong** — Every retry hits PSP again, or always returns stale `undefined` after payment confirmed.

---

## Idempotency & payment state

### Decision rules

| Operation                 | Idempotency key | Rule                                                       |
| ------------------------- | --------------- | ---------------------------------------------------------- |
| Create Payment            | `paymentId`     | Same key → same response; no duplicate PSP authorization   |
| Cancel / Capture / Refund | `requestId`     | Duplicate `requestId` → stored operation response          |
| State store               | Persistent      | PostgreSQL, DynamoDB, VBase on IO — **not** in-memory only |

### State machine (valid transitions)

```text
undefined → approved → settled
undefined → denied
approved → cancelled
```

Enforce transitions — e.g. cannot capture after cancel. Async methods stay `undefined` until PSP confirmation.

### Hard constraints

#### Constraint: `paymentId` idempotency on Create

**Detection** — No lookup before PSP call.

**Correct** — Existing `paymentId` → return stored response verbatim (same `tid`, `nsu`, `authorizationId`).

**Wrong** — Regenerate identifiers on retry or call PSP again.

#### Constraint: Identical duplicate responses

**Detection** — Retry returns new `tid`/`nsu` for same `paymentId`.

**Correct** — Byte-stable stored PPP response for duplicates.

#### Constraint: `requestId` on Cancel/Capture/Refund

Duplicate Gateway operation retries must not double-cancel or double-refund.

---

## PCI compliance & Secure Proxy

### Decision rules

| Environment                             | Card authorization                                                                  |
| --------------------------------------- | ----------------------------------------------------------------------------------- |
| **Non-PCI** (all VTEX IO apps)          | **Must** use `secureProxyUrl` from Create Payment                                   |
| **PCI-certified connector** (QSA AOC)   | May call PSP directly with raw card fields                                          |
| **Post-auth** (cancel, capture, refund) | **No** `secureProxyUrl` — direct PSP API by `tid` / `authorizationId`; no card data |

- Tokens (`numberToken`, `holderToken`, `cscToken`) valid **only** through `secureProxyUrl`.
- **May persist:** `card.bin`, `card.numberLength`, `card.expiration` only.
- **Must not persist or log:** PAN, CVV, holder name, token values, full payment request bodies.

```text
Authorize (Create Payment):
  Connector → POST secureProxyUrl + X-PROVIDER-Forward-To: PSP
  Gateway replaces tokens → PSP

Capture / Cancel / Refund:
  Connector → PSP API directly (credentials, outbound-access on IO)
```

### Hard constraints

#### Constraint: Secure Proxy when `secureProxyUrl` present

**Detection** — Direct PSP URL with tokens in non-PCI hosting.

**Correct** — All card authorization via proxy with `X-PROVIDER-Forward-To` and allowlisted PSP host.

**Wrong** — Bypass proxy on IO or send tokens directly to PSP.

#### Constraint: No Secure Proxy on post-auth

**Detection** — Cancel/capture/refund referencing `secureProxyUrl` or `SecureExternalClient`.

**Correct** — Separate direct PSP client for settlements/refunds/cancellations.

#### Constraint: No storage or logging of card data

**Detection** — DB/cache/logs contain PAN, CVV, holder, or tokens.

**Correct** — Redacted operational logs (paymentId, method, value, bin only).

---

## Connector hosting (architecture)

### VTEX IO (Payment Provider Framework)

- **Default path** for new VTEX-hosted connectors: IO app with `paymentProvider` builder, `PaymentProvider` + `PaymentProviderService`.
- **PCI:** Secure Proxy mandatory for card authorize on IO.
- **Redirect async flows:** custom public route in app routing + `service.json` exposure; register with `PaymentProviderService` routes.
- **Persistence:** VBase or external store for `paymentId` state; policies for `vbase-read-write`, `outbound-access` to PSP hosts.
- **Gateway retry on IO:** use framework **`retry(request)`** semantics where PPP requires — avoid ad-hoc duplicate retry logic.
- **Release:** beta affiliation → test mode/workspace → stable → **homologation** (~30 day SLA typical); account allowlisting for IO connectors via VTEX ticket.
- **Build constraint (IO ops):** Builder Hub compiles with **TypeScript 3.9.7** — affects dependency and syntax choices for engineering; not an architecture pattern but a platform constraint for IO connectors.

### Self-hosted middleware

- Implements same **six PPP payment-flow endpoints** and HTTPS/TLS rules.
- Owns **persistent idempotency store** and callback notification with app keys.
- **PSP integration:** validate base URL vs path versioning once per environment; OAuth/token calls must meet Gateway **~2s** budget on some paths — cache tokens in connector layer.

### PSP integration checklist (architecture)

1. Confirm **test vs production** base URLs and API versioning (no duplicated path segments).
2. Map each PPP operation to PSP operations (authorize, capture, cancel, refund).
3. Define **idempotency** and **state store** before go-live.
4. Classify methods **sync vs async** and set `delayToCancel` per instrument.
5. Plan **PCI path** (Secure Proxy vs certified direct).
6. Plan **homologation** artifacts: connector identity, endpoints, methods, allowed accounts, contact.

---

## Common failure modes

- **Sync approval of async methods** — fulfillment without settlement (Pix/Boleto).
- **`inboundRequestsUrl` as browser return URL** — 400 and stuck `undefined`.
- **Hardcoded or stripped `callbackUrl`** — payments never complete.
- **No callback retry** — single failed POST; relies only on slow Gateway polling.
- **Duplicate PSP charges** — no `paymentId` guard on Create retries.
- **Stale `undefined` on retry** — webhook updated store but Create still returns old status.
- **Misaligned `delayToCancel`** — especially 7-day default for Pix.
- **Secure Proxy on capture/refund** — `secureProxyUrl` undefined post-auth.
- **Direct card handling on IO** — PCI violation and failed token translation.
- **PAN/tokens in logs or DB** — compliance incident.
- **Incomplete PPP responses** — missing delay fields or reconciliation IDs.
- **Partial endpoint set** — homologation and production cancel/refund failures.
- **Manifest mismatch** — Admin methods don’t match PSP capability.

---

## Architecture review checklist

- [ ] Integration model chosen with documented **why not native** (if custom PPP)
- [ ] **PCI scope** stated (SAQ level, Secure Proxy vs AOC direct)
- [ ] All **six payment-flow endpoints** in scope for custom connectors
- [ ] **Async methods** use `undefined`, correct `delayToCancel`, and callback strategy (IO vs non-IO)
- [ ] **`callbackUrl`** stored verbatim; `X-VTEX-signature` preserved
- [ ] **Redirect flows** use custom public route, not `inboundRequestsUrl`
- [ ] **`paymentId` / `requestId` idempotency** with persistent store
- [ ] **Authorize** via Secure Proxy on non-PCI; **post-auth** direct to PSP
- [ ] **No card data** in logs, caches, or databases beyond allowed metadata
- [ ] **Latency** and **HTTPS/TLS** requirements in non-functional design
- [ ] **Homologation** and affiliation/test plan for IO connectors
- [ ] Aligns with **Technical Foundation** in Well-Architected review
