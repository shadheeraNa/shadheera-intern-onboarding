# BookSwap — Storage and Cache Decisions

## Decision summary

BookSwap will use:

- **Azure SQL** as the system of record for relational business data.
- **Azure Blob Storage** as the system of record for JPEG and PNG book-photo files.
- **Azure Cache for Redis** as an optional performance layer for hot catalogue reads.
- **Azure Service Bus** for asynchronous in-app notification and weekly digest work.
- **Azure Communication Services Email** for outbound weekly digest emails.
- **Microsoft Entra External ID** for authentication and JWT issuance.

All correctness-critical state changes remain synchronous Azure SQL transactions. Redis is never authoritative, and email availability never affects `POST /books`.

### Planning assumptions

These are capacity-planning estimates rather than measured production values:

- Approximately 10 apartment buildings are supported initially.
- Each building can contain up to 5,000 book listings, giving approximately 50,000 books across all buildings.
- Approximately 5,000 members are registered across all buildings.
- Each book receives an average of two borrow requests per year.
- Approximately 60% of borrow requests are accepted.
- An uploaded photo averages 1.5 MB, with a 5 MB maximum.
- Every eligible member receives one weekly digest email.

## 1. Data inventory

| Data type | Example record | Volume estimate after 1 year | Read/write ratio |
|---|---|---:|---|
| Authentication identity | Entra user subject, sign-in credentials and token claims | Approximately 5,000 identities | Read-heavy |
| Building reference | `Building(id, name)` | Approximately 10 rows | Very read-heavy, about 100:1 |
| Member application profile | `Member(id, externalSubjectId, buildingId, email, digestEnabled)` | Approximately 5,000 rows | Read-heavy, about 20:1 |
| Book listing | `Book(id, buildingId, ownerId, title, author, isbn, condition, available, createdAt)` | Approximately 50,000 rows | Strongly read-heavy, about 20:1 |
| Book photo binary | One JPEG or PNG file associated with a book | Up to 50,000 blobs; about 75 GB expected or 250 GB at the maximum size | Read-heavy, about 10:1 |
| Book photo metadata | `BookPhoto(bookId, blobName, contentType, sizeBytes, etag, uploadedAt)` | Up to 50,000 rows | Read-heavy, about 10:1 |
| Borrow request | `BorrowRequest(id, bookId, borrowerId, status, message, requestedAt, respondedAt)` | Approximately 100,000 rows | Moderately read-heavy, about 3:2 |
| Loan and borrower history | `Loan(id, bookId, borrowRequestId, ownerId, borrowerId, status, borrowedAt, dueAt, returnedAt)` | Approximately 60,000 rows | Read-heavy, about 5:2 |
| In-app notification | `Notification(id, memberId, type, message, read, relatedResourceType, relatedResourceId, createdAt, readAt)` | Approximately 300,000 rows | Approximately balanced |
| Transactional outbox event | `OutboxEvent(id, eventType, payload, createdAt, publishedAt)` | Approximately 300,000 events; published rows retained briefly and then removed or archived | Write-and-consume |
| Redis cache entry | One cached book or eligible catalogue page | Bounded by TTL and Redis memory policy | Strongly read-heavy |
| Notification queue message | One event awaiting notification processing | Temporary; follows notification activity | Write-and-consume |
| Weekly digest job | One building-level job per week | Approximately 520 jobs | Write-and-consume |
| Digest email command | One member-level email command per week | Up to approximately 260,000 commands | Write-and-consume |

## 2. Storage selection

| Data type | Chosen store | Why this store | Why not the alternatives |
|---|---|---|---|
| Authentication identity and credentials | Microsoft Entra External ID | Provides managed sign-in, credential protection, account recovery and JWT issuance. The API maps the JWT subject to an internal member. | Storing passwords in Azure SQL would make BookSwap responsible for credential security and token issuance. Redis is temporary and unsuitable for credentials. |
| Building reference | Azure SQL | Buildings are structured records referenced by members and books. Foreign keys ensure that those records belong to a valid building. | Cosmos DB would move relationship checks into application code or require duplicated building data. Redis cannot be authoritative because entries may expire or be evicted. |
| Member application profile | Azure SQL | Stores BookSwap-specific fields such as internal member ID, building membership, email address and digest preference, all of which relate to other business records. | Entra should authenticate the user, not hold BookSwap's relational business model. Cosmos DB adds partitioning and denormalisation work without a clear need at this scale. |
| Book listing | Azure SQL | Books have a stable schema and relationships to an owner, building, requests, loans and photo metadata. SQL supports indexes, joins, pagination and transactions that keep availability consistent. | Cosmos DB would require a partition and denormalisation strategy for only about 50,000 records. Redis can lose entries, and Blob Storage cannot support relational catalogue queries. |
| Book photo binary | Azure Blob Storage | Designed for durable binary objects and efficient delivery of JPEG/PNG files up to 5 MB. | Database BLOBs would enlarge SQL backups, restore time and I/O usage. Redis memory is expensive and temporary; Cosmos DB is not an efficient binary file store. |
| Book photo metadata | Azure SQL | The blob name, content type, size, ETag and book relationship are small structured values that should be constrained with the book record. | Blob metadata alone cannot enforce relational ownership. Redis may cache a URL but cannot be the permanent book-to-photo mapping. |
| Borrow request | Azure SQL | Links a book, borrower, owner, status and optional resulting loan. Transactions and constraints support valid status changes and checks for unavailable books or duplicate pending requests. | Cosmos DB would make coordinated request, book and loan changes more complex. Redis is not durable, and Service Bus cannot provide permanent request history or member queries. |
| Loan and borrower history | Azure SQL | Loans have strong relationships to books, requests, owners and borrowers. SQL can atomically create a loan, set the fixed 30-day due date and change book availability. | Cosmos DB would complicate atomic updates across related documents. Redis is not durable, and Service Bus cannot answer loan-history queries. |
| In-app notification and read state | Azure SQL | Notifications are persistent user-facing records that must support listing and `read`/`readAt` updates after the queue message is completed. | Service Bus messages are removed after processing and cannot provide history. Redis eviction must not delete notification state. Cosmos DB is unnecessary at the expected volume. |
| Transactional outbox event | Azure SQL | The business update and its unpublished event can be committed in the same transaction, closing the failure gap between SQL and Service Bus. | Direct publication after a SQL commit can lose an event if Service Bus is unavailable. A Redis or Cosmos DB outbox would not share the Azure SQL business transaction. |
| Hot book and catalogue entries | Azure Cache for Redis | Provides low-latency in-memory reads for selected repeat catalogue requests and supports TTL-based expiry. | Azure SQL remains required because Redis may expire, evict or lose entries. Blob Storage and Service Bus cannot answer request-response catalogue queries. |
| Pending notification and digest work | Azure Service Bus | Provides durable message delivery, locks, retries, delivery counts and dead-letter queues while consumers are unavailable. | Running the work synchronously would couple API latency to workers and email. Redis does not provide equivalent durable delivery and DLQ behaviour; using SQL alone would require custom queue locking and retry logic. |
| Book-photo delivery | Azure Blob Storage | Serves the authoritative photo bytes referenced by the API. | The application database should not be used as a file server. |
| Outbound digest delivery | Azure Communication Services Email | Required managed service for sending the weekly digest after the background worker receives a command. | Operating a custom SMTP service would add security, deliverability, reputation and maintenance responsibilities. This service is a delivery channel, not a business-data store. |

### 2.1 Source of truth

- **Microsoft Entra External ID:** authentication identity, credentials and JWT issuance.
- **Azure SQL:** buildings, member application profiles, book metadata and availability, photo references, borrow requests, loans, loan history, notifications, read state and unpublished outbox events.
- **Azure Blob Storage:** JPEG and PNG photo bytes.
- **Azure Cache for Redis:** temporary copies only; Azure SQL wins if values differ.
- **Azure Service Bus:** temporary holder of work awaiting processing, not the permanent record of books, loans or notifications.
- **Azure Communication Services Email:** outbound delivery only.

### 2.2 Relational consistency

The following changes must use Azure SQL transactions:

1. **Accept a borrow request**
   - Change the selected request to `ACCEPTED`.
   - Create the loan.
   - Set `dueAt` to exactly 30 days after `borrowedAt`.
   - Mark the book unavailable.
   - Decline competing pending requests.
   - Insert the related notification events into the outbox.

2. **Decline a borrow request**
   - Change the request to `DECLINED`.
   - Record `respondedAt`.
   - Insert the related notification event into the outbox.

3. **Return a loan**
   - Change the loan to `RETURNED`.
   - Record `returnedAt`.
   - Mark the book available.
   - Insert the related notification event into the outbox.

Each transaction either commits fully or rolls back fully. This prevents states such as an accepted request without a loan or an active loan whose book still appears available.

### 2.3 Photo-write consistency

Blob Storage and Azure SQL cannot share one normal database transaction. Photo replacement will therefore:

1. Validate ownership, JPEG/PNG content type and the 5 MB limit.
2. Upload the new file using a unique blob name.
3. Update the SQL photo reference and metadata.
4. Invalidate the cached book and catalogue generation.
5. Delete the previous blob only after the SQL update succeeds.

If the upload succeeds but SQL fails, the new unreferenced blob is deleted immediately where possible or removed later by an orphan-blob cleanup job.

## 3. Cache plan

### 3.1 Objective and scope

Azure Cache for Redis will reduce repeated Azure SQL reads on the hottest catalogue paths and help meet the catalogue target of under 300 ms p95.

Redis is optional for correctness. A Redis error is treated as a cache miss, and the API falls back to Azure SQL. Write operations and business-rule checks always use current SQL state.

### 3.2 Cached data

| Cached data | Example | Cache key | TTL | Rationale |
|---|---|---|---:|---|
| Individual book details | `GET /books/{bookId}` | `book:{buildingId}:{bookId}` | 60 seconds | Book metadata changes infrequently and the key can be invalidated directly. |
| First two catalogue pages without free-text search | `GET /books?condition=GOOD&available=true&page=1&pageSize=20` | `catalogue:{buildingId}:{generation}:condition={condition}:available={available}:page={page}:pageSize={pageSize}` | 30 seconds | Newest-first and availability-filtered pages are likely to be repeatedly requested. The shorter TTL limits stale availability and delayed new listings. |

Only pages 1 and 2 without the `search` parameter are cached initially. Arbitrary search terms and deep pages are likely to have low reuse and would create too many cache keys.

The cached `Book` response may include its `photoUrl`, but Redis will not contain the JPEG or PNG bytes.

### 3.3 Cache keys

Every key includes `buildingId` to prevent cross-building data leakage. Filters are normalised before key generation, so `condition=good` and `condition=GOOD` produce the same key. Missing filters use a consistent value such as `ALL`.

Examples:

```text
book:building-10:book-123
catalogue:building-10:7f45...:condition=ALL:available=true:page=1:pageSize=20
```

The catalogue generation token is stored per building without a TTL. Invalidation replaces it with a new unique value. If the token is missing, the API creates a new unique token rather than reusing a fixed default that could expose an older key namespace.

Cache entries contain only fields already permitted in API responses. They never contain passwords, JWTs, addresses, phone numbers or other private identity data.

### 3.4 Cache-aside pseudocode

```javascript
async function getBook(buildingId, bookId) {
  const key = `book:${buildingId}:${bookId}`;

  try {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
  } catch (error) {
    logger.warn("Redis read failed", { key, error });
  }

  const book = await sql.getBookById(buildingId, bookId);
  if (!book) return null;

  try {
    await redis.set(key, JSON.stringify(book), { EX: 60 });
  } catch (error) {
    logger.warn("Redis write failed", { key, error });
  }

  return book;
}

async function listBooks(buildingId, query) {
  const normalised = normaliseCatalogueQuery(query);
  const cacheEligible = !normalised.search && normalised.page <= 2;

  if (!cacheEligible) {
    return sql.listBooks(buildingId, normalised);
  }

  let key;

  try {
    const generation = await getOrCreateCatalogueGeneration(buildingId);
    key = buildCatalogueKey(buildingId, generation, normalised);

    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
  } catch (error) {
    logger.warn("Redis read failed", { buildingId, error });
  }

  // SQL is called once. SQL errors remain SQL errors rather than being
  // misclassified and retried as cache failures.
  const result = await sql.listBooks(buildingId, normalised);

  if (key) {
    try {
      await redis.set(key, JSON.stringify(result), { EX: 30 });
    } catch (error) {
      logger.warn("Redis write failed", { key, error });
    }
  }

  return result;
}
```

The SQL catalogue query still requires suitable indexes because the endpoint must continue to work when Redis is unavailable.

### 3.5 TTL and invalidation

Book entries use a 60-second TTL because metadata changes infrequently. Catalogue pages use 30 seconds because new listings and availability changes should appear quickly. These are starting values that should be reviewed using cache-hit rate, SQL latency, Redis memory use and acceptable staleness. A small random variation can prevent many keys expiring simultaneously.

After the SQL transaction commits, the API applies the following best-effort invalidation:

| Operation | Individual book key | Catalogue generation | Reason |
|---|---|---|---|
| `POST /books` | None | Replace | New listing must appear in newest-first pages. |
| `PATCH /books/{bookId}` | Delete | Replace | Metadata, condition or availability may change. |
| `PUT /books/{bookId}/photo` | Delete | Replace | `photoUrl` or the photo reference changes. |
| `POST /books/{bookId}/borrow-requests` | None | None | A pending request does not change the current `Book` representation. |
| Accept through `PATCH /borrow-requests/{borrowRequestId}` | Delete | Replace | The book becomes unavailable. |
| Decline through `PATCH /borrow-requests/{borrowRequestId}` | None | None | The book representation does not change. |
| Return through `PATCH /loans/{loanId}` | Delete | Replace | The book becomes available. |

Old catalogue generations are no longer read and expire naturally after 30 seconds. This avoids scanning Redis for every possible filter and pagination key.

If SQL fails, nothing is invalidated. If SQL succeeds but invalidation fails, the business operation still succeeds; the failure is logged, the TTL bounds the stale period, and later write validation still uses SQL.

### 3.6 Data not cached

The initial implementation will not cache:

- **Borrow requests:** user-specific, mutable workflow state with low expected reuse.
- **Loans and borrower history:** correctness-sensitive and sufficiently small for indexed SQL queries.
- **Notifications and read state:** private, frequently changing state that would add invalidation complexity.
- **Arbitrary free-text searches:** too many low-reuse combinations of search text, filters and pages.
- **Photo binaries:** Blob Storage is cheaper and designed for files up to 5 MB.
- **Credentials or JWTs:** Entra External ID remains responsible for authentication.
- **Business decisions:** availability, duplicate pending-request checks, loan creation and return validation always use current SQL state.

### 3.7 Redis failure and monitoring

Redis failure reduces performance but does not break valid requests:

1. Cache reads become misses.
2. The API queries Azure SQL.
3. Cache writes and invalidations remain best-effort.
4. Redis errors are logged.
5. A short Redis timeout prevents the cache from consuming most of the 300 ms response-time budget.
6. Cache-aside reads repopulate Redis after recovery.

Monitor cache-hit rate, Redis latency and timeouts, SQL latency on misses, key count and memory use, catalogue p95 latency and catalogue-generation changes. Cache coverage should expand only when measurements show repeated and expensive reads.

## 4. Queue plan

### 4.1 Objective and queue layout

Azure Service Bus will move secondary work out of synchronous API requests. Immediate business-state changes still occur in Azure SQL.

| Queue | Producer | Consumer | Responsibility |
|---|---|---|---|
| `notification-events` | Transactional outbox publisher | Notification worker | Creates persistent in-app notifications after request and loan events |
| `digest-jobs` | Weekly scheduled job within the backend/worker deployment | Digest worker | Selects the ten newest books for each building and starts the weekly digest |
| `digest-email-commands` | Digest worker | Email worker | Sends one member's digest through Azure Communication Services Email |

Separate queues prevent lower-priority digest traffic from delaying in-app notifications, which should normally arrive within two seconds.

### 4.2 Notification messages and processing

A notification event contains identifiers and immutable event facts, not a full database record:

```json
{
  "messageVersion": 1,
  "eventId": "evt-123",
  "eventType": "BORROW_REQUEST_RECEIVED",
  "occurredAt": "2026-07-20T10:30:00Z",
  "buildingId": "building-10",
  "recipientMemberId": "member-25",
  "relatedResourceType": "BORROW_REQUEST",
  "relatedResourceId": "request-450",
  "correlationId": "request-trace-789"
}
```

Messages exclude passwords, JWTs, addresses, phone numbers and unnecessary personal information.

Normal processing is:

```text
API commits business change and outbox row in Azure SQL
        ↓
Outbox publisher sends the event to `notification-events`
        ↓
Notification worker inserts the persistent Notification row in Azure SQL
        ↓
Worker completes the Service Bus message
        ↓
Member retrieves it through GET /notifications
```

The queue message is temporary. The Notification row and its `read`/`readAt` state remain in Azure SQL.

### 4.3 Transactional outbox

A direct sequence creates a reliability gap:

```javascript
await sql.createBorrowRequest(request);
await serviceBus.send(event);
```

If SQL succeeds and Service Bus fails, the request exists but the event may be lost. BookSwap therefore writes the business change and outbox row in the same SQL transaction:

```javascript
async function createBorrowRequest(command) {
  return sql.transaction(async (tx) => {
    const request = await tx.insertBorrowRequest(command);

    await tx.insertOutboxEvent({
      eventId: createUuid(),
      eventType: "BORROW_REQUEST_RECEIVED",
      aggregateType: "BorrowRequest",
      aggregateId: request.id,
      payload: {
        buildingId: request.buildingId,
        recipientMemberId: request.ownerId,
        relatedResourceType: "BORROW_REQUEST",
        relatedResourceId: request.id
      },
      createdAt: new Date(),
      publishedAt: null
    });

    return request;
  });
}
```

Both rows commit or both roll back. A continuously running publisher reads rows where `publishedAt IS NULL`, sends them to Service Bus and then marks them published. If Service Bus is unavailable, the unpublished rows remain in SQL for a later retry. Published rows are retained briefly for operations and then archived or deleted.

### 4.4 Delivery, idempotency and acknowledgement

Service Bus uses lock-based processing:

1. The consumer receives and temporarily locks a message.
2. It performs its SQL work.
3. It completes the message only after the SQL transaction succeeds.
4. If it crashes or the lock expires, the message becomes available again.

Delivery is treated as **at least once**. A publisher or consumer may crash after completing one system operation but before recording completion in the other, causing duplicate delivery.

Consumers must therefore be idempotent. The Notification table stores `sourceEventId` with a unique constraint. A duplicate event either finds the existing row or fails the unique insert harmlessly; it does not create a second notification. Digest sending similarly records a unique `commandId` and known delivery status.

### 4.5 Retry and dead-letter handling

Failures are classified as:

- **Transient:** temporary SQL connection failure, network timeout, worker restart or email-service outage. The message is not completed and is retried, using delayed or increasing retry intervals where practical.
- **Permanent:** malformed payload, missing required identifier, unsupported message version or a repeatable application error. The message is dead-lettered immediately or after the maximum delivery count.

A starting maximum of 10 deliveries is reasonable and should be tuned using production evidence.

After the maximum count, Service Bus moves the message to the dead-letter queue (DLQ). Operations should retain the queue name, message/event ID, event type, reason, delivery count, timestamps and correlation ID. The response is to alert, diagnose, fix the underlying issue and replay only when safe. Messages must not be automatically recycled forever.

### 4.6 Consumer unavailable for 30 minutes

If the notification consumer is down for 30 minutes:

1. API business operations continue to commit in Azure SQL.
2. Their notification intents are preserved as outbox rows.
3. If the publisher and Service Bus are available, messages accumulate in `notification-events`.
4. If Service Bus is unavailable too, rows remain unpublished in the outbox.
5. When the consumer returns, it processes the backlog; additional worker instances can scale out.
6. Idempotency prevents duplicate notifications during recovery.

No committed book, request or loan data is lost or rolled back. The two-second notification target is missed during the outage, so alerts should monitor consumer health, oldest-message age, queue depth, unpublished outbox age and DLQ count.

### 4.7 Weekly digest and email failure

The weekly scheduled job runs within the backend/worker deployment and enqueues one `digest-jobs` message per building. The digest worker queries **Azure SQL**, not Redis, for the ten most recently added books and emits one `digest-email-commands` message per eligible member.

The email worker then calls Azure Communication Services Email. `POST /books` never calls the email service, so listing creation still returns `201 Created` when email is unavailable.

Temporary email failures are retried. Repeated failures go to the DLQ. Digest commands have a limited useful lifetime, such as 48 hours, so an obsolete digest is expired or dead-lettered rather than delivered shortly before the next week's digest.

Because email is an external side effect, perfect exactly-once delivery cannot be guaranteed. The worker records each unique command ID and the known provider result to reduce duplicate sends and flags ambiguous timeouts for investigation.

### 4.8 Ordering, privacy and monitoring

Queue ordering does not decide book availability, request acceptance or loan status; those states are already committed synchronously in SQL. Consumers use `occurredAt` for notification ordering and may process independent messages concurrently.

Monitor:

- Active queue depth and oldest-message age.
- Processing and success rates.
- Retry, redelivery and DLQ counts.
- Consumer health.
- Unpublished outbox count and age.
- Time from business event to Notification-row creation.
- Digest email success and failure rates.

Under normal operation, the event-to-notification duration should remain within two seconds.
