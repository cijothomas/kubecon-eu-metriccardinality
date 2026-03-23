# KubeCon EU Observability Day - Metric Cardinality Capping

Lightning talk for [KubeCon EU Co-located Events 2026](https://colocatedeventseu2026.sched.com/) — Observability Day.

## Abstract

High-cardinality metrics just got safer with OpenTelemetry's cardinality
capping—SDKs won't get OOM-killed anymore—but there's a dangerous catch. When
cardinality limits are hit, new attribute sets are folded into a special overflow
attribute. The problem? Metric backends like Prometheus don't treat this overflow
attribute as special—it's just another label—so queries can silently return
incorrect results.

Imagine querying error counts for `/endpoint1`. Once the cap is hit, new errors
for that endpoint get folded into overflow. A query like
`errors WHERE url="/endpoint1"` might underreport errors—masking real issues.

This lightning talk explains how OTel's cardinality capping works, how it can
create hidden observability traps, how to detect when your queries may be
incorrect, and strategies to avoid falling into this subtle but impactful
pitfall.

## Talk Outline (Lightning Talk — 10 min)

### 1. What is Cardinality? (1 min)

- Introduce the running example: Counter `http.server.requests` with attributes
  `url.path`, `http.method`, `http.status_code`.
- Each unique combination of attribute values = one time series.
  - e.g., `{url=/home, method=GET, status=200}` is one series,
    `{url=/checkout, method=POST, status=200}` is another.
- Cardinality = the number of unique attribute combinations.
  - If you have 100 URL paths × 3 methods × 5 status codes = **1,500 series**.
  - But if `url.path` includes user-specific IDs like `/api/order/{uuid}`
    → cardinality explodes to millions.
- **Impact on backends** — Storage, indexing, and query cost scale with the
  number of unique time series. Backends slow down or reject ingestion.
- **Impact on SDKs** — Each unique attribute combination is a separate
  aggregation in memory. High cardinality → unbounded memory → OOM kills.

### 2. Why OTel SDKs Added Cardinality Capping (1 min)

- Before capping: a single bad metric (e.g., `request.id` as an attribute) could
  OOM-kill the entire application.
- SDKs needed a safety valve — protect the app first, preserve data second.
- The spec is now **stable** and support is rolling out across all SDKs.

### 3. How It Works — The Overflow Attribute (1 min)

- SDK enforces a cap on the number of unique attribute sets **per metric per
  collection cycle** (default: 2000).
- Once the cap is reached, any *new* attribute combination is aggregated into a
  single overflow series with a special attribute:
  `otel.metric.overflow = true`.
- All other attributes on the overflow series are dropped — it's a catch-all
  bucket.

### 4. The Main Problem — Metrics That "Lie" (1.5 min)

- The high-cardinality attribute is the obvious victim — its values get folded
  into overflow, so queries for specific values underreport.
- **Running example throughout the talk:**
  - Counter: `http.server.requests`
  - Attributes: `url.path`, `http.method`, `http.status_code`
  - `url.path` is the high-cardinality attribute (unique per endpoint/resource)
  - Cardinality limit = 3 (low for demo purposes; default is 2000)
- After 3 unique attribute combos fill the cap, the 4th request
  `{url=/api/order/abc123, method=POST, status=500}` with count 15 → **overflow**.
  - "How many requests to `/api/order/abc123`?" → **0** (should be **15**).
  - Your error dashboard for that endpoint shows nothing.

### 5. The Subtle Trap — OTHER Queries Lie Too! (1.5 min)

- It's not just the high-cardinality attribute (`url.path`) that's affected.
- When overflow fires, **all attributes** on the new combination are lost — not
  just the high-cardinality one.
- "How many **500 errors**?" → **0** (should be **15**). Your error alert
  doesn't fire. SLO dashboard shows green while users get errors.
- "How many **POST requests**?" → **130** (should be **145**).
- **Key insight:** One high-cardinality attribute poisons queries on *every*
  attribute of that metric.
- Only the **total** request count (SUM across everything including overflow)
  remains correct — **245** = **245**.

### 6. What Attributes Are NOT Affected? (0.5 min)

- **Resource attributes** — not part of metric aggregation keys, unaffected.
- **Scope attributes** — same, not part of the cardinality calculation.
- Only **metric data point attributes** (the ones that define the time series)
  are subject to capping.

### 7. Solutions & Mitigation (1.5 min)

- **Obvious:** Avoid high-cardinality attributes (use Views to drop/reduce
  dimensions before aggregation).
- **Temporality matters:**
  - **Cumulative** — once a series is created, it persists for the lifetime of
    the SDK. High cardinality accumulates forever.
  - **Delta** — series can be reclaimed when not seen in a collection cycle.
    Cardinality pressure is lower because stale combinations are freed.
- **SDK reclaiming unused series** — In delta temporality, SDKs can reclaim
  slots, making room for new attribute combinations and reducing overflow
  frequency.
- **Be very clear:** OTel SDK cardinality capping is at the **SDK level**. It
  limits what the SDK holds in memory. The backend may see fewer series than
  reality — it's not the backend doing the capping.

### 8. How to Detect Overflow — Aspire Dashboard Demo (1 min)

- .NET Aspire dashboard (language-neutral — works with any OTel SDK via OTLP)
  can detect the `otel.metric.overflow` attribute.
- When overflow is present, the UI **warns the user** that query results may be
  incomplete.
- Show the Aspire dashboard with the overflow warning in action.

### 9. Prometheus Does NOT Treat Overflow Specially (0.5 min)

- Prometheus ingests `otel_metric_overflow="true"` as just another label.
- No special handling, no warnings — queries silently return wrong results.
- PromQL has no built-in way to flag affected queries.
- This is the gap: the backend doesn't know the data is incomplete.

### 10. Conclusions & Takeaways (0.5 min)

- Cardinality capping is a **necessary safety net** — it prevents OOM.
- But it introduces a **silent correctness problem** — metrics can lie.
- One bad attribute poisons queries on **all** attributes of that metric.
- **Know your tools:** Use dashboards that understand overflow (Aspire).
- **Design your metrics carefully:** Drop high-cardinality attributes via Views
  *before* they cause capping.
- **Choose temporality wisely:** Delta enables reclaiming; cumulative
  accumulates pressure.
- The overflow attribute exists for a reason — **monitor it, alert on it, and
  don't ignore it.**

---

## Slide Example — Web App (used throughout the talk)

### Setup

- Counter: `http.server.requests`
- Attributes: `url.path`, `http.method`, `http.status_code`
- **Cardinality limit: 3** (low for demo; default is 2000)
- `url.path` is the high-cardinality attribute

### What happens during one collection cycle:

| # | Request | Attributes | Tracked? |
|---|---------|------------|----------|
| 1 | 100 GET /home returning 200       | `{url=/home, method=GET, status=200}` | Yes (1st unique combo) |
| 2 | 50 POST /api/users returning 201   | `{url=/api/users, method=POST, status=201}` | Yes (2nd unique combo) |
| 3 | 80 POST /checkout returning 200    | `{url=/checkout, method=POST, status=200}` | Yes (3rd — **limit reached**) |
| 4 | 15 POST /api/order/abc123 returning 500 | `{url=/api/order/abc123, method=POST, status=500}` | **No — overflow!** |

### What the SDK exports:

```
{url=/home,      method=GET,  status=200}  → count: 100
{url=/api/users, method=POST, status=201}  → count: 50
{url=/checkout,  method=POST, status=200}  → count: 80
{otel.metric.overflow=true}                → count: 15   ← catch-all bucket
```

### Queries that LIE:

| Query | Expected | Actual | Problem |
|-------|----------|--------|---------|
| "How many **500 errors**?" | **15** | **0** | Error alert doesn't fire! SLO dashboard shows green. |
| "How many **POST requests**?" | **145** | **130** | 15 POSTs lost to overflow |
| "How many requests to **/api/order/abc123**?" | **15** | **0** | Entire endpoint invisible |

### Query that is CORRECT:

| Query | Expected | Actual | Why? |
|-------|----------|--------|------|
| "**Total** requests?" | **245** | **245** | Sum across all series including overflow is always accurate |

### Key takeaway for the slide:

> **Every measurement is accounted for** — either with its original attributes
> or in the overflow bucket. Totals are always correct. But any query filtering
> on specific attributes can silently underreport — and it's not just the
> high-cardinality attribute (`url.path`) that's affected. Queries on
> `http.method` and `http.status_code` are wrong too! Your 500-error alert
> stays silent while users are seeing failures.

---

## Pseudocode for Slides

```
// Creating a metric
counter = meter.create_counter("http.server.requests")

// Recording — each unique {url.path, http.method, http.status_code} is a series
counter.add(100, {url: "/home",       method: "GET",  status: 200})
counter.add(50,  {url: "/api/users",  method: "POST", status: 201})
counter.add(80,  {url: "/checkout",   method: "POST", status: 200})

// Cardinality limit = 3, already reached!
// This next measurement overflows:
counter.add(15,  {url: "/api/order/abc123", method: "POST", status: 500})
//  → folded into {otel.metric.overflow: true}, count += 15
//  → original attributes (url, method, status) are ALL LOST
//  → your 500-error query returns 0!
```

## TODO

- [ ] Create slides (language-neutral, pseudocode examples)
- [ ] Build example scenario showing overflow in action
- [ ] Screenshot/recording of Aspire dashboard detecting overflow
- [ ] Screenshot/recording of Prometheus NOT detecting overflow
- [ ] Prepare the "other queries lie too" example with concrete numbers
- [ ] Rehearse within 10 min
