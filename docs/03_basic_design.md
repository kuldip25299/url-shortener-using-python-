# URL Shortener — Basic System Design

## 1. Introduction

In the previous chapter, we established the fundamental problem:

> Given a long URL, generate a short URL and redirect users from the short URL to the original destination.

The core mapping is:

```text
Short Code → Original URL
```

For example:

```text
aB72x → https://example.com/products/iphone-17-pro
```

Before introducing databases, Redis, load balancers, queues, or other infrastructure, we first need to understand the **basic system design**.

The goal of this chapter is to answer:

* What are the core components?
* What APIs do we need?
* What happens when a short URL is created?
* What happens when a short URL is opened?
* What data do we need?
* What assumptions are we making?
* What limitations does the basic architecture have?

---

# 2. Start With Requirements

A good system-design process starts with requirements.

We should not start by choosing technologies.

Instead:

```text
Requirements
    ↓
System Behavior
    ↓
Data Model
    ↓
Architecture
    ↓
Technology
```

For our URL Shortener, we first define the minimum functionality.

---

# 3. Functional Requirements

The initial system needs two primary operations.

## Requirement 1 — Create a Short URL

A client sends a long URL.

Example:

```text
https://example.com/products/iphone-17-pro
```

The system generates:

```text
aB72x
```

and returns:

```text
https://short.ly/aB72x
```

Conceptually:

```text
Long URL
    |
    v
URL Shortener
    |
    v
Short URL
```

---

## Requirement 2 — Redirect a Short URL

A user opens:

```text
https://short.ly/aB72x
```

The system extracts:

```text
aB72x
```

finds the corresponding original URL:

```text
https://example.com/products/iphone-17-pro
```

and redirects the user.

Flow:

```text
Short URL
    |
    v
Extract Short Code
    |
    v
Find Original URL
    |
    v
Redirect
```

These two operations are enough for our first implementation.

---

# 4. Future Functional Requirements

The initial implementation will be intentionally small.

Later, we can add:

* URL expiration
* URL deletion
* URL revocation
* Authentication
* User ownership
* Custom aliases
* Click analytics
* Rate limiting
* Abuse detection
* API keys
* Monitoring

We should not implement these immediately.

They will be introduced when we need them.

---

# 5. Non-Functional Requirements

The system should eventually satisfy several non-functional requirements.

## Scalability

We should be able to handle increasing traffic by adding application servers.

```text
1 Server
   ↓
2 Servers
   ↓
10 Servers
   ↓
100 Servers
```

---

## Availability

If one application server fails, the entire service should not necessarily become unavailable.

Later we can use multiple application instances behind a load balancer.

---

## Performance

Redirect requests should be fast.

A user opening:

```text
https://short.ly/aB72x
```

should not wait unnecessarily for expensive processing.

---

## Durability

Created URL mappings should survive:

* Application restarts
* Server failures
* Deployment
* Multiple application instances

This requirement will eventually lead us from in-memory storage to persistent storage.

---

## Security

The system should:

* Validate URLs
* Avoid dangerous input
* Prevent abuse
* Avoid predictable identifiers where appropriate
* Rate-limit public APIs
* Protect internal resources

Security will be covered in detail later.

---

# 6. High-Level Architecture

The simplest architecture is:

```text
             ┌───────────────┐
             │    Client     │
             └───────┬───────┘
                     |
                     v
             ┌───────────────┐
             │ URL Shortener │
             │   Application │
             └───────┬───────┘
                     |
                     v
             ┌───────────────┐
             │ URL Storage   │
             └───────────────┘
```

There are only three conceptual components:

1. Client
2. Application
3. Storage

This is enough to understand the basic system.

---

# 7. Component 1 — Client

The client can be:

* Web browser
* Mobile application
* Backend service
* API client

For example:

```text
POST /shorten
```

with:

```json
{
  "url": "https://example.com/products/iphone-17-pro"
}
```

The client receives:

```json
{
  "short_url": "https://short.ly/aB72x"
}
```

---

# 8. Component 2 — URL Shortener Application

The application contains the business logic.

Its responsibilities include:

1. Receive the request.
2. Validate the URL.
3. Generate a short code.
4. Store the mapping.
5. Return the short URL.
6. Handle redirect requests.
7. Find the original URL.
8. Return the redirect response.

Conceptually:

```text
Client
  |
  v
Application
  |
  +---- Validate URL
  |
  +---- Generate Short Code
  |
  +---- Store Mapping
  |
  +---- Redirect Requests
```

---

# 9. Component 3 — Storage

The application needs to store the relationship:

```text
short_code → original_url
```

For example:

```text
aB72x → https://example.com/products/iphone-17-pro
```

Initially, we can use an in-memory dictionary.

Later, we will replace it with a database.

The important architectural principle is:

> Storage is responsible for remembering URL mappings. The application is responsible for the business logic.

---

# 10. API Design

We need two basic endpoints.

## Endpoint 1 — Create Short URL

```text
POST /shorten
```

Request:

```json
{
  "url": "https://example.com/products/iphone-17-pro"
}
```

Response:

```json
{
  "short_url": "https://short.ly/aB72x"
}
```

---

# 11. Endpoint 2 — Redirect

```text
GET /{short_code}
```

For example:

```text
GET /aB72x
```

The server finds:

```text
aB72x
   ↓
https://example.com/products/iphone-17-pro
```

and returns an HTTP redirect.

Conceptually:

```http
HTTP/1.1 302 Found
Location: https://example.com/products/iphone-17-pro
```

---

# 12. Create URL Request Flow

Let's follow the complete request.

The client sends:

```text
POST /shorten
```

with:

```json
{
  "url": "https://example.com/products/iphone-17-pro"
}
```

The application performs:

```text
1. Receive URL
       ↓
2. Validate URL
       ↓
3. Generate Short Code
       ↓
4. Check uniqueness
       ↓
5. Store Mapping
       ↓
6. Build Short URL
       ↓
7. Return Response
```

Complete flow:

```text
                         POST /shorten
                               |
                               v
                       ┌───────────────┐
                       │   Application │
                       └───────┬───────┘
                               |
                         Validate URL
                               |
                               v
                      Generate Short Code
                               |
                               v
                         Check Unique
                               |
                               v
                       Store URL Mapping
                               |
                               v
                       Return Short URL
```

---

# 13. Example Create Operation

Input:

```text
https://example.com/articles/system-design/url-shortener
```

Generated code:

```text
aB72x
```

Stored mapping:

```text
aB72x
   ↓
https://example.com/articles/system-design/url-shortener
```

Response:

```text
https://short.ly/aB72x
```

---

# 14. Redirect Request Flow

Now another user opens:

```text
https://short.ly/aB72x
```

The browser sends:

```text
GET /aB72x
```

The application performs:

```text
1. Extract short code
       ↓
2. Find mapping
       ↓
3. Check whether mapping exists
       ↓
4. Check expiration if supported
       ↓
5. Return redirect
```

Flow:

```text
                    GET /aB72x
                         |
                         v
                 ┌───────────────┐
                 │   Application │
                 └───────┬───────┘
                         |
                  Extract aB72x
                         |
                         v
                  Lookup Mapping
                         |
                    ┌────┴────┐
                    |         |
                  Found     Not Found
                    |         |
                    v         v
               Original URL   404
                    |
                    v
                Redirect
```

---

# 15. What Does the Storage Look Like?

At the conceptual level:

```text
URL Mapping
-----------------------------------------
short_code     original_url
-----------------------------------------
aB72x          https://example.com/...
xY92p          https://example.com/...
K8mQa          https://example.com/...
```

The application performs:

```text
lookup("aB72x")
```

and receives:

```text
https://example.com/products/iphone-17-pro
```

---

# 16. Initial In-Memory Implementation

For the first implementation, we can represent the storage as:

```python
url_mapping = {
    "aB72x": "https://example.com/products/iphone-17-pro",
    "xY92p": "https://example.com/articles/system-design"
}
```

Creating a mapping:

```python
url_mapping["aB72x"] = "https://example.com/products/iphone-17-pro"
```

Reading a mapping:

```python
original_url = url_mapping["aB72x"]
```

This gives us a working model without introducing database complexity.

---

# 17. Why Start With In-Memory Storage?

It might seem strange to use an approach that we already know is not production-ready.

There is a reason.

We want to separate two concepts:

### Business Logic

```text
Generate code
    ↓
Create mapping
    ↓
Lookup code
    ↓
Redirect
```

from:

### Storage Technology

```text
Dictionary
Database
Redis
NoSQL
```

The business logic should remain understandable regardless of the storage implementation.

This makes it easier to see why we later introduce a database.

---

# 18. Limitations of In-Memory Storage

The simple design has several problems.

## Problem 1 — Data Loss

If the application restarts:

```text
Application
     |
     X
Restart
     |
     v
Memory cleared
```

all mappings disappear.

---

## Problem 2 — Multiple Servers

Suppose we have:

```text
Server 1
Server 2
Server 3
```

Each server has separate memory.

If Server 1 creates:

```text
aB72x → URL A
```

Server 2 may not know about it.

```text
             ┌── Server 1
             │    aB72x exists
             |
Client ──────┼── Server 2
             │    aB72x missing
             |
             └── Server 3
```

This makes in-memory storage unsuitable for a horizontally scaled system.

---

## Problem 3 — No Durability

The mappings are not persistent.

A production URL shortener cannot rely only on process memory if users expect their short URLs to continue working.

---

# 19. Introducing Persistent Storage

The next logical step is a database.

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
                     v
             ┌───────────────┐
             │   Database    │
             └───────────────┘
```

Now:

```text
short_code
     ↓
Database
     ↓
original_url
```

The mapping survives application restarts.

It is also shared between application servers.

---

# 20. Basic Database Model

Our initial table can contain:

```text
url_mapping

id
short_code
original_url
created_at
expires_at
```

Conceptually:

```text
+----+------------+---------------------------+---------------------+
| id | short_code | original_url              | created_at          |
+----+------------+---------------------------+---------------------+
| 1  | aB72x      | https://example.com/...   | 2026-08-26 10:00:00 |
| 2  | xY92p      | https://example.com/...   | 2026-08-26 10:05:00 |
+----+------------+---------------------------+---------------------+
```

We will design the actual schema in a later chapter.

---

# 21. Why `short_code` Needs to Be Unique

Consider:

```text
aB72x → URL A
aB72x → URL B
```

Now a request arrives:

```text
GET /aB72x
```

Which URL should we return?

There is no reliable answer.

Therefore:

```text
short_code
```

must be unique.

At the database level, this can eventually be enforced with:

```sql
UNIQUE(short_code)
```

This is better than relying only on application code.

---

# 22. Short Code Generation Is a Separate Problem

Our basic architecture requires a short code:

```text
Original URL
     |
     v
Generate Code
     |
     v
aB72x
```

But how should we generate it?

Possible approaches include:

### Random string

```text
aB72x
```

### UUID

```text
550e8400-e29b-41d4-a716-446655440000
```

### Database ID

```text
123456
```

### Database ID + Base62

```text
123456
   ↓
Base62
   ↓
eJA
```

Each approach has different properties.

We will analyze these options before choosing one.

---

# 23. Basic Data Flow

The complete basic system can now be represented as:

```text
                         ┌───────────────┐
                         │    Client     │
                         └───────┬───────┘
                                 |
                      ┌──────────┴──────────┐
                      |                     |
                      v                     v
               POST /shorten          GET /aB72x
                      |                     |
                      v                     v
              ┌─────────────────────────────────┐
              │          Application             │
              │                                 │
              │  Validate                       │
              │  Generate Code                  │
              │  Lookup                         │
              │  Redirect                       │
              └───────────────┬─────────────────┘
                              |
                              v
                      ┌───────────────┐
                      │   Database    │
                      │               │
                      │ short_code    │
                      │ original_url  │
                      └───────────────┘
```

---

# 24. Read and Write Paths

It is useful to separate the two main paths.

## Write Path

Creating a short URL:

```text
Client
  |
  v
POST /shorten
  |
  v
Validate
  |
  v
Generate Code
  |
  v
Database Write
  |
  v
Return Short URL
```

This is a write-heavy operation.

---

## Read Path

Opening a short URL:

```text
Client
  |
  v
GET /aB72x
  |
  v
Lookup
  |
  v
Database Read
  |
  v
Redirect
```

This is a read operation.

In a large URL-shortening service:

```text
Reads >>> Writes
```

This difference will become extremely important when we introduce caching.

---

# 25. First Scaling Problem

Imagine:

```text
URL creation = 1,000/sec
Redirects = 100,000/sec
```

If every redirect queries the database:

```text
100,000 requests/sec
        |
        v
100,000 database lookups/sec
```

The database may become the bottleneck.

At this point, we should not immediately add random infrastructure.

Instead, we ask:

> What is the actual workload?

Most redirect requests are asking the same basic question:

```text
What URL does aB72x represent?
```

Frequently accessed mappings may be requested again and again.

That makes caching a natural optimization.

---

# 26. Future Architecture With Redis

Later, we can introduce:

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
                ┌─────────┐
                │  Redis  │
                └────┬────┘
                     |
                Cache Miss
                     |
                     v
                ┌─────────┐
                │ Database│
                └─────────┘
```

The request flow becomes:

```text
GET /aB72x
     |
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

We will not implement this yet.

First, we need to understand the basic database-backed system.

---

# 27. Basic API Contract

For the educational implementation, we will use a simple API.

## Create Short URL

```text
POST /shorten
```

Request:

```json
{
  "url": "https://example.com/products/iphone-17-pro"
}
```

Response:

```json
{
  "short_url": "https://short.ly/aB72x"
}
```

---

## Redirect

```text
GET /{short_code}
```

Example:

```text
GET /aB72x
```

Response:

```text
HTTP 302
Location: https://example.com/products/iphone-17-pro
```

---

# 28. Error Cases

Even the basic system needs to define failure behavior.

### Invalid URL

```text
POST /shorten
```

with:

```json
{
  "url": "not-a-valid-url"
}
```

should return a client error.

---

### Unknown Short Code

```text
GET /unknown123
```

should return:

```text
404 Not Found
```

---

### Expired URL

If expiration is supported:

```text
GET /aB72x
```

may return an appropriate error rather than redirecting.

---

### Duplicate Short Code

If a generated code already exists:

```text
aB72x
```

the system must generate another code or safely resolve the collision.

---

# 29. What We Have Designed So Far

Our initial system contains:

```text
Client
   |
   v
API
   |
   +---- Create URL
   |
   +---- Redirect
   |
   v
Database
```

The core data is:

```text
short_code → original_url
```

The core APIs are:

```text
POST /shorten
GET /{short_code}
```

The core challenge is:

```text
Generate
   ↓
Store
   ↓
Lookup
   ↓
Redirect
```

This is the foundation of the entire project.

---

# 30. What We Have NOT Solved Yet

There are still several important engineering problems.

### Short-code generation

How can we generate compact and unique codes?

### Collision handling

What happens if two requests generate the same code?

### Database design

What indexes and constraints do we need?

### Security

How do we prevent malicious or abusive URLs?

### Expiration

How do we handle temporary links?

### Performance

How do we reduce database reads?

### Scalability

How do we support many application servers?

### Analytics

How do we track clicks without slowing redirects?

### Failure handling

What happens when Redis or the database becomes unavailable?

These will be solved progressively.

---

# 31. Design Principle

The most important principle for this project is:

> **Do not solve tomorrow's scaling problem before today's problem exists.**

Our first implementation does not need:

```text
Load Balancer
Redis Cluster
Kafka
Database Sharding
Kubernetes
```

We first need:

```text
A correct URL mapping
        ↓
A reliable short-code strategy
        ↓
A persistent database
        ↓
A fast redirect
```

Then we measure and identify bottlenecks.

Only after identifying a real bottleneck should we introduce additional infrastructure.

---

# 32. Chapter Summary

We now have a basic system design for a URL Shortener.

The fundamental model is:

```text
Short Code → Original URL
```

The system has two primary operations:

```text
POST /shorten
GET /{short_code}
```

The basic architecture is:

```text
Client
   |
   v
Application
   |
   v
Database
```

The create flow is:

```text
Long URL
   ↓
Validate
   ↓
Generate Short Code
   ↓
Store Mapping
   ↓
Return Short URL
```

The redirect flow is:

```text
Short URL
   ↓
Extract Code
   ↓
Lookup Mapping
   ↓
Original URL
   ↓
HTTP Redirect
```

The simple architecture is enough to start implementation, but it has limitations around:

* Short-code generation
* Collision handling
* Persistence
* Performance
* Security
* Scalability

Our next step is therefore to solve one of the most fundamental problems:

> **How should we generate the short code?**
