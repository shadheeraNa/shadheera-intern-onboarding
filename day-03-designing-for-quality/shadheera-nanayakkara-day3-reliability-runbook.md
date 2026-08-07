# BookSwap — Reliability Runbook v0.1

**Day 3 deliverable 2 of 4**
**Scope:** 200 apartment buildings, ahead of a known Sunday tabloid spike (10× normal RPS, ~4 hours). The Day 2 design is unchanged. This runbook describes how three named failures show up, how we detect them, what the design already does to survive them, and what a human does when paged.

**Traffic baseline (from the SLO map):** about 10 requests per second normally, roughly 5 of them search, rising to about 100 at the Sunday spike. Roles match the error budget policy: the **on-call engineer** responds and can raise a freeze, the **BookSwap tech lead** owns declaring and lifting an incident, and the **product owner** is kept informed.

## Failure 1: Azure SQL primary unavailable for 5 minutes

Azure SQL is the one database that holds all the real data: members, books, borrow requests, loans, notifications, and the outbox. Everything else trusts it as the single source of truth. In this failure, nothing can reach that database for five minutes.

### What the user sees

- **People just looking at recent books may not notice at first.** Redis, the cache, is holding temporary copies. But those copies only last a short time: 60 seconds for a single book and 30 seconds for the first couple of catalogue pages. Once that time runs out there is no copy left, so viewing breaks too.
- **Anyone trying to do something fails right away.** Listing a book, requesting a borrow, accepting or declining a request, and marking a loan returned all need the database. Instead of a broken screen or an endless spinner, the app shows a polite "BookSwap is temporarily unavailable, please try again."
- **Search stops working** once the short cache window is gone.

The aim of the design is that users see a clean "try again shortly", never a frozen app and never a duplicate listing after they retry.

### Detection

We do not want to wait for users to complain. Two automatic checks run:

- **Failure counter (App Insights `dependencies`).** This counts how many database calls are failing. If more than half fail over a 3-minute window, it pages the on-call engineer as a Sev1.
```kql
  dependencies
  | where timestamp > ago(3m)
  | where type == "SQL"
  | summarize failRate = 100.0 * countif(success == false) / count()
```
- **Health poke.** An App Insights availability test calls `/health` every minute, which runs a tiny `SELECT 1` to ask the database "are you alive?" Three failed pokes in a row is about 3 minutes, which matches the rule that a human must be alerted within 3 minutes.
- **Whose fault is it.** The Azure SQL "Failed Connections" metric and the Resource Health blade tell us whether Microsoft is having a platform problem or whether we overloaded our own database.

### Mitigation in design (timeouts, retries, circuit breaker, fallback)

- **Timeouts.** The app waits at most **5 seconds** to connect to the database. Once connected, a read query is given **3 seconds** to finish and a write transaction **10 seconds**. Reads are kept short on purpose so a slow query cannot break the 800 ms search promise; writes are allowed longer because they update several tables at once.

- **Retry (only for temporary glitches).** If a call fails with a transient error, the app tries again up to **3 times**, waiting a little longer each time (about 200 ms, then 400 ms, then 800 ms, never more than 2 seconds) with a small random delay added so that many failing requests do not all retry at the same instant. Permanent errors, such as a validation or constraint failure, are never retried because they will fail every time.

- **Circuit breaker (a fuse for the database).** If **5 calls fail within 10 seconds**, the app assumes SQL is genuinely down and stops calling it for **30 seconds**, returning an instant "try again" response during that time. After 30 seconds it lets a single test request through: if that succeeds, normal traffic resumes; if not, it waits another 30 seconds. This is the real hero of this failure, because retrying every request during a five-minute outage would use up all the database connection slots and keep our own app jammed even after SQL recovers.

- **Fast failure response.** While the breaker is open, the API returns `503 Service Unavailable` with a `Retry-After: 30` header, which tells the app to wait 30 seconds before trying again.

- **Read fallback.** If SQL is failing but a value is still in Redis, the app serves that cached copy (slightly stale, but useful). This applies to reads only.

- **No write fallback, by design.** Writes are not saved to a queue during an outage. A write such as a borrow request must check the book's current availability, which is impossible while SQL is down, so the write fails cleanly and the client retries later.

- **Idempotency key (no duplicate listings on retry).** Every `POST /books` carries a unique `Idempotency-Key`. The server remembers that key for 24 hours. If a listing times out and the user retries with the same key, the server returns the original result instead of creating a second copy of the book. This is what makes "failed attempts retryable without duplicates" true, and it is reused in Failure 3.

### Manual response (who is paged, what they do)

1. **The on-call engineer is paged** by the Azure Monitor action group (pager plus Teams) within 3 minutes.
2. **Confirm the alert.** Open the App Insights failure view and the SQL Resource Health blade to decide whether this is a Microsoft platform problem or our own capacity problem (maxed out database tier, connection limit, or a runaway query).
3. **If it is a platform problem,** there is nothing to patch. Check that the circuit breaker is working (fast 503s, no jammed connections), post an update in the incident channel, and wait for the database to come back. Azure SQL failover usually finishes in about 30 to 60 seconds, so a full 5 minutes points to a real incident worth watching.
4. **If it is our capacity,** scale the database up to a larger tier, or stop the runaway query or connection leak that the failure view identified.
5. **The tech lead** formally declares the incident if users are affected, and gives the all-clear once database calls are succeeding normally again.

### Post-incident actions

- Hold a blame-free review within 48 hours, and attach the alert timeline and the retry and breaker log lines to confirm they behaved as designed.
- If we ran out of capacity, decide whether to move to a larger SQL tier or add a backup copy (a failover group or read replica) so reads can survive losing the primary.
- Confirm `/health` really did detect the outage and did not itself expose any private data.
- Log the fix as a reliability task, prioritised by the error budget policy, so it comes before new features if the listing-creation budget was spent.

## Failure 2: Azure Cache for Redis is down

Redis is only a cache. It holds temporary copies of book details and the first couple of catalogue pages to make reads fast. It is never the source of truth, so losing it cannot lose any data. When Redis is down, every read simply goes to Azure SQL instead. The real question is not "is the data safe" (it is), but "can SQL handle the extra reading."

### What the user sees

- **Almost nothing, at normal traffic.** Search and book views still work because the app just reads from SQL instead of the cache. Pages come back a little slower than usual, but well inside the 800 ms search promise.
- **No errors and no data loss.** Nobody sees a "try again" message. This failure is a slowdown, not an outage.
- **The one thing at risk is speed under load.** If Redis is down during the Sunday spike, all that reading piles onto SQL at once, and that is where search could start creeping toward or past 800 ms. That specific spike case is handled in Failure 3; here we cover Redis being down on a normal day.

### Detection

- **Redis dependency failures (App Insights `dependencies`).** Calls to Redis start failing or timing out.
```kql
  dependencies
  | where timestamp > ago(5m)
  | where type == "Redis"
  | summarize failRate = 100.0 * countif(success == false) / count()
```
- **Cache hit rate drops to near zero.** The observability plan tracks cache hit rate; a sudden fall to 0% is a clear "Redis is gone" signal.
- **Azure Cache for Redis metrics.** The "Errors" and "Server Load" metrics on the Redis resource, plus its Resource Health blade, confirm whether the cache itself is unhealthy.
- **Severity is deliberately lower.** Because the app keeps working, this is a **Sev3** at normal traffic. It only escalates to a paging **Sev2** if search p95 latency climbs toward 800 ms or SQL CPU rises sharply, which means SQL is struggling to absorb the load.

### Mitigation in design (timeouts, retries, circuit breaker, fallback)

- **Very short Redis timeout.** Every Redis call is given only **50 milliseconds** to respond. Redis is meant to be instant, so if it does not answer in 50 ms we stop waiting and treat it as a cache miss. This keeps a sick cache from eating into the 800 ms search budget.

- **No retries on the cache.** Unlike the database, Redis has a fallback: the real data is always in SQL. Retrying a dead cache would just waste time, so a failed Redis call immediately falls through to SQL rather than trying again.

- **Circuit breaker to skip Redis entirely.** If Redis fails **10 times within 10 seconds**, the app stops calling it for **30 seconds** and reads straight from SQL. This avoids paying the 50 ms timeout on every single request while Redis is down. After 30 seconds it tries one call to see if Redis is back.

- **SQL is the fallback for reads.** This is the whole point of cache-aside: a cache miss (or a cache outage) is normal, and the app simply reads from SQL. Because the search endpoint has proper indexes, it still works without the cache, just a little slower.

- **Cache writes and invalidation become best-effort.** When the app changes a book, it normally clears the matching cache entry. With Redis down, that clearing is skipped and only logged. This is harmless, because nothing is being cached anyway, so there is no stale copy to worry about.

### Manual response (who is paged, what they do)

1. **The on-call engineer is notified** (a Sev3 ticket, not a middle-of-the-night page, unless it has already escalated to Sev2).
2. **Check that SQL is coping.** Watch the search p95 latency and the SQL CPU / DTU metric. As long as search stays under 800 ms, the system is degraded but healthy and there is time to fix Redis calmly.
3. **If SQL is straining** (search latency rising, SQL CPU high), treat it as the escalation point: scale SQL up temporarily and, if the spike is involved, follow Failure 3.
4. **Recover Redis.** Reboot the Azure Cache for Redis node, or let Azure heal it. On a tier with a replica, Azure fails over automatically; the engineer's job is mainly to confirm recovery and watch the cache hit rate climb back up.
5. **The tech lead** is informed but only declares an incident if it escalates to Sev2.

### Post-incident actions

- Confirm the design behaved as intended: the 50 ms timeout kept latency in budget, the breaker skipped Redis, and SQL absorbed the reads without errors.
- Load-test the uncached search path at spike traffic (about 100 rps). The SLO map already flags this as an untested gap (SLI #2), so this is the concrete follow-up that closes it.
- If Redis is running on a tier without a replica, evaluate moving to one with automatic failover so a single node loss is invisible.
- Consider a cache warm-up step after recovery so the first requests after Redis returns do not all miss at once.

## Failure 3: Sunday tabloid spike, 10x sustained traffic

A tabloid will feature BookSwap next Sunday. Traffic is expected to jump from about 10 requests per second to about 100, and stay there for roughly 4 hours. Nothing is broken here; the system is simply asked to do ten times its normal work at once. Because we know it is coming, most of the response is preparation, and the design should absorb the load automatically rather than waiting for a human to react.

### What the user sees

- **Ideally, nothing unusual.** Pages load normally and new listings save fine, because more copies of the API spin up to share the load.
- **The two promises under pressure** are search staying under 800 ms (SLO #1) and listing creation staying 99.9% successful (SLO #3). Both must hold *during* the spike, not just on a calm day.
- **If we shed load,** some requests may get a brief "you are being rate limited, please wait a moment" (`429`) response rather than a slow page or an error. This is a deliberate, controlled slowdown for a few users to protect the service for everyone.

### Detection

- **Traffic surge (App Insights `requests`).** A sharp rise in request count is the first sign the spike has started.
```kql
  requests
  | where timestamp > ago(15m)
  | summarize rps = count() / 900.0 by bin(timestamp, 1m)
```
- **SLO fast-burn alert.** The important alert is not raw traffic, it is the error budget burning too fast. Per the error budget policy, if search or listing creation would use up its whole 28-day budget in under about 1 hour, the on-call is paged immediately and it is treated as an incident. A single bad hour at 100 rps is enough to do that, which is why the spike is a first-class risk.
- **The bottleneck signals.** Watch SQL CPU / DTU (the most likely choke point, since a cold cache sends searches to SQL), App Service CPU and instance count, and the Service Bus queue depth and oldest-message age for the background worker.
- **Front Door metrics.** Total request count and the number of `429` (rate-limited) responses show whether throttling is kicking in.

### Mitigation in design (autoscale, queue depth, throttling)

- **Autoscale the API (horizontal scaling).** The API is stateless, so we can just add more copies behind the load balancer. Azure App Service autoscale starts at **2 instances** and adds **2 more whenever average CPU stays above 70% for 5 minutes**, up to a maximum of **10 instances**. It scales back down by 1 instance when CPU drops below 40% for 10 minutes. Ten copies comfortably covers ten times the traffic.

- **Scale the background worker on queue depth.** Notifications and digests run through Service Bus, so a busy Sunday makes those queues grow. The worker autoscales on **queue depth: add an instance for every 500 waiting messages**, up to 5 workers. In-app notifications keep arriving quickly; the weekly digest is best-effort and is allowed to lag, so it never competes with live traffic.

- **Rate limiting and WAF at Azure Front Door (throttling).** Front Door sits in front of App Service and caps abusive or runaway traffic before it reaches us: about **100 requests per minute per IP** for general endpoints, a tight **5 per minute per IP on login** (a standard defence against credential attacks), and **20 per minute per member on write endpoints**. Over-limit requests get a `429` with `Retry-After`. This protects the service without ever taking it fully down.

- **Protect the database, the real bottleneck.** Since a cold cache pushes searches onto SQL, the plan is to **pre-warm the cache on Saturday evening** (populate catalogue pages 1 and 2 for all 200 buildings), **confirm the search indexes are in place**, and **scale the SQL tier up for the event** (or add a read replica) so it has headroom. This directly addresses the gap the SLO map flagged as untested.

- **Reuse the SQL timeouts, retries, and breaker from Failure 1.** The same short timeouts, limited exponential-backoff retries, and circuit breaker apply. Under load these matter even more: they stop a brief SQL slowdown from snowballing into a pile-up of stuck requests.

- **Idempotency key keeps listing creation safe under load (SLO #3).** During a spike, timeouts and retries are far more likely, so users and clients will resubmit. Because every `POST /books` carries an `Idempotency-Key` that the server remembers for 24 hours, a resubmitted listing returns the original result instead of creating a duplicate. This is what lets us promise "99.9% success, retryable without duplicates" even when the network is under strain.

### Manual response (who is paged, what they do)

Because this is planned, most of the work happens before Sunday:

1. **Before the event:** the on-call engineer and tech lead confirm autoscale rules are active, the cache is pre-warmed, SQL is scaled up, and the fast-burn alert is armed. A short "war-room" chat channel is opened for the window.
2. **During the event:** the team watches the health dashboard rather than touching anything. As long as autoscale is firing and search stays under 800 ms, no action is needed.
3. **If the budget starts fast-burning,** the on-call treats it as an incident per the policy: check whether the choke point is SQL (scale the tier further) or the API (raise the autoscale maximum), and tighten the Front Door rate limits if the traffic looks abusive rather than genuine.
4. **The tech lead** owns any decision to shed more load or to trigger the error budget freeze, and keeps the product owner informed.

### Post-incident actions

- Compare what actually happened to the prediction: was it really 10x, or more? Feed the real peak rps back into the SLO map traffic baseline so future planning is based on measured numbers.
- Right-size the autoscale maximum and the SQL tier from the real load, rather than the estimate, and decide whether to keep the scaled-up SQL or return to the smaller tier.
- Decide whether to make the cache pre-warm and wider cache coverage permanent, if the uncached search path was the main strain.
- Record how close each SLO came to its budget during the window, as evidence for the next spike.