# URL Shortener — Short Code Generation Strategies

## 1. Introduction

Our URL Shortener can now:

* Store URL mappings in a database.
* Use Redis to cache frequently accessed URLs.
* Run multiple stateless application servers.
* Distribute traffic through a load balancer.

Our current architecture is:

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

Now we need to solve one of the most important problems in a URL Shortener:

> **How do we generate a unique short code for every URL?**

For example:

```text
Long URL:
https://example.com/products/iphone-17-pro

Short URL:
https://short.ly/aB72x
```

The value:

```text
aB72x
```

is the **short code**.

Generating this code sounds simple.

At small scale, it is simple.

At large scale, however, we need to think about:

* Uniqueness
* Collisions
* Concurrency
* Distributed application servers
* Code length
* Predictability
* Security
* Database constraints
* Throughput

---

# 2. What Exactly Are We Generating?

Suppose our API receives:

```text
POST /shorten
```

with:

```text
https://example.com/products/iphone
```

We need to generate something like:

```text
aB72x
```

and store:

```text
short_code                 original_url
-----------                ----------------------------
aB72x                      https://example.com/products/iphone
```

The client receives:

```text
https://short.ly/aB72x
```

Later:

```text
GET /aB72x
```

allows us to find the original URL.

Therefore:

```text
Short Code
    |
    v
Unique Identifier
    |
    v
URL Mapping
```

---

# 3. Requirements for a Short Code

A good short-code generation mechanism should ideally provide:

### 1. Uniqueness

Two different URLs should not accidentally receive the same code.

### 2. Short length

For example:

```text
aB72x
```

is better than:

```text
928374928374928374
```

### 3. High throughput

The system may need to generate:

```text
10,000 codes/sec
```

or more.

### 4. Distributed safety

Multiple application servers may generate codes concurrently.

### 5. Low collision probability

We should minimize the need to regenerate codes.

### 6. Security where required

Predictable codes can expose information.

---

# 4. The Simplest Approach — Random String

One obvious solution is:

```text
Generate random characters
```

For example:

```text
aB72x
```

using:

```text
[a-z]
[A-Z]
[0-9]
```

Suppose our alphabet contains:

```text
26 lowercase
26 uppercase
10 digits
```

Total:

```text
62 characters
```

For a 6-character code:

```text
62^6
```

possible combinations.

That is:

```text
56,800,235,584
```

possible codes.

This provides a large namespace.

---

# 5. Random Code Generation

Conceptually:

```text
Request
   |
   v
Random Generator
   |
   v
"X7kP2a"
   |
   v
Check Database
   |
   +---- Exists → Generate Again
   |
   +---- Doesn't exist
            |
            v
        Store URL
```

This is easy to understand and easy to implement.

But there is an important issue.

---

# 6. Random Collision

Suppose we generate:

```text
aB72x
```

and it already exists.

Then:

```text
INSERT
```

would violate the uniqueness constraint.

Therefore we need:

```text
Generate
   |
   v
Check / Insert
   |
   +---- Collision
   |       |
   |       v
   |    Generate Again
   |
   +---- Success
```

The database must still enforce uniqueness.

---

# 7. Never Trust Application-Level Uniqueness Alone

A common mistake is:

```python
if not database.exists(code):
    database.insert(code)
```

This looks safe.

But consider two application servers.

### Server 1

```text
Check:
aB72x does not exist
```

### Server 2

At exactly the same time:

```text
Check:
aB72x does not exist
```

Both believe the code is available.

Then:

```text
Server 1 → INSERT aB72x
Server 2 → INSERT aB72x
```

One must fail.

This is a classic race condition.

---

# 8. Database Unique Constraint

The database should enforce:

```text
short_code UNIQUE
```

For example:

```sql
CREATE UNIQUE INDEX idx_short_code
ON urls(short_code);
```

Now the database guarantees:

```text
aB72x → only one record
```

even when multiple servers operate concurrently.

This is an important principle:

> **Application logic can attempt uniqueness; the database must enforce uniqueness.**

---

# 9. Collision Handling

Our safe algorithm becomes:

```text
Generate random code
        |
        v
Attempt INSERT
        |
        +---- SUCCESS
        |       |
        |       v
        |    Return code
        |
        +---- UNIQUE VIOLATION
                |
                v
           Generate again
```

We do not need a separate existence check.

The database insert itself is the authoritative uniqueness test.

This avoids a common race condition.

---

# 10. Why Not Check Before Insert?

Consider:

```text
exists(code)
```

followed by:

```text
insert(code)
```

There are two database operations.

Between them, another server can insert the same code.

Therefore:

```text
CHECK → INSERT
```

is not atomic.

Instead:

```text
INSERT
```

with a unique constraint lets the database handle the race safely.

---

# 11. Random Code Length

How long should the short code be?

Suppose we use Base62 characters.

### 4 characters

```text
62^4
```

=

```text
14,776,336
```

Approximately 14.7 million combinations.

### 5 characters

```text
62^5
```

=

```text
916,132,832
```

Approximately 916 million combinations.

### 6 characters

```text
62^6
```

=

```text
56,800,235,584
```

Approximately 56.8 billion combinations.

### 7 characters

```text
62^7
```

=

```text
3,521,614,606,208
```

Approximately 3.5 trillion combinations.

Therefore, even a relatively short code can provide a very large namespace.

---

# 12. Base62

Base62 is commonly used for compact identifiers.

The alphabet is:

```text
abcdefghijklmnopqrstuvwxyz
ABCDEFGHIJKLMNOPQRSTUVWXYZ
0123456789
```

That gives:

```text
62 symbols
```

The advantage is simple:

> More possible values can fit into fewer characters.

For example:

```text
Decimal ID:
1258392
```

can be encoded into a shorter Base62 representation.

---

# 13. Base62 Is an Encoding, Not Encryption

This distinction is very important.

Suppose:

```text
123456
```

is converted into:

```text
w7e
```

That does not mean:

```text
w7e
```

is secure.

Base62 is simply an encoding scheme.

Anyone who knows the encoding algorithm can decode it.

Therefore:

> **Do not use Base62 as a security mechanism.**

If sensitive information must be protected, use proper cryptographic mechanisms.

---

# 14. Sequential Database ID + Base62

Another popular strategy is:

```text
Database ID
     |
     v
Base62 Encode
     |
     v
Short Code
```

Suppose the database generates:

```text
1
2
3
4
5
```

Base62 encoding produces compact strings.

Conceptually:

```text
Database ID     Short Code
-----------     ----------
1               1
2               2
3               3
...
1000            g8
...
```

This is extremely efficient.

But it introduces another question:

> Where does the unique numeric ID come from?

Usually the database sequence / auto-increment mechanism can provide it.

---

# 15. Architecture With Database ID

The flow becomes:

```text
POST /shorten
      |
      v
Insert URL
      |
      v
Database generates ID
      |
      v
ID = 123456
      |
      v
Base62 Encode
      |
      v
"z3k9"
      |
      v
Return short URL
```

However, we have a problem.

The short code is derived from the database ID, so we need to determine exactly when and how the mapping is stored.

There are several ways to design this.

---

# 16. Approach A — Insert Then Generate Code

We could insert:

```text
original_url
```

and let the database generate:

```text
id = 123456
```

Then:

```text
123456
   |
   v
Base62
   |
   v
z3k9
```

Then update:

```text
short_code = z3k9
```

The sequence becomes:

```text
INSERT
  ↓
Get ID
  ↓
Encode ID
  ↓
UPDATE
```

This requires multiple operations.

---

# 17. Approach B — Use ID Directly

We could make the public short code a Base62 representation of the database ID.

Conceptually:

```text
Database:
id = 123456

Public:
short.ly/z3k9
```

The database does not necessarily need a separate random short-code column.

During redirect:

```text
z3k9
  |
  v
Base62 Decode
  |
  v
123456
  |
  v
Database lookup by primary key
```

This can be very efficient.

---

# 18. The Predictability Problem

Sequential IDs have a security/privacy issue.

Suppose:

```text
short.ly/a
short.ly/b
short.ly/c
```

or their longer Base62 equivalents correspond to sequential database IDs.

Someone could potentially enumerate URLs.

For example:

```text
short.ly/abc01
short.ly/abc02
short.ly/abc03
```

If those codes map directly to IDs, users may discover that IDs are sequential.

This can expose:

* Approximate creation volume
* URL enumeration opportunities
* Potential access to links that should not be discoverable

Therefore:

> **A short identifier should not automatically be treated as an authorization mechanism.**

---

# 19. Obfuscating Sequential IDs

One approach is to obfuscate the numeric ID before Base62 encoding.

Conceptually:

```text
Database ID
    |
    v
Obfuscation
    |
    v
Base62
    |
    v
Short Code
```

For example:

```text
123456
   |
   v
Obfuscation function
   |
   v
987654
   |
   v
Base62
   |
   v
xK7p
```

This makes simple sequential enumeration harder.

However, custom obfuscation introduces complexity and should not be confused with cryptographic security.

---

# 20. Random IDs vs Sequential IDs

Let's compare.

| Feature                | Random Code  | Sequential ID + Base62 |
| ---------------------- | ------------ | ---------------------- |
| Implementation         | Simple       | Simple/Moderate        |
| Collision handling     | Required     | Usually unnecessary    |
| Database dependency    | Can be lower | Stronger               |
| Predictability         | Low          | High                   |
| Enumeration risk       | Lower        | Higher                 |
| Code length            | Configurable | Compact                |
| Generation throughput  | High         | Very high              |
| Distributed generation | Easy         | Requires ID strategy   |
| Security by itself     | No           | No                     |

Neither approach is universally correct.

The choice depends on the requirements.

---

# 21. UUID

Another common approach is UUID.

A UUID looks like:

```text
550e8400-e29b-41d4-a716-446655440000
```

It provides an enormous identifier space.

But it is not particularly short.

Using the UUID directly:

```text
short.ly/550e8400-e29b-41d4-a716-446655440000
```

defeats one of the main purposes of a URL Shortener.

We could encode UUID bytes into Base62/Base64-like representations, but the resulting code will still generally be longer than a small sequential/random code.

---

# 22. Snowflake-Style IDs

At very large scale, we may want IDs generated without relying on a single database.

A common family of solutions is:

> Snowflake-style distributed IDs.

The identifier is composed of fields such as:

```text
Timestamp
Worker ID
Sequence
```

Conceptually:

```text
┌────────────┬───────────┬──────────┐
│ Timestamp  │ Worker ID │ Sequence │
└────────────┴───────────┴──────────┘
```

This allows multiple servers to generate unique IDs independently.

For example:

```text
Server 1 → ID A
Server 2 → ID B
Server 3 → ID C
```

without asking the database for every ID.

---

# 23. Why Distributed IDs Matter

Suppose we have:

```text
100 application servers
```

and each receives URL creation requests.

If every request must first contact one central ID generator:

```text
App 1 ──┐
App 2 ──┤
App 3 ──┼──> Central ID Generator
...     │
App 100 ┘
```

the ID generator can become a bottleneck.

A distributed ID strategy allows:

```text
App 1 → Generate locally
App 2 → Generate locally
App 3 → Generate locally
```

while still maintaining uniqueness.

---

# 24. Distributed ID + Base62

We could then use:

```text
Distributed ID
      |
      v
Base62 Encode
      |
      v
Short Code
```

This provides:

* Distributed generation
* High throughput
* No database round trip for ID allocation
* Compact public identifiers

But there is an important tradeoff.

The distributed ID may be larger than a simple sequential database ID.

Therefore, the resulting Base62 code may be longer.

---

# 25. The Important Design Tradeoff

There is no universally "best" ID generator.

We need to balance:

```text
Shortness
Uniqueness
Throughput
Predictability
Security
Complexity
Distributed generation
```

For example:

```text
Sequential DB ID
```

is very simple.

But:

```text
Distributed ID
```

is better suited to extremely high write throughput.

Meanwhile:

```text
Random Base62
```

provides less predictability but requires collision handling.

---

# 26. Collision Probability

Random identifiers introduce an interesting mathematical problem.

Even if the namespace is large, collisions eventually become possible.

This is related to the:

> Birthday Paradox.

Suppose there are:

```text
N
```

possible codes.

As the number of generated codes increases, the probability of at least one collision increases faster than many people intuitively expect.

This does not mean random IDs are bad.

It means:

> **Random ID systems must still handle collisions correctly.**

---

# 27. Why Database Uniqueness Still Matters

Suppose we have:

```text
62^6 ≈ 56.8 billion
```

possible codes.

That sounds enormous.

But our system may eventually create millions or billions of URLs.

We should still have:

```sql
UNIQUE(short_code)
```

because the database provides the final correctness guarantee.

The application can generate a highly unlikely collision.

The database guarantees that the same code cannot be stored twice.

---

# 28. Concurrency

Now consider our horizontally scaled architecture:

```text
                    Load Balancer
                         |
            ┌────────────┼────────────┐
            |            |            |
            v            v            v
          App 1        App 2        App 3
```

Suppose all three servers receive:

```text
POST /shorten
```

at the same time.

Each server generates a code independently.

We must guarantee:

```text
URL A → unique code
URL B → unique code
URL C → unique code
```

This is why the generation mechanism must work correctly under concurrency.

---

# 29. Race Condition Example

Imagine both servers generate:

```text
aB72x
```

at exactly the same time.

Without a database unique constraint:

```text
App 1 → aB72x
App 2 → aB72x
```

we could end up with ambiguity.

With:

```sql
UNIQUE(short_code)
```

the database guarantees:

```text
App 1 → SUCCESS
App 2 → UNIQUE VIOLATION
```

App 2 can generate another code and retry.

---

# 30. Production-Safe Random Generation

A robust algorithm is:

```text
function create_short_url(url):

    repeat:

        code = generate_random_code()

        try:
            insert(code, url)

            return code

        except unique_constraint_violation:

            continue
```

The important detail is:

> The database insert and unique constraint are the final authority.

---

# 31. Retry Limit

We should not retry forever.

For example:

```text
Attempt 1
Attempt 2
Attempt 3
Attempt 4
Attempt 5
```

If all fail:

```text
Return error
```

In a properly sized namespace, repeated collisions should be rare.

A large number of consecutive collisions may indicate:

* Namespace too small
* Bad random generator
* Implementation bug
* Database issue
* Attack
* Unexpected traffic pattern

---

# 32. Randomness Quality

If we use random short codes, we should use an appropriate secure random generator when unpredictability matters.

Avoid relying on weak pseudo-random mechanisms for security-sensitive identifiers.

For Python, for example:

```python
import secrets
```

can be used for cryptographically strong random values.

The exact implementation will be introduced in the code chapter.

---

# 33. Security vs Uniqueness

These are different requirements.

### Uniqueness

Means:

```text
Two URLs should not have the same code.
```

### Unpredictability

Means:

```text
Someone should not easily guess another valid code.
```

A sequential ID provides:

```text
Strong uniqueness
Weak unpredictability
```

A random secure token provides:

```text
Strong uniqueness probability
Better unpredictability
```

but still requires collision handling.

---

# 34. Do Short URLs Need to Be Secure?

It depends on the product.

If:

```text
short.ly/aB72x
```

is simply a public redirect, predictability may not matter much.

But suppose the short URL provides access to:

```text
Private invitation
Private document
Private meeting
Password reset
Payment action
```

Then guessing the code could become a security problem.

In those systems:

> The short code may need sufficient entropy, and authorization should still be enforced separately.

---

# 35. Never Treat the Short Code as Authorization

This is extremely important.

Bad architecture:

```text
GET /aB72x

If code exists:
    allow access
```

If the URL represents private information, the code itself should not be the only authorization mechanism.

Instead:

```text
Short Code
    |
    v
Resolve Resource
    |
    v
Authorization
    |
    v
Allow / Deny
```

A URL Shortener should not accidentally become an insecure access-control system.

---

# 36. Code Alphabet

We should also think about characters.

A Base62 alphabet contains:

```text
a-z
A-Z
0-9
```

But visually confusing characters may sometimes be undesirable in human-readable codes.

For example:

```text
0
O

1
l
I
```

If users manually type codes, we may choose a reduced alphabet.

For machine-generated links sent by email/SMS, Base62 is generally more convenient.

---

# 37. URL-Safe Characters

The generated code will appear inside a URL.

Therefore, we should avoid characters that require unnecessary URL encoding.

A simple alphabet such as:

```text
[a-zA-Z0-9]
```

is URL-friendly.

This makes codes like:

```text
aB72x
```

easy to use.

---

# 38. Case Sensitivity

We must decide whether:

```text
aB72x
```

and:

```text
ab72x
```

are different.

If we use Base62:

```text
a
```

and:

```text
A
```

represent different values.

Therefore:

```text
aB72x != ab72x
```

This increases the namespace significantly.

But it also means users must preserve case correctly.

For automatically generated URLs, this is generally acceptable.

---

# 39. Code Length Strategy

We have several options.

### Fixed length

Every code is:

```text
6 characters
```

Example:

```text
aB72xQ
K8mP2z
```

### Variable length

Codes grow naturally:

```text
1
2
...
z
10
11
...
```

Variable-length codes can be more compact at low volumes.

Fixed-length codes are often easier to reason about and provide predictable URL lengths.

The choice depends on product requirements.

---

# 40. Database Indexing

The short code is part of every redirect request:

```text
GET /aB72x
```

Therefore, the database needs an efficient lookup.

We should have:

```sql
UNIQUE(short_code)
```

This usually creates an index.

Then:

```sql
SELECT original_url
FROM urls
WHERE short_code = 'aB72x';
```

can efficiently locate the record.

---

# 41. Short Code as Primary Key

Could we make:

```text
short_code
```

the primary key?

Yes.

For example:

```sql
CREATE TABLE urls (
    short_code VARCHAR(16) PRIMARY KEY,
    original_url TEXT NOT NULL
);
```

Then:

```text
aB72x
```

directly identifies the record.

This is simple.

However, there may be reasons to retain a separate internal numeric ID, such as:

* Analytics relationships
* Internal references
* Efficient joins
* Easier database operations
* Future architecture changes

We should choose based on the broader system requirements.

---

# 42. Recommended Educational Design

For this repository, we want to demonstrate multiple strategies rather than claim there is only one correct answer.

We will implement and compare:

```text
Strategy 1
Random Base62

Strategy 2
Database ID + Base62

Strategy 3
Distributed ID + Base62
```

This allows us to understand the tradeoffs.

---

# 43. Strategy 1 — Random Base62

Architecture:

```text
Request
   |
   v
Secure Random Generator
   |
   v
Base62 code
   |
   v
Database INSERT
   |
   +---- Collision → Retry
   |
   +---- Success
```

Advantages:

* Simple
* Unpredictable
* Distributed-friendly
* No central ID generator

Disadvantages:

* Collision handling
* Random generation
* Database uniqueness check
* Probability calculations

---

# 44. Strategy 2 — Database ID + Base62

Architecture:

```text
Request
   |
   v
Database ID
   |
   v
Base62
   |
   v
Short Code
```

Advantages:

* Very low collision complexity
* Easy to reason about
* Compact
* Database guarantees uniqueness

Disadvantages:

* More predictable
* Tied to ID-generation mechanism
* Central database can become part of the creation hot path

---

# 45. Strategy 3 — Distributed ID + Base62

Architecture:

```text
Request
   |
   v
Distributed ID Generator
   |
   v
Base62
   |
   v
Short Code
   |
   v
Database
```

Advantages:

* High write throughput
* Distributed generation
* No database round trip just for ID allocation
* Suitable for large-scale systems

Disadvantages:

* More complex
* Worker ID management
* Clock considerations
* Longer IDs may produce longer codes
* More operational complexity

---

# 46. What Happens When We Have Multiple Regions?

Suppose our application is deployed in:

```text
US
Europe
Asia
```

and all regions generate short codes.

A simple database auto-increment may not be enough if each region has independent databases.

We need a globally unique ID strategy.

Possible approaches include:

```text
Central ID service
Distributed ID generator
Region-specific ID ranges
Database sequence service
UUID/ULID-like identifiers
```

This is where distributed ID generation becomes more interesting.

---

# 47. Clock Problems With Distributed IDs

Some distributed ID algorithms use timestamps.

For example:

```text
Timestamp
Worker
Sequence
```

If a server's clock moves backward:

```text
Current time:
12:00:10

Clock suddenly:
12:00:05
```

the generator can potentially produce IDs that violate expected ordering.

Therefore, production distributed ID generators need clock-handling strategies.

This is one reason distributed ID generation is more complex than:

```text
AUTO_INCREMENT
```

---

# 48. Ordering vs Uniqueness

Another important distinction:

A system may require:

```text
Unique IDs
```

but not:

```text
Strictly increasing IDs
```

For URL Shorteners, we generally need uniqueness.

We usually do not need:

```text
URL A ID < URL B ID
```

to guarantee anything about request order.

This means we have more freedom in choosing the ID-generation strategy.

---

# 49. Why We Should Not Overengineer

Imagine our system generates:

```text
10,000 URLs/day
```

We probably do not need:

```text
100-node distributed ID service
```

A database-backed design with Base62 may be perfectly adequate.

System design should always begin with:

```text
Actual workload
      |
      v
Actual bottleneck
      |
      v
Required architecture
```

not:

```text
Popular technology
      |
      v
Force it into system
```

---

# 50. Capacity Planning

Before selecting an ID strategy, estimate:

```text
URLs created/day
URLs created/sec
Peak creation rate
Total URLs/year
Expected lifetime
```

For example:

```text
1 million URLs/day
```

Average:

```text
1,000,000 / 86,400
≈ 11.6 URLs/sec
```

Even if peak traffic is:

```text
100 URLs/sec
```

a normal database-backed ID generator can likely handle the creation workload depending on the database and infrastructure.

The redirect workload may be much larger.

This reinforces an important architectural observation:

> **URL creation and URL redirection have very different traffic patterns.**

---

# 51. Write Traffic vs Read Traffic

Suppose:

```text
URL creation:
1,000/sec

Redirect:
100,000/sec
```

The system is:

```text
Write:
1K/sec

Read:
100K/sec
```

Therefore, optimizing redirect reads with:

```text
Redis
```

may be more important than building an extremely sophisticated ID generator.

This is why we introduced caching before distributed ID generation.

---

# 52. Recommended Initial Design

For our educational URL Shortener, we will initially use:

```text
Secure Random Base62
```

with:

```text
Database UNIQUE(short_code)
```

and:

```text
Retry on collision
```

Why?

Because it teaches several important concepts:

```text
Random ID generation
Collision handling
Database constraints
Concurrency
Distributed application servers
Security considerations
```

Then we can implement the alternative strategies separately for comparison.

---

# 53. Recommended Production Choice

There is no universal production choice.

For a moderate-scale system:

```text
Database ID + Base62
```

can be extremely simple and reliable.

For a system requiring unpredictable links:

```text
Secure random Base62
```

is attractive.

For extremely high write throughput across many nodes/regions:

```text
Distributed ID + Base62
```

may make more sense.

The correct choice depends on:

```text
Scale
Security requirements
Infrastructure
Database architecture
Regional deployment
Operational complexity
```

---

# 54. Short Code Generation Flow

Our first implementation will follow:

```text
              Create Short URL
                     |
                     v
              Generate Code
                     |
                     v
               Random Base62
                     |
                     v
              Database INSERT
                     |
            ┌────────┴────────┐
            |                 |
         Success           Collision
            |                 |
            v                 v
       Return URL          Generate Again
```

Database:

```text
short_code UNIQUE
```

This gives us correctness even with concurrent application servers.

---

# 55. Complete Architecture

Combining everything we have learned:

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
                ┌──────────┴──────────┐
                |                     |
                v                     v
              Redis               Database
              Cache             Source of Truth
                                      |
                                      v
                               UNIQUE short_code
```

Create flow:

```text
Client
  |
  v
Load Balancer
  |
  v
Application
  |
  v
Generate random Base62
  |
  v
Database INSERT
  |
  +---- Collision → Retry
  |
  +---- Success
  |
  v
Return short URL
```

Redirect flow:

```text
Client
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

# 56. Important Lessons

This chapter introduced several fundamental system-design concepts.

### Uniqueness

Use:

```text
Database UNIQUE constraint
```

as the final correctness guarantee.

### Collision handling

Random identifiers must tolerate collisions.

### Concurrency

Multiple application servers can generate identifiers simultaneously.

### Encoding

Base62 provides compact URL-safe representations.

### Security

Short codes should not automatically be considered authorization tokens.

### Distributed generation

At very large scale, IDs can be generated without relying on one database.

### Tradeoffs

Every strategy has advantages and disadvantages.

---

# 57. Final Design Principle

The most important principle is:

> **Do not optimize short-code generation in isolation. Design it according to the actual write workload, security requirements, and database architecture.**

For our URL Shortener:

```text
Redirect traffic
      ↓
Very high
      ↓
Redis + horizontal scaling
```

while:

```text
URL creation traffic
      ↓
Usually much lower
      ↓
Simple reliable ID generation may be enough
```

This distinction prevents unnecessary complexity.

---

# 58. What Comes Next?

We now know how to generate short codes.

But our system has another important problem.

Imagine someone creates:

```text
https://short.ly/aB72x
```

and thousands of users click it.

We need to answer:

```text
How many times was it clicked?

Who clicked it?
When?
From which country?
Which device?
Which browser?
What was the referrer?
```

If we synchronously write analytics data to the database for every redirect:

```text
100,000 redirects/sec
        |
        v
100,000 analytics writes/sec
```

we can create another major bottleneck.

Therefore, the next problem is:

> **How do we build scalable click analytics without slowing down the redirect path?**

This introduces an important system-design topic:

```text
Asynchronous Processing
Message Queues
Event-Driven Architecture
Kafka / RabbitMQ-style systems
Eventual Consistency
```

That will be the focus of the next chapter.
