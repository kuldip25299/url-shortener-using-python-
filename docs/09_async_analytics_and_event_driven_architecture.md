# URL Shortener — Asynchronous Analytics and Event-Driven Architecture

## 1. Introduction

Our URL Shortener has now reached an important stage.

We currently have:

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

The redirect path is optimized:

```text
Request
   |
   v
Redis
   |
   v
Redirect
```

This is fast.

But now we want to add analytics.

For every click, we may want to record:

```text
Short code
Timestamp
IP address
Country
Device
Browser
Operating system
Referrer
User agent
```

This creates a new system-design problem.

---

# 2. The Naive Analytics Approach

The simplest implementation is:

```text
User
  |
  v
GET /aB72x
  |
  v
Redis
  |
  v
Analytics DB
  |
  v
Redirect
```

For every redirect:

```text
1. Resolve URL
2. Store analytics
3. Redirect user
```

This looks simple.

But it creates a serious problem.

---

# 3. Redirects Can Be Extremely High Volume

Suppose our system receives:

```text
100,000 redirects/sec
```

If every redirect performs an analytics database write:

```text
100,000 requests/sec
        |
        v
100,000 DB writes/sec
```

The analytics workload can become the bottleneck.

Even worse, the user is waiting for the redirect.

---

# 4. Analytics Should Not Slow Down Redirects

The most important requirement is:

> **Analytics should not unnecessarily increase redirect latency.**

The ideal flow is:

```text
User
  |
  v
Application
  |
  +---- Redis lookup
  |
  +---- Publish analytics event
  |
  v
Redirect immediately
```

The analytics processing happens separately.

Conceptually:

```text
                    Application
                    /          \
                   /            \
                  v              v
              Redis          Event Queue
                                |
                                v
                         Analytics Workers
                                |
                                v
                         Analytics Storage
```

This is the beginning of:

> Event-Driven Architecture

---

# 5. Synchronous vs Asynchronous Processing

## Synchronous

Everything happens in the request path.

```text
Request
   |
   v
Resolve URL
   |
   v
Write Analytics
   |
   v
Redirect
```

The request waits for analytics processing.

---

## Asynchronous

The application publishes an event.

```text
Request
   |
   v
Resolve URL
   |
   v
Publish Event
   |
   v
Redirect
```

Then:

```text
Event
  |
  v
Queue
  |
  v
Worker
  |
  v
Analytics DB
```

The redirect does not need to wait for the analytics database.

---

# 6. What Is an Event?

An event is a message describing something that happened.

For example:

```json
{
  "event_type": "url_clicked",
  "short_code": "aB72x",
  "timestamp": "2026-09-02T10:00:00Z"
}
```

We can add more information:

```json
{
  "event_type": "url_clicked",
  "short_code": "aB72x",
  "timestamp": "2026-09-02T10:00:00Z",
  "ip": "203.0.113.10",
  "user_agent": "Mozilla/5.0",
  "referrer": "https://google.com"
}
```

The application publishes this event.

Another component processes it.

---

# 7. Event Producer

The application generating the event is called the:

> Producer

In our system:

```text
Application
     |
     v
Analytics Event
```

The application produces:

```text
url_clicked
```

events.

---

# 8. Event Consumer

The component that receives and processes the event is called the:

> Consumer

For example:

```text
Queue
  |
  v
Analytics Worker
```

The worker consumes:

```text
url_clicked
```

events and stores analytics.

---

# 9. Message Queue

Between the producer and consumer we introduce a message broker or queue.

Architecture:

```text
Application
     |
     v
Message Queue
     |
     v
Analytics Worker
     |
     v
Analytics Database
```

The queue acts as a buffer.

---

# 10. Why Do We Need a Queue?

Suppose traffic suddenly increases.

Normal traffic:

```text
10,000 clicks/sec
```

Peak traffic:

```text
100,000 clicks/sec
```

Without a queue:

```text
Application
     |
     v
Analytics DB
```

The database may immediately become overloaded.

With a queue:

```text
Application
     |
     v
Queue
     |
     v
Workers
     |
     v
Analytics DB
```

The queue can temporarily absorb the traffic spike.

---

# 11. Queue as a Buffer

Imagine:

```text
Producer:
100,000 events/sec
```

but workers can process:

```text
80,000 events/sec
```

The remaining:

```text
20,000 events/sec
```

can accumulate in the queue.

Conceptually:

```text
Producer
   |
   | 100K/sec
   v
┌─────────────┐
│    Queue    │
│             │
│ 20K pending │
└──────┬──────┘
       |
       | 80K/sec
       v
   Consumers
```

The system temporarily absorbs the difference.

---

# 12. Queue Does Not Create Infinite Capacity

A queue is not magic.

If:

```text
Producer = 100K/sec
Consumer = 50K/sec
```

then backlog grows:

```text
50K events/sec
```

Eventually the queue can become too large.

Therefore, queue depth must be monitored.

A queue helps absorb temporary spikes.

It does not solve a permanent capacity mismatch.

---

# 13. Eventual Consistency

Once analytics becomes asynchronous, analytics data may not be immediately available.

For example:

```text
10:00:00
User clicks URL

10:00:00.001
Redirect returned

10:00:00.050
Analytics worker processes event

10:00:00.100
Analytics dashboard updated
```

There is a small delay.

This is:

> Eventual Consistency

For analytics, this is usually acceptable.

The redirect itself must be fast.

---

# 14. Redirect Data vs Analytics Data

We should separate two types of data.

## Critical URL Mapping

```text
short_code
original_url
```

This is required to perform the redirect.

Therefore:

```text
Database
+
Redis
```

are part of the critical path.

---

## Analytics

Examples:

```text
click count
country
browser
device
referrer
```

These are not required to perform the redirect.

Therefore:

```text
Analytics
```

can be processed asynchronously.

This distinction is extremely important.

---

# 15. Updated Architecture

Our system becomes:

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
             └──────┬──────┼──────┬──────┘
                    |      |      |
                    v      v      v
                  Redis  Queue
                    |      |
                    |      v
                    |   Workers
                    |      |
                    v      v
                 URL DB  Analytics DB
```

The redirect path is:

```text
Application
    |
    +---- Redis → URL
    |
    +---- Queue → Analytics
```

---

# 16. Redirect Request Flow

Suppose:

```text
GET /aB72x
```

arrives.

The application performs:

```text
1. Extract short code
2. Check Redis
3. Resolve original URL
4. Publish click event
5. Return redirect
```

Conceptually:

```text
GET /aB72x
      |
      v
   Redis
      |
      v
Original URL
      |
      +------> Analytics Event
      |
      v
 HTTP 302
```

The analytics pipeline is separate from the redirect response.

---

# 17. What Should the Event Contain?

A useful event might contain:

```json
{
  "event_id": "evt_123456",
  "event_type": "url_clicked",
  "short_code": "aB72x",
  "timestamp": "2026-09-02T10:00:00Z",
  "ip": "203.0.113.10",
  "user_agent": "Mozilla/5.0",
  "referrer": "https://google.com"
}
```

Depending on requirements, we may also include:

```text
country
region
device_type
browser
os
campaign
user_id
```

But we should avoid collecting unnecessary personal data.

---

# 18. Event ID

Every event should ideally have an identifier:

```text
event_id
```

For example:

```text
evt_01J8XYZ...
```

Why?

Because distributed systems can deliver the same event more than once.

The consumer may need to identify duplicates.

This introduces:

> Idempotent event processing

---

# 19. At-Least-Once Delivery

Many messaging systems prioritize:

> At-least-once delivery

Meaning:

```text
The system tries to ensure the event is not lost,
but an event may occasionally be delivered more than once.
```

For example:

```text
Event A
   |
   v
Worker
   |
   v
Database write
   |
   X
Worker crashes before acknowledging
   |
   v
Queue delivers Event A again
```

Now the worker sees:

```text
Event A
Event A
```

This is normal in distributed systems.

---

# 20. Duplicate Analytics

Suppose a click event is:

```text
evt_123
```

and it is processed twice.

Without protection:

```text
evt_123 → +1 click
evt_123 → +1 click
```

The dashboard shows:

```text
2 clicks
```

even though there was only:

```text
1 click
```

Therefore, the analytics consumer should be designed carefully.

---

# 21. Idempotent Consumer

One approach is to maintain:

```text
event_id UNIQUE
```

in the analytics database.

Conceptually:

```text
Process event
     |
     v
Have we processed event_id?
     |
     +---- YES → Ignore
     |
     +---- NO
            |
            v
        Store event
```

This is another application of the idempotency concept we studied in our previous project.

---

# 22. Event Processing Flow

The worker can follow:

```text
Receive Event
     |
     v
Check event_id
     |
     +---- Already processed
     |          |
     |          v
     |        Ignore
     |
     +---- New Event
                |
                v
          Process Event
                |
                v
          Store Analytics
                |
                v
              ACK
```

The exact order of database write and acknowledgment depends on the messaging system and desired delivery semantics.

---

# 23. Why Acknowledgment Matters

The queue needs to know whether the consumer successfully processed a message.

Conceptually:

```text
Queue
  |
  v
Worker
  |
  +---- Success → ACK
  |
  +---- Failure → Retry
```

If the worker crashes:

```text
Queue
  |
  v
Worker
  X
```

the message should become available for processing again.

---

# 24. Failed Events

Suppose an analytics worker receives:

```text
Event A
```

but the database is temporarily unavailable.

We do not want to lose the event.

Instead:

```text
Event
  |
  v
Worker
  |
  X
Database failure
  |
  v
Retry
```

After several failed attempts, the event may be moved to a:

> Dead Letter Queue

or:

> DLQ

---

# 25. Dead Letter Queue

Architecture:

```text
Main Queue
    |
    v
Worker
    |
    +---- Success → ACK
    |
    +---- Repeated failure
              |
              v
          Dead Letter Queue
```

The DLQ stores messages that could not be processed successfully.

Operations teams can then investigate them.

---

# 26. Why DLQ Is Important

Suppose one malformed event causes:

```text
Worker
   |
   v
FAIL
   |
   v
Retry
   |
   v
FAIL
   |
   v
Retry
```

If the message keeps retrying forever, it can block or waste resources.

A DLQ allows us to isolate problematic events.

---

# 27. Retry Strategy

A basic retry strategy might be:

```text
Attempt 1
   ↓
Failure
   ↓
Wait
   ↓
Attempt 2
   ↓
Failure
   ↓
Wait longer
   ↓
Attempt 3
```

The delay can use exponential backoff.

For example:

```text
1 second
2 seconds
4 seconds
8 seconds
```

with appropriate limits.

---

# 28. Why Not Retry Immediately?

Imagine the analytics database is down.

If 10,000 events all retry immediately:

```text
10,000 requests
      |
      v
Database
      X
```

Then all 10,000 retry again immediately.

This can make the outage worse.

Backoff spreads retries over time.

---

# 29. Multiple Consumers

One worker may not be enough.

Suppose:

```text
Queue
  |
  v
Worker 1
```

can process:

```text
10,000 events/sec
```

but traffic is:

```text
50,000 events/sec
```

We can add:

```text
Queue
  |
  +---- Worker 1
  +---- Worker 2
  +---- Worker 3
  +---- Worker 4
  +---- Worker 5
```

Now processing capacity can increase horizontally.

---

# 30. Consumer Group

In systems that support consumer groups, workers can share the workload.

Conceptually:

```text
                  Queue
                    |
          ┌─────────┼─────────┐
          |         |         |
          v         v         v
       Worker 1  Worker 2  Worker 3
```

Each event is processed by one worker in the group.

This allows horizontal scaling.

---

# 31. Partitioning

At very large scale, queues may be partitioned.

For example:

```text
Topic: url_clicks

Partition 0
Partition 1
Partition 2
Partition 3
```

Events are distributed across partitions.

Consumers can process partitions in parallel.

This architecture is common in high-throughput event-streaming systems.

---

# 32. Ordering

Sometimes we care about event ordering.

Suppose:

```text
Event A
Event B
Event C
```

for the same short URL.

We may want:

```text
A → B → C
```

to be processed in order.

But global ordering across millions of events is expensive.

A better approach may be:

> Preserve ordering only where it matters.

For example:

```text
Partition by short_code
```

so events for the same URL go to the same partition.

Then:

```text
aB72x → Partition 1
xY92p → Partition 2
qK82z → Partition 3
```

This allows parallel processing while maintaining per-key ordering.

---

# 33. Event Streaming vs Simple Queue

There are different messaging models.

### Simple Queue

Good for:

```text
Task processing
Background jobs
Work queues
```

### Event Streaming Platform

Better when we need:

```text
Very high throughput
Partitions
Consumer groups
Event replay
Long-lived event streams
Multiple independent consumers
```

Examples of technologies in the industry include:

```text
Kafka
Amazon Kinesis
Google Pub/Sub
RabbitMQ
SQS
```

Each has different semantics and operational characteristics.

---

# 34. We Should Not Choose Technology First

For this educational project, we should not start with:

```text
"We need Kafka."
```

Instead ask:

```text
What problem do we have?
```

Our problem is:

```text
Redirect traffic is high
Analytics processing should not block redirects
```

Therefore:

```text
Asynchronous event processing
```

is the architectural solution.

Then we can evaluate technologies.

---

# 35. Analytics Storage

Where should analytics data go?

It depends on how we query it.

For example:

```text
How many clicks today?
```

or:

```text
Clicks by country
```

or:

```text
Clicks by browser
```

or:

```text
Clicks per hour for the last 30 days
```

These are analytical queries.

A traditional transactional database may not always be the best long-term storage system for massive event volumes.

---

# 36. Raw Events vs Aggregated Data

We can store:

### Raw events

```text
Event 1
Event 2
Event 3
...
```

This provides maximum flexibility.

But storage grows quickly.

Alternatively, we can aggregate:

```text
2026-09-02
aB72x
clicks = 102,432
```

This reduces storage.

But we lose some raw detail.

---

# 37. Hybrid Analytics Architecture

A scalable architecture might be:

```text
Click Event
     |
     v
Message Queue
     |
     v
Stream Processor
     |
     +------------+
     |            |
     v            v
Raw Storage   Aggregated DB
```

Raw events can be retained for future analysis.

Aggregated data can power dashboards efficiently.

---

# 38. Pre-Aggregation

Suppose we receive:

```text
100,000 clicks/sec
```

Instead of querying 100,000 raw records every time someone opens the dashboard, we can maintain counters.

For example:

```text
short_code = aB72x
date = 2026-09-02
hour = 13
clicks = 12450
```

Then:

```text
Dashboard
   |
   v
Aggregated Analytics DB
```

can return the result quickly.

---

# 39. Counter-Based Analytics

A simple architecture could maintain:

```text
clicks:aB72x:20260902
```

and increment:

```text
INCR clicks:aB72x:20260902
```

Redis is extremely good at counters.

However, Redis alone should not automatically become our permanent analytics store.

If analytics data is important, we need a durable storage strategy.

---

# 40. Redis for Real-Time Counters

A useful architecture can be:

```text
Click Event
     |
     v
Queue
     |
     v
Worker
     |
     +---- Redis → Real-time counter
     |
     +---- Persistent Analytics Storage
```

This allows:

```text
Dashboard
    |
    v
Redis
    |
    v
Very recent statistics
```

while durable storage provides long-term history.

---

# 41. Critical Path vs Non-Critical Path

Our system can now be divided into two paths.

## Critical Path

```text
User
  |
  v
Load Balancer
  |
  v
Application
  |
  v
Redis
  |
  v
Redirect
```

This path must be extremely fast.

---

## Asynchronous Path

```text
Application
     |
     v
Event Queue
     |
     v
Workers
     |
     v
Analytics Storage
```

This path can tolerate small delays.

This separation is one of the most important architectural improvements in our URL Shortener.

---

# 42. Failure Isolation

Suppose the analytics database goes down.

With synchronous analytics:

```text
Redirect
   |
   v
Analytics DB
   X
   |
   v
Redirect fails or becomes slow
```

This is bad.

With asynchronous analytics:

```text
Redirect
   |
   +---- Redis → Redirect
   |
   +---- Queue → Analytics DB
                         X
```

The redirect can continue working.

Events accumulate in the queue until the analytics system recovers.

This is called:

> Failure Isolation

---

# 43. Backpressure

Suppose:

```text
Producer:
100K events/sec

Consumers:
60K events/sec
```

The queue starts growing.

This tells us:

```text
Processing capacity < incoming rate
```

The system needs more workers.

We can scale:

```text
2 workers
   ↓
5 workers
   ↓
10 workers
```

This is one form of handling:

> Backpressure

---

# 44. Monitoring Queue Lag

Important metrics include:

```text
Queue depth
Consumer lag
Events/sec
Processing latency
Failed events
Retry count
DLQ size
```

For example:

```text
Queue depth:
10,000 → healthy

Queue depth:
10,000,000 → investigate
```

A growing queue is an early warning that consumers cannot keep up.

---

# 45. Analytics Processing Latency

Suppose:

```text
Event generated:
10:00:00

Event processed:
10:00:03
```

Processing latency:

```text
3 seconds
```

This is useful to monitor.

For an analytics system, a few seconds may be perfectly acceptable.

For another system, it might not be.

The requirement determines the architecture.

---

# 46. Event Schema Versioning

Events can evolve.

Version 1:

```json
{
  "event_type": "url_clicked",
  "short_code": "aB72x"
}
```

Later we add:

```text
country
device
browser
```

Now consumers must handle both versions.

Therefore, event schemas should be versioned carefully.

For example:

```json
{
  "event_type": "url_clicked",
  "version": 2,
  "short_code": "aB72x"
}
```

This makes future evolution easier.

---

# 47. Do Not Put Huge Payloads in Events

An event should contain the information needed by consumers.

Avoid putting unnecessarily large data into every message.

For example:

```text
Huge HTML document
Large image
Complete request body
```

should generally not be embedded into a click event.

Instead, store large data separately and send a reference if needed.

Smaller events improve:

* Throughput
* Network efficiency
* Queue storage
* Consumer performance

---

# 48. Privacy Considerations

Analytics can contain sensitive information.

Potentially sensitive fields include:

```text
IP address
User ID
Location
User agent
Referrer
```

We should follow the application's privacy and legal requirements.

Possible techniques include:

```text
Data minimization
IP truncation/anonymization
Retention policies
Access controls
Encryption
```

The educational implementation should avoid collecting data that is not necessary to demonstrate the architecture.

---

# 49. Retention

Analytics data does not necessarily need to live forever.

For example:

```text
Raw events:
30 days

Aggregated statistics:
2 years
```

The exact policy depends on product requirements.

Retention policies prevent analytics storage from growing indefinitely.

---

# 50. Complete Event-Driven Architecture

Our system now looks like:

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
               App 1         App 2         App 3
                  |             |             |
                  └─────────────┼─────────────┘
                                |
                       ┌────────┴────────┐
                       |                 |
                       v                 v
                     Redis           Message Queue
                       |                 |
                       |                 v
                       |            Analytics
                       |             Workers
                       |                 |
                       v                 v
                  URL Database     Analytics Storage
```

---

# 51. Redirect Flow

```text
Client
  |
  v
Load Balancer
  |
  v
Application
  |
  +---- Redis HIT
  |       |
  |       v
  |    Original URL
  |
  +---- Publish click event
  |
  v
HTTP Redirect
```

Analytics:

```text
Click Event
     |
     v
Message Queue
     |
     v
Worker
     |
     +---- Deduplication
     |
     +---- Aggregation
     |
     v
Analytics Storage
```

---

# 52. What Happens if Redis Is Down?

This is an important future problem.

Our current flow is:

```text
Application
    |
    v
Redis
```

If Redis fails, we can potentially fall back to:

```text
Database
```

Conceptually:

```text
Redis
  |
  +---- HIT → Redirect
  |
  +---- MISS / unavailable
            |
            v
         Database
            |
            v
         Redis SET
            |
            v
         Redirect
```

But if Redis is completely unavailable, database traffic may suddenly increase dramatically.

This is called:

> Cache Failure / Cache Stampede Risk

We will address this later.

---

# 53. What Happens if the Queue Is Down?

This is another important question.

Suppose:

```text
Application
     |
     X
Message Queue
```

The redirect still needs to work.

We should decide whether analytics is:

```text
Best effort
```

or:

```text
Required
```

For many URL Shorteners, losing a tiny number of analytics events may be preferable to making every redirect fail.

Therefore, a production system may use:

```text
Timeouts
Local buffering
Retry mechanisms
Durable queues
Circuit breakers
```

depending on requirements.

---

# 54. Important Design Decision

We must explicitly define:

> Is analytics part of the correctness requirement?

For example:

### URL Redirect

Must not be lost.

```text
short_code → original_url
```

### Analytics

May tolerate small delays.

```text
click → analytics event
```

This allows us to design them differently.

---

# 55. Exactly Once Is Hard

People often say:

> "We need exactly-once processing."

In distributed systems, true exactly-once behavior across multiple systems can be complicated.

A practical approach is often:

```text
At-least-once delivery
+
Idempotent processing
```

This provides effectively-once results for many workloads.

For our analytics pipeline:

```text
At-least-once event delivery
+
Unique event_id
+
Idempotent consumer
```

is a strong practical design.

---

# 56. Why This Architecture Scales

Suppose:

```text
Redirects = 100K/sec
```

Application servers handle:

```text
URL resolution
```

Redis handles:

```text
Fast lookup
```

Queue handles:

```text
Event buffering
```

Workers handle:

```text
Analytics processing
```

Analytics storage handles:

```text
Historical queries
```

Each component has a focused responsibility.

This allows independent scaling.

---

# 57. Independent Scaling

For example:

```text
Redirect traffic increases
        |
        v
Add Application Servers
```

If analytics processing increases:

```text
Analytics backlog increases
        |
        v
Add Workers
```

If cache traffic increases:

```text
Redis load increases
        |
        v
Scale Redis architecture
```

Each layer can evolve independently.

This is a major benefit of decoupled architecture.

---

# 58. When Should We Introduce a Queue?

Not every small application needs one.

If our system handles:

```text
100 clicks/day
```

a synchronous analytics database write might be perfectly fine.

But if:

```text
100,000 clicks/sec
```

are expected, asynchronous processing becomes much more attractive.

Again:

> Architecture should follow workload.

---

# 59. Production Design Principle

A useful rule is:

> **Keep the user-facing critical path as small and reliable as possible. Move non-critical work to asynchronous processing.**

For our URL Shortener:

```text
Critical:

Resolve URL
→ Redirect
```

Asynchronous:

```text
Analytics
Reporting
Aggregations
Notifications
Fraud analysis
Data pipelines
```

This separation improves scalability and failure isolation.

---

# 60. Chapter Summary

We introduced:

```text
Event-Driven Architecture
Message Queues
Producers
Consumers
Asynchronous Processing
Eventual Consistency
At-Least-Once Delivery
Idempotent Consumers
Retries
Exponential Backoff
Dead Letter Queues
Consumer Groups
Partitioning
Backpressure
Queue Lag
Analytics Aggregation
Failure Isolation
```

Our architecture evolved from:

```text
Application
   |
   v
Redis
   |
   v
Database
```

to:

```text
                         Application
                        /           \
                       /             \
                      v               v
                   Redis          Message Queue
                                      |
                                      v
                                  Workers
                                      |
                                      v
                              Analytics Storage
```

This gives us a much more scalable analytics architecture.

---

# 61. The Next Problem

Our system is becoming highly distributed.

We now have:

```text
Load Balancer
Application Servers
Redis
Database
Message Queue
Workers
Analytics Storage
```

At this point, another important problem appears.

What happens if:

```text
Redis becomes unavailable?
```

Or:

```text
Redis is overloaded?
```

Or:

```text
10,000 requests all miss the cache simultaneously?
```

Suppose a popular short URL receives:

```text
100,000 requests/sec
```

and its cache entry expires.

Suddenly all 100,000 requests may hit the database.

```text
                    100K requests
                          |
                          v
                       Redis
                          |
                        MISS
                          |
          ┌───────────────┼───────────────┐
          |               |               |
          v               v               v
       Database        Database        Database
          ...             ...             ...
```

This can overload the database.

This problem is commonly called:

> **Cache Stampede / Thundering Herd**

The next chapter will focus on:

```text
Cache Failure
Cache Stampede
Cache Penetration
Cache Breakdown
Cache Avalanche
Distributed Locks
Request Coalescing
```

and how to protect the database when the cache layer fails or becomes cold.
