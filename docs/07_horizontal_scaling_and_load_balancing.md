# URL Shortener — Horizontal Scaling and Load Balancing

## 1. Introduction

Our URL Shortener has now evolved through several stages.

Initially:

```text
Client
  |
  v
Application
  |
  v
Database
```

Then we introduced Redis:

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

This solves an important problem:

> We no longer need to query the database for every redirect.

But another problem appears when traffic grows.

What happens if one application server receives:

```text
10,000 requests/sec
```

or:

```text
50,000 requests/sec
```

?

At some point, one application server will become the bottleneck.

We need a way to run multiple application instances.

This introduces:

* Horizontal scaling
* Load balancing
* Stateless application servers
* Health checks
* Failover
* Connection pooling
* High availability

---

# 2. Vertical vs Horizontal Scaling

There are two common ways to scale an application.

## Vertical Scaling

Make the existing server bigger.

For example:

```text
Before:

4 CPU
8 GB RAM

        ↓

After:

16 CPU
32 GB RAM
```

This is called:

> Vertical Scaling

or:

> Scale Up

---

# 3. Horizontal Scaling

Instead of making one server bigger, add more servers.

For example:

```text
Before:

Application 1
```

becomes:

```text
Application 1
Application 2
Application 3
Application 4
```

This is called:

> Horizontal Scaling

or:

> Scale Out

---

# 4. Why Horizontal Scaling?

Suppose one server can handle:

```text
5,000 requests/sec
```

and our traffic becomes:

```text
15,000 requests/sec
```

Instead of requiring one extremely large server:

```text
15,000 req/sec
      |
      v
Huge Server
```

we can use:

```text
15,000 req/sec
      |
      v
Load Balancer
      |
      +---- 5,000 → Server 1
      +---- 5,000 → Server 2
      +---- 5,000 → Server 3
```

This provides both:

* More capacity
* Better availability

---

# 5. The Problem With Multiple Servers

Suppose we simply add:

```text
Application 1
Application 2
Application 3
```

How does the client know which server to call?

We do not want the client to decide.

Instead, we introduce:

> Load Balancer

Architecture:

```text
Client
   |
   v
Load Balancer
   |
   +---- Application 1
   |
   +---- Application 2
   |
   +---- Application 3
```

The load balancer receives the request and chooses an application server.

---

# 6. What Does a Load Balancer Do?

A load balancer sits between clients and application servers.

Its basic responsibility is:

```text
Receive request
      |
      v
Choose healthy application server
      |
      v
Forward request
      |
      v
Return response
```

For example:

```text
GET /aB72x
     |
     v
Load Balancer
     |
     v
Application 2
```

The client does not need to know that Application 2 handled the request.

---

# 7. Updated Architecture

Our system becomes:

```text
                         Client
                           |
                           v
                    ┌──────────────┐
                    │Load Balancer │
                    └──────┬───────┘
                           |
              ┌────────────┼────────────┐
              |            |            |
              v            v            v
          App Server 1 App Server 2 App Server 3
              |            |            |
              └────────────┼────────────┘
                           |
                    ┌──────┴──────┐
                    |             |
                    v             v
                  Redis        Database
```

This is our first horizontally scalable architecture.

---

# 8. Stateless Application Servers

There is an important requirement when using multiple application servers:

> Application servers should be stateless whenever possible.

A stateless server does not depend on local memory or local filesystem state to process the next request.

For example:

```text
Request 1 → Server 1

Request 2 → Server 3

Request 3 → Server 2
```

All servers should be capable of handling all requests.

---

# 9. Why Statelessness Matters

Imagine:

```text
Server 1
Memory:
aB72x → URL A
```

but:

```text
Server 2
Memory:
aB72x → missing
```

Now the behavior depends on which server receives the request.

This makes horizontal scaling difficult.

Instead, shared state should live in shared infrastructure:

```text
Application Servers
      |
      +---- Redis
      |
      +---- Database
```

---

# 10. What State Should Not Live on the Application Server?

Avoid relying on local memory for important persistent state.

For example:

```text
URL mappings
User sessions
Rate-limit counters
Distributed locks
Important application state
```

Instead, use appropriate shared systems.

For our URL Shortener:

```text
URL mappings
      ↓
Database

Cache
      ↓
Redis
```

---

# 11. Local Memory Cache

We could still use a small local memory cache for optimization.

For example:

```text
Application 1
    |
    +---- Local Cache
    |
    +---- Redis
```

But local cache is not authoritative.

If Server 1 restarts:

```text
Local cache disappears
```

That is acceptable because:

```text
Redis → shared cache
Database → source of truth
```

For our first implementation, we will avoid local caching and keep the architecture simple.

---

# 12. Load Balancing Algorithms

A load balancer needs a strategy for choosing servers.

Several algorithms are commonly used.

---

# 13. Round Robin

The simplest strategy is:

```text
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1
Request 5 → Server 2
Request 6 → Server 3
```

This is called:

> Round Robin

Diagram:

```text
             Request
                |
                v
         Load Balancer
                |
       ┌────────┼────────┐
       v        v        v
     App 1    App 2    App 3
       ↑                 |
       └─────────────────┘
```

It works well when servers have similar capacity and requests have relatively similar cost.

---

# 14. Weighted Round Robin

Suppose:

```text
Server 1 = powerful
Server 2 = medium
Server 3 = small
```

We could assign:

```text
Server 1 → weight 3
Server 2 → weight 2
Server 3 → weight 1
```

Traffic might approximately become:

```text
Server 1 → 50%
Server 2 → 33%
Server 3 → 17%
```

This is called:

> Weighted Round Robin

---

# 15. Least Connections

Another strategy is:

> Send the request to the server with the fewest active connections.

For example:

```text
Server 1 → 100 connections
Server 2 →  40 connections
Server 3 →  70 connections
```

The next request may go to:

```text
Server 2
```

This can be useful when request durations vary significantly.

---

# 16. IP Hash

Another approach is to use the client's IP address to determine the server.

Conceptually:

```text
hash(client_ip) → server
```

This can provide some session affinity.

However, we should not rely on IP-based routing for application correctness.

Our application should remain stateless.

---

# 17. Do We Need Sticky Sessions?

Sticky sessions mean:

```text
User A → Always Server 1
```

for the duration of a session.

This can simplify some stateful applications.

But sticky sessions create problems:

* Uneven traffic
* Harder failover
* Reduced flexibility
* Server dependency

For our URL Shortener:

> We do not need sticky sessions.

Any server should be able to handle any request.

---

# 18. Why Our URL Shortener Is Naturally Stateless

Consider:

```text
GET /aB72x
```

The application only needs:

```text
short_code
```

Then it looks up:

```text
Redis
```

or:

```text
Database
```

There is no requirement that:

```text
Request 2
```

must go to the same server as:

```text
Request 1
```

Therefore:

```text
Request 1 → App 1
Request 2 → App 3
Request 3 → App 2
```

all work correctly.

This is exactly what we want.

---

# 19. Health Checks

What happens if:

```text
Application 2
```

crashes?

The load balancer should detect that the server is unhealthy.

This is done through:

> Health Checks

For example:

```text
GET /health
```

Expected response:

```text
HTTP 200
```

If the server stops responding correctly:

```text
Application 2
      |
      X
```

the load balancer removes it from the traffic pool.

---

# 20. Health Check Architecture

```text
                 Load Balancer
                      |
        ┌─────────────┼─────────────┐
        |             |             |
        v             v             v
     App 1          App 2         App 3
      UP             DOWN           UP
        |                           |
        +---------------------------+
                    |
               Receive traffic
```

Traffic should go only to:

```text
App 1
App 3
```

until App 2 becomes healthy again.

---

# 21. Health Endpoint

Our application can expose:

```text
GET /health
```

For example:

```json
{
  "status": "ok"
}
```

This endpoint should be lightweight.

It should not perform expensive business operations.

---

# 22. Liveness vs Readiness

In production container orchestration systems, two concepts are particularly useful.

### Liveness

Answers:

> Is the application process alive?

Example:

```text
GET /health/live
```

### Readiness

Answers:

> Is the application ready to receive traffic?

Example:

```text
GET /health/ready
```

A process may be alive but not ready.

For example:

```text
Application started
       |
       v
Loading configuration
       |
       v
Connecting to dependencies
       |
       v
Ready
```

Only after it is ready should it receive production traffic.

---

# 23. What Should Readiness Check?

This depends on architecture.

For example:

```text
Application
    |
    +---- Redis
    |
    +---- Database
```

A readiness check might verify critical dependencies.

However, we should be careful.

If the readiness endpoint requires every dependency to be perfectly healthy, a temporary dependency problem can cause every application instance to be removed from the load balancer.

Therefore:

> Health-check design should reflect actual application requirements.

For our educational project, we will start with a simple application-level health endpoint.

---

# 24. Failure Scenario

Suppose we have:

```text
App 1 → Healthy
App 2 → Healthy
App 3 → Healthy
```

Traffic:

```text
1000 req/sec
```

Now App 2 crashes.

The load balancer detects:

```text
App 2 → Unhealthy
```

Traffic becomes:

```text
App 1 → ~500 req/sec
App 3 → ~500 req/sec
App 2 → 0 req/sec
```

The system continues operating.

This is one of the primary benefits of horizontal scaling.

---

# 25. Scaling Up

Suppose:

```text
Current traffic = 10,000 req/sec
```

and we have:

```text
2 application servers
```

Each server handles:

```text
5,000 req/sec
```

Now traffic becomes:

```text
20,000 req/sec
```

We can add:

```text
Application 3
Application 4
```

Architecture:

```text
Load Balancer
      |
      +---- App 1
      +---- App 2
      +---- App 3
      +---- App 4
```

This is horizontal scaling.

---

# 26. Auto Scaling

In a real production environment, we may not want to manually add servers.

Instead, infrastructure can scale based on metrics.

For example:

```text
CPU > 70%
       |
       v
Add instance
```

or:

```text
Requests per second > threshold
       |
       v
Add instance
```

When traffic decreases:

```text
CPU < 30%
       |
       v
Remove instance
```

This is:

> Auto Scaling

---

# 27. Statelessness Makes Auto Scaling Easier

Imagine the application server stores important state locally.

Then removing a server could destroy important data.

But if:

```text
Application = stateless
```

we can safely:

```text
Add Server
Remove Server
Replace Server
Restart Server
```

because the important state is stored externally.

Our architecture follows:

```text
Application
   ↓
Stateless

Redis
   ↓
Shared Cache

Database
   ↓
Persistent Data
```

---

# 28. Connection Pooling

Horizontal scaling introduces another problem.

Every application server needs connections to:

```text
Redis
Database
```

Suppose:

```text
10 application servers
```

and each server opens:

```text
100 database connections
```

Then:

```text
10 × 100 = 1,000 DB connections
```

The database may become overloaded even if CPU usage is low.

Therefore:

> Connection pooling is important.

---

# 29. Database Connection Pool

Instead of opening a new database connection for every request:

```text
Request
   |
   v
Create DB connection
   |
   v
Query
   |
   v
Close connection
```

we maintain a pool:

```text
Application
     |
     v
Connection Pool
  |   |   |   |
  v   v   v   v
 DB connections
```

Requests reuse existing connections.

---

# 30. Why Connection Pooling Matters

Creating database connections has overhead.

A connection pool:

```text
Request
   |
   v
Borrow connection
   |
   v
Execute query
   |
   v
Return connection
```

is generally much more efficient.

But the pool must be configured carefully.

Too few connections:

```text
Requests wait
```

Too many connections:

```text
Database overloaded
```

---

# 31. Pool Size and Number of Servers

Suppose:

```text
20 application servers
```

and:

```text
50 DB connections/server
```

Then:

```text
20 × 50 = 1,000 possible connections
```

If the database only safely supports:

```text
500
```

connections, the architecture is incorrectly configured.

This is an important system-design lesson:

> Scaling application servers also scales dependency connections.

---

# 32. Redis Connections

The same principle applies to Redis.

Suppose:

```text
20 application servers
```

all connect to Redis.

We need to manage:

```text
Redis connections
Connection pools
Timeouts
Retries
```

The goal is to avoid creating excessive connection overhead.

---

# 33. Timeouts

Distributed systems must assume that network calls can fail or become slow.

For example:

```text
Application
    |
    | Redis request
    v
Redis
```

If Redis stops responding, the application should not wait forever.

We need timeouts.

Conceptually:

```text
Redis timeout = 100 ms
```

If the request exceeds the timeout:

```text
Redis call fails
       |
       v
Fallback / error handling
```

The exact timeout depends on the workload and environment.

---

# 34. Retries

Suppose Redis temporarily fails.

We might retry:

```text
Attempt 1
   ↓
Failure

Attempt 2
   ↓
Success
```

But retries must be bounded.

Bad design:

```text
Retry forever
```

This can make an outage worse.

A safer approach is:

```text
Timeout
+
Limited retries
+
Backoff
```

---

# 35. Retry Storm

Imagine:

```text
10,000 requests
```

all fail against Redis.

If every request immediately retries:

```text
10,000 × retry
```

the Redis server receives another huge burst.

This can create a:

> Retry Storm

Therefore, retries must be used carefully.

---

# 36. Exponential Backoff

One common strategy is:

```text
Retry 1 → wait 10 ms
Retry 2 → wait 20 ms
Retry 3 → wait 40 ms
```

The delay increases between attempts.

This reduces synchronized retry pressure.

We do not need a complex retry framework for our initial implementation, but the concept is important for production system design.

---

# 37. Load Balancer Failure

We have discussed application failure.

But what if the load balancer itself fails?

If there is only one load balancer:

```text
Internet
   |
   v
Load Balancer
   X
```

the entire application becomes unreachable.

Therefore, production load balancers are generally deployed with high availability.

Conceptually:

```text
                Internet
                   |
            Load Balancer Layer
              /            \
             /              \
           LB 1            LB 2
             \              /
              \            /
               Applications
```

The exact implementation depends on the cloud provider or infrastructure platform.

---

# 38. High Availability

High availability means:

> Avoid having a single component whose failure takes down the entire service.

We should examine every major component:

```text
Load Balancer
Application Servers
Redis
Database
```

For example:

```text
1 application server
```

is a single point of failure.

Better:

```text
3 application servers
```

Similarly:

```text
single Redis instance
```

may become a single point of failure.

Later we will discuss Redis replication and failover.

---

# 39. Database Is Now the Next Bottleneck

Our architecture is currently:

```text
Load Balancer
      |
      +---- App 1
      +---- App 2
      +---- App 3
              |
          ┌───┴────┐
          |        |
        Redis    Database
```

Redis handles most redirect reads.

But the database still handles:

* URL creation
* Cache misses
* Updates
* Administrative operations
* Cleanup
* Other future features

As the dataset grows, database scaling becomes the next challenge.

---

# 40. Read Replicas

One possible solution is database read replicas.

Architecture:

```text
                  Application
                       |
              ┌────────┴────────┐
              |                 |
              v                 v
           Primary          Read Replica
              |
              v
           Writes
```

But we must understand an important issue:

> Replicas can introduce replication lag.

For our URL Shortener, that matters because a newly created URL may not immediately appear on a replica.

Therefore, read-replica architecture needs careful read/write routing.

We will cover this later.

---

# 41. Why Redis Still Matters With Read Replicas

Read replicas do not replace caching.

Consider:

```text
100,000 redirects/sec
```

Even with multiple replicas:

```text
100K requests
     |
     v
Read replicas
```

we still perform database work for every request.

With Redis:

```text
100K requests
     |
     v
Redis
     |
     +---- Most requests served
     |
     +---- Small percentage → Database
```

Caching and database replication solve different problems.

---

# 42. Application Deployment

Horizontal scaling also changes how we deploy the application.

Instead of manually configuring:

```text
Server 1
Server 2
Server 3
```

we want repeatable deployments.

A common approach is:

```text
Source Code
    |
    v
Build
    |
    v
Container Image
    |
    v
Deploy
    |
    v
Multiple Instances
```

This ensures every application instance runs the same version.

---

# 43. Immutable Application Instances

Ideally:

```text
Application Image v1
```

is identical across all instances.

For example:

```text
App 1 → v1
App 2 → v1
App 3 → v1
```

When deploying v2:

```text
App 1 → v2
App 2 → v2
App 3 → v2
```

This avoids configuration drift.

---

# 44. Rolling Deployment

We can deploy gradually.

For example:

```text
Before:

App 1 → v1
App 2 → v1
App 3 → v1
```

Deploy v2:

```text
App 1 → v2
App 2 → v1
App 3 → v1
```

Then:

```text
App 1 → v2
App 2 → v2
App 3 → v1
```

Finally:

```text
App 1 → v2
App 2 → v2
App 3 → v2
```

This is commonly called a:

> Rolling Deployment

---

# 45. Graceful Shutdown

When removing an application server, we should not abruptly terminate it while requests are being processed.

A graceful shutdown looks like:

```text
Server receives shutdown signal
        |
        v
Stop accepting new traffic
        |
        v
Finish existing requests
        |
        v
Close connections
        |
        v
Exit
```

This prevents unnecessary failed requests during deployments or scaling events.

---

# 46. Request Flow in the Scaled System

Let's follow a real redirect.

User requests:

```text
https://short.ly/aB72x
```

### Step 1

Client sends request.

```text
Client
  |
  v
Load Balancer
```

### Step 2

Load balancer selects:

```text
Application 2
```

### Step 3

Application checks Redis:

```text
url:aB72x
```

### Step 4

Redis returns:

```text
https://example.com/products/iphone
```

### Step 5

Application returns:

```text
HTTP 302
```

No database query occurs.

---

# 47. Cache Miss in the Scaled System

Suppose:

```text
url:xY92p
```

is not cached.

Flow:

```text
Client
  |
  v
Load Balancer
  |
  v
Application 3
  |
  v
Redis MISS
  |
  v
Database
  |
  v
Original URL
  |
  v
Redis SET
  |
  v
Redirect
```

The next request can be served from Redis regardless of which application server receives it.

That is the key benefit of a shared cache.

---

# 48. Why Shared Redis Is Important

Without shared Redis:

```text
App 1 → Local Cache
App 2 → Local Cache
App 3 → Local Cache
```

With shared Redis:

```text
App 1 ──┐
App 2 ──┼──> Redis
App 3 ──┘
```

Now:

```text
App 1 populates cache
```

and:

```text
App 3 can use the same cache
```

This makes horizontal scaling much more efficient.

---

# 49. Current Architecture

Our architecture has now evolved significantly:

```text
                         Internet
                            |
                            v
                    ┌──────────────┐
                    │Load Balancer │
                    └──────┬───────┘
                           |
             ┌─────────────┼─────────────┐
             |             |             |
             v             v             v
          App 1          App 2         App 3
             |             |             |
             └─────────────┼─────────────┘
                           |
                  ┌────────┴────────┐
                  |                 |
                  v                 v
                Redis            Database
                Cache            Source of Truth
```

---

# 50. Responsibilities

## Load Balancer

Responsible for:

```text
Traffic distribution
Health checks
Failover
TLS termination
```

depending on the infrastructure implementation.

---

## Application Servers

Responsible for:

```text
API logic
URL generation
Validation
Cache-aside logic
Redirect handling
```

They should remain stateless.

---

## Redis

Responsible for:

```text
Fast URL lookups
Cache
TTL
Reducing database reads
```

---

## Database

Responsible for:

```text
Persistent URL mappings
Data integrity
Unique short codes
Durable storage
```

---

# 51. What This Architecture Solves

We have solved several problems.

### Problem 1 — Database read pressure

Solution:

```text
Redis
```

### Problem 2 — One application server bottleneck

Solution:

```text
Horizontal scaling
```

### Problem 3 — Application server failure

Solution:

```text
Multiple instances
+
Health checks
```

### Problem 4 — Uneven traffic

Solution:

```text
Load balancer
```

### Problem 5 — Server-local state

Solution:

```text
Stateless application
+
Shared Redis
+
Database
```

---

# 52. What This Architecture Does Not Solve

We still have important challenges.

### Challenge 1

How do we generate short codes efficiently?

```text
Millions of URLs
```

### Challenge 2

What happens when the database becomes too large?

```text
100M+
1B+
records
```

### Challenge 3

What happens if Redis fails?

### Challenge 4

How do we prevent abuse?

### Challenge 5

How do we collect analytics at scale?

### Challenge 6

How do we handle extremely high traffic?

These lead to the next stages of the design.

---

# 53. Important System Design Lesson

A common mistake is to jump directly to:

```text
Kubernetes
Kafka
Redis Cluster
Sharding
Microservices
```

without identifying the problem they solve.

We are intentionally doing the opposite.

We started with:

```text
Simple Application
```

Then:

```text
Database
```

Then:

```text
Redis
```

Then:

```text
Multiple Application Servers
```

Each component was introduced because a specific bottleneck appeared.

This is the correct way to reason about scalable architecture.

---

# 54. Scaling Decision Tree

Our design process can be summarized as:

```text
Traffic increases
      |
      v
One application server overloaded?
      |
      +---- YES
      |      |
      |      v
      |  Add instances
      |      |
      |      v
      |  Load Balancer
      |
      v
Database reads too high?
      |
      +---- YES
      |      |
      |      v
      |    Redis
      |
      v
Database itself overloaded?
      |
      +---- YES
             |
             v
       Read Replicas /
       Partitioning /
       Sharding
```

The architecture should evolve based on measured bottlenecks.

---

# 55. Production Principle

One of the most important principles from this chapter is:

> **Scale stateless compute horizontally and keep shared state in appropriate external systems.**

For our URL Shortener:

```text
Compute
   ↓
Stateless application servers

Cache
   ↓
Redis

Persistent data
   ↓
Database
```

This separation allows each layer to scale independently.

---

# 56. Chapter Summary

Our URL Shortener started as:

```text
Client
  |
  v
Application
  |
  v
Database
```

Then became:

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

Now it has evolved into:

```text
                         Internet
                            |
                            v
                    ┌──────────────┐
                    │Load Balancer │
                    └──────┬───────┘
                           |
             ┌─────────────┼─────────────┐
             |             |             |
             v             v             v
          App 1          App 2         App 3
             |             |             |
             └─────────────┼─────────────┘
                           |
                  ┌────────┴────────┐
                  |                 |
                  v                 v
                Redis            Database
                Cache            Source
                               of Truth
```

The key concepts introduced are:

```text
Vertical Scaling
Horizontal Scaling
Load Balancing
Round Robin
Weighted Round Robin
Least Connections
Stateless Architecture
Health Checks
Liveness
Readiness
Connection Pooling
Timeouts
Retries
Retry Storms
High Availability
Auto Scaling
Rolling Deployment
Graceful Shutdown
```

The next major bottleneck is now the **database itself**.

If our URL Shortener grows from:

```text
1 million URLs
```

to:

```text
100 million
```

or:

```text
1 billion+
```

we need to think carefully about:

```text
Database capacity
Read replicas
Write scaling
Partitioning
Sharding
```

Before jumping to sharding, however, we need to solve another fundamental problem:

> **How should we generate unique short codes efficiently at large scale, especially when multiple application servers are creating URLs concurrently?**

That takes us to the next chapter: **ID generation and short-code generation strategies**.
