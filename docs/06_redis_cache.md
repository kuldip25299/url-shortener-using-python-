# URL Shortener — Redis Cache and Cache-Aside Pattern

## 1. Introduction

In the previous chapters, we built a database-backed URL Shortener.

Our current architecture is:

```text
Client
   |
   v
Application
   |
   v
Database
```

For every redirect request:

```text
GET /aB72x
```

the application queries the database:

```text
aB72x
  ↓
Database
  ↓
Original URL
```

This works well for a small system.

But URL Shorteners have an important characteristic:

> **Redirect traffic is usually much higher than URL creation traffic.**

For example:

```text
URL creation:
1,000 requests/sec

Redirects:
100,000 requests/sec
```

If every redirect hits the database:

```text
100,000 redirects/sec
        |
        v
100,000 database reads/sec
```

The database can become the bottleneck.

To solve this, we introduce **Redis as a cache**.

---

# 2. What Is Redis?

Redis is an in-memory data store commonly used for:

* Caching
* Sessions
* Rate limiting
* Counters
* Distributed locks
* Temporary data
* Queues
* Pub/Sub
* Fast key-value lookups

For our URL Shortener, we will initially use Redis for one specific purpose:

> **Cache the mapping between a short code and the original URL.**

For example:

```text
aB72x
   ↓
https://example.com/products/iphone-17-pro
```

Redis can retrieve this mapping much faster than repeatedly querying a database.

---

# 3. Redis Does Not Replace the Database

This is one of the most important concepts in this project.

We should not think:

```text
Database → Redis
```

as:

> "Let's move the database into Redis."

Instead:

```text
Database
    ↓
Source of Truth

Redis
    ↓
Cache
```

The database permanently stores the URL mapping.

Redis temporarily stores frequently accessed mappings.

The architecture becomes:

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
                ┌────┴────┐
                |         |
                v         v
             Redis     Database
             Cache     Source of Truth
```

---

# 4. Why Cache the Redirect Path?

Consider these two operations.

### Create Short URL

```text
POST /shorten
```

This might happen relatively infrequently.

### Redirect

```text
GET /aB72x
```

This may happen thousands or millions of times.

Therefore:

```text
Reads >>> Writes
```

This is called a **read-heavy workload**.

Caching is particularly useful for read-heavy systems.

---

# 5. Without Redis

Our current redirect flow is:

```text
User
 |
 | GET /aB72x
 v
Application
 |
 | SELECT ...
 v
Database
 |
 | original_url
 v
Application
 |
 v
HTTP Redirect
```

If 100,000 users request URLs:

```text
100,000 requests
       |
       v
100,000 database queries
```

The database now has to process every redirect lookup.

---

# 6. With Redis

With Redis:

```text
User
 |
 | GET /aB72x
 v
Application
 |
 | GET aB72x
 v
Redis
 |
 +---- HIT ----> Original URL
 |
 +---- MISS
          |
          v
       Database
          |
          v
        Redis
          |
          v
     Original URL
```

Now the database is accessed only when Redis does not have the mapping.

---

# 7. Cache-Aside Pattern

The caching strategy we will use is called:

> **Cache-Aside**

It is also sometimes called:

> **Lazy Loading**

The application is responsible for:

1. Checking the cache.
2. Reading from the database if the cache misses.
3. Updating the cache.
4. Returning the result.

Redis itself does not automatically know when the database changes.

The application coordinates the two.

---

# 8. Cache-Aside Read Flow

The standard flow is:

```text
1. Request
      |
      v
2. Check Redis
      |
      +---- HIT
      |      |
      |      v
      |   Return URL
      |
      +---- MISS
             |
             v
        Query Database
             |
             v
        Store in Redis
             |
             v
        Return URL
```

This is the most important Redis flow in our URL Shortener.

---

# 9. Cache Hit

Suppose Redis already contains:

```text
Key:
url:aB72x

Value:
https://example.com/products/iphone-17-pro
```

A user requests:

```text
GET /aB72x
```

The application checks Redis:

```text
GET url:aB72x
```

Redis returns:

```text
https://example.com/products/iphone-17-pro
```

This is called a:

> **Cache Hit**

Flow:

```text
GET /aB72x
      |
      v
    Redis
      |
      v
    FOUND
      |
      v
  Original URL
      |
      v
   Redirect
```

No database query is required.

---

# 10. Cache Miss

Now suppose Redis does not contain:

```text
url:xY92p
```

The application performs:

```text
GET url:xY92p
```

Redis returns:

```text
NULL
```

This is a:

> **Cache Miss**

The application then queries the database:

```text
Redis MISS
    |
    v
Database
    |
    v
Original URL
```

Then it stores the result in Redis:

```text
Database
    |
    v
Redis SET
    |
    v
Future requests become cache hits
```

---

# 11. Complete Cache-Aside Flow

Let's follow a complete example.

User requests:

```text
https://short.ly/aB72x
```

Application extracts:

```text
aB72x
```

Then:

```text
Redis GET url:aB72x
```

Suppose the result is:

```text
NULL
```

So:

```text
Cache MISS
```

The application queries:

```sql
SELECT original_url
FROM url_mapping
WHERE short_code = 'aB72x';
```

Database returns:

```text
https://example.com/products/iphone-17-pro
```

Application then executes:

```text
Redis SET url:aB72x
```

with:

```text
https://example.com/products/iphone-17-pro
```

Finally:

```text
HTTP 302
Location: https://example.com/products/iphone-17-pro
```

---

# 12. Second Request

Now another user requests:

```text
GET /aB72x
```

Application checks Redis:

```text
GET url:aB72x
```

Redis returns immediately:

```text
https://example.com/products/iphone-17-pro
```

No database query.

Flow:

```text
Request
  |
  v
Redis
  |
 HIT
  |
  v
Original URL
  |
  v
Redirect
```

This is where caching provides the performance benefit.

---

# 13. Cache Key Design

We need a key for every cached URL.

A simple strategy is:

```text
url:{short_code}
```

Examples:

```text
url:aB72x
url:xY92p
url:K8mQa
```

This makes the purpose of the key clear.

We should avoid vague keys such as:

```text
aB72x
```

because a Redis instance may eventually contain many different types of data.

Using a namespace is cleaner:

```text
url:aB72x
```

---

# 14. Cache Value

The value can simply be the original URL.

For example:

```text
Key:
url:aB72x

Value:
https://example.com/products/iphone-17-pro
```

We do not initially need to cache the entire database record.

We only need what the redirect operation requires.

This follows an important principle:

> Cache the data needed for the hot path, not the entire database row by default.

---

# 15. Should We Cache JSON?

We could store:

```json
{
  "original_url": "https://example.com/products/iphone-17-pro",
  "expires_at": null,
  "is_active": true
}
```

instead of just:

```text
https://example.com/products/iphone-17-pro
```

Whether this is useful depends on our application.

If redirect validation requires:

```text
is_active
expires_at
```

then caching those fields can reduce database access.

However, it introduces a cache-consistency problem.

For our initial implementation, we will keep the cached object simple.

---

# 16. Cache TTL

We should generally avoid keeping cached data forever.

Redis supports:

> **TTL — Time To Live**

For example:

```text
url:aB72x
TTL = 3600 seconds
```

After one hour, Redis automatically expires the key.

Conceptually:

```text
SET url:aB72x URL
       |
       v
   TTL = 1 hour
       |
       v
Key expires
```

This prevents stale cache entries from remaining indefinitely.

---

# 17. Why TTL Matters

Suppose the original URL changes.

Database:

```text
aB72x → URL B
```

but Redis still contains:

```text
aB72x → URL A
```

Without expiration, users could continue receiving the old value.

TTL limits how long stale information can remain in the cache.

However:

> TTL alone does not guarantee immediate consistency.

We will solve that using cache invalidation.

---

# 18. Cache Invalidation

Cache invalidation means removing or updating cached data when the source data changes.

Suppose:

```text
Database:
aB72x → URL A

Redis:
aB72x → URL A
```

Now the URL is changed:

```text
Database:
aB72x → URL B
```

Redis still has:

```text
aB72x → URL A
```

We have a stale cache.

A common strategy is:

```text
Update Database
      |
      v
Delete Redis key
```

For example:

```text
UPDATE database
      |
      v
DEL url:aB72x
```

The next request produces a cache miss:

```text
Redis MISS
    |
    v
Database
    |
    v
URL B
    |
    v
Redis
```

---

# 19. Why Delete Instead of Update?

There are two possible strategies.

### Strategy A — Update Cache

```text
Database update
      |
      v
Redis update
```

### Strategy B — Invalidate Cache

```text
Database update
      |
      v
Redis delete
```

For a simple cache-aside design, deletion is often easier.

The next request reloads the authoritative value from the database.

This reduces the number of places where update logic must remain perfectly synchronized.

---

# 20. Create Flow With Cache

Suppose we create:

```text
POST /shorten
```

with:

```text
https://example.com/products/iphone
```

The application:

```text
1. Generate short code
2. Store mapping in database
3. Return short URL
```

Do we need to immediately populate Redis?

Not necessarily.

We can use lazy loading.

That means:

```text
Database
    |
    v
Redis is populated only when requested
```

This avoids filling Redis with URLs that may never be accessed.

---

# 21. Why Lazy Loading Is Useful

Suppose we create:

```text
1,000,000 URLs
```

but only:

```text
50,000
```

are actually clicked.

If we populate Redis for every URL:

```text
1,000,000 database records
        |
        v
1,000,000 Redis entries
```

we may waste memory.

With cache-aside:

```text
1,000,000 URLs in database
        |
        v
Only requested URLs enter Redis
```

This is more memory efficient.

---

# 22. Cache Hit Ratio

An important metric is:

> **Cache Hit Ratio**

Formula:

```text
Cache Hit Ratio =
Cache Hits / Total Cache Requests
```

For example:

```text
Total requests = 1,000,000
Cache hits     = 950,000
```

Then:

```text
Hit Ratio = 95%
```

This means only:

```text
50,000
```

requests reached the database.

---

# 23. Why Hit Ratio Matters

Suppose:

```text
100,000 redirects/sec
```

### Without Redis

```text
100,000 DB reads/sec
```

### With 95% cache hit ratio

```text
100,000 requests
      |
      +---- 95,000 → Redis
      |
      +---- 5,000  → Database
```

This can dramatically reduce database load.

The exact performance depends on infrastructure, network latency, Redis configuration, database configuration, and workload.

But the architectural benefit is clear:

> Cache reduces repeated database reads.

---

# 24. Cache Eviction

Redis has limited memory.

Suppose Redis can store:

```text
10 GB
```

but our URL dataset grows to:

```text
100 GB
```

We cannot keep everything in Redis.

Redis therefore supports eviction policies.

Common policies include:

```text
LRU
LFU
TTL-based expiration
```

For our project, we do not need to configure every Redis policy initially.

The important concept is:

> Redis is a bounded cache, not an infinite storage system.

---

# 25. LRU

LRU means:

> Least Recently Used

The basic idea is:

```text
When Redis is full
      |
      v
Remove less recently used entries
```

Suppose:

```text
URL A → frequently accessed
URL B → frequently accessed
URL C → never accessed
```

When memory becomes constrained, `URL C` is a better candidate for eviction.

This matches many URL Shortener workloads because popular links tend to receive repeated requests.

---

# 26. Redis Failure

What happens if Redis becomes unavailable?

This is an important production question.

Our database remains the source of truth.

Therefore:

```text
Redis unavailable
      |
      v
Application
      |
      v
Database
```

The system can still potentially serve redirects, although with higher latency and higher database load.

This is one of the major benefits of treating Redis as a cache rather than the primary store.

---

# 27. Graceful Cache Failure

A robust redirect path can conceptually behave like:

```text
Request
   |
   v
Try Redis
   |
   +---- Available → HIT → Redirect
   |
   +---- Available → MISS
   |                    |
   |                    v
   |                 Database
   |
   +---- Unavailable
             |
             v
          Database
```

The application should not fail all redirects simply because the cache is unavailable.

---

# 28. Cache Failure Should Not Cause Database Failure

Suppose Redis fails during a traffic spike.

Normally:

```text
100,000 requests/sec
```

with:

```text
95% cache hit
```

means:

```text
5,000 DB reads/sec
```

If Redis goes down:

```text
100,000 DB reads/sec
```

may suddenly hit the database.

This can overload the database.

This is called a:

> **Cache failure amplification problem**

Therefore, production systems may need additional protection such as:

* Rate limiting
* Connection pooling
* Circuit breakers
* Request coalescing
* Database replicas
* Graceful degradation
* Cache recovery strategies

We will discuss these as the architecture evolves.

---

# 29. Cache Stampede

Another important problem occurs when a popular cache entry expires.

Suppose:

```text
url:aB72x
```

is extremely popular.

Thousands of requests arrive simultaneously:

```text
Request 1
Request 2
Request 3
...
Request 10,000
```

The cached entry expires.

All requests see:

```text
CACHE MISS
```

Then all requests query the database:

```text
10,000 requests
       |
       v
Database
```

This is called:

> **Cache Stampede**

or:

> **Thundering Herd**

---

# 30. Basic Cache Stampede Flow

```text
Popular URL
    |
    v
Redis entry expires
    |
    v
10,000 concurrent requests
    |
    +---- MISS
    +---- MISS
    +---- MISS
    +---- MISS
    |
    v
Database receives many identical queries
```

This can temporarily overload the database.

---

# 31. Preventing Cache Stampede

There are several strategies.

### Strategy 1 — Request Coalescing

Allow one request to load the value while others wait.

```text
Request A
   |
   v
Load database
   |
   v
Populate Redis
```

Other requests wait for the result.

---

### Strategy 2 — Distributed Lock

Use a Redis lock so that only one request refreshes the cache.

Conceptually:

```text
Request A → Acquire lock → Database
Request B → Wait
Request C → Wait
```

---

### Strategy 3 — TTL Jitter

Instead of expiring many keys at exactly the same time, add small random variations to TTL.

For example:

```text
Base TTL = 3600 seconds
```

could become:

```text
3582 seconds
3611 seconds
3640 seconds
```

This spreads expiration over time.

We will not implement all of these in the first version.

---

# 32. Cache Consistency

We now have two copies of the same logical data:

```text
Database
    |
    +---- aB72x → URL A
    |
Redis
    |
    +---- aB72x → URL A
```

This introduces a consistency question.

What happens if:

```text
Database → URL B
Redis    → URL A
```

?

This is why cache invalidation is an important part of system design.

For our URL Shortener:

```text
Database = authoritative
Redis = derived/cache copy
```

The database always wins.

---

# 33. Cache-Aside Write Strategy

For updates, we can use:

```text
Update Database
      |
      v
Delete Cache
```

For creation:

```text
Create Database Record
      |
      v
Return Short URL
```

The cache can be populated later on the first redirect.

For reads:

```text
Check Cache
      |
      +---- HIT → Return
      |
      +---- MISS
              |
              v
          Database
              |
              v
          Populate Cache
              |
              v
            Return
```

This is the standard cache-aside pattern.

---

# 34. Redis Key Namespace

As our system grows, Redis may store more than URLs.

For example:

```text
url:aB72x
rate_limit:user:123
lock:url:aB72x
analytics:clicks:aB72x
```

Using namespaces makes the data easier to understand.

Our URL cache should therefore use:

```text
url:{short_code}
```

Example:

```text
url:aB72x
```

---

# 35. Redis Data Type

For our initial cache, we only need:

```text
String
```

Example:

```text
SET url:aB72x "https://example.com/products/iphone"
```

Then:

```text
GET url:aB72x
```

returns the URL.

We do not need Redis Hashes, Lists, Sets, or Streams for this basic cache.

---

# 36. Example Redis Commands

Set a URL:

```bash
redis-cli SETEX url:aB72x 3600 "https://example.com/products/iphone"
```

Read it:

```bash
redis-cli GET url:aB72x
```

Check TTL:

```bash
redis-cli TTL url:aB72x
```

Delete it:

```bash
redis-cli DEL url:aB72x
```

The exact Redis commands may vary slightly depending on Redis version and client library.

---

# 37. Python Example

Using Python and a Redis client, the logic can be represented as:

```python
def get_original_url(short_code):
    key = f"url:{short_code}"

    cached_url = redis.get(key)

    if cached_url:
        return cached_url

    url = database.get_original_url(short_code)

    if url is None:
        return None

    redis.setex(key, 3600, url)

    return url
```

The important logic is:

```text
Cache
  ↓
Hit? → Return
  ↓
Miss
  ↓
Database
  ↓
Populate Cache
  ↓
Return
```

---

# 38. Redirect Handler

The HTTP handler can conceptually do:

```python
def redirect(short_code):
    original_url = get_original_url(short_code)

    if original_url is None:
        return not_found()

    return redirect_to(original_url)
```

This keeps the caching logic separate from the HTTP response logic.

The architecture becomes:

```text
HTTP Layer
     |
     v
URL Service
     |
     +---- Redis
     |
     +---- Database
```

---

# 39. Why Not Put Redis Logic Everywhere?

We should avoid code like:

```python
redis.get(...)
redis.set(...)
redis.delete(...)
```

spread throughout every application module.

Instead, the cache behavior should have a clear responsibility.

For example:

```text
URL Service
     |
     v
URL Cache
     |
     v
Redis
```

This makes the system easier to maintain and test.

For our educational project, however, we will keep the implementation intentionally simple and avoid unnecessary abstraction layers.

---

# 40. Read Path After Redis

Our architecture is now:

```text
                       GET /aB72x
                            |
                            v
                    ┌───────────────┐
                    │ Application   │
                    └───────┬───────┘
                            |
                            v
                       ┌─────────┐
                       │  Redis  │
                       └────┬────┘
                            |
                     ┌──────┴──────┐
                     |             |
                    HIT           MISS
                     |             |
                     v             v
                Original URL   Database
                     |             |
                     |             v
                     |           Redis
                     |             |
                     └──────┬──────┘
                            |
                            v
                         Redirect
```

This is our first meaningful scalability improvement.

---

# 41. Write Path After Redis

The write path remains simple:

```text
POST /shorten
      |
      v
Validate URL
      |
      v
Generate Short Code
      |
      v
Database INSERT
      |
      v
Return Short URL
```

We do not need to write to Redis immediately.

This gives us:

```text
Database
   ↓
Source of Truth

Redis
   ↓
Read Cache
```

---

# 42. Why This Architecture Is Good

We have separated responsibilities.

### Database

Responsible for:

* Durable storage
* Unique short codes
* URL mappings
* Source of truth

### Redis

Responsible for:

* Fast reads
* Reducing database load
* Temporary cached mappings

### Application

Responsible for:

* Business logic
* Cache-aside behavior
* Validation
* Redirects

This separation makes the system easier to scale.

---

# 43. What Happens During Application Scaling?

Suppose we have:

```text
Application 1
Application 2
Application 3
```

All applications connect to the same:

```text
Redis
```

and:

```text
Database
```

Architecture:

```text
                  ┌── Application 1 ──┐
                  │                   │
Client ───────────┼── Application 2 ──┼── Redis
                  │                   │
                  └── Application 3 ──┘
                                      |
                                      v
                                   Database
```

Now all application servers can share cached URL mappings.

This solves one of the major problems with local in-memory caching.

---

# 44. Local Memory Cache vs Redis

We could technically cache URLs inside each application server.

For example:

```text
Server 1 → Memory
Server 2 → Memory
Server 3 → Memory
```

But then the caches are separate.

```text
Server 1:
aB72x → URL A

Server 2:
aB72x → missing

Server 3:
aB72x → URL A
```

Redis provides a shared cache:

```text
Server 1 ──┐
Server 2 ──┼──> Redis
Server 3 ──┘
```

This is more appropriate for horizontally scaled applications.

---

# 45. Redis Is Still a Dependency

Introducing Redis improves performance but adds another component.

Before:

```text
Application
     |
Database
```

Now:

```text
Application
   |
   +---- Redis
   |
   +---- Database
```

We have gained:

* Faster reads
* Reduced database load
* Shared cache

But we now need to consider:

* Redis availability
* Redis memory
* Eviction
* TTL
* Cache invalidation
* Cache stampede
* Monitoring

This is an important system-design lesson:

> **Every optimization introduces new failure modes and operational complexity.**

---

# 46. Basic Production Architecture

At this stage, our production-oriented architecture looks like:

```text
                         Internet
                            |
                            v
                     Load Balancer
                            |
             ┌──────────────┼──────────────┐
             |              |              |
             v              v              v
        Application 1  Application 2  Application 3
             |              |              |
             └──────────────┼──────────────┘
                            |
                       ┌────┴────┐
                       |         |
                       v         v
                    Redis     Database
                    Cache     Source
                              of Truth
```

This is still a simplified architecture.

We will add components only when a specific scaling problem requires them.

---

# 47. Important Design Decision

Our current design follows this rule:

```text
Database = durable truth
Redis = performance optimization
```

Therefore:

> If Redis is deleted, the system should be able to rebuild the cache from the database.

For example:

```text
Redis data lost
      |
      v
Cache empty
      |
      v
Requests generate cache misses
      |
      v
Database
      |
      v
Redis rebuilt gradually
```

This is called **cache warming through normal traffic** or lazy repopulation.

---

# 48. Cache Warm-Up

Suppose Redis restarts and all cached entries disappear.

Initially:

```text
Redis = EMPTY
```

Requests arrive:

```text
Request
   |
   v
Redis MISS
   |
   v
Database
   |
   v
Redis SET
```

After some time:

```text
Redis
 ├── popular URL A
 ├── popular URL B
 ├── popular URL C
 └── popular URL D
```

The cache naturally warms up.

For some systems, explicit cache warming may be useful, but it is not necessary for our initial design.

---

# 49. Monitoring

Once Redis becomes part of the architecture, we should monitor it.

Important metrics include:

### Cache hit ratio

```text
hits / total requests
```

### Cache misses

```text
misses/sec
```

### Redis memory usage

```text
used_memory
```

### Evictions

```text
evicted_keys
```

### Redis latency

```text
GET latency
SET latency
```

### Database load

Especially:

```text
queries/sec
connections
CPU
latency
```

The goal is not just to add Redis.

The goal is to verify that Redis is actually reducing database pressure.

---

# 50. Benchmarking the Cache

We should eventually compare:

### Version 1

```text
Application → Database
```

against:

### Version 2

```text
Application → Redis → Database
```

We can measure:

```text
Request latency
Database queries
Redis hit ratio
Throughput
CPU usage
```

For example:

```text
                 Database Only     Redis Cache
------------------------------------------------
Latency              X ms             Y ms
DB reads/sec         100K             5K
Cache hit ratio       N/A             95%
```

The actual values should come from benchmarks rather than assumptions.

This will make the repository useful as a practical system-design demonstration.

---

# 51. What We Have Learned

By adding Redis, we introduced several important distributed-system concepts:

```text
Caching
Cache-Aside
Cache Hit
Cache Miss
TTL
Cache Invalidation
Cache Eviction
Cache Stampede
Thundering Herd
Cache Failure
Source of Truth
Horizontal Scaling
Read-Heavy Workload
```

These concepts appear in many production systems, not only URL Shorteners.

---

# 52. Current Architecture

Our URL Shortener has now evolved from:

```text
Client
  |
  v
Application
  |
  v
Database
```

to:

```text
Client
  |
  v
Application
  |
  +---- Redis
  |
  +---- Database
```

with the read path:

```text
Request
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
          |
          v
        Redis
          |
          v
       Redirect
```

---

# 53. What We Have Not Solved Yet

Redis solves the repeated-read problem, but several challenges remain.

We still need to answer:

### How do we generate IDs at very large scale?

```text
Millions/billions of URLs
```

### How do we handle multiple application servers?

```text
Horizontal scaling
```

### How do we protect the API?

```text
Rate limiting
```

### How do we handle malicious URLs?

```text
Abuse prevention
```

### How do we track clicks?

```text
Analytics
```

### How do we handle database growth?

```text
Read replicas
Partitioning
Sharding
```

### How do we handle Redis failure?

```text
High availability
Redis replication
Redis Cluster
```

These will be introduced only when the corresponding problem appears.

---

# 54. Chapter Summary

A URL Shortener is naturally a read-heavy system.

If every redirect hits the database:

```text
100K redirects/sec
        |
        v
100K DB reads/sec
```

Redis can reduce this load.

The cache-aside pattern is:

```text
Request
   |
   v
Redis
   |
   +---- HIT → Return
   |
   +---- MISS
           |
           v
        Database
           |
           v
        Redis
           |
           v
         Return
```

The responsibilities are:

```text
Database
   ↓
Source of Truth

Redis
   ↓
Fast Cache

Application
   ↓
Coordinates Both
```

The most important design principle is:

> **Redis should improve performance, not become the only place where URL mappings exist.**

If Redis fails or loses its data, the database should still contain the authoritative URL mapping.

Our architecture is now:

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
                    ┌────────────┴────────────┐
                    |                         |
                    v                         v
              ┌──────────┐             ┌──────────┐
              │  Redis   │             │ Database │
              │  Cache   │             │  Truth   │
              └──────────┘             └──────────┘
```

The next important question is:

> **How do we scale the application itself when one server is no longer enough?**

That will lead us to **load balancing, horizontal scaling, stateless application servers, and high availability**.
