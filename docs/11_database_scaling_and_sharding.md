# URL Shortener — Database Scaling and Sharding

Our URL shortener currently has a simple architecture:

```text
Client
  │
  ▼
Application
  │
  ├──► Redis
  │
  └──► Database
```

This works very well for an initial system.

But eventually the system can grow.

For example:

```text
100 million URLs
1 billion URLs
10 billion redirects
Millions of requests per second
```

At that point, a single database may become a bottleneck.

The question becomes:

> How do we scale the database when one database server is no longer enough?

This chapter explains read replicas, database partitioning, sharding, consistent hashing, hot partitions, and how these concepts apply to a production URL shortener.

---

# 1. Why Does the Database Become a Problem?

A URL shortener has two major operations.

## Create a Short URL

```text
POST /shorten

Long URL
   │
   ▼
Generate short code
   │
   ▼
Database INSERT
```

## Redirect

```text
GET /abc123

Short code
   │
   ▼
Find mapping
   │
   ▼
Long URL
   │
   ▼
HTTP Redirect
```

The redirect operation is much more frequent.

For example:

```text
100,000 URL creations/day

10,000,000 redirects/day
```

The ratio can be heavily read-oriented.

Therefore the database architecture should optimize for:

```text
Very high read volume
+
Moderate write volume
```

---

# 2. First Scaling Step: Add Redis

Before immediately sharding the database, we should ask:

> Are we unnecessarily hitting the database?

Suppose we have:

```text
1,000,000 redirect requests
```

If Redis has a:

```text
99% cache hit rate
```

then approximately:

```text
990,000 requests
```

can be served from Redis.

Only approximately:

```text
10,000 requests
```

need to reach the database.

Architecture:

```text
                    Requests
                       │
                       ▼
                    Redis
                       │
                 ┌─────┴─────┐
                 │           │
                HIT         MISS
                 │           │
                 ▼           ▼
              Redirect    Database
```

This is why caching should generally be considered before database sharding.

---

# 3. What If Redis Is Not Enough?

Suppose traffic continues increasing.

We might eventually have:

```text
Redis
  │
  ▼
Database
```

with:

```text
Database CPU = 90%
Database connections = 95%
Database latency = increasing
```

Now we need to scale the database.

There are two broad approaches:

```text
Vertical Scaling
Horizontal Scaling
```

---

# 4. Vertical Scaling

Vertical scaling means making the database server bigger.

For example:

```text
Before:

8 CPU
32 GB RAM
500 GB SSD
```

Upgrade to:

```text
32 CPU
128 GB RAM
2 TB SSD
```

Advantages:

```text
Simple
Easy to operate
Minimal application changes
```

Disadvantages:

```text
Hardware limits
Expensive at large scale
Single-machine bottleneck
Potentially larger failure domain
```

Vertical scaling is often a good first step.

We should not introduce distributed complexity before we need it.

---

# 5. Horizontal Scaling

Horizontal scaling means adding more database servers.

Instead of:

```text
One Database
```

we have:

```text
Database 1
Database 2
Database 3
Database 4
```

Now the challenge is:

> How do we decide which database handles which request?

There are two major strategies:

```text
Read Replication
+
Sharding
```

They solve different problems.

---

# 6. Read Replicas

Read replicas copy data from a primary database.

Architecture:

```text
                 Application
                      │
             ┌────────┴────────┐
             │                 │
           Writes            Reads
             │                 │
             ▼                 ▼
        ┌──────────┐    ┌─────────────┐
        │ Primary  │───►│ Read Replica│
        └──────────┘    └─────────────┘
```

For multiple replicas:

```text
                  Primary
                 /   |   \
                /    |    \
               ▼     ▼     ▼
             R1      R2     R3
```

The primary handles writes.

Replicas handle reads.

---

# 7. Read Replicas for URL Shorteners

A redirect is primarily a read operation.

So we could have:

```text
Create URL
    │
    ▼
Primary DB
```

while:

```text
Redirect
    │
    ▼
Read Replica
```

Architecture:

```text
                 URL Service
                      │
             ┌────────┴────────┐
             │                 │
           Write              Read
             │                 │
             ▼                 ▼
         Primary DB       Read Replicas
                            │  │  │
                            ▼  ▼  ▼
                           R1 R2 R3
```

This can significantly increase read capacity.

---

# 8. Replication Lag

Read replicas introduce an important problem:

> Replicas may not contain the latest data immediately.

Suppose:

```text
T1:
Create abc123

Primary:
abc123 → example.com
```

The replica may still be:

```text
Replica:
abc123 → NOT FOUND
```

for a short period.

This is called:

```text
Replication Lag
```

---

# 9. Read-After-Write Problem

Consider this flow:

```text
User creates:

abc123
```

The write goes to:

```text
Primary
```

Immediately afterward:

```text
User clicks:

abc123
```

The redirect request goes to:

```text
Read Replica
```

If replication has not completed:

```text
Replica → NOT FOUND
```

The user may receive:

```text
404
```

even though the URL was just created successfully.

---

# 10. Handling Read-After-Write

There are several strategies.

## Strategy 1: Read From Primary

Immediately after creation:

```text
Create → Primary
Redirect → Primary
```

This guarantees the latest value but increases primary read traffic.

## Strategy 2: Temporary Cache

After creating:

```text
Write DB
   │
   ▼
Redis
```

Then the immediate redirect can hit Redis.

```text
Redirect
   │
   ▼
Redis HIT
```

This is especially useful for a URL shortener.

## Strategy 3: Session/Request Routing

Temporarily route the user's reads to the primary after a write.

The correct choice depends on consistency requirements.

---

# 11. When Read Replicas Are Not Enough

Suppose we have:

```text
10 billion URL mappings
```

Even with replicas, every replica may need to contain:

```text
10 billion rows
```

That creates challenges around:

```text
Storage
Indexes
Backups
Replication traffic
Recovery time
Query performance
```

At this point we may need:

```text
Sharding
```

---

# 12. What Is Database Sharding?

Sharding means splitting data across multiple database servers.

Instead of:

```text
Database
├── 10 billion URLs
```

we can have:

```text
Shard 1
├── URLs

Shard 2
├── URLs

Shard 3
├── URLs

Shard 4
├── URLs
```

Each shard stores only part of the total dataset.

This is horizontal partitioning across independent database nodes.

---

# 13. Why Sharding Helps

Without sharding:

```text
All URLs
   │
   ▼
One Database
```

With sharding:

```text
                 URL Service
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
       Shard 1    Shard 2    Shard 3
```

Now:

```text
Shard 1 → handles subset of URLs
Shard 2 → handles subset of URLs
Shard 3 → handles subset of URLs
```

Storage and query workload are distributed.

---

# 14. Choosing a Shard Key

The most important design decision is:

> What determines which shard contains a URL?

Possible shard keys:

```text
Numeric ID
Short code
Hash of short code
User ID
Tenant ID
```

For a URL shortener, the short code is a natural candidate.

For example:

```text
abc123
```

We can calculate:

```text
hash("abc123")
```

and use that to determine the shard.

---

# 15. Hash-Based Sharding

A simple approach is:

```text
shard = hash(short_code) % number_of_shards
```

Suppose:

```text
Number of shards = 4
```

Then:

```text
hash("abc123") % 4 = 0
```

Therefore:

```text
abc123 → Shard 0
```

Another URL:

```text
xyz789
```

might produce:

```text
hash("xyz789") % 4 = 2
```

Therefore:

```text
xyz789 → Shard 2
```

Architecture:

```text
                    URL Service
                         │
                         ▼
                  Shard Router
                         │
             ┌───────────┼───────────┐
             │           │           │
             ▼           ▼           ▼
          Shard 0     Shard 1     Shard 2
```

---

# 16. The Problem With Modulo Sharding

Modulo hashing looks simple.

But consider:

```text
4 shards
```

with:

```text
hash(key) % 4
```

Now we add another shard:

```text
5 shards
```

The formula becomes:

```text
hash(key) % 5
```

Many keys will map to different shards.

For example:

```text
Before:

abc123 → Shard 2
```

After adding a shard:

```text
abc123 → Shard 4
```

But the data is still on Shard 2.

Now the application needs to move a huge amount of data.

This is called:

```text
Resharding
```

---

# 17. What Is Consistent Hashing?

Consistent hashing is a technique designed to reduce the amount of data that needs to move when nodes are added or removed.

Instead of:

```text
hash(key) % N
```

we map nodes and keys onto a logical hash ring.

Conceptually:

```text
                  Hash Ring

               ┌────────────┐
            ┌──►   Shard 1  ├──┐
            │  └────────────┘  │
            │                  │
            │                  ▼
       Shard 0               Shard 2
            ▲                  │
            │                  │
            └──────────────────┘
```

Keys are also mapped onto this ring.

Each key belongs to the next node clockwise.

---

# 18. Why Consistent Hashing Helps

Suppose we have:

```text
Shard A
Shard B
Shard C
```

Then add:

```text
Shard D
```

With a good consistent hashing strategy, only a portion of keys need to move.

Instead of:

```text
Almost everything moves
```

we get:

```text
Only affected key ranges move
```

This is especially useful in distributed caching and sharded systems.

---

# 19. Virtual Nodes

Real systems often use virtual nodes.

Instead of putting one physical server on the hash ring:

```text
Shard A
```

we represent it with multiple positions:

```text
A1
A2
A3
A4
A5
...
```

For example:

```text
Hash Ring

A1
      B1
            A2
                  C1
        B2
  A3
             C2
```

This improves distribution.

Without virtual nodes, one physical node could accidentally own a very large portion of the hash space.

---

# 20. Sharding by Short Code

For our URL shortener, the logical architecture can become:

```text
                    Request
                       │
                       ▼
                 URL Service
                       │
                       ▼
                 Shard Router
                       │
             ┌─────────┼─────────┐
             │         │         │
             ▼         ▼         ▼
          Shard 1   Shard 2   Shard 3
```

The router calculates:

```text
hash(short_code)
```

and determines the correct shard.

For:

```text
/abc123
```

the application can directly identify the database containing that mapping.

---

# 21. Why We Don't Want to Query Every Shard

A bad architecture would be:

```text
Request /abc123

      │
      ├──► Shard 1
      ├──► Shard 2
      ├──► Shard 3
      ├──► Shard 4
      └──► Shard 5
```

This is called a scatter-gather pattern.

It creates unnecessary work.

Instead:

```text
/abc123
   │
   ▼
Shard Router
   │
   ▼
Shard 3
```

Only one shard is queried.

This is one of the main benefits of a good shard key.

---

# 22. Hot Partition Problem

Sharding does not automatically guarantee even traffic.

Suppose:

```text
Shard 1 → 25%
Shard 2 → 25%
Shard 3 → 25%
Shard 4 → 25%
```

Looks perfect.

But then a viral URL happens to belong to:

```text
Shard 2
```

Now:

```text
Shard 2 → 80% traffic
```

while:

```text
Shard 1 → 5%
Shard 3 → 5%
Shard 4 → 10%
```

This is a:

```text
Hot Partition
```

or:

```text
Hot Shard
```

---

# 23. Why URL Shorteners Are Vulnerable to Hot Keys

URL shorteners can have extremely uneven access patterns.

For example:

```text
abc123 → 10 million requests
def456 → 5 million requests
xyz789 → 1,000 requests
```

A single popular URL can become a hot key.

This is why Redis caching is so important.

If:

```text
abc123
```

is served from Redis:

```text
Millions of requests
       │
       ▼
     Redis
       │
       X
    Database
```

the database shard does not receive all that traffic.

---

# 24. Sharding + Redis

A scalable architecture may look like:

```text
                         Clients
                            │
                            ▼
                     Load Balancer
                            │
                            ▼
                      URL Service
                            │
                            ▼
                         Redis
                            │
                    ┌───────┴───────┐
                    │               │
                   HIT             MISS
                    │               │
                    ▼               ▼
                 Redirect      Shard Router
                                   │
                      ┌────────────┼────────────┐
                      ▼            ▼            ▼
                   Shard 1      Shard 2      Shard 3
```

The cache absorbs most hot-key traffic.

The database handles cache misses.

---

# 25. Should Redis Also Be Sharded?

At sufficiently large scale, yes.

A single Redis server may eventually become a bottleneck.

We could have:

```text
Redis Cluster

Node 1
Node 2
Node 3
Node 4
...
```

Keys are distributed across Redis nodes.

Conceptually:

```text
              Redis Cluster
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      Redis 1    Redis 2    Redis 3
```

This allows Redis capacity and throughput to scale horizontally.

---

# 26. Database Sharding vs Redis Sharding

These solve different problems.

## Database Sharding

Distributes the persistent dataset.

```text
10 billion URLs
       │
       ▼
Shard 1 + Shard 2 + Shard 3
```

## Redis Sharding

Distributes cached data.

```text
Huge cache
   │
   ▼
Redis 1 + Redis 2 + Redis 3
```

A production system may use both.

---

# 27. Replicas Per Shard

We can combine sharding and replication.

For example:

```text
                   Shard 1
                  /       \
                 ▼         ▼
            Primary       Replica


                   Shard 2
                  /       \
                 ▼         ▼
            Primary       Replica


                   Shard 3
                  /       \
                 ▼         ▼
            Primary       Replica
```

This gives us:

```text
Sharding
+
Read scaling
+
Failure tolerance
```

---

# 28. Complete Scaled Architecture

A more advanced URL shortener can look like:

```text
                         Internet
                            │
                            ▼
                     Load Balancer
                            │
                 ┌──────────┼──────────┐
                 │          │          │
                 ▼          ▼          ▼
              App 1      App 2      App 3
                 │          │          │
                 └──────────┼──────────┘
                            │
                            ▼
                     Redis Cluster
                            │
                     Cache Miss
                            │
                            ▼
                      Shard Router
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
          Shard 1        Shard 2        Shard 3
           /   \          /   \          /   \
          P     R        P     R        P     R
```

Where:

```text
P = Primary
R = Replica
```

This architecture can scale far beyond a single database.

---

# 29. How URL Creation Works With Sharding

Suppose the user submits:

```text
https://example.com/very/long/url
```

The application generates:

```text
abc123
```

Then:

```text
hash("abc123")
```

determines:

```text
Shard 2
```

The write flow becomes:

```text
Client
  │
  ▼
URL Service
  │
  ▼
Generate short code
  │
  ▼
Shard Router
  │
  ▼
Shard 2 Primary
  │
  ▼
Store mapping
  │
  ▼
Redis
```

---

# 30. How Redirect Works

For:

```text
GET /abc123
```

the application first checks Redis.

```text
Request
   │
   ▼
Redis
   │
   ├── HIT
   │    │
   │    ▼
   │  Redirect
   │
   └── MISS
        │
        ▼
    Shard Router
        │
        ▼
     Shard 2
        │
        ▼
     Long URL
        │
        ▼
      Redis
        │
        ▼
     Redirect
```

The shard is determined from the same short code.

---

# 31. The Importance of a Stable Routing Function

There is one critical requirement:

```text
Create URL
```

and:

```text
Read URL
```

must use the same routing logic.

For example:

```text
Shard = route(short_code)
```

must produce the same shard for the same short code.

Otherwise:

```text
Create:
abc123 → Shard 2
```

but:

```text
Read:
abc123 → Shard 4
```

would fail.

Therefore shard-routing logic should be centralized.

---

# 32. Shard Metadata

A mature system may maintain metadata describing the shard layout.

For example:

```text
Shard Map

hash range A → Shard 1
hash range B → Shard 2
hash range C → Shard 3
```

The application or routing layer uses this information to determine where data lives.

When topology changes:

```text
Shard 4 added
```

the shard map can be updated.

---

# 33. Resharding

Eventually we may need more capacity.

For example:

```text
Current:

Shard 1
Shard 2
Shard 3
```

Traffic grows.

We add:

```text
Shard 4
```

Now some existing data needs to move.

This is:

```text
Resharding
```

The process must be carefully designed.

We should consider:

```text
Data movement
Application routing
Consistency
Downtime
Replication
Backfill
Validation
Rollback
```

---

# 34. Online Resharding

A production system should ideally avoid:

```text
Stop application
Move all data
Restart application
```

Instead, we want an online process.

Conceptually:

```text
Old Shard
    │
    │ Copy data
    ▼
New Shard
    │
    │ Validate
    ▼
Switch routing
    │
    ▼
New requests → New Shard
```

This is much more complex but allows the system to remain available.

---

# 35. Do We Actually Need Sharding?

This is an important system-design question.

The answer is:

> Not necessarily.

Suppose our system has:

```text
10 million URLs
100,000 redirects/day
```

Sharding would probably be unnecessary complexity.

A much simpler architecture could be:

```text
Application
    │
    ▼
Redis
    │
    ▼
PostgreSQL/MySQL
```

Even at larger scale, read replicas and good indexing may be sufficient.

---

# 36. When Should We Consider Sharding?

Sharding becomes more attractive when:

```text
Database storage is approaching limits
```

or:

```text
Single-node CPU is insufficient
```

or:

```text
IOPS are insufficient
```

or:

```text
Replication becomes expensive
```

or:

```text
Query workload cannot be handled by replicas
```

or:

```text
Dataset is too large for practical single-node operation
```

Sharding should solve a demonstrated bottleneck.

---

# 37. Scaling Strategy

A sensible evolution might be:

### Stage 1

```text
Single Database
```

### Stage 2

```text
Database
+
Indexes
+
Redis
```

### Stage 3

```text
Primary
+
Read Replicas
+
Redis
```

### Stage 4

```text
Redis Cluster
+
Database Sharding
+
Replicas
```

### Stage 5

```text
Multiple Regions
+
Global Routing
+
Geo-Distributed Data
```

Do not jump directly to Stage 5.

---

# 38. Indexing Still Matters

Sharding does not eliminate the need for good database indexes.

For example:

```sql
SELECT long_url
FROM urls
WHERE short_code = ?;
```

The database should have an index on:

```text
short_code
```

Ideally:

```text
UNIQUE(short_code)
```

This allows efficient lookup.

A bad design would be:

```text
Full table scan
```

for every redirect.

Even a sharded database can become slow if each shard performs expensive queries.

---

# 39. Data Model

A simple table might look like:

```text
urls

id
short_code
long_url
created_at
expires_at
user_id
status
```

Important index:

```text
UNIQUE(short_code)
```

Potential indexes:

```text
user_id
created_at
expires_at
```

depending on query patterns.

The key principle is:

> Index based on actual access patterns.

---

# 40. Why the Short Code Is a Good Lookup Key

Our redirect request is:

```text
/abc123
```

Therefore the system already has:

```text
abc123
```

This means we do not need a secondary lookup to determine the mapping.

We can directly use:

```text
short_code
```

for:

```text
Cache key
Database lookup
Shard routing
```

For example:

```text
short_code = abc123

       │
       ├──► Redis key
       │
       ├──► Database index
       │
       └──► Shard routing
```

This is a clean design.

---

# 41. Avoiding Cross-Shard Queries

Cross-shard queries are expensive.

Suppose analytics asks:

```text
How many URLs did user 123 create?
```

If URLs are sharded by short code, the user's URLs may exist across many shards.

The application may need:

```text
Shard 1 ──┐
Shard 2 ──┤
Shard 3 ──┤──► Aggregate
Shard 4 ──┤
Shard 5 ──┘
```

This can become expensive.

Therefore transactional URL lookup and analytics queries should often be separated.

This is one reason our previous chapter introduced:

```text
Asynchronous Analytics
```

---

# 42. Separate Transactional and Analytical Workloads

The redirect path should remain simple:

```text
Short Code
   │
   ▼
Redis
   │
   ▼
Database
   │
   ▼
Redirect
```

Analytics should be asynchronous:

```text
Redirect
   │
   ├──► User redirect
   │
   └──► Analytics event
             │
             ▼
           Queue
             │
             ▼
        Analytics DB
```

This prevents analytical queries from interfering with URL resolution.

---

# 43. Failure of One Shard

Suppose:

```text
Shard 2 ❌
```

What happens?

If there is no replica:

```text
Requests → Shard 2
             │
             X
```

Some URLs become unavailable.

With replication:

```text
Shard 2 Primary ❌
        │
        ▼
Shard 2 Replica
```

Traffic can be redirected to the replica.

This provides fault tolerance.

---

# 44. What About Redis Failure?

Our previous chapter covered this.

The overall strategy becomes:

```text
Redis available
    │
    ▼
Serve cache
```

If Redis fails:

```text
Redis ❌
   │
   ▼
Database
```

But database capacity must be protected with:

```text
Rate limiting
Connection limits
Circuit breakers
Timeouts
Load shedding
```

This demonstrates an important architecture principle:

> Every layer needs a failure strategy.

---

# 45. Multi-Region Scaling

At very large scale, one geographic region may not be enough.

We could eventually have:

```text
                   Global Users
                        │
                        ▼
                  Global Router
                  /           \
                 ▼             ▼
              US Region     Asia Region
                 │             │
                 ▼             ▼
             Redis + DB     Redis + DB
```

This introduces much harder problems:

```text
Cross-region replication
Data ownership
Consistency
Failover
Latency
Conflict resolution
Traffic routing
Disaster recovery
```

We should not introduce multi-region architecture unless the business actually requires it.

---

# 46. The Main Scaling Principle

The URL shortener should scale in layers.

```text
                  Traffic
                     │
                     ▼
              Load Balancer
                     │
                     ▼
             Application Servers
                     │
                     ▼
                Redis Cache
                     │
                     ▼
               Read Replicas
                     │
                     ▼
                Sharded DB
```

Each layer solves a different bottleneck.

```text
Load Balancer
→ Application scaling

Redis
→ Read latency + database load

Read Replicas
→ Read scaling

Sharding
→ Dataset + database capacity

Replication
→ Availability
```

---

# 47. A Common Mistake

A common system-design mistake is saying:

> "We have billions of URLs, so let's use sharding."

That is incomplete.

The correct reasoning is:

```text
Traffic
  │
  ▼
Measure bottleneck
  │
  ▼
Can caching solve it?
  │
  ▼
Can read replicas solve it?
  │
  ▼
Can vertical scaling solve it?
  │
  ▼
If not
  │
  ▼
Consider sharding
```

Architecture should follow the bottleneck.

Not the other way around.

---

# 48. Final Architecture

For a large-scale URL shortener, we can now visualize the system as:

```text
                           INTERNET
                              │
                              ▼
                       Global / DNS LB
                              │
                              ▼
                        Load Balancer
                              │
                 ┌────────────┼────────────┐
                 │            │            │
                 ▼            ▼            ▼
              App 1        App 2        App 3
                 │            │            │
                 └────────────┼────────────┘
                              │
                              ▼
                       Redis Cluster
                              │
                       ┌──────┴──────┐
                       │             │
                      HIT           MISS
                       │             │
                       ▼             ▼
                   Redirect     Shard Router
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
                 Shard 1          Shard 2          Shard 3
                 /     \          /     \          /     \
                P       R        P       R        P       R
                │       │        │       │        │       │
                └───────┴────────┴───────┴────────┴───────┘
                              │
                              ▼
                       Async Analytics
                              │
                              ▼
                            Queue
                              │
                              ▼
                        Analytics DB
```

This architecture separates:

```text
Request handling
Caching
Persistent storage
Database scaling
High availability
Analytics
```

---

# 49. What We Have Learned

We started with:

```text
Application
   │
   ▼
Database
```

Then introduced:

```text
Redis
```

Then:

```text
Read Replicas
```

Then:

```text
Sharding
```

Then:

```text
Replication per shard
```

And finally:

```text
Redis Cluster
+
Sharded Database
+
Async Analytics
```

Each step solves a specific problem.

---

# 50. Final Takeaway

A scalable URL shortener is not just:

```text
Short URL → Database → Redirect
```

A production architecture must answer:

```text
How many requests can we handle?

Where is the data stored?

How do we reduce database reads?

What happens when Redis fails?

What happens when the database fails?

How do we scale reads?

How do we scale storage?

How do we distribute data?

How do we avoid hot partitions?

How do we add new shards?

How do we recover from failures?
```

The important system-design lesson is:

> **Scale the component that is actually becoming the bottleneck, and introduce distributed-system complexity only when the simpler architecture is no longer sufficient.**

---

# 51. The Next Problem

We now have a highly scalable storage architecture.

But there is another important problem.

Imagine:

```text
10 million URL creations/day
```

and every application server generates short codes independently.

How do we guarantee:

```text
abc123
```

is not generated twice?

And how do we generate short codes efficiently across:

```text
100 application servers
```

without:

```text
Database bottlenecks
Collisions
Global locks
Duplicate IDs
```

This brings us to the next major system-design topic:

```text
Distributed ID Generation
```

We will compare:

```text
Auto Increment IDs
UUID
Random IDs
Hash IDs
Snowflake IDs
Pre-generated ID blocks
Database sequences
Distributed ID generators
```

and design a scalable short-code generation system for our URL shortener.
