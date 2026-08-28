# URL Shortener — Database Design

## 1. Introduction

We now know how the basic URL Shortener works and how a short code can be generated.

The next question is:

> **What exactly should we store in the database?**

The database is responsible for maintaining the mapping between a short code and its original URL.

The fundamental relationship is:

```text
short_code → original_url
```

For example:

```text
aB72x → https://example.com/products/iphone-17-pro
```

For the first production-oriented version of our system, we will use a relational database.

The goal of this chapter is to understand:

* What data needs to be stored
* Table structure
* Primary keys
* Unique constraints
* Indexes
* URL lookup performance
* Expiration
* Database transactions
* Concurrent requests
* What the database should and should not be responsible for

---

# 2. What Information Do We Actually Need?

At the minimum, every shortened URL needs:

```text
ID
Short Code
Original URL
Created At
```

Optionally, we may also need:

```text
Expires At
User ID
Is Active
Click Count
Updated At
```

However, we should avoid storing fields simply because they might be useful someday.

Start with the minimum required data.

---

# 3. Basic Data Model

Our initial table will look like:

```text
url_mapping

+----+------------+---------------------------+---------------------+
| id | short_code | original_url              | created_at          |
+----+------------+---------------------------+---------------------+
| 1  | aB72x      | https://example.com/...   | 2026-08-28 10:00:00 |
| 2  | xY92p      | https://google.com        | 2026-08-28 10:05:00 |
| 3  | K8mQa      | https://github.com/...    | 2026-08-28 10:10:00 |
+----+------------+---------------------------+---------------------+
```

The important relationship is:

```text
id
 ↓
short_code
 ↓
original_url
```

---

# 4. Why Do We Need an Internal ID?

You might ask:

> If `short_code` uniquely identifies the URL, why do we need `id`?

We technically could use:

```text
short_code
original_url
```

only.

But having an internal numeric primary key provides several advantages.

For example:

```text
id = 12345
short_code = dnh
```

The numeric ID can act as the internal identity of the record.

It is useful for:

* Database relationships
* Internal references
* Analytics
* Ownership
* Auditing
* Future schema extensions

The internal ID does not need to be exposed publicly.

---

# 5. Primary Key

We can define:

```sql
id BIGINT PRIMARY KEY AUTO_INCREMENT
```

Conceptually:

```text
id
↓
1
2
3
4
5
```

The database guarantees that each record has a unique primary key.

Using `BIGINT` instead of a smaller integer gives us a much larger ID range.

For a system that may grow significantly, this is a sensible default.

---

# 6. Short Code Column

The short code could be stored as:

```sql
short_code VARCHAR(16) NOT NULL
```

For example:

```text
aB72x
xY92p
K8mQa
```

The exact length depends on the identifier-generation strategy.

For our project, we can initially use:

```text
VARCHAR(16)
```

This leaves room for future changes without making the column unnecessarily large.

---

# 7. Short Code Must Be Unique

This is one of the most important database constraints.

We need:

```sql
UNIQUE(short_code)
```

Why?

Because the application performs:

```text
GET /aB72x
```

and expects exactly one mapping.

We cannot allow:

```text
aB72x → URL A
aB72x → URL B
```

Therefore:

```text
short_code
     |
     v
UNIQUE
```

must be enforced by the database.

---

# 8. Why Not Rely Only on Application Code?

We might write:

```python
if not short_code_exists(code):
    create_url(code)
```

This appears safe.

But under concurrency:

```text
Request A
Request B
```

can both execute:

```text
check code
```

before either one inserts the record.

Both might see:

```text
Does aB72x exist?

NO
```

Then both try:

```text
INSERT aB72x
```

This is a race condition.

The database must therefore enforce the invariant:

```text
short_code must be unique
```

---

# 9. Database Constraint as Final Protection

The correct architecture is:

```text
Application
    |
    | Check
    v
Does code exist?
    |
    v
Database
    |
    | UNIQUE constraint
    v
Final correctness
```

The application check can improve efficiency.

The database constraint guarantees correctness.

This pattern is extremely common in production systems.

---

# 10. Original URL

The original URL should be stored as text.

For example:

```sql
original_url TEXT NOT NULL
```

Example:

```text
https://example.com/products/iphone-17-pro?color=black&storage=512gb
```

URLs can be longer than normal strings, so `TEXT` is a reasonable starting point.

Depending on the database and application requirements, a sufficiently large `VARCHAR` can also be used.

The important point is:

> Do not assume every URL will be short.

---

# 11. Created Timestamp

We should store when the short URL was created.

Example:

```sql
created_at TIMESTAMP NOT NULL
```

This allows us to answer questions such as:

```text
When was this URL created?
```

It also becomes useful for:

* Cleanup
* Analytics
* Debugging
* Auditing
* Reporting
* Expiration policies

---

# 12. Expiration

Some URL Shorteners support temporary URLs.

For example:

```text
Created:
2026-08-28 10:00

Expires:
2026-09-28 10:00
```

We can represent this with:

```sql
expires_at TIMESTAMP NULL
```

If:

```text
expires_at IS NULL
```

we can interpret it as:

```text
No expiration
```

If an expiration timestamp exists:

```text
current_time > expires_at
```

then the URL should no longer redirect.

---

# 13. Active Status

Another useful field is:

```sql
is_active BOOLEAN NOT NULL DEFAULT TRUE
```

This allows us to disable a short URL without deleting its database record.

For example:

```text
aB72x
```

could exist in the database but be disabled.

The redirect flow becomes:

```text
Lookup short code
      |
      v
Record exists?
      |
      v
Is active?
      |
      +---- NO → 404 / 410
      |
      +---- YES
              |
              v
          Check expiry
              |
              v
           Redirect
```

This is useful for administrative controls and abuse handling.

---

# 14. Recommended Initial Schema

For our educational project, we can use:

```sql
CREATE TABLE url_mapping (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code VARCHAR(16) NOT NULL UNIQUE,
    original_url TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);
```

This gives us a good foundation without overengineering.

---

# 15. Why We Should Not Store Everything

A common mistake in system design is creating a table like:

```text
id
short_code
original_url
user_id
ip_address
country
browser
device
click_count
last_clicked_at
campaign
referrer
utm_source
utm_medium
utm_campaign
...
```

before we actually need these fields.

This creates unnecessary complexity.

Instead:

```text
Core URL mapping
        ↓
Add features when requirements demand them
```

For example, analytics can later have a separate table or event pipeline.

---

# 16. URL Creation Flow With Database

Now our create flow becomes:

```text
Client
  |
  | POST /shorten
  v
Application
  |
  | Validate URL
  v
Generate Short Code
  |
  v
Database INSERT
  |
  v
Return Short URL
```

Example request:

```json
{
  "url": "https://example.com/products/iphone-17-pro"
}
```

Application generates:

```text
aB72x
```

Database stores:

```text
id = 1
short_code = aB72x
original_url = https://example.com/products/iphone-17-pro
```

Response:

```json
{
  "short_url": "https://short.ly/aB72x"
}
```

---

# 17. Redirect Flow With Database

When a user opens:

```text
https://short.ly/aB72x
```

the application executes a lookup similar to:

```sql
SELECT original_url, expires_at, is_active
FROM url_mapping
WHERE short_code = 'aB72x';
```

If the record is valid:

```text
original_url
      ↓
https://example.com/products/iphone-17-pro
```

The application returns:

```text
HTTP 302
Location: https://example.com/products/iphone-17-pro
```

---

# 18. Indexing

The redirect path is our most important read operation.

We frequently execute:

```sql
WHERE short_code = ?
```

Therefore, `short_code` must be indexed.

Fortunately:

```sql
UNIQUE(short_code)
```

normally creates a unique index.

So:

```text
GET /aB72x
     |
     v
Database
     |
     v
Index on short_code
     |
     v
Matching row
```

The database does not need to scan the entire table.

---

# 19. Without an Index

Imagine:

```text
10 million records
```

and we execute:

```sql
SELECT *
FROM url_mapping
WHERE short_code = 'aB72x';
```

Without an appropriate index, the database may need to inspect many rows.

Conceptually:

```text
Row 1
 ↓
Row 2
 ↓
Row 3
 ↓
...
 ↓
Row 10,000,000
```

This becomes expensive.

---

# 20. With an Index

With an index:

```text
short_code index
       |
       v
    aB72x
       |
       v
Matching record
```

The database can locate the relevant row efficiently.

This is why indexing is critical for a URL Shortener.

---

# 21. Primary Key Index vs Short Code Index

We now have two important indexes.

### Primary Key

```text
id
```

Used for internal record identification.

### Unique Index

```text
short_code
```

Used for public URL lookup.

The redirect path primarily uses:

```text
short_code
```

not:

```text
id
```

Therefore, the `short_code` index is essential.

---

# 22. Should We Index `original_url`?

Usually, no.

Our primary query is:

```text
short_code → original_url
```

We are not normally asking:

```text
original_url → short_code
```

Therefore, an index on `original_url` would not provide much value for the core redirect flow.

Indexes should support actual query patterns.

Do not add indexes simply because a column exists.

---

# 23. Should We Index `created_at`?

Possibly.

If we later need queries such as:

```sql
SELECT *
FROM url_mapping
WHERE created_at < ?;
```

for cleanup or reporting, an index may become useful.

But we should add it based on actual access patterns.

The principle is:

> Index for queries, not for columns.

---

# 24. Database Read Path

The redirect request is a read-heavy operation.

The path is:

```text
GET /aB72x
     |
     v
Application
     |
     v
Database
     |
     v
short_code index
     |
     v
URL record
     |
     v
original_url
     |
     v
HTTP Redirect
```

This is the critical path we will optimize later.

---

# 25. Database Write Path

Creating a URL requires a write.

```text
POST /shorten
     |
     v
Application
     |
     v
Generate Code
     |
     v
INSERT
     |
     v
Database
```

The database must guarantee:

```text
id unique
short_code unique
```

This is why database constraints are part of the business correctness of the system.

---

# 26. Transaction Considerations

Creating a short URL should be atomic.

Conceptually:

```text
BEGIN
   |
   v
Create record
   |
   v
COMMIT
```

If something goes wrong:

```text
BEGIN
   |
   v
Create record
   |
   X
Failure
   |
   v
ROLLBACK
```

For our simple implementation, the database driver or ORM can manage the transaction.

The important concept is:

> We should not return a successful short URL before the mapping has been successfully persisted.

---

# 27. Example Failure

Suppose the application generates:

```text
aB72x
```

but the database insert fails.

If the application still returns:

```text
https://short.ly/aB72x
```

the user now has a URL that does not work.

That is incorrect.

Correct flow:

```text
Generate Code
      |
      v
Database Insert
      |
      +---- FAILURE → Return Error
      |
      +---- SUCCESS
              |
              v
        Return Short URL
```

---

# 28. Handling Duplicate Short Codes

Suppose our strategy is random generation.

The application generates:

```text
aB72x
```

The database says:

```text
UNIQUE constraint violation
```

The application should:

```text
1. Generate another code
2. Attempt insert again
3. Repeat within a safe retry limit
```

Example:

```text
Generate aB72x
       |
       v
Collision
       |
       v
Generate xY92p
       |
       v
Success
```

We should never retry forever.

A production system should have a bounded retry policy.

---

# 29. Bounded Retry

For example:

```python
MAX_RETRIES = 5
```

Conceptually:

```text
Attempt 1
   ↓
Collision

Attempt 2
   ↓
Collision

Attempt 3
   ↓
Success
```

If all attempts fail:

```text
Return server error
```

This protects the application from unexpected infinite loops.

---

# 30. What Happens if the Database Is Down?

Suppose:

```text
POST /shorten
```

arrives while the database is unavailable.

We cannot safely create a persistent short URL.

The application should not pretend the operation succeeded.

The response should indicate a temporary server-side failure.

For example:

```text
HTTP 503 Service Unavailable
```

The exact API response format will be defined in our implementation.

---

# 31. Database Is Not the Only Storage We May Eventually Need

At small scale:

```text
Application
     |
     v
Database
```

may be enough.

At larger scale:

```text
Application
     |
     +---- Redis
     |
     +---- Database
```

may be more appropriate.

Redis can cache frequently accessed mappings:

```text
aB72x → https://example.com/...
```

Then:

```text
GET /aB72x
      |
      v
    Redis
      |
      +---- HIT → Redirect
      |
      +---- MISS
              |
              v
           Database
```

We will introduce this later.

---

# 32. Why Database First?

We should establish a durable source of truth before introducing caching.

The architecture should be:

```text
Database
   ↓
Source of Truth
```

and eventually:

```text
Redis
   ↓
Cache

Database
   ↓
Source of Truth
```

Redis should not initially be treated as the permanent source of URL mappings.

This separation makes the system easier to reason about.

---

# 33. Analytics Should Not Overload This Table

Suppose we want to count clicks.

A naive approach might add:

```text
click_count
```

and update it on every redirect:

```sql
UPDATE url_mapping
SET click_count = click_count + 1
WHERE short_code = ?;
```

This looks simple.

But imagine:

```text
100,000 redirects/sec
```

Now every redirect produces a database write.

The redirect path becomes:

```text
Lookup
+
Update counter
+
Redirect
```

This can create a serious database bottleneck.

Therefore:

> Analytics should eventually be decoupled from the critical redirect path.

We will explore this later using caching, asynchronous events, queues, or streaming systems.

---

# 34. Potential Future Analytics Model

Instead of:

```text
Redirect
   |
   v
Database UPDATE
```

we can eventually do:

```text
Redirect
   |
   +----> Return immediately
   |
   +----> Publish click event
               |
               v
          Event Queue
               |
               v
        Analytics Storage
```

This is a completely separate system-design topic that naturally grows from our URL Shortener.

---

# 35. Data Lifecycle

A URL record can move through different states.

```text
Created
   |
   v
Active
   |
   +----> Expired
   |
   +----> Disabled
   |
   +----> Deleted
```

We do not necessarily have to physically delete every record immediately.

For example:

```text
is_active = FALSE
```

can disable a link while preserving the record for auditing.

---

# 36. Soft Delete vs Hard Delete

### Soft Delete

Keep the record:

```text
id = 1
short_code = aB72x
is_active = false
```

Advantages:

* Auditing
* Recovery
* Historical information

### Hard Delete

Remove the row:

```sql
DELETE FROM url_mapping
WHERE id = 1;
```

Advantages:

* Less storage
* Simpler long-term data retention

The correct choice depends on product and compliance requirements.

For our educational project, we can support soft deletion first.

---

# 37. Basic Entity Relationship

At this stage, we have one main entity:

```text
URL Mapping
```

with:

```text
URL Mapping
----------------
id
short_code
original_url
created_at
expires_at
is_active
```

Later, if user accounts are introduced:

```text
User
  |
  | 1:N
  v
URL Mapping
```

But we will not introduce users until the requirement exists.

---

# 38. Database Design Principles

This simple table demonstrates several important production principles.

### Principle 1

Use a primary key.

```text
id
```

### Principle 2

Enforce business-critical uniqueness.

```text
UNIQUE(short_code)
```

### Principle 3

Index the critical lookup path.

```text
short_code
```

### Principle 4

Store timestamps.

```text
created_at
expires_at
```

### Principle 5

Keep the schema minimal.

Do not store unnecessary information.

### Principle 6

Do not make the database responsible for everything.

Caching, analytics, and asynchronous processing can be separated later.

---

# 39. Final Schema

Our initial schema is:

```sql
CREATE TABLE url_mapping (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    short_code VARCHAR(16) NOT NULL UNIQUE,

    original_url TEXT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    expires_at TIMESTAMP NULL,

    is_active BOOLEAN NOT NULL DEFAULT TRUE
);
```

This is intentionally simple.

It is enough to build the first database-backed version.

---

# 40. Example Records

After creating several URLs:

```text
+----+------------+--------------------------------------+---------------------+---------------------+-----------+
| id | short_code | original_url                         | created_at          | expires_at          | is_active |
+----+------------+--------------------------------------+---------------------+---------------------+-----------+
| 1  | aB72x      | https://example.com/products/iphone  | 2026-08-28 10:00:00 | NULL                | true      |
| 2  | xY92p      | https://github.com/example/project   | 2026-08-28 10:05:00 | NULL                | true      |
| 3  | K8mQa      | https://example.com/invite           | 2026-08-28 10:10:00 | 2026-09-28 10:10:00 | true      |
+----+------------+--------------------------------------+---------------------+---------------------+-----------+
```

---

# 41. Core Query — Create

The application eventually performs something equivalent to:

```sql
INSERT INTO url_mapping (
    short_code,
    original_url,
    expires_at
)
VALUES (
    ?,
    ?,
    ?
);
```

The database generates:

```text
id
```

automatically.

---

# 42. Core Query — Redirect

The redirect operation performs:

```sql
SELECT
    original_url,
    expires_at,
    is_active
FROM url_mapping
WHERE short_code = ?;
```

The application then validates:

```text
Record exists?
       |
       v
Active?
       |
       v
Expired?
       |
       v
Redirect
```

---

# 43. Why This Schema Is Enough for Version 1

The first version does not need:

```text
Redis
Kafka
Load Balancer
Sharding
Read Replicas
Kubernetes
Distributed ID Generator
```

We first need a correct system:

```text
POST /shorten
      |
      v
Database
      |
      v
Short URL

GET /short_code
      |
      v
Database
      |
      v
Original URL
      |
      v
Redirect
```

Once this works, we can benchmark it and identify bottlenecks.

---

# 44. What We Will Learn From This Database Design

This chapter introduces several system-design concepts:

```text
Primary Keys
Unique Constraints
Indexes
Transactions
Concurrency
Race Conditions
Persistence
Data Lifecycle
Soft Delete
Read Path
Write Path
Source of Truth
```

These concepts are much more important than the table itself.

The table is simple.

The engineering decisions around the table are what matter.

---

# 45. Next Evolution

Our architecture currently looks like:

```text
                  ┌───────────────┐
                  │    Client     │
                  └───────┬───────┘
                          |
                          v
                  ┌───────────────┐
                  │ Application   │
                  └───────┬───────┘
                          |
                          v
                  ┌───────────────┐
                  │   Database    │
                  │               │
                  │ url_mapping   │
                  └───────────────┘
```

The next major question is:

> **What happens when the number of redirect requests becomes much larger than the number of URL creation requests?**

For example:

```text
URL creation:
1,000 requests/sec

Redirect:
100,000 requests/sec
```

If every redirect reaches the database, the database may become the bottleneck.

That leads naturally to the next architectural component:

```text
Redis Cache
```

and the **Cache-Aside pattern**.

---

# 46. Chapter Summary

Our URL Shortener needs a persistent mapping:

```text
short_code → original_url
```

The initial database schema is:

```text
url_mapping

id
short_code
original_url
created_at
expires_at
is_active
```

The most important constraint is:

```text
UNIQUE(short_code)
```

The most important lookup is:

```sql
WHERE short_code = ?
```

and therefore `short_code` needs an efficient index.

The database provides:

* Persistence
* Uniqueness
* Concurrency protection
* Reliable storage
* Efficient lookup

But the database should not become responsible for:

* High-volume caching
* Real-time analytics
* Every click counter update
* Asynchronous event processing

Those concerns can be introduced as the system evolves.

The architecture now gives us a solid foundation:

```text
Client
  |
  v
Application
  |
  v
Database
  |
  +---- short_code → original_url
```

The next challenge is performance.

> **How can we make the redirect path extremely fast without querying the database for every request?**

That is where Redis and the **Cache-Aside pattern** enter the system.
