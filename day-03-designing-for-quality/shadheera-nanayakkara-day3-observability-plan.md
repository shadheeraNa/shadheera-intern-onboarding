# BookSwap — Observability Plan

**Author:** Shadheera Nanayakkara
**Day 3 deliverable 4 of 4**
**Scope:** 200 apartment buildings, ahead of a known Sunday tabloid spike (10× normal RPS, ~4 hours). The Day 2 design is unchanged. This plan puts a measurable signal on every promise in the SLO map, an alert on every SLO, and states clearly what is deliberately left unalerted and what PII is redacted before logging.

Two principles carry over from the SLO map and security review:

- Success is measured on `5xx` only. A `4xx` is the API correctly rejecting a bad request and must never page anyone or burn budget.
- Privacy and auth are zero-tolerance guardrails. A single occurrence is an incident, so those alerts fire on the first event, not on a percentage.

---

## Setup

### Signals (required tests / validations)

The three observability pillars (logs, metrics, traces) each have at least two entries. Every signal ties back to a specific SLI from the SLO map.

| # | Signal type | Source | What it answers | Sample query / metric name |
|---|-------------|--------|-----------------|----------------------------|
| 1 | Metric | Application Insights | Search latency p95 (SLI #1) | `requests \| where name == "GET /books" \| summarize percentile(duration, 95) by bin(timestamp, 1m)` |
| 2 | Metric | App Insights | Listing creation success rate (SLI #3) | `requests \| where timestamp > ago(28d) \| where name == "POST /books" \| summarize good = countif(toint(resultCode) < 500), total = count() \| extend sli_pct = round(100.0 * good / total, 3)` |
| 3 | Log | App Insights `traces` | Authn failures with member ID (SLI #7) | `traces \| where customDimensions.event == "auth.failed"` |
| 4 | Trace | App Insights `dependencies` | Slow request breakdown across SQL and Redis (SLI #2) | `dependencies \| where timestamp > ago(1h) \| summarize avg(duration), percentile(duration, 95) by type, target` |
| 5 | Metric | Service Bus | Email digest queue depth | `ServiceBusQueue "notifications-digest" ActiveMessageCount` (Azure Monitor platform metric) |
| 6 | Log | Azure Monitor Logs / App Insights `traces` | Audit completeness: are loan and auth events logged with `requestId` and `memberId`? (SLI #9) | `traces \| where timestamp > ago(28d) \| where tostring(customDimensions.event) in ("auth.failed", "loan.created", "loan.returned") \| extend hasReqId = isnotempty(tostring(customDimensions.requestId)), hasMember = isnotempty(tostring(customDimensions.memberId)) \| summarize complete = countif(hasReqId and hasMember), total = count() \| extend completeness_pct = round(100.0 * complete / total, 3)` |
| 7 | Trace | App Insights (custom metric) | In-app notification delivery latency under 2 s (SLI #4) | `customMetrics \| where name == "notification.deliveryLatencyMs" \| summarize percentile(value, 95) by bin(timestamp, 5m)` |

Row 7 is captured as a custom metric (`notification.deliveryLatencyMs`) rather than a pure distributed trace. It is grouped under the trace pillar because it measures latency across the notification pipeline (outbox → worker → in-app write); a full span-level trace of that path is a later refinement.

### Logs — Azure Monitor Logs

**Schema.** Every log line carries a fixed set of fields so events are correlatable and SLI #9 is directly queryable: `timestamp`, `severity`, `requestId` (correlation ID), `memberId` (or `anonymous` for events that occur before authentication, never invented), `event` (for example `auth.failed`, `loan.created`, `loan.returned`), `route` (for example `POST /books`), `resultCode`, and `durationMs`.

**Retention.** Two tiers. Operational logs are kept 30 days, which covers the longest SLO window (28 days rolling) with margin. Audit events (`auth.failed`, `loan.created`, `loan.returned`) are kept 90 days, since they exist for security and support reconstruction rather than SLO calculation, and access to them is restricted to the security and operations roles.

**Redaction rules (applied before any log line is written).** The principle is: log identifiers, never contents or contact details.

- **Never logged:** email, address, phone, JWTs and other tokens, connection strings, and full request or response bodies (which may contain any of these).
- **Logged for correlation:** `memberId`, `requestId`, `route`, `resultCode`, `durationMs`.
- The weekly digest worker logs the recipient's `memberId`, never the email address.
- Application Insights dependency tracking is configured to record the SQL command *shape* only, with parameter values disabled, so a member ID or email used in a `WHERE` clause never lands in telemetry.

These rules implement the PII finding from the security review (Section 7) and satisfy the guardrail that a member's private data must never appear in logs or telemetry.

### Metrics — Azure Application Insights

Three kinds of metric are collected:

- **Request metrics** (auto-collected): count, duration, and success flag for every HTTP call. These power SLIs #1, #2, #3, and #6.
- **Dependency metrics** (auto-collected): duration and success of every outbound call to Azure SQL, Redis, and Service Bus. These power the slow-request breakdown (signal 4) and show whether SQL or Redis is the bottleneck during the Sunday spike.
- **Custom metrics** (emitted in code): `notification.deliveryLatencyMs`, emitted by the notification worker as `insertedAt − occurredAt`, powering SLI #4.

### Traces — Application Insights distributed tracing

Distributed tracing follows a single request across the API and its dependencies (SQL, Redis, Service Bus) so latency can be attributed to the right component.

**Sample rate:** 100% of failed requests and of write operations (`POST`/`PATCH`) are always traced, since these are rare and high-value. For the high-volume read path (`GET /books`), adaptive sampling is used, targeting roughly 10% under normal load and automatically reducing the rate during the Sunday spike to control cost and ingestion volume. Sampling never drops error traces, so a rare failure is never lost to sampling.

---

## Results Summary

| Metric | Target | Achieved |
|--------|--------|----------|
| SLOs covered by an alert | 100% | 100% (10 of 10 SLOs) |
| Alerts with a clear runbook link | 100% | 9 of 10 (photo-upload intentionally unlinked — see alert table note) |
| Dashboards for ops | 1 health, 1 business | 2 defined (health + business) |

**Health dashboard** (answers "is the system healthy?" in under 5 minutes, satisfying the Day 3 detection NFR): search latency (SLI #1), search and listing error rates (#2, #3), listings-endpoint availability (#9), and current error-budget burn.

**Business dashboard** (answers "are members using it and is the core loop working?"): listings created per hour, borrow requests per hour, notification delivery latency (#4), and digest queue depth.

---

## Alert proposal

Every SLO in the map is protected by exactly one alert. Guardrail alerts (#4, #6, #8) fire on the first occurrence rather than on a percentage, because a single event is already an incident.

| # | Alert | Condition | Severity | Notification | Runbook |
|---|-------|-----------|----------|--------------|---------|
| 1 | Search latency SLO burn | p95 of `GET /books` > 800 ms for 5 min, or 28-day budget would exhaust in < 1 hr at current rate | Sev2 | Pager + Teams | `reliability/runbook.md#failure-3-sunday-tabloid-spike-10x-sustained-traffic` |
| 2 | Search availability | `GET /books` `5xx` rate > 0.5% over 5 min | Sev2 | Pager + Teams | `reliability/runbook.md#failure-2-azure-cache-for-redis-is-down` |
| 3 | Listing creation failure | `POST /books` `5xx` rate > 0.1% over 10 min | Sev2 | Pager + Teams | `reliability/runbook.md#failure-1-azure-sql-primary-unavailable-for-5-minutes` |
| 4 | Duplicate listing detected | any `(memberId, Idempotency-Key)` maps to > 1 created listing | Sev1 | Pager | `reliability/runbook.md#failure-1-azure-sql-primary-unavailable-for-5-minutes` |
| 5 | Notification delivery slow | p95 `notification.deliveryLatencyMs` > 2000 ms over 15 min | Sev3 | Teams | `reliability/runbook.md#failure-3-sunday-tabloid-spike-10x-sustained-traffic` |
| 6 | Cross-member data exposure | `authz.violation` event count ≥ 1 | Sev1 | Pager + security channel | `security/review.md` (Authz, Section 2) |
| 7 | Photo upload failure | `PUT /books/{id}/photo` `5xx` rate > 1% over 15 min | Sev3 | Teams | `— (no matching runbook section; brief scopes runbook to 3 failures)` |
| 8 | Authentication bypass | `2xx` on a protected endpoint with no valid JWT, count ≥ 1 | Sev1 | Pager + security channel | `security/review.md` (Authn, Section 1) |
| 9 | Listings endpoint outage | `GET /books` availability = 0 (no `2xx`) for 3 min | Sev1 | Pager | `reliability/runbook.md#failure-1-azure-sql-primary-unavailable-for-5-minutes` |
| 10 | Audit log incomplete | audit completeness < 100% over 1 hr (events missing `requestId` or `memberId`) | Sev3 | Teams | `security/review.md` (PII, Section 7) |

**Severity spread (deliberate, not everything is a page):**

- **Sev1 (wake on-call immediately):** #4, #6, #8, #9 — the three zero-tolerance guardrails plus a full listings outage.
- **Sev2 (page on-call):** #1, #2, #3 — a member-facing SLO is breaking.
- **Sev3 (Teams, handled in work hours):** #5, #7, #10 — real but low-stakes; no data at risk and nothing is down.

---

## What we are deliberately NOT alerting on

1. **The weekly email digest.** The digest is best-effort with no SLO (SLO map, NFR #4). A late or delayed digest harms no member, so digest lag and digest-queue backlog are shown on the business dashboard but never page anyone.
2. **Individual `4xx` responses.** A `400`, `401`, or `404` is the API correctly rejecting a bad request, not a failure (only `5xx` burns the error budget). Single client errors are never alerted; they remain queryable in logs if a pattern needs investigation.
3. **Redis cache hit rate.** When the cache is cold or down, search falls back to SQL and still returns correct results (SLI #2). A low hit rate makes search slower, not broken, and if it slows enough to matter, the search-latency alert (#1) already catches the real member-visible symptom. We alert on latency, not on the cache metric itself.

---

## How this plan ties to the other deliverables

- Nine of ten alerts resolve to a real heading in reliability/runbook.md (Failures 1–3) or to the relevant section of security/review.md (Authn, Authz, PII). Photo-upload failure is intentionally unlinked, because the runbook is scoped to three failures per the brief and Blob Storage is not among them.
- The redaction rules restate the security review's PII finding (Section 7) as operational logging config.
- The signals reuse the KQL defined in the SLO map for SLIs #1, #3, and #9 verbatim, so the two documents stay consistent.
- The guardrail alerts (#4, #6, #8) correspond to the zero-tolerance SLIs (#3b, #5, #7) in the SLO map, which have no percentage budget by design.
