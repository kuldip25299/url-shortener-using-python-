

# URL Shortener — Short Code Generation

## 1. Introduction

One of the most important parts of a URL Shortener is generating the **short code**.

Given:

```text
https://example.com/products/iphone-17-pro
````

we need to generate something like:

```text
aB72x
```

so that the final URL becomes:

```text
https://short.ly/aB72x
```

The short code is the key used to find the original URL.

The fundamental relationship is:

```text
aB72x
  ↓
https://example.com/products/iphone-17-pro
```

Therefore, our short-code generation strategy must consider:

* Uniqueness
* Collision handling
* Length
* URL safety
* Generation speed
* Scalability
* Predictability
* Security

---

# 2. What Exactly Is a Short Code?

A short code is simply an identifier.

For example:

```text
aB72x
```

The system stores:

```text
short_code = aB72x
original_url = https://example.com/products/iphone-17-pro
```

When the user visits:

```text
https://short.ly/aB72x
```

the application extracts:

```text
aB72x
```

and performs a lookup.

Conceptually:

```text
GET /aB72x
     |
     v
Extract "aB72x"
     |
     v
Lookup
     |
     v
Original URL
     |
     v
Redirect
```

The short code itself does not need to contain the original URL.

It is simply a compact identifier for the stored mapping.

---

# 3. Requirements for a Good Short Code

A production-quality short-code strategy should satisfy several requirements.

## 3.1 Uniqueness

Different URL records should not accidentally use the same code.

Bad:

```text
abc12 → URL A
abc12 → URL B
```

Good:

```text
abc12 → URL A
xyz89 → URL B
```

---

## 3.2 Short Length

The purpose of the system is to produce shorter URLs.

For example:

```text
aB72x
```

is much more appropriate than:

```text
550e8400-e29b-41d4-a716-446655440000
```

---

## 3.3 URL Safe

The generated code should work safely inside a URL.

A common character set is:

```text
0-9
a-z
A-Z
```

This gives us:

```text
62 possible characters
```

This encoding is commonly called **Base62**.

---

## 3.4 Fast Generation

A URL Shortener may receive many creation requests.

Generating a code should therefore be inexpensive.

---

## 3.5 Collision Handling

Even if we use random generation, collisions are possible.

The system must have a strategy for dealing with them.

---

## 3.6 Suitable for Distributed Systems

Eventually we may have:

```text
Application 1
Application 2
Application 3
...
Application N
```

The generation strategy should continue to work when multiple servers generate IDs concurrently.

---

# 4. Approach 1 — Random String

The simplest strategy is generating a random string.

For example:

```text
aB72x
K91pq
xY7mQ
```

Using Python:

```python
import secrets
import string

characters = string.ascii_letters + string.digits

short_code = "".join(
    secrets.choice(characters)
    for _ in range(6)
)
```

This generates a six-character code.

For example:

```text
aB72x9
```

---

# 5. Random Code Space

If we use Base62 characters:

```text
0-9
a-z
A-Z
```

we have:

```text
62 characters
```

For a 6-character code:

```text
62^6
```

possible combinations exist.

That is:

```text
56,800,235,584
```

possible codes.

So the theoretical space is very large.

However, a large space does **not** mean collisions are impossible.

---

# 6. Collision Example

Suppose we generate:

```text
aB72x9
```

and store:

```text
aB72x9 → URL A
```

Later, random generation produces:

```text
aB72x9
```

again.

Now we have a collision.

We cannot store:

```text
aB72x9 → URL B
```

without overwriting the first mapping.

Therefore, the application needs to detect the collision.

---

# 7. Collision Handling

A simple strategy is:

```text
Generate code
      |
      v
Check database
      |
      +---- Code does not exist
      |          |
      |          v
      |       Store it
      |
      +---- Code already exists
                 |
                 v
             Generate again
```

Example:

```text
Generate aB72x
       |
       v
Already exists
       |
       v
Generate xY92p
       |
       v
Does not exist
       |
       v
Store
```

This is a perfectly valid strategy for many applications.

However, we must also consider concurrency.

---

# 8. Why Application-Level Checks Are Not Enough

Consider two requests arriving at almost exactly the same time.

```text
Request A
Request B
```

Both generate:

```text
aB72x
```

Then both check:

```text
Does aB72x exist?
```

Both may receive:

```text
NO
```

because neither request has inserted the value yet.

Then:

```text
Request A → INSERT aB72x
Request B → INSERT aB72x
```

This creates a race condition.

Therefore:

> The database should enforce uniqueness.

For example:

```sql
UNIQUE(short_code)
```

The application can still check before inserting for efficiency, but the database constraint is the final protection.

---

# 9. Approach 2 — UUID

Another common approach is using a UUID.

Example:

```text
550e8400-e29b-41d4-a716-446655440000
```

UUIDs provide an enormous identifier space and are designed to be globally unique with extremely high probability.

However, they are not particularly short.

Using them directly:

```text
https://short.ly/550e8400-e29b-41d4-a716-446655440000
```

does not provide a very good URL-shortening experience.

Therefore, UUIDs are usually not ideal as the visible short code.

---

# 10. Approach 3 — Database Auto-Increment ID

Another simple approach is to let the database generate:

```text
1
2
3
4
5
...
```

Suppose a URL gets:

```text
id = 12345
```

We could create:

```text
https://short.ly/12345
```

This is extremely simple.

The database guarantees uniqueness.

However, there are some disadvantages.

---

# 11. Problem With Sequential IDs

Sequential identifiers are predictable.

If someone knows:

```text
https://short.ly/10001
```

they can easily try:

```text
https://short.ly/10002
https://short.ly/10003
https://short.ly/10004
```

This makes enumeration easy.

For a public URL Shortener, this may expose information about the system.

For example:

```text
How many URLs have been created?
```

or potentially allow users to discover other records if access controls are poorly designed.

Therefore, exposing raw database IDs is often undesirable.

---

# 12. Approach 4 — Database ID + Base62

A much more interesting approach is:

```text
Database ID
    ↓
Base62 Encoding
    ↓
Short Code
```

Suppose the database generates:

```text
12345
```

Instead of using:

```text
12345
```

we encode it using Base62:

```text
12345
  ↓
dnh
```

The resulting URL becomes:

```text
https://short.ly/dnh
```

This provides a compact representation of the numeric ID.

---

# 13. What Is Base62?

Base62 uses:

```text
0-9
a-z
A-Z
```

That gives us:

```text
62 characters
```

The character set can be represented as:

```text
0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
```

Each character represents one digit in base 62.

The advantage is that a large integer can be represented using fewer characters.

---

# 14. Decimal vs Base62

Consider a number:

```text
1000000
```

In decimal, it requires:

```text
7 characters
```

Base62 can represent the same number using fewer characters.

As numbers become larger, the difference becomes more useful.

Conceptually:

```text
Decimal ID
    |
    v
1000000
    |
    v
Base62
    |
    v
4c92
```

The exact encoded value depends on the chosen Base62 character mapping.

---

# 15. Base62 Conversion

The conversion works using repeated division.

Suppose:

```text
number = N
base = 62
```

We repeatedly calculate:

```text
remainder = number % 62
number = number // 62
```

Each remainder identifies one Base62 character.

Eventually, the characters are reversed to produce the encoded string.

Conceptually:

```text
Number
  |
  v
Divide by 62
  |
  v
Remainder
  |
  v
Base62 character
  |
  v
Repeat
```

---

# 16. Example Base62 Implementation

A simple implementation can look like:

```python
BASE62_ALPHABET = (
    "0123456789"
    "abcdefghijklmnopqrstuvwxyz"
    "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
)


def encode_base62(number: int) -> str:
    if number == 0:
        return BASE62_ALPHABET[0]

    result = []

    while number > 0:
        number, remainder = divmod(number, 62)
        result.append(BASE62_ALPHABET[remainder])

    return "".join(reversed(result))
```

Example:

```python
print(encode_base62(12345))
```

The result is a compact string representation of the number.

---

# 17. Decoding Base62

Because Base62 is an encoding scheme, we can also decode it.

Conceptually:

```text
Short Code
    |
    v
Base62 Decode
    |
    v
Original Integer ID
```

A simple implementation:

```python
BASE62_ALPHABET = (
    "0123456789"
    "abcdefghijklmnopqrstuvwxyz"
    "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
)

BASE62_LOOKUP = {
    character: index
    for index, character in enumerate(BASE62_ALPHABET)
}


def decode_base62(value: str) -> int:
    number = 0

    for character in value:
        number = number * 62 + BASE62_LOOKUP[character]

    return number
```

For example:

```python
encoded = encode_base62(12345)

decoded = decode_base62(encoded)

print(encoded)
print(decoded)
```

The decoded value should return:

```text
12345
```

---

# 18. Database ID + Base62 Flow

The complete approach becomes:

```text
Create URL
    |
    v
Database
    |
    v
Generate ID = 12345
    |
    v
Base62 Encode
    |
    v
dnh
    |
    v
https://short.ly/dnh
```

For redirect:

```text
https://short.ly/dnh
    |
    v
Extract dnh
    |
    v
Base62 Decode
    |
    v
12345
    |
    v
Database lookup
    |
    v
Original URL
```

This is an elegant design.

However, it has an important architectural consideration.

---

# 19. Does Base62 Make IDs Secret?

No.

This is important.

Base62 is **encoding**, not encryption.

If:

```text
12345 → dnh
```

then someone who understands the encoding can decode:

```text
dnh → 12345
```

Therefore:

```text
Base62 ≠ Security
```

If sequential IDs are encoded using Base62:

```text
dnh
dni
dnj
dnk
```

they may still be predictable.

If the application requires unpredictable short codes, we need a different strategy.

---

# 20. Approach 5 — Random + Base62

Another option is generating a random integer and encoding it using Base62.

Conceptually:

```text
Secure Random Number
        |
        v
Base62 Encode
        |
        v
Short Code
```

For example:

```text
Random Integer
      |
      v
827364829374
      |
      v
Base62
      |
      v
X7kP2a
```

This provides a compact and less predictable identifier.

However, collisions are still theoretically possible.

Therefore:

```text
Unique database constraint
+
Collision retry
```

is still important.

---

# 21. Random vs Sequential IDs

Let's compare the common approaches.

| Approach                | Short |                     Unique | Predictable | Distributed Friendly |
| ----------------------- | ----: | -------------------------: | ----------: | -------------------: |
| Random string           |   Yes |              Probabilistic |         Low |                  Yes |
| UUID                    |    No | Extremely high probability |         Low |                  Yes |
| DB auto-increment       |   Yes |                        Yes |        High |              Depends |
| DB ID + Base62          |   Yes |                        Yes |        High |              Depends |
| Random integer + Base62 |   Yes |              Probabilistic |         Low |                  Yes |

There is no universally perfect strategy.

The correct choice depends on the requirements.

---

# 22. What Should We Choose for This Educational Project?

For this repository, we should deliberately implement multiple approaches.

This is an educational system-design project, so simply choosing one algorithm hides important engineering trade-offs.

We can demonstrate:

```text
Version 1
Random Short Code
```

Then:

```text
Version 2
Database ID + Base62
```

Then discuss:

```text
Version 3
Distributed ID Generation
```

This lets us understand why the architecture evolves.

---

# 23. Recommended Initial Implementation

For the first production-oriented implementation, we can use:

```text
Database-generated numeric ID
             +
         Base62
```

Architecture:

```text
Long URL
    |
    v
Create Database Record
    |
    v
Database ID
    |
    v
Base62 Encode
    |
    v
Short Code
    |
    v
Return Short URL
```

Example:

```text
Database ID = 12345

12345
  ↓
Base62
  ↓
dnh

https://short.ly/dnh
```

The reason for choosing this approach initially is that it gives us:

* Deterministic generation
* No random collision during ID creation
* Very fast encoding
* Compact codes
* Simple implementation
* Easy database integration

But we should also understand its limitations.

---

# 24. Important Problem With Database ID Generation

The Base62 approach depends on obtaining a unique database ID.

If we have:

```text
Application 1
Application 2
Application 3
```

all creating URLs, the database must safely generate unique IDs.

A relational database can handle this using:

```text
AUTO_INCREMENT
```

or an equivalent sequence mechanism.

However, this introduces a dependency:

```text
Create URL
    |
    v
Database
    |
    v
Generate ID
```

The database becomes part of the ID-generation path.

At high scale, this may eventually become a bottleneck or a distributed-systems concern.

---

# 25. Distributed ID Generation

At larger scale, we may want:

```text
Application 1 ──┐
Application 2 ──┤
Application 3 ──┼──> ID Generator
Application 4 ──┤
Application N ──┘
```

The ID generator needs to produce unique IDs without requiring every request to synchronously coordinate through the same database row.

Possible strategies include:

* Snowflake-style IDs
* Database sequences
* Dedicated ID-generation service
* UUIDs
* Random identifiers

This is an advanced topic and does not need to be implemented in the first version.

---

# 26. Collision Handling With Database Constraints

Regardless of the strategy, the database should protect the uniqueness invariant.

Conceptually:

```sql
CREATE TABLE url_mapping (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code VARCHAR(20) NOT NULL UNIQUE,
    original_url TEXT NOT NULL
);
```

The important part is:

```sql
UNIQUE(short_code)
```

Now even if two concurrent requests attempt:

```text
aB72x
```

the database cannot accept duplicate values.

The application can detect the failure and retry when using a probabilistic generation strategy.

---

# 27. Concurrency Example

Consider:

```text
Request A
Request B
```

Both arrive at the same time.

Random generation produces:

```text
Request A → xY92p
Request B → xY92p
```

Both attempt:

```text
INSERT xY92p
```

The database guarantees:

```text
First insert  → SUCCESS
Second insert → UNIQUE constraint violation
```

The application then:

```text
Generate another code
        |
        v
Retry
```

This is an important system-design pattern:

> **Use application logic for normal operation and database constraints as the final correctness boundary.**

---

# 28. Is Collision Actually a Big Problem?

It depends on the identifier space and the number of generated URLs.

Suppose we use:

```text
6 Base62 characters
```

The total space is:

```text
62^6
=
56,800,235,584
```

That is approximately:

```text
56.8 billion
```

possible combinations.

However, due to the **birthday paradox**, collision probability becomes meaningful much earlier than the total space would suggest.

For a random identifier system, the approximate probability of at least one collision is:

```text
P ≈ 1 - e^(-k² / (2N))
```

where:

```text
k = number of generated IDs
N = size of identifier space
```

This is why we should not say:

> "There are 56 billion combinations, so collisions cannot happen."

They can.

The correct approach is:

```text
Large identifier space
+
Uniqueness constraint
+
Retry
```

---

# 29. Why We Care About Short Length

Suppose we use Base62.

Possible code spaces:

```text
Length 1 → 62
Length 2 → 3,844
Length 3 → 238,328
Length 4 → 14,776,336
Length 5 → 916,132,832
Length 6 → 56,800,235,584
Length 7 → 3,521,614,606,208
Length 8 → 218,340,105,584,896
```

Increasing the code length dramatically increases the available identifier space.

Therefore, we can choose an appropriate length based on expected scale.

---

# 30. Fixed Length vs Variable Length

We do not necessarily need every code to have exactly the same length.

With Base62 encoding:

```text
1
```

might become:

```text
1
```

while a larger ID could become:

```text
aB72x
```

This produces variable-length codes.

That is completely valid.

Alternatively, we can pad values to a fixed length.

For example:

```text
000001
00000A
00ab72
```

Fixed-length identifiers can simplify some designs but are not required.

For our educational implementation, variable-length Base62 is simpler.

---

# 31. Security Consideration

Short-code generation and authorization are separate concerns.

Even if we use unpredictable codes, we should not rely on obscurity for authorization.

For example:

```text
https://short.ly/aB72x
```

being difficult to guess does not mean the underlying resource is securely protected.

If a short URL should only be accessible by an authorized user, the application needs proper authorization.

Therefore:

```text
Random Code
    ≠
Authentication
    ≠
Authorization
```

This distinction is important in production systems.

---

# 32. Enumeration Protection

If the system uses sequential IDs:

```text
1
2
3
4
5
```

or Base62-encoded sequential IDs:

```text
1
2
3
4
5
```

users may be able to enumerate URLs.

If enumeration is a concern, we can use:

```text
Secure Random Identifier
```

or another opaque-ID strategy.

For example:

```text
aB72x
P9kLm
x7Qa2
```

are harder to enumerate.

But again:

> Unpredictability reduces enumeration risk; it does not replace access control.

---

# 33. Recommended Architecture for the Repository

We will use the following progression.

## Stage 1 — Simple Random Code

```text
Long URL
   |
   v
Random Code
   |
   v
Storage
```

Purpose:

```text
Understand the basic concept.
```

---

## Stage 2 — Database ID + Base62

```text
Long URL
   |
   v
Database
   |
   v
Numeric ID
   |
   v
Base62
   |
   v
Short Code
```

Purpose:

```text
Understand deterministic ID generation.
```

---

## Stage 3 — Analyze Scale

We will ask:

```text
What if we generate millions of URLs?
```

Then:

```text
What if we have multiple application servers?
```

Then:

```text
What if database-generated IDs become a bottleneck?
```

---

## Stage 4 — Distributed ID Generation

We can then study:

```text
Snowflake-style IDs
```

and other distributed ID strategies.

This is where the URL Shortener becomes a useful system-design case study rather than just a CRUD application.

---

# 34. Final Recommended Flow

For the initial database-backed implementation:

```text
                       CREATE

                    Long URL
                       |
                       v
                Validate URL
                       |
                       v
               Create DB Record
                       |
                       v
                Numeric ID
                       |
                       v
                Base62 Encode
                       |
                       v
                  Short Code
                       |
                       v
              Return Short URL
```

For redirects:

```text
                       REDIRECT

                    Short URL
                       |
                       v
                 Extract Code
                       |
                       v
                 Base62 Decode
                       |
                       v
                  Numeric ID
                       |
                       v
                Database Lookup
                       |
                       v
                 Original URL
                       |
                       v
                    Redirect
```

---

# 35. Key Takeaways

A short code is simply an identifier:

```text
Short Code → Original URL
```

There are multiple ways to generate it:

```text
Random String
UUID
Database ID
Database ID + Base62
Random Integer + Base62
Distributed ID
```

For our educational project, the most useful progression is:

```text
Random Code
     ↓
Understand Collisions
     ↓
Database ID
     ↓
Base62
     ↓
Database-backed Shortener
     ↓
Distributed ID Generation
```

The most important engineering lessons are:

1. Short codes must be unique.
2. Random identifiers can collide.
3. Database uniqueness constraints provide a correctness boundary.
4. Base62 provides compact encoding.
5. Base62 is encoding, not encryption.
6. Sequential IDs are predictable.
7. Unpredictable identifiers can reduce enumeration risk.
8. Identifier generation becomes more interesting as the system scales.
9. A database can safely generate IDs for an initial implementation.
10. Very large distributed systems may require a distributed ID-generation strategy.

The key principle is:

> **Choose the simplest identifier-generation strategy that satisfies the current requirements, then evolve it when scale or security requirements justify the additional complexity.**

The next chapter will turn this design into a real **database schema and persistence layer**, where we will define exactly what information the URL Shortener needs to store and how we enforce uniqueness and efficient lookups.

```
```
