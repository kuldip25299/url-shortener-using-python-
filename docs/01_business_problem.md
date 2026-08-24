# URL Shortener — Business Problem

## 1. Why Do We Need a URL Shortener?

Modern web applications generate URLs that can become very long.

A URL may contain:

* Multiple path segments
* Query parameters
* Tracking parameters
* Campaign information
* Resource identifiers
* Filters
* Authentication or session-related information

For example:

```text
https://example.com/products/electronics/mobile-phones/apple/iphone-17-pro?campaign=summer-sale&source=email&region=india
```

Technically, this URL works perfectly.

However, sharing such a URL is not always convenient.

A shorter URL would be much easier to share:

```text
https://short.ly/aB72x
```

The short URL should still take the user to the same destination.

---

# 2. The Business Problem

Imagine a company that wants to provide users with shareable links.

Users should be able to submit a long URL:

```text
https://example.com/products/electronics/mobile-phones/apple/iphone-17-pro?campaign=summer-sale
```

and receive:

```text
https://short.ly/aB72x
```

When another user opens:

```text
https://short.ly/aB72x
```

the system should redirect them to:

```text
https://example.com/products/electronics/mobile-phones/apple/iphone-17-pro?campaign=summer-sale
```

The system therefore needs to maintain a mapping:

```text
Short Code
    |
    v
Original URL
```

For example:

```text
aB72x
   |
   v
https://example.com/products/electronics/mobile-phones/apple/iphone-17-pro?campaign=summer-sale
```

---

# 3. Basic User Journey

The system has two primary operations.

## Operation 1 — Create a Short URL

A user submits a long URL.

```text
Client
   |
   | POST /shorten
   |
   v
URL Shortener
   |
   | Generate short code
   |
   v
Store URL mapping
   |
   v
Return short URL
```

Example request:

```json
{
  "url": "https://example.com/products/iphone-17-pro"
}
```

Example response:

```json
{
  "short_url": "https://short.ly/aB72x"
}
```

---

# 4. Operation 2 — Open the Short URL

A user opens:

```text
https://short.ly/aB72x
```

The application receives:

```text
GET /aB72x
```

The system looks up the short code:

```text
aB72x
```

and finds the original URL:

```text
https://example.com/products/iphone-17-pro
```

The application then redirects the user.

The complete flow is:

```text
Browser
   |
   | GET /aB72x
   v
URL Shortener
   |
   | Lookup aB72x
   v
URL Mapping
   |
   | Original URL
   v
HTTP Redirect
   |
   v
Original Website
```

---

# 5. Why Can't We Simply Store the Short URL?

A short URL by itself doesn't contain enough information to know where the user should be redirected.

For example:

```text
https://short.ly/aB72x
```

does not tell us what `aB72x` means.

The system therefore needs a mapping:

```text
aB72x
   ↓
https://example.com/products/iphone-17-pro
```

This mapping is the fundamental data structure behind a URL shortener.

---

# 6. The Simplest Possible Solution

We could initially store mappings in memory:

```python
url_mapping = {
    "aB72x": "https://example.com/products/iphone-17-pro",
    "xY92p": "https://example.com/products/macbook-air",
}
```

Then:

```text
GET /aB72x
```

would perform:

```text
aB72x
  ↓
url_mapping["aB72x"]
  ↓
Original URL
```

This is enough to demonstrate the basic idea.

However, this approach has serious limitations.

---

# 7. Problem With In-Memory Storage

Suppose our application stores:

```text
aB72x → https://example.com/...
```

inside Python memory.

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

the mappings disappear.

The short URLs that were previously generated would stop working.

That means the system needs persistent storage.

---

# 8. Multiple Application Servers

Now imagine the application becomes popular.

Instead of one server, we have three:

```text
             ┌── Server 1
             |
Client ──────┼── Server 2
             |
             └── Server 3
```

If each server stores URL mappings in its own memory, we have another problem.

Suppose Server 1 creates:

```text
aB72x → https://example.com/page
```

Then the user opens the short URL and the request reaches Server 2.

Server 2 doesn't know about `aB72x`.

```text
Server 1
   |
   | aB72x exists
   v

Server 2
   |
   | aB72x does not exist
   v
404
```

Therefore, shared persistent storage becomes necessary when we have multiple application servers.

---

# 9. Basic Persistent Architecture

We can introduce a database:

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
                    │ short_code    │
                    │ original_url  │
                    └───────────────┘
```

Now the mapping survives application restarts and can be shared by multiple application servers.

---

# 10. Another Important Problem — Short Code Generation

The system needs to generate short codes.

For example:

```text
aB72x
xY92p
K8mQa
```

The generated code must be unique.

We cannot have:

```text
aB72x → URL A
aB72x → URL B
```

because the system would not know which URL to redirect to.

Therefore, we need a reliable short-code generation strategy.

This creates another system-design question:

> How do we generate unique, compact identifiers at scale?

We will explore several approaches later.

---

# 11. Read Traffic Will Be Much Higher

A URL shortener usually has two major operations:

```text
Create short URL
        ↓
Write operation
```

and:

```text
Open short URL
        ↓
Read operation
```

In a typical system, the same short URL may be opened many times.

For example:

```text
One short URL created
        |
        +---- User 1 opens
        +---- User 2 opens
        +---- User 3 opens
        +---- User 4 opens
        +---- ...
        +---- Thousands of opens
```

Therefore:

```text
Reads >>> Writes
```

This makes URL shortening a good example of a **read-heavy system**.

Later, this characteristic will lead us toward caching.

---

# 12. Database Can Become a Bottleneck

Initially, the database may be completely sufficient.

For example:

```text
Request
   |
   v
Application
   |
   v
Database
   |
   v
Original URL
```

But if the application receives a very large number of redirects:

```text
100,000 redirects/sec
```

we don't necessarily want every request to hit the database.

That creates unnecessary database load.

Eventually, we can introduce a cache:

```text
Request
   |
   v
Redis
   |
   +---- HIT ----> Redirect
   |
   +---- MISS
          |
          v
       Database
```

We will introduce this optimization only after understanding why it is needed.

---

# 13. Security and Abuse

A URL shortener also creates security challenges.

Because the service hides the destination URL behind a short identifier, attackers could potentially use it to distribute malicious links.

For example:

```text
https://short.ly/aB72x
```

could redirect to a malicious website.

Therefore, a production URL-shortening service needs to think about:

* URL validation
* Abuse prevention
* Rate limiting
* Malicious URL detection
* Spam prevention
* Short-code enumeration
* Authentication and authorization
* Monitoring

Security is not an optional feature added at the end.

It is part of the system design.

---

# 14. Functional Requirements

Our initial system should support the following functionality.

## Create Short URL

Given a valid long URL, generate a unique short URL.

```text
POST /shorten
```

Input:

```json
{
  "url": "https://example.com/long/path"
}
```

Output:

```json
{
  "short_url": "https://short.ly/aB72x"
}
```

---

## Redirect

Given a short code, find the original URL and redirect the client.

```text
GET /aB72x
```

---

## URL Expiration

The system should eventually support URLs that expire after a specified period.

Example:

```text
created_at = 2026-08-24 10:00
expires_at = 2026-08-25 10:00
```

After expiration, the short URL should no longer redirect.

---

# 15. Non-Functional Requirements

The system should eventually be designed for:

### Scalability

The application should support increasing traffic by adding more application servers.

### Availability

A failure of one application server should not make the entire service unavailable.

### Performance

Redirect requests should be processed quickly.

### Durability

Created URL mappings should not disappear when application servers restart.

### Security

The system should prevent common URL-shortener abuse and validate user input.

### Observability

The system should provide enough metrics and logs to understand system health.

---

# 16. Initial Scale Assumption

For the educational project, we will use a hypothetical workload rather than trying to design for infinite scale.

Assume:

```text
URL creation:
1,000 requests/sec

URL redirects:
100,000 requests/sec
```

This gives us an approximate ratio:

```text
1 write
   :
100 reads
```

The exact numbers are not important.

The purpose is to demonstrate why architectural decisions change as traffic increases.

---

# 17. The Core System Design Problem

At the beginning, our system looks simple:

```text
Client
   |
   v
Application
   |
   v
Database
```

But as requirements grow, new questions appear:

```text
How do we generate unique short codes?
                |
                v
How do we persist mappings?
                |
                v
How do we efficiently redirect?
                |
                v
How do we handle millions of reads?
                |
                v
Can Redis reduce database load?
                |
                v
How do we scale application servers?
                |
                v
How do we handle failures?
                |
                v
How do we collect analytics?
                |
                v
How do we secure the service?
```

These questions will drive the architecture throughout this repository.

---

# 18. What We Are NOT Building Initially

We will not begin by building a massive distributed system.

We will not immediately introduce:

* Kubernetes
* Kafka
* Multiple databases
* Microservices
* Distributed locks
* Complex service abstractions
* Multiple caching layers
* Global infrastructure

Instead, we will follow:

```text
Simple
  ↓
Working
  ↓
Identify Problem
  ↓
Improve
  ↓
Measure
  ↓
Scale
```

Every technology introduced later should solve a specific problem.

---

# 19. The Engineering Journey

The repository will evolve approximately like this:

### Version 1

```text
Python Dictionary
```

Understand the core URL-shortening logic.

### Version 2

```text
Python Application
       |
       v
Database
```

Add persistence.

### Version 3

```text
Application
    |
    v
Database
```

Add validation, expiration, and security.

### Version 4

```text
Application
    |
    v
Redis
    |
    v
Database
```

Reduce database reads.

### Version 5

```text
             ┌── Application 1
             |
Client → LB ─┼── Application 2
             |
             └── Application 3
                    |
                 Redis
                    |
                 Database
```

Scale application servers horizontally.

### Version 6

```text
Redirect
   |
   +------> Response
   |
   +------> Analytics Event
                  |
                  v
                Queue
                  |
                  v
               Worker
```

Move non-critical processing out of the request path.

---

# 20. Key Takeaway

A URL shortener looks like a simple application:

```text
Long URL
    ↓
Short URL
    ↓
Redirect
```

But it provides a useful system-design problem because it forces us to think about:

* Unique identifier generation
* Persistence
* Database indexing
* Read-heavy workloads
* Caching
* Horizontal scaling
* Availability
* Security
* Analytics
* Asynchronous processing
* Failure handling

The important lesson is not how to write a few lines of code that shorten a URL.

The important lesson is:

> **How do we take a simple URL-mapping application and evolve it into a secure, highly available, and scalable production system?**

That is the problem we will solve throughout this repository.
