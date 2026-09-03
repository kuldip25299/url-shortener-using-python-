# URL Shortener — Cache Failure, Cache Stampede, Penetration and Avalanche

A Redis cache can make a URL shortener extremely fast.

But adding a cache also introduces a new class of distributed-system problems.

A production URL shortener cannot assume:

```text
Redis is always available
```

or:

```text
If cache misses, everything is fine
```

At high traffic, a single popular short URL can receive thousands or millions of requests.

If the cache fails or expires at the wrong time, all those requests can suddenly hit the database.

That can create a much bigger problem than the original cache miss.

This chapter explains the most important cache failure patterns and how production systems handle them.

---

# 1. Why Cache Failures Matter

Our basic architecture currently looks like:

```text
                ┌──────────────┐
                │    Client    │
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │ Load Balancer│
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │ URL Service  │
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │    Redis     │
                └──────┬───────┘
                       │
                 Cache Miss
                       │
                       ▼
                ┌──────────────┐
                │   Database   │
                └──────────────┘
```

The normal flow is:

```text
Request
   │
   ▼
Redis
   │
   ├── HIT ──────► Return URL
   │
   └── MISS
         │
         ▼
      Database
         │
         ▼
      Redis
         │
         ▼
      Return URL
```

This works well when cache misses are relatively rare.

But consider:

```text
1 popular URL
10,000 requests/second
Redis key expires
```

Now thousands of requests may simultaneously query the database.

This is where cache stampede begins.

---

# 2. What Is Cache Stampede?

A cache stampede happens when many requests try to rebuild the same expired or missing cache entry at the same time.

For example:

```text
Short URL:

abc123

Redis:

abc123 → EXPIRED
```

Now:

```text
Request 1 ──┐
Request 2 ──┤
Request 3 ──┤
Request 4 ──┤
Request 5 ──┤
Request 6 ──┤
Request 7 ──┤
Request 8 ──┤
             ▼
          Redis MISS
             │
             ▼
          Database
```

Every request sees:

```text
MISS
```

So every request queries the database.

Instead of:

```text
10,000 requests
        │
        ▼
     Redis
        │
        ▼
  1 database query
```

we can accidentally create:

```text
10,000 requests
        │
        ▼
     Redis MISS
        │
        ├──► DB
        ├──► DB
        ├──► DB
        ├──► DB
        ├──► DB
        ├──► DB
        └──► ...
```

This is a cache stampede.

---

# 3. Cache Stampede vs Thundering Herd

These terms are closely related.

## Cache Stampede

Usually refers specifically to many requests rebuilding an expired or missing cache entry simultaneously.

```text
Cache expires
     │
     ▼
Many requests
     │
     ▼
Database overload
```

## Thundering Herd

A broader problem where many waiting requests are released at the same time and all compete for the same resource.

For example:

```text
One resource becomes available
          │
          ▼
Thousands of requests wake up
          │
          ▼
All compete for resource
```

A cache stampede is one common form of the thundering-herd problem.

---

# 4. Why This Is Dangerous for a URL Shortener

URL shorteners often have highly skewed traffic.

For example:

```text
URL A → 1,000,000 requests
URL B → 500,000 requests
URL C → 10,000 requests
URL D → 100 requests
URL E → 1 request
```

This is not evenly distributed.

A small number of URLs may become extremely popular.

For example, imagine a company sends a marketing campaign:

```text
https://short.example/abc123
```

Millions of users click it.

If:

```text
abc123
```

expires from Redis at exactly the wrong moment, millions of requests may attempt to load it from the database.

The database may become the bottleneck.

---

# 5. Basic Protection: Double-Check Cache

One simple technique is to check the cache again after acquiring a lock.

The flow is:

```text
Request
   │
   ▼
Redis GET
   │
   └── MISS
        │
        ▼
   Acquire Lock
        │
        ▼
   Redis GET again
        │
        ├── HIT
        │
        └── MISS
              │
              ▼
           Database
              │
              ▼
          Redis SET
              │
              ▼
         Release Lock
```

Why check Redis twice?

Because another request may have populated the cache while we were waiting for the lock.

Without the second check:

```text
Request A gets lock
Request B waits

Request A:
DB → Redis

Request A releases lock

Request B:
DB → Redis
```

Request B performs unnecessary database work.

With the second check:

```text
Request A:
DB → Redis

Request A releases lock

Request B:
Redis GET
   │
   └── HIT

Return immediately
```

This is called **double-checked locking** in this context.

---

# 6. Distributed Lock

A distributed lock allows only one request to rebuild a cache entry.

For example:

```text
             Redis
               │
        ┌──────┴──────┐
        │             │
     Lock Key      URL Cache
        │             │
        │             │
    url-lock:abc123   abc123
```

When a request gets a cache miss:

```text
SET url-lock:abc123 <unique-token> NX EX 5
```

Conceptually:

```text
NX = create only if key does not exist

EX 5 = automatically expire lock after 5 seconds
```

Only one request should successfully acquire the lock.

Example:

```text
Request A → LOCK → SUCCESS
Request B → LOCK → FAIL
Request C → LOCK → FAIL
Request D → LOCK → FAIL
```

Request A rebuilds the cache.

The other requests should not all immediately hit the database.

---

# 7. Why Lock Expiration Is Important

Never create a distributed lock without considering failure.

Imagine:

```text
Request A
   │
   ▼
Acquires lock
   │
   ▼
Database query
   │
   X
Process crashes
```

If the lock never expires:

```text
url-lock:abc123
```

could remain forever.

Then:

```text
Request B
   │
   ▼
Cannot acquire lock
   │
   ▼
Request blocked forever
```

Therefore distributed locks generally need a timeout.

Example:

```text
Lock TTL = 5 seconds
```

If the owner crashes:

```text
5 seconds
    │
    ▼
Lock automatically expires
```

Another request can recover.

---

# 8. Lock Ownership

A lock should ideally contain a unique value.

For example:

```text
url-lock:abc123 = request-7f82a
```

When releasing the lock, the application should make sure it is releasing **its own lock**.

Bad approach:

```text
DEL url-lock:abc123
```

Potential problem:

```text
Request A owns lock

Request A takes too long

Lock expires

Request B acquires lock

Request A finishes

Request A executes DEL

Request B's lock is deleted
```

This can create race conditions.

A safer design associates the lock with a unique owner token and releases it only if the stored value still matches that token.

---

# 9. Request Coalescing

Distributed locking is not the only solution.

Another approach is request coalescing.

The idea is:

```text
Many requests
      │
      ▼
One request performs database lookup
      │
      ▼
Other requests wait
      │
      ▼
Cache populated
      │
      ▼
All requests receive result
```

Instead of:

```text
Request A ──► DB
Request B ──► DB
Request C ──► DB
Request D ──► DB
```

we want:

```text
Request A ──► DB
Request B ──┐
Request C ──┤
Request D ──┤
             │
             ▼
          Result
```

This is often called:

```text
Single-flight
```

or:

```text
Request coalescing
```

The key idea is:

> For the same cache key, allow only one request to perform the expensive work.

---

# 10. Cache Penetration

Cache penetration is a different problem.

It happens when requests repeatedly ask for data that does not exist.

For example:

```text
/abc123 → exists
/xyz789 → does not exist
```

An attacker could generate:

```text
/random1
/random2
/random3
/random4
/random5
...
```

Every request produces:

```text
Redis MISS
     │
     ▼
Database
     │
     ▼
NOT FOUND
```

The database keeps receiving requests for data that does not exist.

---

# 11. Example of Cache Penetration

Suppose our database contains:

```text
abc123
def456
xyz789
```

An attacker sends:

```text
aaaaaa
bbbbbb
cccccc
dddddd
eeeeee
```

For each request:

```text
Redis:
MISS

Database:
NOT FOUND
```

Then another request:

```text
aaaaaa
```

again produces:

```text
Redis:
MISS

Database:
NOT FOUND
```

The application repeatedly asks the database the same question:

> Does this URL exist?

---

# 12. Solution: Negative Caching

We can cache the fact that a URL does not exist.

For example:

```text
short:url:aaaaaa → NOT_FOUND
```

with a short TTL.

Example:

```text
TTL = 30 seconds
```

Now:

```text
Request
   │
   ▼
Redis
   │
   └── NOT_FOUND
         │
         ▼
      Return 404
```

No database query is necessary.

This is called:

```text
Negative caching
```

---

# 13. Why Negative Cache TTL Should Be Short

Imagine a URL does not exist:

```text
abc123 → NOT_FOUND
```

We cache it for:

```text
24 hours
```

Later an administrator creates:

```text
abc123
```

But users may continue receiving:

```text
404
```

until the negative cache expires.

Therefore negative cache entries generally use relatively short TTLs.

For example:

```text
Normal URL cache:
1 hour

Negative cache:
30 seconds
```

The exact values depend on the application.

---

# 14. Bloom Filters

Another technique for cache penetration is a Bloom filter.

A Bloom filter can quickly answer:

```text
Could this key exist?
```

It does not necessarily tell us that the key definitely exists.

It can produce:

```text
Definitely does not exist
```

or:

```text
Maybe exists
```

Architecture:

```text
Request
   │
   ▼
Bloom Filter
   │
   ├── Definitely not present
   │        │
   │        ▼
   │       404
   │
   └── Maybe present
            │
            ▼
          Redis
            │
            ▼
         Database
```

This can protect the database from large volumes of obviously invalid requests.

---

# 15. Bloom Filter Limitation

Bloom filters have an important property:

They can have false positives.

Example:

```text
Bloom Filter:

abc123 → maybe exists
```

even if it does not actually exist.

But they should not produce false negatives when correctly maintained.

So:

```text
Definitely doesn't exist
```

is useful.

But:

```text
Maybe exists
```

still requires checking Redis/database.

---

# 16. Cache Breakdown

Cache breakdown usually refers to a very hot key expiring and causing many requests to hit the backend simultaneously.

Example:

```text
Popular URL:

abc123

Traffic:

100,000 requests/sec
```

Redis:

```text
abc123 → EXPIRED
```

Now:

```text
100,000 requests
       │
       ▼
   Cache MISS
       │
       ▼
   Database
```

This can overwhelm the database.

Cache stampede, cache breakdown, and thundering herd are sometimes used differently by different teams, but the underlying concern is similar:

> Too many requests suddenly bypass the cache and hit the origin.

---

# 17. Cache Avalanche

A cache avalanche happens when many cache entries expire or become unavailable around the same time.

For example:

```text
10:00:00

Redis:

URL A → expires
URL B → expires
URL C → expires
URL D → expires
URL E → expires
...
```

Suddenly:

```text
Thousands of keys
      │
      ▼
Cache MISS
      │
      ▼
Database
```

This is much broader than one hot key.

---

# 18. Why Fixed TTLs Can Cause Avalanche

Suppose an application loads 1 million URLs at:

```text
10:00 AM
```

and gives every cache entry:

```text
TTL = 1 hour
```

Then many keys may expire around:

```text
11:00 AM
```

This creates synchronized expiration.

Instead of:

```text
Requests distributed over time
```

we get:

```text
             11:00
               │
               ▼
        Massive expiration
               │
               ▼
          DB traffic spike
```

---

# 19. TTL Jitter

One simple solution is TTL jitter.

Instead of:

```text
TTL = 3600 seconds
```

for every key:

```text
TTL = 3600 + random(0, 300)
```

For example:

```text
URL A → 3682 sec
URL B → 3741 sec
URL C → 3615 sec
URL D → 3890 sec
URL E → 3664 sec
```

Now expiration is spread over time.

Instead of:

```text
11:00:00
11:00:00
11:00:00
11:00:00
```

we get:

```text
11:00:15
11:01:04
11:02:11
11:04:30
...
```

This reduces synchronized load.

---

# 20. Cache Avalanche From Redis Failure

An avalanche can also happen if Redis itself becomes unavailable.

Normal:

```text
Request
   │
   ▼
Redis
   │
   ▼
URL
```

Redis failure:

```text
Request
   │
   ▼
Redis ❌
   │
   ▼
Database
```

If traffic is high:

```text
100,000 requests/sec
        │
        ▼
      Redis ❌
        │
        ▼
100,000 DB requests/sec
```

This can cause database failure.

And then:

```text
Redis fails
    ↓
Database overloaded
    ↓
Database becomes slow
    ↓
Application requests become slow
    ↓
Connections accumulate
    ↓
Application becomes unhealthy
```

This is a cascading failure.

---

# 21. Fail-Open vs Fail-Closed

When Redis is unavailable, we need to decide what the application should do.

## Fail Open

Continue to the database.

```text
Redis unavailable
      │
      ▼
Database
```

This preserves correctness.

But it can overload the database.

## Fail Closed

Reject or degrade requests.

```text
Redis unavailable
      │
      ▼
Return temporary error
```

This protects the database.

But users may receive errors even though the URL exists.

For a URL shortener, the typical strategy is:

```text
Try Redis
   │
   ├── available → use Redis
   │
   └── unavailable
          │
          ▼
       Database
```

combined with database protection mechanisms.

---

# 22. Database Protection

If Redis fails, the database needs protection.

Possible techniques include:

```text
Connection Pool Limits
        │
        ▼
Rate Limiting
        │
        ▼
Circuit Breaker
        │
        ▼
Request Timeouts
        │
        ▼
Load Shedding
```

The goal is not simply:

> Keep accepting every request.

The goal is:

> Keep the overall system alive.

---

# 23. Circuit Breaker

A circuit breaker can detect that a dependency is unhealthy.

Example:

```text
Application
     │
     ▼
Database
     │
     X
  failing
```

After enough failures:

```text
Circuit OPEN
```

Requests can then fail fast instead of waiting for slow database responses.

Conceptually:

```text
Normal:

Request → DB


Failure:

Request → DB → timeout
Request → DB → timeout
Request → DB → timeout


Circuit opens:

Request → FAIL FAST
Request → FAIL FAST
Request → FAIL FAST
```

This prevents threads/connections from being consumed by repeated slow operations.

---

# 24. Stale-While-Revalidate

Another useful cache strategy is:

```text
Serve stale value
+
Refresh cache in background
```

Suppose:

```text
abc123 → https://example.com
```

is technically expired.

Instead of immediately making every request wait for a database query:

```text
Request
   │
   ▼
Stale cache
   │
   ├──► Return existing value
   │
   └──► Background refresh
```

The user gets a fast response.

Meanwhile:

```text
Background Worker
       │
       ▼
   Database
       │
       ▼
     Redis
```

This can dramatically reduce stampede pressure for data that can tolerate slightly stale reads.

For a URL mapping, stale data may often be acceptable depending on deletion/update semantics.

---

# 25. Cache Warming

Another technique is cache warming.

Before a large event:

```text
Marketing campaign
      │
      ▼
Expected popular URLs
```

We can preload important mappings:

```text
Database
    │
    ▼
Redis
```

before traffic arrives.

For example:

```text
Campaign starts at 10:00 AM

At 09:55:

Warm:
abc123
def456
xyz789
```

At 10:00:

```text
Millions of requests
        │
        ▼
      Redis
        │
        ▼
      HIT
```

This reduces the chance of a cold-cache event.

---

# 26. Hot-Key Protection

A hot key is a key receiving extremely high traffic.

Example:

```text
abc123
```

receives:

```text
500,000 requests/sec
```

Even Redis may need special consideration for extremely hot keys.

Possible techniques include:

```text
Local in-process cache
```

or:

```text
Multiple cache replicas
```

or:

```text
Request coalescing
```

For example:

```text
Application Instance 1
       │
       ▼
Local Cache
       │
       ▼
Redis
```

If the same URL is requested repeatedly by the same application instance, a very short-lived local cache can reduce Redis traffic.

---

# 27. Local Cache

A two-level cache can look like:

```text
             Application
                  │
                  ▼
            L1 Local Cache
                  │
             Cache Miss
                  │
                  ▼
            L2 Redis Cache
                  │
             Cache Miss
                  │
                  ▼
              Database
```

Where:

```text
L1 = in-memory application cache
L2 = distributed Redis cache
```

Example:

```text
L1 TTL = 5 seconds
L2 TTL = 1 hour
```

This can be very effective for extremely hot URLs.

However, local caches introduce another concern:

```text
Cache consistency
```

because every application instance may have a different copy.

---

# 28. Cache Invalidation

Suppose a short URL points to:

```text
https://example.com/page1
```

Then an administrator changes it:

```text
https://example.com/page2
```

Database:

```text
abc123 → page2
```

But Redis may still contain:

```text
abc123 → page1
```

Now users receive stale data.

One approach is:

```text
Update Database
      │
      ▼
Delete Redis Key
```

For example:

```text
DB UPDATE
   │
   ▼
Redis DEL abc123
```

The next request loads the new value.

---

# 29. Cache-Aside Pattern

The URL shortener can use the cache-aside pattern.

Read:

```text
GET
 │
 ▼
Cache
 │
 ├── HIT → return
 │
 └── MISS
       │
       ▼
    Database
       │
       ▼
     Cache
       │
       ▼
     Return
```

Write:

```text
Write Database
      │
      ▼
Invalidate Cache
```

This is simple and commonly used.

---

# 30. Protecting Against Cache Stampede

A production URL shortener can combine several techniques.

For example:

```text
                 Request
                    │
                    ▼
              L1 Local Cache
                    │
                MISS
                    │
                    ▼
               Redis Cache
                    │
                MISS
                    │
                    ▼
             Request Coalescing
                    │
                    ▼
              Distributed Lock
                    │
                    ▼
                Database
                    │
                    ▼
             Populate Redis
                    │
                    ▼
              Return Result
```

Not every system needs every layer.

The architecture should match the traffic and failure requirements.

---

# 31. Practical Strategy for Our URL Shortener

For our educational URL shortener, we can use the following approach.

## Normal request

```text
Request
   │
   ▼
Redis GET
   │
   └── HIT
         │
         ▼
      Redirect
```

## Cache miss

```text
Request
   │
   ▼
Redis MISS
   │
   ▼
Acquire lock
   │
   ▼
Check Redis again
   │
   ├── HIT → Return
   │
   └── MISS
         │
         ▼
      Database
         │
         ▼
      Redis SET
         │
         ▼
      Return
```

## Invalid URL

```text
Request
   │
   ▼
Redis
   │
   └── MISS
         │
         ▼
      Database
         │
         └── NOT FOUND
                │
                ▼
        Negative Cache
                │
                ▼
             404
```

---

# 32. Recommended TTL Strategy

There is no universal TTL.

For our example:

```text
URL mapping:

TTL = 1 hour
```

Negative cache:

```text
TTL = 30 seconds
```

Lock:

```text
TTL = 5 seconds
```

Local cache, if used:

```text
TTL = 5 seconds
```

These are example values, not production defaults.

Real TTLs should be selected based on:

```text
Traffic
Data update frequency
Consistency requirements
Database capacity
Cache capacity
Failure behavior
```

---

# 33. What Happens During a Redis Failure?

Let's walk through the complete failure.

Normal:

```text
             ┌─────────┐
Request ────►│  Redis  │
             └────┬────┘
                  │ HIT
                  ▼
               Redirect
```

Redis fails:

```text
             ┌─────────┐
Request ────►│  Redis  │ ❌
             └─────────┘
                  │
                  ▼
             Database
                  │
                  ▼
               Redirect
```

But now we need protection.

```text
                 Request
                    │
                    ▼
                 Redis
                    │
                 FAILURE
                    │
                    ▼
              Rate Limiter
                    │
                    ▼
             Connection Pool
                    │
                    ▼
                Database
```

This prevents unlimited database pressure.

---

# 34. Monitoring Cache Health

Caching is not something we should deploy and forget.

Important metrics include:

```text
Cache hit rate
Cache miss rate
Cache latency
Redis errors
Redis connection count
Redis memory usage
Evictions
Hot keys
Database QPS
Database latency
Cache stampede events
Lock contention
Lock wait time
```

For example:

```text
Cache Hit Rate:

99%
```

might be healthy.

But suddenly:

```text
99% → 70% → 30%
```

could indicate:

```text
Redis failure
TTL problem
Eviction problem
Deployment issue
Traffic pattern change
Cache invalidation bug
```

---

# 35. Cache Hit Rate

One of the most important metrics is:

```text
Cache Hit Rate
```

Formula:

```text
Cache Hit Rate =
Cache Hits / Total Cache Requests
```

Example:

```text
Cache Hits = 990,000
Total Requests = 1,000,000
```

Therefore:

```text
Hit Rate = 99%
```

A high hit rate means fewer requests reach the database.

But:

> High cache hit rate alone does not guarantee a healthy system.

Latency, memory pressure, evictions, and dependency health also matter.

---

# 36. Detecting a Stampede

Suppose we monitor:

```text
Redis MISS rate
Database QPS
Database latency
```

Normally:

```text
Redis MISS = 1%
Database QPS = 1,000
```

Suddenly:

```text
Redis MISS = 40%
Database QPS = 40,000
```

This is a strong signal of cache trouble.

We can alert on:

```text
Cache miss rate > threshold
```

combined with:

```text
Database QPS increase
```

rather than relying on only one metric.

---

# 37. The Important Principle

A cache should improve the system.

It should not become a single point of failure that takes the whole system down.

The architecture should therefore assume:

```text
Cache can fail.
```

And design accordingly:

```text
Cache failure
     │
     ▼
Controlled degradation
     │
     ▼
Database protected
     │
     ▼
Application remains available
```

---

# 38. Complete Read Path

Putting everything together:

```text
                         Request
                            │
                            ▼
                    ┌───────────────┐
                    │ L1 Local Cache│
                    └───────┬───────┘
                            │
                          MISS
                            │
                            ▼
                    ┌───────────────┐
                    │ Redis / L2    │
                    └───────┬───────┘
                            │
                     ┌──────┴──────┐
                     │             │
                    HIT           MISS
                     │             │
                     ▼             ▼
                  Redirect   Request Coalescing
                                  │
                                  ▼
                            Distributed Lock
                                  │
                                  ▼
                            Double Check
                                  │
                             ┌────┴────┐
                             │         │
                            HIT       MISS
                             │         │
                             ▼         ▼
                          Return    Database
                                       │
                                       ▼
                                  Populate Redis
                                       │
                                       ▼
                                    Return
```

For invalid URLs:

```text
Database
   │
   └── NOT FOUND
          │
          ▼
   Negative Cache
          │
          ▼
        404
```

---

# 39. What We Should Actually Build

We should not implement every advanced technique immediately.

For our educational project, a good progression is:

### Version 1

```text
Database
+
Redis
+
Cache Aside
```

### Version 2

Add:

```text
TTL
+
Negative Caching
```

### Version 3

Add:

```text
Distributed Lock
+
Double-Checked Cache
```

### Version 4

Add:

```text
Request Coalescing
```

### Version 5

Add:

```text
Metrics
+
Failure Simulation
+
Load Testing
```

This lets us demonstrate why each technique exists rather than blindly adding complexity.

---

# 40. Important System Design Lesson

The important lesson is not:

> Use Redis.

The real lesson is:

> Every optimization introduces new failure modes.

Without Redis:

```text
Database
   │
   └── Simple
```

With Redis:

```text
Application
    │
    ├── Local Cache
    │
    ├── Redis
    │
    └── Database
```

Now we have additional concerns:

```text
Cache miss
Cache expiration
Cache invalidation
Cache failure
Cache stampede
Cache penetration
Cache avalanche
Hot keys
Stale data
Lock contention
```

A good system designer considers these problems before they happen in production.

---

# 41. Summary

We learned several important cache failure patterns.

### Cache Stampede

Many requests rebuild the same expired cache entry.

```text
Cache expires
     ↓
Many requests
     ↓
Database overload
```

### Thundering Herd

Many requests compete for the same resource simultaneously.

### Cache Penetration

Requests repeatedly query data that does not exist.

Solution:

```text
Negative caching
Bloom filters
```

### Cache Breakdown

A hot cache entry expires and causes a large backend spike.

Solution:

```text
Distributed lock
Request coalescing
Stale-while-revalidate
Cache warming
```

### Cache Avalanche

Many cache entries expire or become unavailable around the same time.

Solution:

```text
TTL jitter
Cache warming
Graceful degradation
```

### Redis Failure

The database may suddenly receive the entire traffic load.

Solution:

```text
Timeouts
Connection limits
Circuit breakers
Rate limiting
Load shedding
```

The overall principle is:

```text
Cache should absorb traffic.

Cache failure should NOT become
database failure.
```

---

# 42. The Next Problem

Our URL shortener can now handle:

```text
URL Creation
        ↓
Short Code Generation
        ↓
Database Mapping
        ↓
Redis Caching
        ↓
Cache Failures
        ↓
Async Analytics
```

But we have not yet discussed one of the most important scaling problems:

> **How do we handle millions or billions of URLs and redirect requests across multiple servers and database instances?**

At that point we need to think about:

```text
Database Scaling
        ↓
Read Replicas
        ↓
Database Sharding
        ↓
Consistent Hashing
        ↓
Redis Cluster
        ↓
Horizontal Scaling
        ↓
Load Balancing
        ↓
Hot Partition Problems
```

The next chapter will focus on **database scaling and sharding for a URL shortener**.
