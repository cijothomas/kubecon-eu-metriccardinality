---
theme: seriph
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

---
layout: cover
class: text-center
---

<img src="/title-banner.png" style="width: 500px; margin: 0 auto 1rem;" />

# "Metrics That Lie" — Understanding OpenTelemetry Cardinality Capping

**Cijo Thomas, Microsoft**

---

# Have Your Metrics Ever Lied to You?

<br>

You query for error count — it says **zero**. Error rate looks fine.

<br>

But users are reporting failures. Logs confirm errors.

<br>

You use OTel to report metrics. It always worked.

**So where did the errors go?**

<!--
Let me start with a question. Have you ever queried your metrics for error count and it said zero? Error rate looks fine. But users are telling you something is broken. You check the logs — errors everywhere. Your app uses OTel. Metrics always worked. So where did the errors go? This talk is about a feature in OpenTelemetry that can make this happen — silently, without any warning. Let me show you how.
-->

---

# Our Running Example

A simple web app counter:

```
counter: http.server.requests
attributes: url.path, http.method, success
```

Each unique combination of attribute values = **one metric data point**

| url.path | http.method | success | → data point |
|----------|-------------|---------|---------------|
| /home | GET | true | Data point #1 |
| /checkout | POST | true | Data point #2 |
| /api/users | POST | true | Data point #3 |
| ... | ... | ... | ... |

<!--
Let me set up the example we'll use throughout this talk. A simple counter tracking HTTP requests with three attributes: URL path, method, and whether it succeeded. Each unique combination becomes a separate metric data point.
-->

---

# What is Cardinality?

**Cardinality** = number of unique attribute combinations

<br>

100 URL paths × 3 methods × 2 success values = **600 data points** ✅

<br>

That's manageable. But what if some attribute has **high cardinality**?

<!--
Cardinality is just the number of unique combinations. With 100 paths, 3 methods, and 2 success values, that's 600 data points. Totally manageable. But what happens when one of those attributes has unbounded values?
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

Hundreds of thousands of paths × 3 methods × 2 success values = **unbounded data points** 💥

<v-click>

### Impact on the SDK

- Each unique combo is a separate aggregation **in memory**
- High cardinality → high memory → **OOM kill** 💀
- One bad attribute can bring down your entire application

</v-click>

<!--
But what happens when url.path isn't just a handful of known routes? If it includes user-specific IDs \u2014 like order IDs or session tokens \u2014 suddenly you have hundreds of thousands of unique values. That means a huge number of data points, each held as a separate aggregation in memory by the SDK. This is how a single metric can OOM-kill your application.
-->

---

# Even "Safe" Attributes Can Explode

Your attributes look low-cardinality: known paths, standard methods, boolean success.

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

`{url=/, method=FOO, success=false}` → new data point!

**Depending on your app, even seemingly safe attributes can trigger an explosion.**

</v-click>

<!--
And it's not just obviously high-cardinality attributes. Even attributes that look safe can explode. If an attacker sends requests with arbitrary HTTP methods — FOO, BAR, random strings — your server returns 405, but the metric still records it with whatever method the attacker sent. Each unique method is a new data point in memory. Depending on your app setup, even seemingly safe attributes can trigger a cardinality explosion.
-->

---

# What Happens If You Run This?

<div style="font-size: 1.5em;">

```
counter = meter.create_counter("http.server.requests")

for i in 1 to 1_000_000_000:
    counter.add(1, { request_id: new_guid(), method: "GET", success: true })
```

</div>

<br>

<v-click>

### Every iteration creates a new unique attribute combination 💥

- 1 billion iterations = **1 billion data points in memory**
- SDK memory grows without bound
- Eventually: **OOM kill** — your app is dead

</v-click>

<!--
Let me ask you — what happens if you run this code? It's just a simple loop with a counter. But notice the attribute: id is a new GUID every iteration. That means every single iteration creates a brand new unique attribute combination that the SDK has to hold in memory. A billion iterations means a billion data points. Your app will be OOM-killed long before it finishes. This is obviously an extreme example, but it shows how easy it is to kill your application with a single bad attribute. In reality, less obvious things like user IDs or full URL paths with query strings do the same thing, just more slowly.
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

SDK caps unique attribute sets at **N per metric per collection cycle** (default: 2,000). For this example: **limit = 3**

<v-click>

| # | Request | Count | Tracked? |
|---|---------|-------|----------|
| 1 | GET /home → success | ×100 | ✅ 1st combo |
| 2 | POST /api/users → success | ×50 | ✅ 2nd combo |
| 3 | POST /checkout → success | ×80 | ✅ 3rd — **limit!** |
| 4 | POST /api/orders → **failure** | ×15 | ❌ **Overflow** |
| 5 | GET /api/status → success | ×20 | ❌ **Overflow** |
| 6 | GET /home → success | ×30 | ✅ **Already tracked** |

</v-click>

<!--
Here's how cardinality capping works. The SDK limits the number of unique attribute combinations per metric per collection cycle. The default is 2,000, but I'm using 3 to keep it simple. We have five types of requests. The first three each get their own data point. But the fourth and fifth are new combinations — they overflow. One is a failed POST, the other is a successful GET. Two very different requests, but they both end up in the same overflow bucket.
-->

---

# What the SDK Exports

Recall: `/home→×130`, `/api/users→×50`, `/checkout→×80`, `/api/orders→×15` ❌, `/api/status→×20` ❌

These become **4 exported data points**:

```
{url=/home,      method=GET,  success=true}   → 130
{url=/api/users, method=POST, success=true}   → 50
{url=/checkout,  method=POST, success=true}   → 80
{otel.metric.overflow=true}                   → 35   ← 15 + 20 folded together
```

## The overflow attribute: **`otel.metric.overflow`**

<br>

- 3 normal data points with their original attributes

<v-click>

- 1 overflow data point — **two completely different requests, now indistinguishable**
  - 15 POST /api/orders → failure &nbsp;} both folded into the same bucket
  - 20 GET /api/status → success &nbsp;}

</v-click>

<!--
The SDK exports four data points — not six. The two overflowed requests get folded into the same overflow bucket. 15 plus 20 equals 35. A failed POST and a successful GET are now a single number: 35. You can't tell them apart. Meanwhile, GET /home correctly aggregates to 130 because it was already tracked. The sixth request just adds to an existing data point — no new slot needed. Everything that overflows is merged into one indistinguishable bucket. This is what makes the next part so dangerous.
-->

---

# The Main Problem — Metrics That "Lie"

What the backend sees:

```
/home:GET:true → 130 | /api/users:POST:true → 50 | /checkout:POST:true → 80 | overflow → 35
```

Queries for endpoints that overflowed return **nothing**:

<br>

| Query | Expected | Actual | |
|-------|----------|--------|-|
| Requests to `/api/orders`? | **15** | **0** | ❌ |

<br>

Your query for that endpoint returns **nothing**.

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

Recall what the backend sees:

```
/home:GET:true → 130 | /api/users:POST:true → 50 | /checkout:POST:true → 80 | overflow → 35
```

<br>

| Query by `success` | Expected | Actual | Problem |
|---------------------|----------|--------|---------|
| How many **successes**? | **280** | **260** | 20 from overflow invisible |
| How many **failures**? | **15** | **0** | 🚨 Error alert doesn't fire! |
| **Total** requests? | **295** | **295** | ✅ Always correct |

<v-click>

### 🔑 Key Insight

> Overflow poisons queries on **every** attribute of that metric — not just the one that caused it.

</v-click>

<!--
This is the slide I really want you to remember. When overflow fires, ALL attributes on that measurement are lost. So if you query by success, failures return zero — because those 15 failed requests lost their success attribute in overflow. Your error alert never fires. Success count is wrong too — 280 expected but only 260 visible, because 20 successful requests are hidden in overflow. The only query that remains correct is the total — 295 equals 295. Overflow poisons queries on EVERY attribute of that metric, not just the one that caused it.
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

# Temporality & Cardinality Capping

<br>

| | Cumulative | Delta |
|-|-----------|-------|
| Data point lifetime | Forever (since SDK start) | Reset each collection cycle |
| Cardinality pressure | Accumulates over time | Only active combos count |
| Reclaiming slots | ❌ Never | ✅ Stale combos freed |
| Overflow risk | Higher — grows forever | Lower — resets each cycle |

<br>

<v-click>

### ⚠️ Important

OTel SDK cardinality capping is at the **SDK level** — not the backend.

The backend may see far more data points from multiple instances, resource attributes, etc.

</v-click>

<!--
Temporality matters a lot for cardinality capping. With cumulative temporality, once a data point is created it lives forever in the SDK — cardinality only grows. With delta, the SDK resets each cycle and can reclaim slots for inactive combinations. This dramatically reduces overflow pressure. And one critical point: this capping happens at the SDK level. It limits what one process holds in memory. Your backend can still see far more data points from multiple instances and resource attributes.
-->

---

# Conclusions & Takeaways

<br>

1. **Cardinality capping is a necessary safety net** — it prevents OOM

<v-click>

2. But it introduces a **subtle observability gap**

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

<!--
To wrap up: cardinality capping is a necessary safety net — it prevents OOM. But it introduces a silent correctness problem. One bad attribute can poison queries on all attributes of that metric. Use tools that understand overflow, like Aspire. Design your metrics carefully. Choose your temporality wisely. And above all — monitor the overflow attribute, alert on it, and don't ignore it. Thank you!
-->

---
layout: cover
class: text-center
---

<img src="/title-banner.png" style="width: 400px; margin: 0 auto 1rem;" />

# Thank You & Feedback

<br>

<img src="/feedback-qr.png" alt="Feedback QR Code" style="width: 300px; margin: 0 auto;" />

<br>

**https://sched.co/2DYA4**

**Cijo Thomas, Microsoft**
