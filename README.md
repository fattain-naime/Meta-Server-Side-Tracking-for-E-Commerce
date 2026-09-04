# Meta Server-Side Tracking for E-Commerce - Deep Research Report (September 2026)

> **Project:** shopika.com.bd
> **Research window:** August 31 – September 2026, against the **live** Meta developer documentation
> **Scope:** Everything Meta (Facebook) currently recommends for setting up server-side event tracking on an e-commerce website — architecture choice, installation, configuration, payloads, user-data handling, deduplication, testing, verification, limits, and pitfalls.
> **Source policy:** Official Meta developer documentation only (developers.facebook.com). Each quoted page's own "Updated" date is recorded. **No backdated or deprecated guidance is used.**

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Terminology](#2-terminology)
3. [What Meta Recommends in 2026 — The Redundant Setup](#3-what-meta-recommends-in-2026--the-redundant-setup)
4. [Method Comparison for an E-Commerce Site](#4-method-comparison-for-an-e-commerce-site)
5. [Graph API Versioning — v26.0 Is Current](#5-graph-api-versioning--v260-is-current)
6. [Conversions API Endpoint and Request Specification](#6-conversions-api-endpoint-and-request-specification)
7. [Server Event Parameters](#7-server-event-parameters)
8. [Customer Information Parameters and Hashing Rules](#8-customer-information-parameters-and-hashing-rules)
9. [fbp, fbc and Meta ClickID — Full Handling Guide](#9-fbp-fbc-and-meta-clickid--full-handling-guide)
10. [external_id and Advanced Matching](#10-external_id-and-advanced-matching)
11. [Standard E-Commerce Events and Object Properties](#11-standard-e-commerce-events-and-object-properties)
12. [Event Deduplication (Browser + Server)](#12-event-deduplication-browser--server)
13. [Event Timing, Batching and Retry Semantics](#13-event-timing-batching-and-retry-semantics)
14. [Testing — the Test Events Tool and test_event_code](#14-testing--the-test-events-tool-and-test_event_code)
15. [Verification and Event Quality (EMQ, Dataset Quality API)](#15-verification-and-event-quality-emq-dataset-quality-api)
16. [Data Processing Options / Limited Data Use](#16-data-processing-options--limited-data-use)
17. [API Limits and Rate Limiting](#17-api-limits-and-rate-limiting)
18. [Conversions API Gateway and Signals Gateway](#18-conversions-api-gateway-and-signals-gateway)
19. [Deprecated or Discouraged Approaches — Do Not Use](#19-deprecated-or-discouraged-approaches--do-not-use)
20. [Consolidated Best-Practices Checklist for E-Commerce](#20-consolidated-best-practices-checklist-for-e-commerce)
21. [Appendix A — How Modules/MetaPixel Implements This Research](#appendix-a--how-modulesmetapixel-implements-this-research)
22. [Appendix B — Source Documents and Fetch Dates](#appendix-b--source-documents-and-fetch-dates)

---

## 1. Executive Summary

For a live e-commerce website in September 2026, Meta's official position is unambiguous:

- **Meta recommends running the Meta Pixel (browser) *together with* the Conversions API (server) on the same Meta Pixel / Dataset ID.** Meta calls this a **"redundant setup"**, and its deduplication documentation states: *"For optimal ad performance, we recommend that advertisers implement the Conversions API alongside their Meta Pixel. We call this a 'redundant setup'."* (Deduplication guide, updated Jun 28, 2026.)
- The current API surface is **Graph API v26.0** (released **July 29, 2026**; previous release v25.0 was February 2026). All new integrations should target v26.0 endpoints.
- The server endpoint is a single POST edge:
  `https://graph.facebook.com/{API_VERSION}/{PIXEL_ID}/events?access_token={TOKEN}`
  The Conversions API now carries **web, app, offline/physical-store and business-messaging events through this single endpoint** — the standalone App Events API and Offline Conversions API are **no longer recommended for new integrations**.
- **Every event should carry an `event_id`**, paired with `event_name`. This is Meta's **recommended** deduplication method, and it also makes batch retries safe (valid events already stored are dropped as duplicates instead of double-counted).
- User data must follow Meta's exact hashing contract: **SHA-256, lowercase, trimmed, no extra spaces** for `em`, `ph`, `fn`, `ln`, `ge`, `db`, `ct`, `st`, `zp`, `country`; `external_id` hashing *recommended*; and `client_ip_address`, `client_user_agent`, `fbp`, `fbc` sent **raw (never hashed)**.
- The ClickID chain (`fbclid` → `_fbc` cookie → `fbc` server parameter) has a strict value format — `fb.{subdomainIndex}.{creationTime}.{fbclid}` — and when the `_fbc` cookie is absent, `fbc` **must be constructed server-side** from the landing URL's `fbclid` query parameter (guide updated **Jan 9, 2026**).
- Web events sent via CAPI **require** `client_user_agent`, `action_source`, and `event_source_url` alongside the event body (Parameters reference, updated Jun 30, 2026).
- `test_event_code` is **testing-only** and must be removed in production — importantly, **events sent with a test code are NOT dropped**: *"Events sent with test_event_code are not dropped. They flow into Events Manager and are used for targeting and ads measurement purposes."*
- For e-commerce catalog ads, `Purchase` **requires** `currency` + `value`, and `contents` **or** `content_ids` is **required for Advantage+ catalog ads** (also strongly recommended on `AddToCart`, `Search`, `ViewContent`).

The remainder of this document is the complete, source-backed detail behind each of these points, ending with the mapping to the `Modules/MetaPixel` implementation deployed on this site.

---

## 2. Terminology

| Term | Meaning (2026 usage) |
|---|---|
| **Meta Pixel** | The browser-side JavaScript tag (`fbq('track', …)`) plus its first-party cookies `_fbp` / `_fbc`. Identified by a numeric Pixel ID. |
| **Dataset** | The server-side twin of a Pixel — a container of events in Events Manager. For web properties the Dataset ID **equals the Pixel ID**, which is why the CAPI endpoint accepts the pixel ID. "Dataset" is the term Meta now uses across Events Manager and its quality APIs. |
| **Conversions API (CAPI)** | The Marketing-API-based server interface for sending events to a Dataset through the Graph API `/events` edge. |
| **Redundant setup** | Pixel (browser) **and** CAPI (server) sending the same logical events, deduplicated by Meta. Meta's recommended architecture. |
| **event_id** | Advertiser-supplied unique identifier per event; the join key Meta uses to collapse duplicate browser/server copies. |
| **Event Match Quality (EMQ)** | 10-point per-event quality score in Events Manager that reflects how completely user_data parameters are filled. |
| **Dataset Quality API** | Graph API surface to read EMQ/event coverage metrics programmatically. |
| **CAPI Gateway** | Self-hosted (Docker, AWS App Runner / EKS, GCP) event-forwarding gateway that sits between your site and Meta and relays events; an alternative to direct CAPI calls. |
| **Signals Gateway** | Newer Meta-hosted multi-channel gateway product (successor direction to CAPI Gateway) covering web/app/CRM pipelines. |
| **Advanced Matching** | Optional Pixel feature that hashes browser-known PII (email/phone/name) into the pixel payload to raise match rates; the server-side equivalent is filling `user_data`. |
| **ClickID (`fbclid`)** | Meta-generated parameter appended to landing URLs from FB/IG ads; the case-sensitive raw value must never be modified. |

---

## 3. What Meta Recommends in 2026 — The Redundant Setup

Meta's 2026 guidance for e-commerce websites, in order of preference:

1. **Meta Pixel + Conversions API together (redundant setup) with event_id-based deduplication.** This is the recommended architecture stated verbatim in the deduplication guide. Browser events capture users who block or restrict server contexts; server events recover users who block browser JavaScript or drop cookies. Sending both maximizes event coverage and match quality — *provided* both ends share the same `event_id` + `event_name` so Meta can collapse duplicates.
2. **Conversions API only** is acceptable when a browser pixel genuinely cannot run, but it forfeits browser-side signals (`fbp`, pixel-observed advanced matching) and scores lower on event coverage in Events Manager.
3. **Conversions API Gateway** (self-hosted) or **Signals Gateway** (Meta-hosted) for teams that want a managed relay layer instead of calling Graph API directly from application servers. Functionally the same payload contract; the gateway adds token storage, retry, and multi-account routing.
4. **Meta partner integrations / GTM server container** ("Conversions API for Google Tag Manager" exists as a first-class documented path) — relevant for agencies, not required for a Laravel application that can call the API directly.

Additional 2026 positions that shape the recommendation:

- The Conversions API is now the **single funnel for web, app, offline and messaging events**; Meta explicitly says the App Events and Offline Conversions APIs are *"no longer recommended for new integrations."*
- Each Graph API version is *"supported for at least two years"* for the Conversions API specifically (Marketing API release cycle is aligned with Graph API; the exception is valid for CAPI only) — so pinning v26.0 today is safe for years.
- Docs now ship a **"Copy for LLM / View as Markdown"** affordance on every page, and the **v26.0 release highlights include the Conversions API Parameter Builder** — a library that auto-forms `fbc`/`fbp`, `event_source_url`, and normalizes/hashes PII. Its exact logic is what a server-side integration must otherwise implement by hand (and what `Modules/MetaPixel` implements natively in PHP).

---

## 4. Method Comparison for an E-Commerce Site

| Criterion | Pixel only | CAPI only | **Pixel + CAPI (recommended)** | CAPI Gateway |
|---|---|---|---|---|
| Where events originate | Browser JS | Your server | Both | Server (relayed) |
| Event coverage | Loses ad-blocker/JS-off users | Loses cookie-restricted browser signal | **Maximum** | Maximum (same as direct CAPI) |
| Match quality inputs | Cookie `_fbp`/`_fbc` + optional Advanced Matching | Full `user_data` (PII, IP, UA) you already hold | **Union of both** | Same as CAPI |
| Deduplication needed | No (single source) | No (single source) | **Yes — event_id + event_name** | Yes (same contract) |
| Latency to Meta | Immediate | Batched (recommended ≤ 1 h) | Immediate + batched | Gateway-managed |
| Infrastructure | None | HTTPS POST from app | HTTPS POST from app | Docker/AWS/GCP instance to maintain |
| Token handling | n/a | App must store access token | App must store access token | Gateway stores token |
| Fit for Shopika (Laravel, single BD storefront) | Current core basic pixel (PageView only, no event_ids) | Partial | **Chosen architecture** | Over-engineered for one site |

**Decision for this project:** redundant setup implemented by the `Modules/MetaPixel` module — browser pixel events + server CAPI events with shared deterministic `event_id`s, one Dataset ID, one system-user access token, `test_event_code` support for staging verification.

---

## 5. Graph API Versioning — v26.0 Is Current

- **Latest version: v26.0**, released **July 29, 2026** (Graph API Changelog / Versions page; blog post "Introducing Graph API v26.0 and Marketing API v26.0", 2026-07-29).
- Previous release: **v25.0**, introduced **February 18, 2026**.
- Available version ladder as listed: v26.0 → v25.0 → v24.0 → v23.0 → v22.0 → … (the changelog lists every version back to v2.1).
- **CAPI-specific support window:** *"Marketing and Graph APIs have different version deprecation schedules. Our release cycle is aligned with the Graph API, so **every version is supported for at least two years**. This exception is only valid for the Conversions API."* (Using the API, updated Jul 17, 2026.)
- **v26.0 notable change relevant to commerce:** the **Commerce Order Management API is deprecated** (47 endpoints; calls on v26.0+ blocked from July 29, 2026; removed from all remaining versions on **October 27, 2026**) as Facebook/Instagram Checkout for Shops has been sunset. This does **not** affect the Conversions API events edge — it only removes the old on-Facebook shop order-management surface.
- Practical rule: request URL pins the version segment (`/v26.0/`); never use unversioned calls; never mix versions between browser pixel config and server endpoint (the pixel JS is versionless, but your CAPI calls must stay on one pinned version).

---

## 6. Conversions API Endpoint and Request Specification

**Endpoint** (Using the API, updated Jul 17, 2026):

```
POST https://graph.facebook.com/{API_VERSION}/{PIXEL_ID}/events?access_token={TOKEN}
Content-Type: application/json   (or multipart form field `data`)

{
  "data": [ { …event… }, { …event… } ],
  "test_event_code": "TEST123"      // testing only — REMOVE in production
}
```

Canonical single Purchase event (fields shown exactly as in Meta's v26.0 example):

```json
{
  "event_name": "Purchase",
  "event_time": 1633552688,
  "event_id": "event.id.123",
  "event_source_url": "https://example.com/product/123",
  "action_source": "website",
  "user_data": {
    "client_ip_address": "192.19.9.9",
    "client_user_agent": "<original request User-Agent>",
    "em": ["309a0a5c3e211326ae75ca18196d301a9bdbd1a882a4d2569511033da23f0abd"],
    "ph": ["254aa248acb47dd654ca3ea53f48c2c26d641d23d7e2e93a1ec56258df7674c4",
           "6f4fcb9deaeadc8f9746ae76d97ce1239e98b404efe5da3ee0b7149740f89ad6"],
    "fbc": "fb.1.1554763741205.AbCdEfGhIjKlMnOpQrStUvWxYz1234567890",
    "fbp": "fb.1.1558571054389.1098115397"
  },
  "custom_data": {
    "value": 100.2,
    "currency": "USD",
    "content_ids": ["product.id.123"],
    "content_type": "product",
    "contents": [{"id": "product.id.123", "quantity": 1, "delivery_category": "home_delivery"}]
  },
  "opt_out": false
}
```

Key structural rules:

- `data` is an **array of up to 1,000 events** per request.
- `access_token` goes in the **query string** (or form field); it must be a token with permission to post to that pixel/dataset — in practice a **system-user token** from Business Settings.
- `em` and `ph` are **arrays** — multiple hashed values (e.g. raw phone and normalized international phone, or several emails) may be supplied to improve matching.
- `opt_out: true` suppresses the event from ads use; default false.
- Response verification: Events Manager shows received events within **~20 minutes**; the Test Events tool shows them in real time.
- Errors: invalid events cause a request error, but events in the same batch that carry an `event_id` and are valid **are still accepted** — fix invalid ones and resend the whole batch; stored ones are dropped as duplicates (see §13).

---

## 7. Server Event Parameters

From the Parameters reference (updated Jun 30, 2026), the per-event server parameters are:

| Parameter | Notes |
|---|---|
| `event_name` | Standard or custom event name (see §11). |
| `event_time` | **Unix timestamp in seconds** of when the event actually happened. Max **7 days** in the past for web events (older ⇒ error for the *entire request*). Offline/physical-store events: upload within **62 days**. |
| `event_id` | Unique per logical event; **dedup key** (see §12). |
| `user_data` | Object of customer information parameters (see §8). |
| `custom_data` | Object of event properties: `value`, `currency`, `contents`, `content_ids`, `content_type`, `content_name`, `content_category`, `num_items`, `search_string`, `order_id`, `predicted_ltv`, `delivery_category` (inside `contents[]`), etc. |
| `event_source_url` | **Required for web events.** The browser URL where the event happened. |
| `action_source` | **Required.** For a website integration always `website`. By sending you *agree the value is accurate to the best of your knowledge*. |
| `opt_out` | Boolean; skip ads processing. |
| `data_processing_options` (+ `_country`, `_state`) | LDU controls (see §16). |
| `referrer_url` | Referring page URL (optional, newer addition). |
| `customer_segmentation` | Optional audience hints (newer addition). |

**Requirement statement for web events** (quoted): *"Website events shared using the Conversions API require the `client_user_agent`, `action_source`, and `event_source_url` parameters, while non-web events require only `action_source`."* These parameters *"contribute to improving the quality of events used for ad delivery and may improve campaign performance."*

**Original event data parameters** (used for dedup diagnostics and merging): `event_name`, `event_time`, `order_id`, `event_id`. Sending `order_id` on Purchase events is recommended for e-commerce reconciliation.

---

## 8. Customer Information Parameters and Hashing Rules

Complete `user_data` parameter table (Parameters reference, updated Jun 30, 2026):

| Parameter | Field | Hashing |
|---|---|---|
| `em` | Email | **SHA-256 required** |
| `ph` | Phone number | **SHA-256 required** |
| `fn` | First name | **SHA-256 required** |
| `ln` | Last name | **SHA-256 required** |
| `ge` | Gender | **SHA-256 required** |
| `db` | Date of birth | **SHA-256 required** |
| `ct` | City | **SHA-256 required** |
| `st` | State | **SHA-256 required** |
| `zp` | Zip code | **SHA-256 required** |
| `country` | Country (2-letter ISO) | **SHA-256 required** |
| `external_id` | Your stable first-party ID for the person | **Hashing recommended** |
| `client_ip_address` | Client IP | **Do not hash** |
| `client_user_agent` | Client UA | **Do not hash** |
| `fbc` | Click ID (from `_fbc` / `fbclid`) | **Do not hash** |
| `fbp` | Browser ID (from `_fbp`) | **Do not hash** |
| `subscription_id` | Subscription ID | Do not hash |
| `fb_login_id` | Facebook Login ID | Do not hash |
| `lead_id` | Lead ID | Do not hash |
| `anon_id` | Install ID (app only) | Do not hash |
| `madid` | Mobile advertiser ID (app only) | Do not hash |
| `page_id`, `page_scoped_user_id`, `ctwa_clid`, `ig_account_id`, `ig_sid` | Messaging surfaces | Do not hash |

**Normalization before hashing** (to actually receive the match-credit, values must be):

1. **Trim** leading/trailing whitespace.
2. **Lowercase** the value.
3. Remove extra spaces.
4. **Emails:** trim leading/trailing AND remove all space characters; lowercase.
5. **Phone numbers:** remove symbols, letters and leading zeros; include the country code (digits only, e.g. `8801XXXXXXXXX` for Bangladesh). Multiple phone variants may be sent in the `ph` array.
6. **Country:** two-letter ISO 3166-1 alpha-2 code in lowercase, then hashed.
7. Then **SHA-256** the normalized string (hex output) — for values that are empty after normalization, omit the key entirely.

**Practical consequence for Bangladesh storefronts:** customer-entered phones like `+8801712345678`, `8801712345678`, `01712345678` must all collapse to `8801712345678` before hashing — the deployed `MetaHasher` implements exactly this normalization (including double-prefix collapse), and `country` is derived from the shipping address via an ISO map (`bangladesh` → `bd` → SHA-256).

---

## 9. fbp, fbc and Meta ClickID — Full Handling Guide

Source: *"ClickID and the fbp and fbc Parameters"*, updated **Jan 9, 2026**.

Meta recommends: *"always send `_fbc` and `_fbp` browser cookie values in the `fbc` and `fbp` event parameters, respectively, when available. These values are subject to change over multiple browser sessions, so we recommend refreshing a user's profile with the latest value whenever possible."*

### 9.1 ClickID retrieval (server-side, priority order)

1. **From the URL query parameter `fbclid`** — read it from the HTTP request query string server-side on first landing.
   - ⚠️ *"ClickID value is case sensitive — do not apply any modifications before using, such as lower or upper case."*
2. **From the `_fbc` cookie** — present when (a) Meta Pixel is installed (Pixel auto-stores ClickID into `_fbc`), or (b) you stored the formatted value yourself per best practice below.

### 9.2 Formatting `fbc`

When `_fbc` is absent, `fbc` **can and should still be sent** if `fbclid` is in the current page URL. Format:

```
fbc = fb.{subdomainIndex}.{creationTime}.{fbclid}
```

- `version` — always the prefix `fb`.
- `subdomainIndex` — which domain level the cookie is defined on: `com` = 0, `example.com` = 1, `www.example.com` = 2. **If generating server-side without saving an `_fbc` cookie, use `1`.**
- `creationTime` — UNIX epoch **milliseconds** when `_fbc` was stored; if no cookie, the timestamp when you **first observed/received** this `fbclid`.
- `{fbclid}` — the raw, case-preserved query value.

Example: `fb.1.1554763741205.IwAR2F4-dbP0l7Mn1IawQQGCINEz7PYXQvwjNwB_qa2ofrHyiLjcbCRxTDMgk`

### 9.3 Storing ClickID

It is *"highly recommended"* to set `_fbc` as an **HTTP response cookie with 90-day expiration** — but **only if**:
- `_fbc` doesn't exist and ClickID was retrieved from the URL's `fbclid`; **or**
- the URL `fbclid` differs from the one embedded in the existing `_fbc` (the `fbclid` is the substring after the last `.`).

Alternative: store the formatted ClickID server-side and always send the most recent value.

### 9.4 `fbp` (Browser ID)

When the Pixel runs with first-party cookies it auto-creates `_fbp`. Format:

```
fbp = fb.{subdomainIndex}.{creationTime}.{randomnumber}
```

`randomnumber` is generated by the Pixel SDK to make every `_fbp` unique (e.g. `fb.1.1596403881668.1116446470`). Server events simply **pass the cookie value through raw** — never generate or modify `fbp` yourself when a real pixel-created cookie exists.

> Note for 2026: Meta's new **Parameter Builder** library (v26.0 highlight) automates exactly this fbc/fbp formation plus PII normalization, with a documented appendix suffix on generated values. `Modules/MetaPixel` implements the equivalent logic natively (construct-fbc-from-fbclid with server subdomainIndex 1, first-observation timestamp, raw `_fbp` passthrough).

---

## 10. external_id and Advanced Matching

- `external_id` is **any stable, first-party identifier** you already assign to a person (customer ID, CRM ID). Hashing is **recommended** (SHA-256), and the same identifier must be usable on both browser and server events.
- Purpose: it is the second deduplication key (with `event_name` — see §12.2) **and** a powerful cross-device/cross-session match signal for attribution and custom audiences.
- **Advanced Matching** (Pixel feature) additionally lets the browser pixel pick up PII the visitor typed in-forms; on the server side the equivalent is populating `user_data` fully. For logged-in e-commerce flows, `external_id` = customer ID + hashed email + hashed phone is the standard triple.
- Best practice: use **one `external_id` scheme across all events** (browse, cart, checkout, purchase, registration) — the deployed module uses the authenticated customer ID, falling back to a stable per-session identifier for guests.

---

## 11. Standard E-Commerce Events and Object Properties

Source: Meta Pixel Reference (page fetched current, dated Jul 16, 2024 stamp but live 2026 content) + Conversions API parameter docs. When Pixel + CAPI run together, Meta recommends including the **`eventID`** as the **fourth argument** of the `fbq('track')` call.

### 11.1 Event surface for an e-commerce site

| Event | Fires when | Object properties (Optional unless noted) |
|---|---|---|
| `PageView` | Every page load (pixel base code) | — |
| `ViewContent` | Product-details page visit | `content_ids`, `content_type`, `contents`, `currency`, `value` — **Advantage+ catalog ads require `contents` or `content_ids`** |
| `Search` | On-site search | `content_ids`, `content_type`, `contents`, `currency`, `search_string`, `value` — catalog ads require `contents` or `content_ids` |
| `AddToCart` | Product added to cart | `content_ids`, `content_type`, `contents`, `currency`, `value` — **catalog ads require `contents`** |
| `AddToWishlist` | Product wishlisted | `content_ids`, `contents`, `currency`, `value` |
| `InitiateCheckout` | Entering checkout flow | `content_ids`, `contents`, `currency`, `num_items`, `value` |
| `AddPaymentInfo` | Payment info saved in checkout | `content_ids`, `contents`, `currency`, `value` |
| `Purchase` | Purchase completed / confirmation page | `content_ids`, `content_type`, `contents`, `currency`, `num_items`, `value` — **`currency` and `value` REQUIRED**; catalog ads require `contents` or `content_ids` |
| `CompleteRegistration` | Registration completed | `currency`, `value` |
| `Lead` | Sign-up completed | `currency`, `value` |
| Others available | `Contact`, `CustomizeProduct`, `Donate`, `FindLocation`, `Schedule`, `StartTrial`, `SubmitApplication`, `Subscribe` | per-reference |

### 11.2 Object property reference (`custom_data`)

| Property | Type | Meaning |
|---|---|---|
| `content_ids` | array of strings/ints | Product IDs (SKUs) associated with the event, e.g. `['ABC123','XYZ789']` |
| `content_name` | string | Product/page name |
| `content_category` | string | Category of the page/product |
| `content_type` | string | `product` **or** `product_group` — must match what the IDs refer to; if omitted, *"Meta will match the event to every item that has the same ID, independent of its type"* |
| `contents` | array of objects | `id` + `quantity` required; may carry `delivery_category` (e.g. `home_delivery`) and EAN |
| `currency` | string | ISO currency of `value` (e.g. `BDT`) |
| `value` | number | Monetary value of the event |
| `num_items` | integer | Item count, used with `InitiateCheckout` |
| `search_string` | string | The search query, used with `Search` |
| `order_id` | string | (original event data parameter) your order reference on `Purchase` |
| `predicted_ltv` | number | Predicted lifetime value (subscription events) |
| `delivery_category` | string | Inside `contents[]`: `home_delivery`, etc. |

**Mapping completeness rule** ("no wrong mapping"): every standard event above maps 1:1 to the storefront surface — product page ⇒ `ViewContent`; search results ⇒ `Search`; add-to-cart ajax response ⇒ `AddToCart`; wishlist ⇒ `AddToWishlist`; checkout-details step ⇒ `InitiateCheckout`; payment step ⇒ `AddPaymentInfo`; order creation ⇒ `Purchase` (server-authoritative, with `order_id`, full `contents`, BDT currency, numeric value); signup ⇒ `CompleteRegistration`; login has no Meta standard event ⇒ custom `Login` event.

---

## 12. Event Deduplication (Browser + Server)

Source: *"Handling Duplicate Pixel and Conversions API Events"*, updated **Jun 28, 2026**.

### 12.1 Method 1 — event_id + event_name (**RECOMMENDED**)

- Add `event_id` to **both** the browser event (Pixel 4th argument `eventID`) and the server event (`event_id`).
- Meta deduplicates when **both** match: Pixel `eventID` == CAPI `event_id` **and** Pixel event name == CAPI `event_name`, sent to the **same Pixel ID**.
- Browser call shape:

  ```js
  fbq('track', 'Purchase', {value: 12, currency: 'USD'}, {eventID: 'EVENT_ID'});
  // or for a specific pixel:
  fbq('trackSingle', 'SPECIFIC_PIXEL_ID', 'Purchase', {…}, {eventID: 'EVENT_ID'});
  // image pixel form:
  // https://www.facebook.com/tr?id=PIXEL_ID&ev=Purchase&eid=EVENT_ID
  ```
- If no parameters beyond identity are needed: `fbq('track', 'Lead', {}, {eventID: 'EVENT_ID'});`
- Window: deduplication works **within 48 hours** of the first received event with a given `event_id`.
- Preference rule: *"If server and browser events do not differ meaningfully in their content, we generally prefer the event that is received first."*

### 12.2 Method 2 — fbp and/or external_id (secondary)

- Send `event_name` + `fbp` and/or `external_id` consistently on **both** ends; Meta compares the combinations and drops server duplicates of previously received browser events.
- **Limitations (documented verbatim):**
  - Works essentially only for **browser-first → server-second** ordering; a server event is **not** discarded because a later browser duplicate arrives.
  - Does **not** dedupe within a single source (two identical browser events or two identical server events are both kept).
- Because of those limitations, event_id+event_name is the primary method and fbp/external_id is a supplementary signal.

### 12.3 What happens without dedup

Duplicate events inflate conversions and corrupt optimization. Conversely, advertisers who never send the same event twice through both channels *"do not need to set up deduplication for those events."*

### 12.4 Verification

Meta provides a **"Verifying Your Setup"** flow: Events Manager → per-event view shows **Deduplication Rate** (share of server events successfully merged with a browser twin). Target: as close to the overlap your traffic actually has as possible; 0% on events that should be dual-sourced signals a broken `event_id` join.

**Implementation contract adopted on Shopika:** every logical user action generates ONE deterministic `event_id` (e.g. `purchase-{orderId}`, `vc-{productId}-{requestId}`, `atc-{productId}-{cartId}`), emitted **identically** into the browser pixel call and the server batch → guaranteed join regardless of which channel Meta receives first.

---

## 13. Event Timing, Batching and Retry Semantics

- `event_time` = actual transaction time, **Unix seconds**; may predate the send to enable batching.
- **7-day rule:** any event older than 7 days ⇒ **error for the entire request, nothing processed**. (Physical-store/offline events: 62 days.)
- **Batch size:** up to **1,000 events** per `data` array.
- **Timeliness:** *"for optimal performance, we recommend you send events as soon as they occur and ideally within an hour of the event occurring."*
- **Retry safety:** send `event_id` on every event. *"If any event in a batch is invalid, the request returns an error — but valid events with an event_id are still accepted. Fix the invalid events and resend the whole batch; the events we already stored are dropped as duplicates, with no double-counting."*
- The deployed module adds a **48-hour server-side self-guard log** (`meta_capi_logs`) mirroring Meta's dedup window so repeated flushes of the same logical event can never leave the server twice.

---

## 14. Testing — the Test Events Tool and test_event_code

- Tool location: **Events Manager → Data Sources → [your Pixel] → Test Events**. It generates a **test ID**.
- Usage: include `"test_event_code": "<TEST_ID>"` (top level, sibling of `data`) in server payloads; browser-side, use `fbq('init', 'PIXEL_ID', {}, {testEventCode: 'TEST_ID'})`.
- **Critical production warning (quoted):** *"The test_event_code field should be used only for testing. You need to remove it when sending your production payload."* AND *"Events sent with test_event_code are not dropped. They flow into Events Manager and are used for targeting and ads measurement purposes."* — a forgotten test code does **not** protect production data.
- Payload Helper tool generates correctly-shaped test payloads (and SDK code via "Get Code").
- Verification flow: send event → watch it appear in Test Events with parameter-level diagnostics (missing `user_data`, hashing errors, `action_source` problems are surfaced per event).

---

## 15. Verification and Event Quality (EMQ, Dataset Quality API)

1. **Events Manager → Data Sources → Pixel → Overview:** raw / matched / attributed event counts; **Connection Method** column shows the channel (browser vs server) each event came in on; drill into any event for parameter diagnostics. Events verifiable within **~20 minutes** of starting.
2. **Event Match Quality (EMQ):** per-event 0–10 score driven by how completely `user_data` is filled (email, phone, name, city/state/zip/country, external_id, fbp/fbc, IP+UA). Each added, correctly-hashed parameter raises the score. The Test Events tool displays EMQ live.
3. **Dataset Quality API** (Graph API): programmatic read of EMQ and related quality signals for server-sourced datasets.
4. **Ads Dataset Event Coverage** (Graph API Reference v26.0): *"Event coverage is the 7-day average percent of pixel events that are covered by Conversions API events"* — the KPI of how much browser-only traffic your server channel recovers.
5. **Deduplication verification:** per-event dedup rate (see §12.4).

**E-commerce quality recipe (all items implemented by the module):** always `client_user_agent` + IP; `_fbp` everywhere; `_fbc`/constructed `fbc` on ad landings; `external_id` for logged-in users; full hashed PII at checkout/purchase; `order_id` + full `contents` on Purchase; BDT currency + numeric value.

---

## 16. Data Processing Options / Limited Data Use

Per-event fields inside each event of `data`:

- **Explicitly NOT enabling LDU:** `"data_processing_options": []` (or omit the fields).
- **LDU with Meta geolocation:** `"data_processing_options": ["LDU"], "data_processing_options_country": 0, "data_processing_options_state": 0`.
- **LDU with manual location** (example — California): `"data_processing_options": ["LDU"], "data_processing_options_country": 1, "data_processing_options_state": 1000`.

Bangladesh-only storefront: not required for BD regulations, but the module config exposes the fields for future compliance needs.

---

## 17. API Limits and Rate Limiting

- *"There is no specific rate limit for the Conversions API. Conversions API calls are counted as Marketing API calls."*
- Marketing API has its **own rate-limiting logic** and is **excluded from Graph API throttling** — CAPI traffic never consumes Graph API budget.
- The only hard cap: **1,000 events per request** (`data` array).
- Business SDK advanced features designed for CAPI users: **Asynchronous Requests**, **Concurrent Batching**, **HTTP Service Interface** (PHP ≥ 7.2, Node ≥ 7.6, Java ≥ 8, Python ≥ 2.7, Ruby ≥ 2). PHP 5 SDK support deprecated since January 2019.

---

## 18. Conversions API Gateway and Signals Gateway

**Conversions API Gateway** — self-hosted relay between your servers and Meta:

- **Hosting options (2026):** AWS App Runner, AWS EKS (multi-account host onboarding), GCP; Docker images with **automatic and manual update** channels; custom domain support; DNS setup guide; cost monitoring; scaling control.
- **Capabilities:** event upload/relay, **system health information**, multi-account management (agencies), domain allow/block lists, data routing per account, host/user management, SMTP setup for alerts, uninstall guide.
- **Enhancements documented:** *Enhance Events with Advanced Matching Data*, *Include Facebook Login Data in the Conversions API* (gateway-side payload enrichment).
- **When to choose it:** many source properties/accounts, centralized token custody, infrastructure teams wanting retries/out of app process. **When NOT:** a single-storefront Laravel app — direct CAPI calls are simpler, fewer moving parts, identical payload contract.

**Signals Gateway** — Meta's newer hosted multi-channel pipeline product (web pixel onboarding, GTM integration, SDK onboarding, BigQuery destination, event filtering, message center, diagnostics). Directionally the managed successor to self-hosted gateways; not required for this project.

---

## 19. Deprecated or Discouraged Approaches — Do Not Use

| Approach | Status (September 2026) |
|---|---|
| **App Events API** (standalone) for new integrations | *"No longer recommended"* — use Conversions API (single endpoint covers app events) |
| **Offline Conversions API** (standalone) for new integrations | *"No longer recommended"* — use Conversions API for offline events |
| **Commerce Order Management API** | **Deprecated in v26.0** (47 endpoints blocked on v26.0+ since Jul 29, 2026; all remaining versions Oct 27, 2026) with the sunset of on-Facebook checkout |
| **PHP 5 Business SDK** | Deprecated since January 2019 |
| **Unversioned Graph API calls** | Never supported for production; always pin e.g. `/v26.0/` |
| **`test_event_code` left in production payloads** | Forbidden by docs; test events are **not dropped** — they pollute production measurement |
| **Hashing `fbp`/`fbc`/IP/UA** | Wrong — these must be sent raw; hashing destroys match signal |
| **Modifying `fbclid` case / whitespace** | ClickID is case-sensitive; any modification breaks attribution |
| **Relying on fbp/external_id dedup alone** | Documented limitations (browser-first only, single-source no-op); event_id+event_name is the recommended method |

---

## 20. Consolidated Best-Practices Checklist for E-Commerce

**Architecture**
- [x] Run Pixel (browser) + CAPI (server) on the **same Dataset/Pixel ID** — redundant setup.
- [x] Pin **v26.0**; expect ≥ 2-year support on that version for CAPI.
- [x] Use a **system-user access token** stored server-side only; never expose the CAPI token to the browser.

**Events**
- [x] Fire the full e-commerce surface: PageView, ViewContent, Search, AddToCart, AddToWishlist, InitiateCheckout, AddPaymentInfo, Purchase, CompleteRegistration (+ custom events where no standard fits, e.g. Login).
- [x] Purchase carries **`currency` + `value` (required)**, `contents`/`content_ids` (+`content_type`), `num_items`, and `order_id`.
- [x] Catalog ads readiness: `contents` **or** `content_ids` on ViewContent/AddToCart/Search too.
- [x] Never double-fire one logical event within a channel; each logical event = one deterministic `event_id`.

**Deduplication**
- [x] `event_id` + `event_name` identical across browser and server (Pixel 4th arg `eventID`).
- [x] Understand the **48-hour** dedup window and "first received wins" preference.
- [x] Use `fbp`/`external_id` as supplementary matching, not as the primary dedup method.

**user_data**
- [x] SHA-256 + lowercase + trim for em/ph/fn/ln/ge/db/ct/st/zp/country; ISO-2 lowercase country; phone with country code, digits only (BD normalization: strip `+`/`0` prefixes, ensure `880…`).
- [x] `external_id` (hashed) for logged-in users; consistent across all events.
- [x] `client_ip_address`, `client_user_agent`, `fbp`, `fbc` raw.
- [x] Multiple values allowed in `em[]` / `ph[]` arrays to raise match rate.

**ClickID / Cookies**
- [x] Capture `fbclid` server-side on landing; construct `fbc = fb.1.{observed-ms}.{fbclid}` when `_fbc` absent.
- [x] Set `_fbc` cookie (90-day expiry) only when absent or when `fbclid` changed.
- [x] Pass `_fbp` through untouched.

**Operations**
- [x] Send events ASAP (≤ 1 hour ideal); batch ≤ 1,000; `event_time` accurate (never > 7 days old).
- [x] Retry-safe batching: on batch error, fix invalid events and resend; valid `event_id`-tagged events survive re-sends.
- [x] Validate with **Test Events** before production; **remove `test_event_code`** afterwards.
- [x] Monitor EMQ, Event Coverage (7-day % of pixel events covered by CAPI), and dedup rate in Events Manager / Dataset Quality API.
- [x] One source of truth per pixel: disable/merge any legacy basic-pixel snippet firing PageView-only duplicates (overlap detection).

---

## Appendix A — How Modules/MetaPixel Implements This Research

The research above was implemented on **2026-08-31** as `Modules/MetaPixel` (nwidart v10 module, 20 files, **zero core-file modifications**). Cross-reference for future maintenance:

| Research requirement | Module implementation |
|---|---|
| v26.0 endpoint, 8 s timeout | `config/config.php` (`api_base`, `default_api_version` = v26.0, released 2026-07-29), `MetaCapiService` batch POST to `https://graph.facebook.com/v26.0/{DATASET_ID}/events?access_token=…` |
| Token-based setup (Dataset ID + Access Token) | Admin → Meta_Pixel_CAPI → Settings (`addon_settings` row via `MetaConfigService`; `Setting` model `HasUuid` firstOrNew/save — raw `updateOrInsert` breaks on the char(36) UUID PK) |
| Redundant setup + event_id dedup | `MetaTrackingMiddleware` (web group) computes **shared deterministic event IDs** (`pv-`, `vc-`, `atc-`, `wl-`, `ic-`, `api-`, `purchase-`, `registration-`, `login-` prefixes) and queues the identical ID into both the browser pixel call (`eventID` 4th argument) and the server batch |
| SHA-256 hashing + BD phone normalization | `MetaHasher` (lowercase/trim, phone → `880XXXXXXXXXX` digits-only incl. double-prefix collapse, country name → ISO-2 via map → hash); verified against expected digests |
| Full `user_data` | `MetaUserDataBuilder` — hashed em/ph/fn/ln/ct/st/zp/country + hashed `external_id` (customer id) + raw IP/UA/`_fbp`/`_fbc`; guests fall back to session-stable `external_id` |
| fbc construction from `fbclid` (Jan 9, 2026 spec) | `MetaEventCollector` — raw `_fbp` passthrough; `_fbc` cookie preferred; else construct `fb.1.{firstObservedMs}.{fbclid}` from the landing URL (server subdomainIndex = 1) |
| Web-event required trio | `event_source_url` = current URL, `action_source` = `website`, `client_user_agent` = original request UA on every server event |
| Full e-commerce event surface + per-event toggles | config `events` map: pageview, view_content, search, add_to_cart, add_to_wishlist, initiate_checkout, add_payment_info, purchase, complete_registration, login (custom) |
| Server-authoritative Purchase with complete data | `OrderObserver::created` → Purchase with `order_id`, `contents[]` (id/quantity), `content_ids`, `content_type=product`, numeric `value`, `currency` auto = site currency (BDT) — reuses the verified `Order->orderDetails->productAllStatus` data chain (same chain proven in Task 17) |
| CompleteRegistration / Login | `CustomerObserver` (registration), `CustomerLoginListener` (customer guard) |
| test_event_code testing-only | Settings field + "Send test event" button; kept out of production flushes |
| Retry/self-dedup guard | 48 h self-guard against `meta_capi_logs` mirroring Meta's dedup window |
| Single-source-of-truth / overlap | Module detects the legacy core basic pixel entry (PageView-only) and suppresses its own duplicate browser PageView when the same pixel ID is present |
| Race safety (theme consumes `order_success_ids`) | Session snapshot `metapixel_purchase_ids` with 30-min TTL |
| Admin visibility | Routes `admin/meta-pixel/{settings,logs,verify}`; logs page shows last requests + responses |

Rollback / module package: `scripts/task18/MetaPixel.tar.gz`; deployed file md5s recorded in the Task 18 worklog entry.

---

## Appendix B — Source Documents and Fetch Dates

All pages fetched from `developers.facebook.com` on **2026-08-31** (raw dumps archived in `scripts/task18/doc_*.json`, extracted text in `scripts/task18/doc_txt/`):

| # | Document | URL path | Page "Updated" stamp |
|---|---|---|---|
| 1 | Conversions API — Using the API | `/documentation/ads-commerce/conversions-api/using-the-api` | **Jul 17, 2026** |
| 2 | Handling Duplicate Pixel and Conversions API Events | `/documentation/ads-commerce/conversions-api/…/deduplication` | **Jun 28, 2026** |
| 3 | Parameters (CAPI) | `/documentation/ads-commerce/conversions-api/parameters` | **Jun 30, 2026** |
| 4 | ClickID and the fbp and fbc Parameters | `/documentation/ads-commerce/conversions-api/…/fbp-fbc` | **Jan 9, 2026** |
| 5 | Meta Pixel — Reference (standard events + object properties) | `/documentation/meta-pixel/reference` | page stamp Jul 16, 2024, live 2026 content |
| 6 | Graph API Changelog / Versions / v26.0 | `/docs/graph-api/changelog`, `/docs/graph-api/changelog/versions` | v26.0 released **Jul 29, 2026** |
| 7 | Blog — Introducing Graph API v26.0 and Marketing API v26.0 | `/blog/post/2026/07/29/introducing-graph-api-v26-and-marketing-api-v26` | Jul 29, 2026 |
| 8 | Blog — Introducing Graph API v25.0 | `/blog/post/2026/02/18/introducing-graph-api-v25-and-marketing-api-v25` | Feb 18, 2026 |
| 9 | Graph API Reference v26.0 — Ads Dataset Event Coverage | `/docs/marketing-api/reference/ads-dataset-event-coverage` | v26.0 |
| 10 | Dataset Quality API | `/documentation/ads-commerce/conversions-api/dataset-quality-api` | v26.0 |

*End of research report 
