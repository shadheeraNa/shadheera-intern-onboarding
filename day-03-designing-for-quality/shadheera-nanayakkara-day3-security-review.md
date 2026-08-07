# BookSwap — Security Review of the API

**Author:** Shadheera Nanayakkara
**Day 3 deliverable 3 of 4**
**Scope:** 200 apartment buildings, ahead of a known Sunday tabloid spike. This is a design-stage security review. The BookSwap backend has not been implemented yet, so this document reviews the Day 2 design and the OpenAPI contract, names the risks each area carries, and specifies the concrete controls the implementation must include. The only component actually exercised is the Prism mock, covered in the ZAP section at the end.

## How to read this review

Each of the seven required categories is written as a single block with four fields: the question being asked, the finding, a severity (High / Medium / Low), and a specific mitigation. Findings are grounded in BookSwap's own artifacts (the OpenAPI spec, the storage decisions, the reliability runbook, and the SLO map) rather than generic checklist statements.

Severity reflects realistic impact, not "everything is High." The distribution is deliberate:

| # | Category | Severity |
|---|----------|----------|
| 1 | Authentication | Medium |
| 2 | Authorization | High |
| 3 | Injection | Low |
| 4 | Secrets | Low |
| 5 | Transport | Low |
| 6 | Rate limiting | Medium |
| 7 | PII | Medium |

The single High is the Broken Object Level Authorization (BOLA) finding, backed by a concrete request. Two zero-tolerance guardrails from the SLO map run underneath this review: authentication bypass and cross-member data exposure must never happen (target = 0), so the Authz and PII findings tie directly back to them.

---

## 1. Authentication (Authn)

**Question:** Is every non-public endpoint protected by JWT?

**Finding:**

The OpenAPI contract applies `bearerAuth` as a global security requirement, so every operation inherits it and no operation opts out. `/health` is the only unauthenticated endpoint, which matches the Day 3 requirement, and it executes only a `SELECT 1` liveness probe, so it exposes no member data.

However, the contract can only guarantee that a bearer token is *present*, not that it is *valid*. The `bearerFormat: JWT` declaration is documentation, not enforcement. This was confirmed during Day 2 mock testing: the Prism mock accepted the placeholder value `Bearer test-token` and returned `401` only when the header was absent entirely. Signature verification, issuer (`iss`), audience (`aud`), expiry (`exp`), and algorithm checks are therefore not covered by the design contract and must be enforced in API middleware. If that middleware is missing, misconfigured, or accepts unsigned tokens (`alg: none`), a forged token would pass authentication.

**Severity:** Medium

**Mitigation:**

Validate every incoming JWT in Express middleware before the request reaches any handler, using the Microsoft Entra External ID JWKS (public signing keys) endpoint. Specifically: verify the RS256 signature against the JWKS keys; reject any token whose header declares `alg: none` or a symmetric algorithm; validate `iss` against the Entra tenant issuer and `aud` against the BookSwap API's client ID; reject tokens where `exp` is in the past (enforcing the 1-hour lifetime). Return `401` with the existing `Unauthorized` error schema on any failure. Keep `/health` on an allow-list that bypasses this middleware.

---

## 2. Authorization (Authz)

**Question:** Does every `/{id}`-shaped endpoint check object ownership?

**Finding:**

Several endpoints identify a resource by ID in the path and return or modify data that belongs to a specific member. The OpenAPI descriptions state the intended ownership rules (for example, `GET /books/{bookId}/loans` is documented as being "for a book owned by the authenticated member"), but the contract cannot enforce these rules. Enforcement depends on the implementation adding an ownership check to each query, and the naive query, filtering only by the resource ID, would skip it. This is a Broken Object Level Authorization (BOLA) risk.

Concrete example. Member A owns book `4d2b878b-7a1d-4570-b5d2-68ebdb347a55`. Member B is an unrelated member with a valid JWT. Member B sends:

```http
GET /books/4d2b878b-7a1d-4570-b5d2-68ebdb347a55/loans
Authorization: Bearer <Member B's valid token>
```

If the loan query filters only by `book_id`, Member B receives Member A's full borrower history. Authentication succeeds, but authorization is broken. This violates the Day 3 requirement that a member must never see another member's loan history.

Building-level scoping (from the storage decisions, including building-scoped cache keys) does not fix this. It isolates buildings from each other, but two members of the *same* building both pass the building check, so it does not stop a same-building neighbour reading another member's data. Since BookSwap members share a building by design, the ownership check is required in addition to building scoping, not instead of it.

The same pattern affects other ID-based endpoints:

| Endpoint | Authorised caller | Risk if ownership is not checked |
|----------|-------------------|----------------------------------|
| `GET /books/{bookId}/loans` | Book owner | Any member reads any book's borrower history |
| `GET /borrow-requests/{borrowRequestId}` | Borrower or owner of the book | A third member reads a request they are not part of |
| `PATCH /borrow-requests/{borrowRequestId}` | Owner of the requested book | A member accepts or declines someone else's request |
| `PATCH /loans/{loanId}` | Book owner | A member marks another member's loan returned |
| `PATCH /notifications/{notificationId}` | Recipient member | A member alters another member's notification state |

**Severity:** High

**Mitigation:**

Enforce ownership in the database query for every ID-based endpoint, not in a separate code check that can be skipped on some paths. Keep both boundaries as separate predicates in the `WHERE` clause, so a caller who fails either one receives zero rows:

```sql
SELECT l.*
FROM loans l
JOIN books b ON b.id = l.book_id
WHERE b.id = @bookId
  AND b.building_id = @callerBuildingId   -- tenant isolation (defence in depth)
  AND b.owner_id    = @callerMemberId;     -- object-level ownership (stops BOLA)
```

Derive `@callerMemberId` and `@callerBuildingId` from the validated JWT subject, never from request parameters. When the caller is not entitled to the object, return `404 Not Found` rather than `403 Forbidden` on privacy-sensitive reads, so the response does not reveal that the resource exists. Apply the equivalent check to every endpoint above, using the correct owner or recipient column (for notifications, the recipient `member_id`; for borrow requests, the book's `owner_id` or the request's `borrower_id`). As a spec change, state the ownership rule in the `description` of every ID-based endpoint and make the `403` / `404` behaviour consistent; the enforcement logic stays in code, verified by BOLA tests.

---

## 3. Injection

**Question:** Are all DB queries parameterised?

**Finding:**

The design commits to parameterised database access. The storage decisions document passes values as arguments (for example `sql.getBookById(buildingId, bookId)` and `sql.listBooks(buildingId, normalised)`) rather than concatenating them into query strings, and the outbox code inserts through parameterised calls. So the intended pattern is safe.

The residual risk is the small number of places where developers commonly break the parameterisation rule by accident, and two of them exist in this API:

- **Free-text search** on `GET /books?search=...`. Building a `LIKE` pattern by concatenating the search term (`"...LIKE '%" + search + "%'"`) is a common source of SQL injection, and this endpoint takes arbitrary user text.
- **Dynamic filtering and sorting.** The list endpoints accept `condition`, `available`, `status`, `role`, and pagination. If any optional `WHERE` clause or an `ORDER BY` is assembled from these values as raw text, that is an injection surface. `ORDER BY` is a particular risk because a column name cannot be parameterised the way a value can.

No concatenated query was found in the design; this is a control to preserve during implementation, not a discovered vulnerability.

**Severity:** Low

**Mitigation:**

Use parameterised queries for every value, with no exceptions, exactly as the storage design already does:

```javascript
const q = "SELECT * FROM books WHERE title = @title";
db.query(q, { title: req.query.title });   // driver treats input as a value, never SQL
```

For search, keep the user text as a bound parameter and put only the wildcards in the query:

```javascript
const q = "SELECT * FROM books WHERE title LIKE @pattern";
db.query(q, { pattern: `%${searchTerm}%` }); // searchTerm is still a bound value
```

For sorting, never concatenate a column name from user input. Validate the requested sort field against a fixed allow-list of permitted columns and map it to a known-safe column name before it reaches the query. Enforce parameterised access in code review and, if available, a static-analysis lint rule so a concatenated query cannot merge unnoticed.

---

## 4. Secrets

**Question:** Where are connection strings stored?

**Finding:**

The design stores secrets correctly by constraint: the Day 3 requirements mandate Azure Key Vault and forbid `.env` files in production. The relevant secrets are the Azure SQL, Redis, and Service Bus connection strings, the Blob Storage access key, and any Microsoft Entra client secret. None of these should appear in source code, configuration files, or logs.

Using Key Vault does not by itself close every gap, and the residual risks are:

- **Bootstrapping.** If the app authenticates to Key Vault with its own secret (a Key Vault credential in config), the problem has only moved. The app must reach Key Vault without holding any secret of its own.
- **Local development and CI.** Production uses Key Vault, but a developer `.env` can be committed by accident, and a deployment pipeline can print a secret into its build log.
- **Telemetry.** A connection string captured in a startup error log or an Application Insights dependency record is a leak; this must be prevented by the observability plan's redaction rules.
- **Rotation.** Secrets must be rotated on a schedule, and any suspected leak must be treated as compromised and rotated immediately.

No hard-coded secret was found in the design; this is about preserving the mandated control through implementation and operations.

**Severity:** Low

**Mitigation:**

Store all connection strings and keys in Azure Key Vault, and have Azure App Service read them at startup using a **Managed Identity**, so no bootstrap secret exists anywhere in the app or its config. Grant that identity read-only access to only the secrets it needs.

Keep secrets out of source control (add `.env` to `.gitignore`; scan commits for secrets in CI). Inject CI/CD credentials from the pipeline's own secret store and mark them so they are masked in logs, never echoed. In the observability plan, add connection strings and keys to the redaction list so they cannot appear in logs or telemetry. Set a rotation schedule (at least quarterly) and rotate immediately on any suspected exposure; Key Vault versioning makes this a configuration change rather than a code change.

---

## 5. Transport

**Question:** Is TLS enforced at Front Door?

**Finding:**

Transport encryption is part of the design. Azure Front Door sits in front of App Service and terminates TLS, the API is published only on HTTPS (`https://api.bookswap.local`), and the container diagram shows the internal service calls encrypted as well: SQL over TLS, Redis over TLS, and Service Bus (AMQP) over TLS. So member traffic and bearer tokens are encrypted on the public edge and on the backend data hops.

Two residual risks remain even when TLS is enabled:

- **Plain-HTTP handling.** If a request arrives on `http://` and Front Door redirects rather than rejects it, the first request, possibly carrying an `Authorization` header, travelled unencrypted before the redirect. HTTPS should be enforced, not just offered.
- **The Front Door to App Service hop.** Front Door terminates TLS and then forwards the request to App Service. That internal leg must also use HTTPS, so traffic is encrypted end to end rather than only up to Front Door.

**Severity:** Low

**Mitigation:**

Configure Front Door to enforce HTTPS-only and reject plain HTTP (or, at minimum, redirect and never accept authenticated requests over HTTP), and send the HSTS response header (`Strict-Transport-Security`) so clients refuse plain HTTP on future requests. Enable HTTPS on the origin so the Front Door to App Service hop is also encrypted, and set App Service to "HTTPS Only." Keep the backend service connections on TLS as the diagram already shows (SQL, Redis, Service Bus), and use a current TLS version (1.2 or higher). No plaintext transport should exist on any hop.

---

## 6. Rate limiting

**Question:** Are auth and write endpoints rate-limited?

**Finding:**

The design includes rate limiting at Azure Front Door. The reliability runbook specifies concrete limits for the Sunday spike: about 100 requests per minute per IP for general endpoints, 5 per minute per IP on login, and 20 per minute per member on write endpoints. So a rate-limiting layer exists and write endpoints are covered.

Two gaps remain when these limits are viewed as security controls rather than only spike protection:

- **The "login" limit targets an endpoint this API does not expose.** Authentication is handled by Microsoft Entra External ID, so BookSwap has no login route to protect. The auth-adjacent risk that *does* apply here is repeated requests with missing or invalid tokens (probing) and repeated authorization failures, which are not explicitly rate-limited.
- **Sensitive ID-based read endpoints are only loosely bounded.** `GET /books/{bookId}/loans` and the other `/{id}` reads fall under the general 100/min-per-IP bucket. That is generous for an attacker enumerating IDs to harvest data, which is exactly the access pattern behind the BOLA risk in the Authorization finding. Rate limiting is the defence-in-depth layer behind the ownership check; a loose limit weakens that layer.

**Severity:** Medium

**Mitigation:**

Keep the existing Front Door limits, but add tighter, security-focused rules on the sensitive surfaces:

- Apply a stricter per-member limit on the write endpoints that cause side effects (`POST /books`, `POST /books/{bookId}/borrow-requests`, and the `PATCH` endpoints), for example 20 per minute per member as the runbook already suggests, so a single member cannot flood listings, borrow requests, or the notification queue.
- Add a tighter limit on the ID-based read endpoints (for example the loan-history and borrow-request reads), well below the general 100/min bucket, so enumeration of another member's resources is slowed even if a future code change weakens the ownership check.
- Rate-limit repeated `401` / `403` responses per IP (a burst of auth or authorization failures is a probing signal), and log those events so they can also feed detection.
- Return `429 Too Many Requests` with a `Retry-After` header on all of these, as the design already does for the spike.

---

## 7. PII (Personally Identifiable Information)

**Question:** What PII appears in responses, logs, or queues?

**Finding:**

The high-exposure surfaces are well-controlled by design. API responses expose only identifiers: the `Book`, `Loan`, `BorrowRequest`, and `Notification` schemas carry member and resource IDs, never email, address, or phone, and `GET /books/{bookId}/loans` explicitly states that borrower addresses and phone numbers are not returned. Queue messages are equally disciplined: the notification event carries only IDs and is documented as excluding passwords, tokens, addresses, and phone numbers. So responses and queues are not the main risk.

The residual risk is in logs and telemetry:

- **Audit logs contain PII by requirement.** The Day 3 audit rule logs every auth failure and every loan create/return with request ID and member ID. The member ID identifies a person's activity, so these logs deliberately hold PII and need access control and a retention limit, which are not yet specified.
- **Error logging can capture request or response bodies.** An exception handler that logs the full request, or a stack trace including payload data, can leak the contents of a borrow-request message or a profile field.
- **Telemetry can capture query parameters.** Application Insights dependency tracking can record SQL command text; if parameter values (an email or member ID in a `WHERE` clause) are captured, PII lands in the monitoring store.
- **The digest email worker handles member email addresses.** If it logs the recipient address on success or failure (for example "sent digest to alice@example.com"), that is a PII leak in a log line.

**Severity:** Medium

**Mitigation:**

Keep responses and queue messages ID-only, as designed. For logs and telemetry, add explicit rules:

- Define a redaction list (email, address, phone, tokens, connection strings, full request/response bodies) and strip these before any log line is written. Log identifiers (member ID, request ID) for correlation, but never contact details or message contents.
- In the digest worker, log the member ID, never the email address.
- Configure Application Insights to record the query shape without parameter values (disable capture of SQL command parameters), so telemetry never stores the values used in a query.
- Treat audit logs as sensitive: restrict access to security and operations roles only, and set a defined retention period rather than keeping them indefinitely.
- State these redaction rules in the observability plan so logging and monitoring are built to them from the start.

---

## 8. OWASP ZAP baseline scan

A ZAP baseline scan (passive, non-intrusive) was run against the Prism mock server, not a live backend, so the results reflect what a contract mock can expose rather than the real API's authentication, authorization, or injection behaviour. The full output is attached as `zap-baseline-report.html` (ZAP version 2.17.0).

**Command used:**

```bash
docker run --rm -t -v "%CD%:/zap/wrk" ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t http://host.docker.internal:4010 -r zap-baseline-report.html
```

**Result summary:** 65 passive checks passed, with 0 High, 1 Medium, and 1 Informational alert.

| Alert | Risk level | Instances |
|-------|-----------|-----------|
| Cross-Domain Misconfiguration | Medium | 3 |
| Storable and Cacheable Content | Informational | 2 |

**Scan coverage caveat.** ZAP's spider began at the root path `/`, which this API does not define, so it returned `404`. The report confirms that 100% of observed responses were `4xx` and only two endpoints were reached; the mock also has no `robots.txt` or `sitemap.xml`. Every alert below was therefore raised against `404` error responses (`/`, `/robots.txt`, `/sitemap.xml`), not against real API responses such as `/books`. This limits how much the scan can say about the design, but each alert still maps to a real production control.

**Alert 1 — Cross-Domain Misconfiguration (Medium, 3 instances).** ZAP found the response header `Access-Control-Allow-Origin: *`, a wildcard CORS policy that permits any website to make cross-domain read requests. As a statement about the BookSwap design this is a false positive: the wildcard is the Prism mock's default behaviour, not something the OpenAPI specification configures. However, the underlying concern is real for the deployed API, which serves private, authenticated member data. If production returned `Access-Control-Allow-Origin: *`, a website loaded in a member's browser could attempt cross-origin reads of the API. The fix is to restrict CORS to the known BookSwap front-end origin rather than a wildcard. This is recorded in the threat register as a production configuration item.

**Alert 2 — Storable and Cacheable Content (Informational, 2 instances).** ZAP noted that the responses did not send explicit cache-control headers, so a shared cache could store them. On the mock this was raised only on empty `404` pages, so it exposes no data and is a false positive here. In production it points at a genuine control: private responses, especially loan history and borrow requests, should send `Cache-Control: no-store` so that no proxy or shared cache retains another member's data. This reinforces the PII finding in Section 7.

**Conclusion.** No genuine vulnerability was found in the design, which is expected against a contract mock that enforces no business logic and returns only error pages to an unauthenticated crawler. The scan's value was twofold: it confirmed the tooling runs cleanly end to end, and it surfaced two response-header controls, CORS restriction and cache-control, that the real API must set correctly. Both are recorded in the threat register and should be re-verified with an active scan once a real backend exists.

---

## Summary and priorities

The design is privacy-conscious and uses the right Azure controls, so most categories carry residual implementation or operational risk rather than open holes. The one exception is authorization: object-level ownership is not enforced by the contract and must be added in code for every ID-based endpoint, which is the highest-priority item before launch.

Priority order for implementation:

1. **Authorization (High).** Add query-level ownership checks to every ID-based endpoint and verify them with BOLA tests.
2. **Authentication, Rate limiting, PII (Medium).** Enforce full JWT validation; tighten rate limits on sensitive reads and writes; define log and telemetry redaction rules.
3. **CORS restriction (Medium, from the ZAP scan).** Lock `Access-Control-Allow-Origin` to the BookSwap front-end origin in production rather than a wildcard.
4. **Injection, Secrets, Transport, response caching (Low).** Preserve parameterised queries, Managed Identity plus Key Vault, and end-to-end HTTPS as the design already intends, and send `Cache-Control: no-store` on private responses.

The two Low and Medium items in points 3 and 4 that came from the ZAP scan are configuration controls on the real deployment, not gaps in the current design; they are listed here so they are not lost between now and implementation.
