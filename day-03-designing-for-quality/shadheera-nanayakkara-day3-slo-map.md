# BookSwap — SLI/SLO Map

**Day 3 deliverable 1 of 4**
**Scope:** 200 apartment buildings, ahead of a known Sunday tabloid spike (10× normal RPS, sustained ~4 hours). The Day 2 design is unchanged; this document defines the measurable promises it must keep.

Two rules run through the map:

- Success-rate SLIs count only `5xx` as failures. A `400`/`401`/`422` is the API correctly rejecting a bad request, so it must not burn the error budget.
- Security and privacy are not percentages. Authentication bypass and cross-member data exposure are zero-tolerance guardrails (target = 0). "99.9% of members can't see each other's data" would be a breach, not an SLO.

---

## 1. NFR inventory

Every Day 2 NFR mapped to the behaviour it protects, with the Day 3 refinement that becomes the SLO target in Section 2.

| # | NFR (Day 2 → Day 3 refinement) | User-visible behaviour |
|---|--------------------------------|------------------------|
| 1 | **Search latency.** Day 2: under 300 ms p95, one building. Day 3: 99% under 800 ms over 28 days, at 10× RPS. | Results come back quickly, including on the busy Sunday. |
| 2 | **Search with a cold/absent cache.** Day 3: results stay useful when Redis is cold or down. | Search still returns correct SQL-backed results, not an error, when the cache fails. |
| 3 | **Listing creation.** Day 2: must succeed even if email is down. Day 3: 99.9% success; retryable without duplicates. | A book is saved even when email is struggling; retrying a timed-out submit never creates two copies. |
| 4 | **Notification latency.** Day 2: in-app within 2 s; digest best-effort. | The owner sees "new borrow request" almost immediately. The weekly digest has no SLO. |
| 5 | **Privacy / isolation.** Day 2: no address/phone in responses. Day 3: never see another member's loan history. | A member only ever sees their own private data. |
| 6 | **Photo uploads.** Day 2: ≤ 5 MB JPEG/PNG, never in the database. | Normal photos upload; oversized/wrong-type files are rejected cleanly. |
| 7 | **Authentication.** Day 3: JWT on every endpoint except `/health`; tokens expire within 1 hour. | Unauthenticated calls are rejected; a stolen token stops working within an hour. |
| 8 | **Detection.** Day 3: a full listings outage pages on-call within 3 min; ops confirm health in under 5 min. | A human is alerted before most members notice. |
| 9 | **Audit.** Day 3: every auth failure and every loan create/return is logged with request ID and member ID. | Support and security can reconstruct who did what. |

The 10× spike is not a separate SLO. It is the condition under which SLOs #1 and #3 must still hold.

---

## 2. SLI / SLO table

### Traffic baseline (defined once, reused by the runbook and observability plan)

BookSwap is a low-traffic community app. The baseline is derived, not measured:

- **Normal load:** ~10 rps total across all 200 buildings, of which ~5 rps is search (`GET /books`).
- **Sunday spike:** 10× = ~100 rps total, ~4 hours.

~100 rps is a small absolute number, so the Sunday risk is **cold cache and SQL read contention**, not raw throughput. This shapes both the "out of budget" bet below and Failure 3 in the runbook.

### Indicators and objectives

| # | SLI definition | Measurement source | SLO target | Window / condition | Error budget |
|---|----------------|--------------------|-----------|--------------------|--------------|
| 1 | % of `GET /books` requests that returned `2xx` and completed in < 800 ms | App Insights `requests` | ≥ 99% | 28 days rolling, must hold at 10× RPS | 1% ≈ 121k of ~12.1M requests / 28d (see below) |
| 2 | % of `GET /books` that did not return `5xx`, including while the Redis dependency is failing | App Insights `requests` + `dependencies` | ≥ 99.5% | 28 days rolling | 0.5%; a Redis outage must not consume it |
| 3 | % of `POST /books` that did not return `5xx` | App Insights `requests` | ≥ 99.9% | 28 days rolling, incl. spike | 0.1% |
| 3b | Duplicate listings created for the same `(memberId, Idempotency-Key)` | Custom event `book.created` correlated by key; count keys mapping to > 1 row | 0 duplicates | Continuous | Zero tolerance |
| 4 | % of notification events where (row `createdAt` − outbox `occurredAt`) < 2 s | Custom metric `notification.deliveryLatencyMs` | ≥ 99% | 7 days rolling | 1% |
| 5 | Count of `2xx` responses exposing another member's loan history, address, or phone | Custom event `authz.violation` | 0 | Continuous | Zero tolerance |
| 6 | % of valid uploads (≤ 5 MB, JPEG/PNG) returning `2xx`; oversized/wrong-type return 413/415 | App Insights `requests`, `PUT /books/{id}/photo` | ≥ 99% | 28 days rolling | 1% |
| 7 | Count of `2xx` on protected endpoints served without a valid JWT | Custom event `auth.result` from auth middleware | 0 | Continuous | Zero tolerance |
| 8 | Time from listings-endpoint outage to on-call page fired | Azure Monitor alert config (Deliverable 4) | < 3 min | Per incident | A missed/late page triggers alert-pipeline repair |
| 9 | % of `auth.failed`, `loan.created`, `loan.returned` events with both `requestId` and `memberId` | Azure Monitor Logs reconciliation vs SQL loan state | 100% | 28 days rolling | Zero tolerance |

Notes:

- SLI #1/#2/#3 depend on the App Insights SDK naming operations exactly `GET /books`, `POST /books`, etc.
- SLI #4 needs the worker to emit `notification.deliveryLatencyMs = insertedAt − occurredAt`. `occurredAt` already exists on the Day 2 outbox event.
- For a failed auth attempt there is often no member, so `memberId` is logged as `anonymous` — never invented.
- The guardrails (#3b, #5, #7) have no KQL below on purpose: you cannot query production for a PII leak without logging the PII, which is itself the leak. They are prevented in code and verified by the security review's contract and BOLA tests.

### Queryable definitions (KQL)

**SLI #1 — Search latency (28-day):**
```kql
requests
| where timestamp > ago(28d)
| where name == "GET /books"
| summarize
    good  = countif(success == true and duration < 800),   // duration is ms, a number
    total = count()
| extend sli_pct = round(100.0 * good / total, 3)           // target >= 99.000
```

**SLI #3 — Listing creation success (28-day):**
```kql
requests
| where timestamp > ago(28d)
| where name == "POST /books"
| summarize
    good  = countif(toint(resultCode) < 500),               // resultCode is a string; cast it
    total = count()
| extend sli_pct = round(100.0 * good / total, 3)           // target >= 99.900
```

**SLI #9 — Audit completeness (28-day):**
```kql
traces
| where timestamp > ago(28d)
| where tostring(customDimensions.event) in ("auth.failed", "loan.created", "loan.returned")
| extend hasReqId  = isnotempty(tostring(customDimensions.requestId)),
         hasMember = isnotempty(tostring(customDimensions.memberId))
| summarize
    complete = countif(hasReqId and hasMember),
    total    = count()
| extend completeness_pct = round(100.0 * complete / total, 3)   // target = 100
```

### Error budget in real numbers

At ~5 rps, search sees `5 × 60 × 60 × 24 × 28` ≈ **12.1M requests / 28 days**. The 99% SLO gives a **1% budget ≈ 121k requests** that may exceed 800 ms, roughly 4,300/day. That sounds large but is proportionally tiny for a slow (not failed) search on a community app.

The spike is where it bites: at ~100 rps a single bad hour is ~360k requests, enough to blow the whole 28-day budget on its own. That is why the spike is a first-class risk, not an edge case.

---

## 3. Error budget policy

**Healthy:** ship features normally.

**Fast-burn (spending too fast):** if search or listing would exhaust its 28-day budget in under ~1 hour at the current rate (realistic during the spike), the on-call engineer is paged and it is treated as an incident immediately, without waiting for the window to close.

**Exhausted (halt-and-fix):**

1. Change freeze on the affected service: only reliability and security fixes ship, no new features, until the SLI is back above its SLO for 7 consecutive days or the root cause is fixed and confirmed by a load test.
2. The next sprint's top priority for that service becomes the runbook follow-ups (Deliverable 2), not the feature backlog.

**Owners:** the on-call engineer raises the freeze; the BookSwap tech lead owns declaring and lifting it (only after the recovery condition); the product owner is informed but cannot overrule a reliability freeze. The decision and its lift are recorded in the incident channel.

**Guardrails are separate.** #3b, #5, #7, #9 have zero budget. One confirmed violation is a security/correctness incident handled immediately, not a budget conversation. You do not spend budget on leaking a phone number.

---

## 4. Out of budget right now

I would bet we cannot meet **SLI #2 today: search stays ≥ 99.5% available under 800 ms when the cache is cold, at 10× load.** The Day 2 cache only covers catalogue pages 1–2 without a search term, so a flood of specific title lookups from the tabloid will mostly miss the cache and hit Azure SQL directly, and that path has never been load-tested at ~100 rps.

Why this and not another SLO: throughput (~100 rps) is small, so the system is not throughput-bound. The exposure is a cold or bypassed cache sending uncached search queries straight to SQL. Closing the gap before Sunday (verified search indexes, a scaled SQL tier or read replica, and a load test of the uncached path) is tracked in the reliability runbook, Failure 3.
