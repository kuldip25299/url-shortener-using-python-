# URL Shortener using Python

A simple, production-oriented URL Shortener built with Python to understand how a small web application can evolve into a secure, scalable distributed system.

This repository is designed for **learning system design through implementation**.

Instead of jumping directly into Redis, load balancers, queues, and distributed architecture, we start with a simple URL shortener and progressively identify its limitations and improve the system.

---

## 🎯 What Are We Building?

A URL Shortener converts a long URL such as:

```text
https://example.com/products/electronics/mobile-phones/apple/iphone-17-pro?campaign=summer-sale&source=email
```

into a short URL:

```text
https://short.ly/aB72x
```

When a user opens the short URL:

```text
https://short.ly/aB72x
```

the application finds the original URL and redirects the user:

```text
Short URL
    |
    v
/aB72x
    |
    v
Find original URL
    |
    v
Redirect
    |
    v
https://example.com/products/...
```

The goal of this repository is not simply to build a working URL shortener.

The goal is to understand **how to design, implement, secure, optimize, and scale such a system in production**.

---

# 🎯 Learning Objectives

By completing this repository, you will understand:

* How URL shortening works
* How short codes are generated
* Random short-code generation
* Base62 encoding
* Database design for URL mappings
* Database indexing
* Unique constraints
* Redirect handling
* URL validation
* URL expiration
* Security considerations
* Redis caching
* Cache-aside pattern
* Cache hit and cache miss
* Read-heavy system design
* Horizontal scaling
* Load balancing
* Database scaling considerations
* Analytics architecture
* Asynchronous processing
* Failure handling
* Monitoring
* Production architecture

---

# 🏗️ System Evolution

We intentionally build the system in stages.

```text
Simple URL Shortener
        |
        v
Database-backed URL Shortener
        |
        v
Secure URL Shortener
        |
        v
Redis Cached URL Shortener
        |
        v
Horizontally Scalable URL Shortener
        |
        v
Production Architecture
```

Each stage solves a specific problem introduced by the previous stage.

---

# 📚 Repository Structure

The repository is organized into documentation, examples, application code, tests, and benchmarks.

```text
url-shortener-using-python/
│
├── README.md
│
├── docs/
│   ├── 01_business_problem.md
│   ├── 02_what_is_url_shortener.md
│   ├── 03_basic_design.md
│   ├── 04_short_code_generation.md
│   ├── 05_database_design.md
│   ├── 06_redirect_flow.md
│   ├── 07_security.md
│   ├── 08_caching.md
│   ├── 09_scalability.md
│   ├── 10_analytics.md
│   ├── 11_failure_handling.md
│   └── 12_production_architecture.md
│
├── examples/
│   ├── 01_basic_url_shortener.py
│   ├── 02_random_short_code.py
│   ├── 03_base62.py
│   ├── 04_database_url_shortener.py
│   ├── 05_redirect.py
│   ├── 06_url_expiration.py
│   ├── 07_url_validation.py
│   ├── 08_redis_cache.py
│   └── 09_cached_url_shortener.py
│
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── database.py
│   ├── models.py
│   ├── services.py
│   └── main.py
│
├── tests/
│   ├── test_short_code.py
│   ├── test_url_validation.py
│   └── test_url_shortener.py
│
├── benchmark/
│   └── benchmark_redirects.py
│
├── docker-compose.yml
├── requirements.txt
└── LICENSE
```

> The repository will be built incrementally. Some directories and files shown above will be introduced in later chapters.

---

# 🧩 Learning Roadmap

## Phase 1 — Understand the Problem

### 01. Business Problem

Understand why URL shorteners are needed and what problem they solve.

We will discuss:

* Long URLs
* Sharing problems
* Readability
* Short links
* Redirects
* Basic requirements

---

### 02. What is a URL Shortener?

Understand the fundamental concept:

```text
Long URL
    |
    v
Generate Short Code
    |
    v
Store Mapping
    |
    v
Return Short URL
```

And the redirect flow:

```text
Short URL
    |
    v
Lookup Short Code
    |
    v
Original URL
    |
    v
Redirect
```

---

### 03. Basic System Design

Before writing production code, design the simplest possible system.

```text
Client
  |
  v
Application
  |
  v
URL Mapping
```

We will identify:

* Core components
* Request flow
* Data required
* Create URL operation
* Redirect operation
* Basic API design

---

# Phase 2 — Build the Core System

## 04. Basic URL Shortener

Start with the simplest implementation.

Initially we can use an in-memory Python dictionary:

```python
{
    "aB72x": "https://example.com/very/long/url"
}
```

This implementation is intentionally simple.

The purpose is to understand the core logic before introducing databases and infrastructure.

---

## 05. Short Code Generation

We need to generate a short identifier:

```text
Long URL
    |
    v
Short Code
```

We will compare approaches such as:

* Random strings
* UUIDs
* Numeric identifiers
* Database IDs
* Base62

We will also discuss:

* Collision probability
* Uniqueness
* Predictability
* Security implications

---

## 06. Base62 Encoding

Base62 allows us to represent numeric IDs using:

```text
0-9
a-z
A-Z
```

For example:

```text
Database ID
    |
    v
Base62 Encoding
    |
    v
Short Code
```

This gives us compact identifiers suitable for URLs.

---

## 07. Database Design

The in-memory implementation cannot survive application restarts or multiple application servers.

We introduce persistent storage.

Example conceptual schema:

```text
url_mapping

id
short_code
original_url
created_at
expires_at
```

Important concepts:

* Primary keys
* Unique constraints
* Indexes
* Query performance
* Data persistence

The critical lookup will be:

```sql
SELECT original_url
FROM url_mapping
WHERE short_code = ?;
```

Therefore `short_code` must be efficiently indexed.

---

# Phase 3 — Production Basics

## 08. Redirect System

Implement the short URL request:

```text
GET /aB72x
```

Flow:

```text
Client
   |
   v
GET /aB72x
   |
   v
Application
   |
   v
Database
   |
   v
Original URL
   |
   v
HTTP Redirect
```

We will also discuss HTTP redirect behavior, including:

* 301
* 302
* 307
* Redirect caching implications

---

## 09. URL Validation

Users should not be able to submit arbitrary invalid values.

We will discuss validation for:

* HTTP URLs
* HTTPS URLs
* Invalid schemes
* Malformed URLs
* Empty URLs
* Extremely long URLs

We will also discuss why the application should **not automatically fetch the destination URL** simply to validate it.

---

## 10. Security

Security is an important part of a production URL shortener.

We will discuss:

### Malicious URLs

Attackers can use URL shorteners to hide malicious destinations.

### Open Redirect Concerns

The application must clearly define what it is allowed to redirect to.

### Short-Code Enumeration

Predictable identifiers such as:

```text
/1
/2
/3
/4
```

can make enumeration easy.

### Abuse Prevention

A public URL shortener can become a target for:

* Bots
* Spam
* Automated URL creation
* Phishing
* Denial-of-service attempts

### Input Validation

Never blindly trust user-provided URLs.

---

## 11. URL Expiration

Support temporary URLs:

```text
created_at
expires_at
```

Request flow:

```text
GET /aB72x
      |
      v
Check expiration
      |
   ┌──┴──┐
   |     |
Expired Valid
   |     |
  404  Redirect
```

This also creates an opportunity to discuss TTL-based caching later.

---

# Phase 4 — Scaling

## 12. Redis Caching

A URL shortener is typically **read-heavy**.

For example:

```text
100 URLs created/sec
```

but potentially:

```text
100,000 redirects/sec
```

The database can become a bottleneck if every redirect requires a database lookup.

We introduce Redis:

```text
Client
  |
  v
Application
  |
  v
Redis
  |
  +---- Cache HIT ----> Redirect
  |
  +---- Cache MISS
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

## 13. Cache Strategy

We will implement and explain the **cache-aside pattern**.

### Cache Hit

```text
Request
   |
   v
Redis
   |
   v
Found
   |
   v
Redirect
```

### Cache Miss

```text
Request
   |
   v
Redis
   |
   v
Not Found
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

We will discuss:

* Cache TTL
* Cache invalidation
* Hot URLs
* Cache hit ratio
* Stale data
* Redis failure fallback

---

## 14. Read-Heavy Architecture

We will analyze why URL shorteners are usually read-heavy systems.

The system has two major operations:

```text
Create short URL
       ↓
Write

Open short URL
       ↓
Read
```

In most real-world systems:

```text
Reads >>> Writes
```

This affects our architecture.

---

## 15. Horizontal Scaling

When one application server is no longer enough:

```text
             ┌── Application 1
             │
Client → Load Balancer ── Application 2
             │
             └── Application 3
```

All application instances should be stateless.

Shared state should live in external systems such as:

```text
Redis
Database
```

This allows us to scale application servers horizontally.

---

# Phase 5 — Advanced Features

## 16. Analytics

We may want to know:

```text
How many times was this URL opened?
```

Possible analytics:

* Click count
* Timestamp
* Referrer
* Device
* Browser
* Country

However, analytics should not unnecessarily slow down the redirect path.

We will explore asynchronous processing:

```text
Request
   |
   +------> Redirect
   |
   +------> Analytics Event
                    |
                    v
                 Queue
                    |
                    v
                 Worker
```

---

## 17. Asynchronous Processing

Analytics is a good example of work that doesn't necessarily need to happen synchronously.

Instead of:

```text
Redirect
   |
   v
Save Analytics
   |
   v
Return Response
```

we can use:

```text
Redirect
   |
   +----> Queue Event
             |
             v
          Worker
             |
             v
       Analytics Storage
```

This introduces:

* Producers
* Consumers
* Background workers
* Queues
* Event processing

---

## 18. Failure Handling

Production systems must handle failures.

We will discuss scenarios such as:

### Redis failure

```text
Redis unavailable
      |
      v
Fallback to Database
```

### Database failure

Cached URLs may still be served when appropriate.

### Duplicate short code

Generate another code or retry safely.

### Expired URL

Return an appropriate response.

### Application server failure

Load balancer routes traffic to another healthy instance.

---

## 19. Monitoring

A production URL shortener should be observable.

Important metrics include:

```text
Requests/sec
Redirect latency
Cache hit ratio
Database latency
Error rate
URL creation rate
Redirect rate
Expired URLs
```

We will discuss what should be monitored and why.

---

# Phase 6 — Final System Design

## 20. Production Architecture

At the end of the repository, we will combine everything into a production-oriented architecture.

```text
                         ┌───────────────┐
                         │    Client     │
                         └───────┬───────┘
                                 |
                                 v
                         ┌───────────────┐
                         │ Load Balancer │
                         └───────┬───────┘
                                 |
                    ┌────────────┼────────────┐
                    |            |            |
                    v            v            v
                App Server   App Server   App Server
                    |            |            |
                    └────────────┼────────────┘
                                 |
                         ┌───────v───────┐
                         │     Redis     │
                         │     Cache     │
                         └───────┬───────┘
                                 |
                              Cache Miss
                                 |
                         ┌───────v───────┐
                         │    Database   │
                         │ URL Mappings  │
                         └───────────────┘
```

Analytics can be separated from the critical redirect path:

```text
Redirect Request
       |
       +-----------> Redirect Response
       |
       +-----------> Analytics Event
                         |
                         v
                       Queue
                         |
                         v
                       Worker
                         |
                         v
                  Analytics Storage
```

---

# 🔬 Benchmarking

This repository will also include benchmarking so that we can measure the impact of our architectural decisions.

We will compare scenarios such as:

```text
Database Only
      vs
Redis + Database
```

Metrics can include:

* Average latency
* P95 latency
* P99 latency
* Requests per second
* Cache hit ratio
* Database queries

The purpose is not to produce unrealistic benchmark numbers.

The purpose is to understand:

> **Why does the architecture change when traffic increases?**

---

# 🔐 Security Principles

The project will follow these principles:

1. Never blindly trust user input.
2. Validate URLs before storing them.
3. Only allow supported URL schemes.
4. Use sufficiently strong short-code generation.
5. Avoid predictable identifiers when security requires opacity.
6. Apply rate limiting to public APIs.
7. Consider abuse and phishing scenarios.
8. Avoid exposing unnecessary internal information.
9. Never fetch arbitrary user-provided URLs from the backend without a strong security reason and SSRF protections.
10. Treat redirect destinations as untrusted input.

---

# 📈 Scaling Philosophy

We will follow a simple engineering principle throughout this project:

```text
Start Simple
     |
     v
Make It Work
     |
     v
Measure
     |
     v
Find Bottleneck
     |
     v
Optimize
     |
     v
Scale
```

We will not introduce distributed infrastructure just because it is available.

Every component should solve a specific problem.

For example:

```text
Database
    ↓
Persistence

Redis
    ↓
Reduce repeated database reads

Load Balancer
    ↓
Distribute application traffic

Queue
    ↓
Move non-critical work out of request path
```

---

# 🧪 Running the Project

The project will be developed incrementally.

As each phase is completed, the README and documentation will be updated with the exact commands required to run and test that stage.

The final application will provide APIs conceptually similar to:

```text
POST /shorten
```

Create a short URL.

```text
GET /{short_code}
```

Redirect to the original URL.

The exact API implementation will be introduced during the implementation chapters.

---

# 🛠️ Technology Stack

The project will primarily use:

### Backend

* Python
* FastAPI

### Database

* MySQL

### Cache

* Redis

### Testing

* pytest

### Benchmarking

* Python benchmarking tools

### Infrastructure

* Docker
* Docker Compose

The repository intentionally avoids unnecessary frameworks and infrastructure until they are required by the problem we are solving.

---

# 👨‍💻 Who Is This Repository For?

This repository is intended for:

* Backend developers
* Python developers
* Software engineers learning system design
* Developers preparing for system-design interviews
* Developers learning Redis
* Developers interested in scalable web applications
* Developers who want to understand production architecture

You do **not** need to know distributed systems beforehand.

The repository starts from the basics and gradually introduces more advanced concepts.

---

# 💡 Key System Design Questions

By the end of the project, you should be able to answer questions such as:

### How do we generate unique short URLs?

Use a suitable short-code strategy such as random identifiers or Base62 encoding with collision handling.

### Why do we need a database?

To persist the mapping between short codes and original URLs.

### Why is `short_code` indexed?

Because redirect requests frequently query the database using the short code.

### Why use Redis?

To reduce repeated database reads for frequently accessed URLs.

### Why is URL shortening read-heavy?

Because URL creation happens relatively infrequently compared with URL redirects.

### How do we scale the application?

Use stateless application servers behind a load balancer.

### What happens if Redis goes down?

The application can fall back to the database depending on the availability and consistency requirements.

### How do we handle analytics?

Move analytics processing away from the critical redirect path using asynchronous processing.

### How do we prevent abuse?

Use URL validation, rate limiting, monitoring, abuse detection, and appropriate security controls.

---

# 🚀 Final Goal

The final goal is not simply:

> "Build a URL shortener."

The goal is to understand the engineering decisions behind a scalable system.

We will start with:

```text
Python Dictionary
```

and progressively evolve it into:

```text
                    ┌───────────────┐
                    │    Clients    │
                    └───────┬───────┘
                            |
                            v
                    ┌───────────────┐
                    │ Load Balancer │
                    └───────┬───────┘
                            |
                 ┌──────────┼──────────┐
                 |          |          |
                 v          v          v
              App 1      App 2      App 3
                 |          |          |
                 └──────────┼──────────┘
                            |
                    ┌───────v───────┐
                    │     Redis     │
                    └───────┬───────┘
                            |
                       Cache Miss
                            |
                    ┌───────v───────┐
                    │    MySQL      │
                    └───────────────┘
```

while keeping the design understandable at every stage.

---

# 📖 Recommended Learning Order

Follow the repository in this order:

```text
01. Business Problem
02. What is URL Shortener?
03. Basic System Design
04. Basic Implementation
05. Short Code Generation
06. Base62
07. Database Design
08. Redirect Flow
09. URL Validation
10. Security
11. Expiration
12. Redis Caching
13. Cache Strategy
14. Read-Heavy Architecture
15. Horizontal Scaling
16. Analytics
17. Asynchronous Processing
18. Failure Handling
19. Monitoring
20. Production Architecture
```

Each step builds on the previous one.

---

# ⭐ Philosophy of This Repository

> **Don't start with a distributed system. Start with a simple system and understand why it eventually needs to become distributed.**

That is the core idea behind this project.
