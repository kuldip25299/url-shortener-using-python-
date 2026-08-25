# URL Shortener — What Is a URL Shortener?

## 1. Introduction

A URL Shortener is a service that converts a long URL into a short, easy-to-share URL.

For example:

```text
Long URL:

https://example.com/products/electronics/mobile-phones/apple/iphone-17-pro?campaign=summer-sale&source=email
```

can become:

```text
Short URL:

https://short.ly/aB72x
```

When a user opens the short URL, the URL Shortener finds the original URL and redirects the user to it.

The basic idea is simple:

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

Later:

```text
Short URL
   |
   v
Find Short Code
   |
   v
Get Original URL
   |
   v
Redirect User
```

---

# 2. The Core Idea

A URL Shortener maintains a mapping between:

```text
Short Code → Original URL
```

For example:

```text
aB72x → https://example.com/products/iphone-17-pro
```

The short URL is constructed using the short code:

```text
https://short.ly/aB72x
```

The domain:

```text
https://short.ly
```

is controlled by the URL-shortening service.

The path:

```text
/aB72x
```

identifies the stored URL.

---

# 3. Two Main Operations

A URL Shortener has two fundamental operations.

## Operation 1 — Create a Short URL

The user provides a long URL.

```text
POST /shorten
```

Example request:

```json
{
  "url": "https://example.com/products/iphone-17-pro"
}
```

The service generates a short code:

```text
aB72x
```

and returns:

```json
{
  "short_url": "https://short.ly/aB72x"
}
```

The system internally stores:

```text
aB72x
    ↓
https://example.com/products/iphone-17-pro
```

---

# 4. Operation 2 — Redirect

When somebody opens:

```text
https://short.ly/aB72x
```

the browser sends:

```text
GET /aB72x
```

The URL Shortener extracts:

```text
aB72x
```

and looks it up.

```text
aB72x
   |
   v
URL Mapping
   |
   v
https://example.com/products/iphone-17-pro
```

The server then returns an HTTP redirect.

Conceptually:

```text
Browser
   |
   | GET /aB72x
   v
URL Shortener
   |
   | Lookup
   v
Original URL
   |
   | HTTP Redirect
   v
Browser
   |
   v
Original Website
```

---

# 5. Why Do We Need a Mapping?

A short code such as:

```text
aB72x
```

does not contain the original URL.

It is simply an identifier.

Therefore, the service needs to remember:

```text
aB72x → https://example.com/products/iphone-17-pro
```

This mapping can initially be represented using a Python dictionary:

```python
url_mapping = {
    "aB72x": "https://example.com/products/iphone-17-pro"
}
```

When a request arrives:

```python
original_url = url_mapping["aB72x"]
```

The application gets the destination URL.

This is enough to demonstrate the fundamental concept.

---

# 6. URL Shortener as a Key-Value Mapping

At its core, a URL Shortener behaves like a key-value system.

The key is:

```text
short_code
```

The value is:

```text
original_url
```

For example:

```text
+----------+------------------------------------------------+
| Key      | Value                                          |
+----------+------------------------------------------------+
| aB72x    | https://example.com/products/iphone            |
| xY92p    | https://example.com/articles/system-design     |
| K8mQa    | https://example.com/docs/architecture          |
+----------+------------------------------------------------+
```

The redirect operation becomes a lookup:

```text
short_code
     |
     v
Find value
     |
     v
original_url
```

This simple model is important because many of the scaling decisions later will revolve around making this lookup extremely fast.

---

# 7. Basic Request Flow

Let's look at the complete flow for creating a URL.

```text
                 Create URL
                     |
                     v
              ┌──────────────┐
              │    Client    │
              └──────┬───────┘
                     |
                     | POST /shorten
                     | original URL
                     v
              ┌──────────────┐
              │ Application  │
              └──────┬───────┘
                     |
                     | Generate short code
                     v
              ┌──────────────┐
              │ URL Mapping  │
              └──────┬───────┘
                     |
                     | Store mapping
                     v
              ┌──────────────┐
              │   Storage    │
              └──────┬───────┘
                     |
                     | short URL
                     v
              ┌──────────────┐
              │    Client    │
              └──────────────┘
```

The redirect flow is different:

```text
                 Open URL
                     |
                     v
              ┌──────────────┐
              │    Client    │
              └──────┬───────┘
                     |
                     | GET /aB72x
                     v
              ┌──────────────┐
              │ Application  │
              └──────┬───────┘
                     |
                     | Lookup aB72x
                     v
              ┌──────────────┐
              │ URL Mapping  │
              └──────┬───────┘
                     |
                     | Original URL
                     v
              ┌──────────────┐
              │ HTTP Redirect│
              └──────┬───────┘
                     |
                     v
              Original Website
```

---

# 8. Creating a Short URL

Let's break the creation process into individual steps.

## Step 1 — Receive the Original URL

The client sends:

```json
{
  "url": "https://example.com/products/iphone-17-pro"
}
```

---

## Step 2 — Validate the URL

The application checks whether the supplied URL is acceptable.

For example:

```text
https://example.com
```

is valid.

Whereas:

```text
not-a-url
```

should be rejected.

Security and validation will be covered in more detail later.

---

## Step 3 — Generate a Short Code

The application generates something such as:

```text
aB72x
```

The code should be sufficiently unique for the system.

---

## Step 4 — Store the Mapping

The application stores:

```text
aB72x
    ↓
https://example.com/products/iphone-17-pro
```

---

## Step 5 — Return the Short URL

The client receives:

```text
https://short.ly/aB72x
```

The creation flow is therefore:

```text
Original URL
     |
     v
Validate
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

---

# 9. Opening a Short URL

Now consider:

```text
https://short.ly/aB72x
```

The browser makes:

```text
GET /aB72x
```

The application extracts:

```text
aB72x
```

Then it performs a lookup:

```text
aB72x
   |
   v
https://example.com/products/iphone-17-pro
```

Finally, it returns a redirect response.

The browser follows that redirect and opens:

```text
https://example.com/products/iphone-17-pro
```

The important point is:

> The URL Shortener does not need to render the destination page. It only needs to tell the browser where to go.

---

# 10. HTTP Redirect

The application can respond with an HTTP redirect such as:

```http
HTTP/1.1 302 Found
Location: https://example.com/products/iphone-17-pro
```

The browser sees the `Location` header and navigates to the destination.

Conceptually:

```text
Client
  |
  | GET /aB72x
  v
URL Shortener
  |
  | 302 Location: https://example.com/products/iphone-17-pro
  v
Client
  |
  | GET /products/iphone-17-pro
  v
Destination Server
```

We will discuss different redirect status codes later.

---

# 11. Why Not Encode the Entire URL?

One obvious question is:

> Why don't we simply encode the original URL into the short URL?

For example:

```text
Original URL
     |
     v
Encode
     |
     v
Short URL
```

Encoding does not necessarily make data smaller.

For example, Base64 encoding can actually make data slightly larger.

A URL Shortener generally does something different:

```text
Long URL
    |
    v
Generate Small Identifier
    |
    v
Store Mapping
```

The short URL contains an identifier, not the entire original URL.

This distinction is fundamental.

---

# 12. Why Does the System Need Storage?

Suppose we generate:

```text
aB72x
```

and return:

```text
https://short.ly/aB72x
```

Later, a user requests:

```text
GET /aB72x
```

How does the application know the destination?

It needs the mapping:

```text
aB72x → original URL
```

Therefore, some form of storage is required if we want the mapping to survive application restarts and be shared across multiple application servers.

Possible storage options include:

* In-memory storage
* Relational database
* NoSQL database
* Key-value database
* Distributed cache

We will start with the simplest option and introduce persistent storage when its limitations become clear.

---

# 13. Why Not Use Only a Database ID?

Another approach is:

```text
Database ID = 12345
```

and generate:

```text
https://short.ly/12345
```

This works technically.

However, sequential IDs reveal information.

For example:

```text
/1000
/1001
/1002
/1003
```

make it easy to guess other identifiers.

Depending on the application, this may expose information about:

* Number of URLs created
* Resource identifiers
* Creation patterns
* Other users' URLs

Therefore, production systems may use opaque identifiers or Base62-encoded values combined with other strategies.

We will explore this later.

---

# 14. Short Code Requirements

A good short code should generally be:

### Unique

Two different URLs should not accidentally receive the same code.

```text
aB72x → URL A

aB72x → URL B
```

must not happen.

---

### Compact

The purpose of the service is to make URLs shorter.

```text
aB72x
```

is preferable to:

```text
550e8400-e29b-41d4-a716-446655440000
```

for a URL-shortening service.

---

### URL-Safe

The generated characters should work safely inside URLs.

Common characters include:

```text
0-9
a-z
A-Z
```

---

### Efficient to Generate

The system may generate thousands or millions of short URLs.

Code generation should therefore be fast.

---

### Difficult to Guess When Required

For public systems, predictable identifiers can make enumeration easier.

For example:

```text
/abc001
/abc002
/abc003
```

are easy to enumerate.

Depending on security requirements, random or sufficiently opaque identifiers may be preferable.

---

# 15. The Read-Heavy Nature of URL Shorteners

Consider a URL:

```text
https://short.ly/aB72x
```

It may be shared on:

* Social media
* Email
* Websites
* Messages
* Advertisements

The URL may be opened thousands or millions of times.

But it may have been created only once.

For example:

```text
Create:

1 request
    |
    v
aB72x created


Reads:

aB72x
  |
  +---- Request 1
  +---- Request 2
  +---- Request 3
  +---- ...
  +---- Request 100,000
```

Therefore:

```text
Read traffic >>> Write traffic
```

This is one of the most important characteristics of the system.

It will eventually influence our caching and scaling strategy.

---

# 16. Basic Architecture

At the beginning, we can model the system as:

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

The storage implementation can initially be very simple.

As the system grows, it will evolve into:

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
          ┌──────────┼──────────┐
          |          |          |
          v          v          v
       App 1      App 2      App 3
          |          |          |
          └──────────┼──────────┘
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

We will not implement this architecture immediately.

Each component will be introduced when we encounter the problem it solves.

---

# 17. Core Data Model

The fundamental information we need is:

```text
short_code
original_url
created_at
expires_at
```

Conceptually:

```text
+------------+-----------------------------------------+
| short_code | original_url                            |
+------------+-----------------------------------------+
| aB72x      | https://example.com/products/iphone     |
| xY92p      | https://example.com/articles/design     |
| K8mQa      | https://example.com/docs/architecture   |
+------------+-----------------------------------------+
```

Additional fields will be introduced later when we add:

* Expiration
* Ownership
* Analytics
* Revocation
* Authentication

---

# 18. What Happens When the URL Does Not Exist?

Suppose the user requests:

```text
GET /unknown123
```

and the system cannot find the code.

The system should not attempt to redirect.

Conceptually:

```text
GET /unknown123
        |
        v
Lookup
        |
        v
Not Found
        |
        v
404 Not Found
```

This is another reason why the redirect endpoint must perform a reliable lookup.

---

# 19. What Happens When the URL Has Expired?

Suppose:

```text
short_code = aB72x
expires_at = 2026-08-25 12:00
```

and the request arrives after expiration.

The application should detect that the URL is no longer valid.

```text
GET /aB72x
      |
      v
Lookup
      |
      v
Expired
      |
      v
Do not redirect
```

The exact expiration behavior will be designed later.

---

# 20. URL Shortener vs URL Redirector

It is useful to distinguish the two concepts.

A **URL redirector** simply redirects one URL to another.

A **URL Shortener** additionally provides:

```text
Long URL
    |
    v
Short Identifier
    |
    v
Persistent Mapping
    |
    v
Redirect
```

The short-code generation and mapping storage are what make URL shortening interesting from a system-design perspective.

---

# 21. The Problems We Will Solve Next

Now that we understand the basic concept, several technical questions remain.

### Question 1

How should we generate the short code?

```text
Long URL
   |
   v
?????
   |
   v
aB72x
```

### Question 2

How do we guarantee uniqueness?

```text
aB72x
```

must not accidentally point to two different URLs.

### Question 3

Where should we store the mapping?

```text
aB72x → Original URL
```

### Question 4

How should we design the database?

### Question 5

How should redirects work?

### Question 6

How do we protect the service from malicious URLs and abuse?

### Question 7

How do we handle millions of redirects?

### Question 8

Can Redis reduce database load?

### Question 9

How do we scale application servers?

### Question 10

How do we handle analytics without slowing redirects?

These questions will drive the remaining implementation.

---

# 22. Key Takeaways

A URL Shortener is fundamentally a mapping system:

```text
Short Code → Original URL
```

The two primary operations are:

```text
POST /shorten
```

to create a short URL, and:

```text
GET /{short_code}
```

to redirect the user.

The basic flow is:

```text
                 CREATE

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


                 REDIRECT

Short URL
   |
   v
Extract Short Code
   |
   v
Lookup Mapping
   |
   v
Get Original URL
   |
   v
HTTP Redirect
```

The implementation itself is simple.

The interesting system-design challenges appear when we ask:

> **How do we make this system persistent, secure, fast, highly available, and scalable?**

That is what we will solve in the next chapters.
