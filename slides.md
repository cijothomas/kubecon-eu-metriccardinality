---
theme: default
title: "Metrics That Lie: OTel's Cardinality Capping Trap"
info: |
  KubeCon EU 2026 — Observability Day Lightning Talk
  by Cijo Thomas
class: text-center
drawings:
  persist: false
transition: slide-left
mdc: true
---
layout: cover
class: text-center
background: /title-bg.png
style: "background-size: cover; background-position: center;"
---

<div style="background: rgba(0, 0, 0, 0.6); padding: 2rem 3rem; border-radius: 12px; display: inline-block;">

# Metrics That Lie

## OTel's Cardinality Capping Trap

<br>

**Cijo Thomas**

</div>

---

# Have You Ever Trusted a Dashboard That Was Wrong?

<br>

Your error rate dashboard says **zero 500 errors**. Alerts are silent.

<br>

But users are reporting failures.

<br>

You use OTel to report metrics. It always worked.

**So where did the errors go?**

<!--
Let me start with a question. Have you ever trusted a dashboard that turned out to be wrong? Your error rate says zero. Alerts are silent. But users are telling you something is broken. You check the metrics — nothing. Your app uses OTel. Metrics always worked. So where did the errors go? This talk is about a feature in OpenTelemetry that can make this happen — silently, without any warning. Let me show you how.
-->

---

# Our Running Example

A simple web app counter:

```
counter: http.server.requests
attributes: url.path, http.method, http.status_code
```

Each unique combination of attribute values = **one time series**

| url.path | http.method | http.status_code | → series |
|----------|-------------|------------------|----------|
| /home | GET | 200 | Series #1 |
| /checkout | POST | 200 | Series #2 |
| /api/users | POST | 201 | Series #3 |
| ... | ... | ... | ... |

<!--
Let me set up the example we'll use throughout this talk. A simple counter tracking HTTP requests with three attributes: URL path, method, and status code. Each unique combination becomes a separate time series.
-->

---

# What is Cardinality?

**Cardinality** = number of unique attribute combinations

<br>

100 URL paths × 3 methods × 5 status codes = **1,500 series** ✅

<br>

That's manageable. But what if some attribute has **high cardinality**?

<!--
Cardinality is just the number of unique combinations. With 100 paths, 3 methods, and 5 status codes, that's 1,500 series. Totally manageable. But what happens when one of those attributes has unbounded values?
-->

---

# Cardinality Explosion

What if `url.path` includes user-specific IDs?

<br>

```
/api/order/abc123
/api/order/def456
/api/order/xyz789
/api/order/...     ← hundreds of thousands of unique values
```

<br>

Hundreds of thousands of paths × 3 methods × 5 status codes = **unbounded series** 💥

<v-click>

### Impact on the SDK

- Each unique combo is a separate aggregation **in memory**
- High cardinality → high memory → **OOM kill** 💀
- One bad attribute can bring down your entire application

</v-click>

<!--
But what happens when url.path isn't just a handful of known routes? If it includes user-specific IDs — like order IDs or session tokens — suddenly you have hundreds of thousands of unique values. That means a huge number of series, each held as a separate aggregation in memory by the SDK. This is how a single metric can OOM-kill your application.
-->

---

# Even "Safe" Attributes Can Explode

Your attributes look low-cardinality: known paths, standard methods, finite status codes.

<br>

But what if an attacker sends requests with **garbage HTTP methods**?

```
curl -X FOO  https://your-app.com/
curl -X BAR  https://your-app.com/
curl -X AAAA https://your-app.com/
...thousands of unique methods
```

<v-click>

Server returns 405 — but the metric still records it:

`{url=/, method=FOO, status=405}` → new series!

**Depending on your app, even seemingly safe attributes can trigger an explosion.**

</v-click>

<!--
And it's not just obviously high-cardinality attributes. Even attributes that look safe can explode. If an attacker sends requests with arbitrary HTTP methods — FOO, BAR, random strings — your server returns 405, but the metric still records it with whatever method the attacker sent. Each unique method is a new series in memory. Depending on your app setup, even seemingly safe attributes can trigger a cardinality explosion.
-->

---

# What Happens If You Run This?

<br>

```
counter = meter.create_counter("http.server.requests")

for i in 1 to 1_000_000_000:
    counter.add(1, { request_id: new_guid(), method: "GET", status: 200 })
```

<br>

<v-click>

### Every iteration creates a new unique attribute combination 💥

- 1 billion iterations = **1 billion series in memory**
- SDK memory grows without bound
- Eventually: **OOM kill** — your app is dead

</v-click>

<!--
Let me ask you — what happens if you run this code? It's just a simple loop with a counter. But notice the attribute: id is a new GUID every iteration. That means every single iteration creates a brand new unique attribute combination that the SDK has to hold in memory. A billion iterations means a billion series. Your app will be OOM-killed long before it finishes. This is obviously an extreme example, but it shows how easy it is to kill your application with a single bad attribute. In reality, less obvious things like user IDs or full URL paths with query strings do the same thing, just more slowly.
-->

---

# OTel's Safety Net — Cardinality Capping

<br>

To protect applications from this, OTel SDKs added **cardinality capping**.

Protect the app first, preserve data second.

<br>

### Status

- **Stable** specification
- Implemented in: **Go, Java, JS, C++, .NET, Rust**
- Expected to land in remaining SDKs in future

<!--
To protect applications from this, OpenTelemetry SDKs added cardinality capping as a safety feature. The priority is clear: protect the application first, preserve metric data second. The spec is now stable, and Go, Java, JavaScript, C++, .NET, and Rust already have it implemented.
-->

---

# How It Works — Cardinality Limit = 3

SDK caps unique attribute sets at **N per metric per collection cycle** (default: 2,000)

For this example: **limit = 3**

<br>

<v-click>

| # | Request | Count | Attributes | Tracked? |
|---|---------|-------|------------|----------|
| 1 | GET /home → 200 | ×100 | `{url=/home, method=GET, status=200}` | ✅ 1st combo |
| 2 | POST /api/users → 201 | ×50 | `{url=/api/users, method=POST, status=201}` | ✅ 2nd combo |
| 3 | POST /checkout → 200 | ×80 | `{url=/checkout, method=POST, status=200}` | ✅ 3rd — **limit!** |
| 4 | POST /api/orders → 500 | ×15 | `{url=/api/orders, method=POST, status=500}` | ❌ **Overflow** |
| 5 | GET /api/status → 200 | ×20 | `{url=/api/status, method=GET, status=200}` | ❌ **Overflow** |
| 6 | GET /home → 200 | ×30 | `{url=/home, method=GET, status=200}` | ✅ **Already tracked** |

</v-click>

<!--
Here's how cardinality capping works. The SDK limits the number of unique attribute combinations per metric per collection cycle. The default is 2,000, but I'm using 3 to keep it simple. We have five types of requests. The first three each get their own series. But the fourth and fifth are new combinations — they overflow. One is a POST returning 500, the other is a GET returning 200. Two very different requests, but they both end up in the same overflow bucket.
-->

---

# What the SDK Exports

The 6 request types become **4 exported series**:

<br>

```
{url=/home,      method=GET,  status=200}  → 130
{url=/api/users, method=POST, status=201}  → 50
{url=/checkout,  method=POST, status=200}  → 80
{otel.metric.overflow=true}                → 35   ← 15 + 20 folded together
```

<br>

- 3 normal series with their original attributes

<v-click>

- 1 overflow series — **two completely different requests, now indistinguishable**
  - 15 POST /api/orders → 500 &nbsp;} both folded into the same bucket
  - 20 GET /api/status → 200 &nbsp;&nbsp;}
- Total: 130 + 50 + 80 + 35 = **295** ✅

</v-click>

<!--
The SDK exports four series — not six. The two overflowed requests get folded into the same overflow bucket. 15 plus 20 equals 35. A POST returning 500 and a GET returning 200 are now a single number: 35. You can't tell them apart. Meanwhile, GET /home correctly aggregates to 130 because it was already tracked. The sixth request just adds to an existing series — no new slot needed. Everything that overflows is merged into one indistinguishable bucket. This is what makes the next part so dangerous.
-->

---

# The Main Problem — Metrics That "Lie"

Queries for endpoints that overflowed return **nothing**:

<br>

| Query | Expected | Actual | |
|-------|----------|--------|-|
| Requests to `/api/orders`? | **15** | **0** | ❌ |

<br>

Your error dashboard for that endpoint shows **nothing**.

<v-click>

But that's the "obvious" case — the endpoint overflowed, so maybe you'd expect some data loss...

<br>

### The real question is:

## What about your *other* queries? 🤔

</v-click>

<!--
The obvious victim is the overflowed attribute itself. If you query for requests to /api/orders, you get zero. That endpoint is invisible. Your dashboard shows nothing. But here's the thing — you might say "well, that endpoint overflowed, of course it got capped." That's the obvious case. The real danger is what happens to your OTHER queries.
-->

---

# OTHER Queries Lie Too!

When overflow fires, **ALL attributes** on the combination are lost.

Recall exported series: `/home:GET:200→130`, `/api/users:POST:201→50`, `/checkout:POST:200→80`, `overflow→35`

<br>

| Query by `http.status_code` | Expected | Actual | Problem |
|------------------------------|----------|--------|---------|
| How many **200** responses? | **230** | **210** | 20 from overflow invisible |
| How many **201** responses? | **50** | **50** | ✅ Happened to be tracked |
| How many **500** responses? | **15** | **0** | 🚨 Error alert doesn't fire! |
| **Total** requests? | **295** | **295** | ✅ Always correct |

<v-click>

### 🔑 Key Insight

> Overflow poisons queries on **every** attribute of that metric — not just the one that caused it.

</v-click>

<!--
This is the slide I really want you to remember. When overflow fires, ALL attributes on that measurement are lost. So if you query by status code, the 500 errors return zero — because those 15 requests lost their status_code attribute in overflow. Same for the 200 count — 230 expected but only 210 visible, because 20 successful requests are hidden in overflow too. The only query that remains correct is the total — 295 equals 295. Overflow poisons queries on EVERY attribute of that metric, not just the one that caused it.
-->

---

# What Attributes Are NOT Affected?

<br>

**Resource attributes** like `service.name`, `host.name` are **not** subject to cardinality capping.

They are not part of the metric data point — they live at a different level.

<br>

Only **metric data point attributes** — the ones you pass when recording a measurement — count toward the cardinality limit.

<!--
A quick clarification: resource attributes like service.name and host.name are safe — they're not part of the cardinality calculation. Only the data point attributes — the ones you pass when recording a measurement — count toward the cardinality limit.
-->

---

# How to Detect It

Before trusting any query, check: **did overflow occur in this time window?**

<br>

Query for the presence of `otel.metric.overflow = true` on your metric.

<v-click>

- If it's there → **any attribute-based query on that metric may be incomplete**
- If it's not → your queries are trustworthy for that period

</v-click>

<!--
So how do you protect yourself? Before trusting any query result, first check whether overflow occurred in that time window. Query for the presence of the otel.metric.overflow attribute on your metric. If it's there, any attribute-based query on that metric could be incomplete. If it's not, you're safe. This is a manual check, but it works.
-->

---

# This Could Be Automated

The `otel.metric.overflow` attribute is **standardized** — tools can detect it automatically.

<br>

Imagine: your dashboard warns you **before you even query**:

*"Cardinality capping detected — results for this metric may be incomplete."*

<v-click>

<br>

## Does your tooling actually do this today?

</v-click>

<!--
But you shouldn't have to check manually every time. The overflow attribute is standardized. Any tool could detect it automatically and warn you before you even run a query. Imagine your dashboard just showing a warning — cardinality capping detected, results may be incomplete. Does your tooling actually do this today?
-->

---

# One Example Tool That Does — Aspire Dashboard

<img src="/aspire-overflow.png" alt="Aspire Dashboard showing cardinality capping warning" style="width: 85%; margin: 0 auto;" />

<!--
Here's one example. The Aspire dashboard detects the overflow attribute and shows this red warning — cardinality capping has been detected, some dimensions have been dropped. It's not the only tool that could do this — but it's one that already does.
-->

---

# Solutions & Mitigation

### If possible, avoid high-cardinality dimensions

<v-click>

### Temporality matters

| | Cumulative | Delta |
|-|-----------|-------|
| Series lifetime | Forever (since SDK start) | Reset each collection cycle |
| Cardinality pressure | Accumulates over time | Only active combos count |
| Reclaiming slots | ❌ Never | ✅ Stale combos freed |

</v-click>

<v-click>

### ⚠️ Important distinction

OTel SDK cardinality capping is at the **SDK level**. It limits what one process holds in memory. The backend may see far more series from multiple instances, resource attributes, etc.

</v-click>

<!--
So what can you do about it? First, the obvious: avoid high-cardinality attributes. Use Views to drop or reduce dimensions before aggregation. Second, temporality matters a lot. With cumulative temporality, once a series is created it lives forever in the SDK — cardinality only grows. With delta, the SDK resets each cycle and can reclaim slots for inactive combinations. This dramatically reduces overflow pressure. And one critical point to be very clear about: this capping happens at the SDK level. It limits what one process holds in memory. Your backend can still see far more series from multiple instances and resource attributes.
-->

---

# Conclusions & Takeaways

<br>

1. **Cardinality capping is a necessary safety net** — it prevents OOM

<v-click>

2. But it introduces a **silent correctness problem** — metrics can lie

</v-click>

<v-click>

3. One bad attribute poisons queries on **ALL** attributes of that metric

</v-click>

<v-click>

4. **Check your tooling** — the overflow attribute is standardized; does your backend/vendor detect it?

</v-click>

<v-click>

5. The overflow attribute exists for a reason — **monitor it, alert on it, don't ignore it**

</v-click>

<br>

### Thank you!

**Cijo Thomas** · KubeCon EU 2026 · Observability Day

<!--
To wrap up: cardinality capping is a necessary safety net — it prevents OOM. But it introduces a silent correctness problem. One bad attribute can poison queries on all attributes of that metric. Use tools that understand overflow, like Aspire. Design your metrics carefully. Choose your temporality wisely. And above all — monitor the overflow attribute, alert on it, and don't ignore it. Thank you!
-->

---
class: text-center
---

# Feedback

<br>

<img src="/feedback-qr.png" alt="Feedback QR Code" style="width: 300px; margin: 0 auto;" />

<br>

**https://sched.co/2DYA4**
