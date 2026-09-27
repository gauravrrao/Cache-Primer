# Caching: Penetration, Avalanche & Breakdown

Caching is one of the most common techniques used to improve application performance and reduce database load.

However, introducing a cache also creates a few important failure scenarios. Three of the most frequently discussed problems in backend/system-design interviews are:

1. **Cache Penetration**
2. **Cache Avalanche**
3. **Cache Breakdown (Hot Key Problem)**

This README explains these three problems in detail, including how they happen, why they are dangerous, and the common techniques used to prevent them.

---

# 1. Cache Penetration

## What is Cache Penetration?

**Cache penetration occurs when a request repeatedly asks for data that does not exist in the database, causing both the cache and database to be queried repeatedly.**

The important point is:

> The requested data does not exist in either the cache or the database.

Normally, a cache works like this:

```text
Client
   |
   v
Check Cache
   |
   |-- HIT --> Return Data
   |
   |-- MISS
        |
        v
     Database
        |
        v
     Store in Cache
```

But with cache penetration:

```text
Client
   |
   v
Check Cache
   |
   |-- MISS
        |
        v
     Database
        |
        |-- Data does not exist
        |
        v
     Return "Not Found"
```

If the same invalid request keeps coming:

```text
Request A --> Cache MISS --> DB --> Not Found
Request A --> Cache MISS --> DB --> Not Found
Request A --> Cache MISS --> DB --> Not Found
Request A --> Cache MISS --> DB --> Not Found
Request A --> Cache MISS --> DB --> Not Found
```

The database receives unnecessary traffic.

---

## Example

Imagine an API:

```http
GET /users/123
```

Suppose user `123` does not exist.

The request flow is:

```text
GET /users/123
       |
       v
Redis
       |
    MISS
       |
       v
PostgreSQL
       |
       v
User doesn't exist
       |
       v
404 Not Found
```

Now imagine an attacker sends:

```text
GET /users/999999999
GET /users/999999998
GET /users/999999997
GET /users/999999996
...
```

If none of these users exist, every request can reach the database.

With enough requests:

```text
Invalid Requests
       |
       v
   Cache MISS
       |
       v
   Database
       |
       v
Database Load ↑
       |
       v
Database Latency ↑
       |
       v
Possible DB Overload
```

This is **cache penetration**.

---

# Why Does Cache Penetration Happen?

The most common reason is that applications usually cache only successful database results.

For example:

```javascript
const cachedUser = await redis.get(`user:${userId}`);

if (cachedUser) {
    return JSON.parse(cachedUser);
}

const user = await db.users.findById(userId);

if (!user) {
    return null;
}

await redis.set(
    `user:${userId}`,
    JSON.stringify(user),
    "EX",
    300
);

return user;
```

If the user doesn't exist:

```text
Redis -> MISS
Database -> Not Found
Redis -> Nothing stored
```

Therefore, the next request repeats the entire process.

---

# How to Prevent Cache Penetration

There are several common solutions.

## Solution 1: Cache Null / Negative Results

Instead of caching only existing users, cache the fact that the user does not exist.

```text
user:999999
       |
       v
Redis
       |
       v
"NULL"
```

For example:

```javascript
const cachedUser = await redis.get(`user:${userId}`);

if (cachedUser === "NULL") {
    return null;
}

if (cachedUser) {
    return JSON.parse(cachedUser);
}

const user = await db.users.findById(userId);

if (!user) {
    await redis.set(`user:${userId}`, "NULL", "EX", 60);
    return null;
}

await redis.set(
    `user:${userId}`,
    JSON.stringify(user),
    "EX",
    300
);

return user;
```

The negative result should generally have a **short TTL**.

For example:

```text
Existing user:
TTL = 5 minutes

Non-existing user:
TTL = 30-60 seconds
```

Why?

Because the resource might be created later.

---

## Solution 2: Bloom Filter

A **Bloom filter** is a probabilistic data structure that can efficiently answer:

> "Could this key exist?"

It is useful when the system contains a very large number of valid keys.

Example:

```text
                 +----------------+
Request -------->| Bloom Filter   |
                 +----------------+
                    |
          +---------+---------+
          |                   |
       "NO"                 "MAYBE"
          |                   |
          v                   v
       Reject              Redis
                              |
                              v
                           Database
```

If the Bloom filter says:

```text
"Definitely does not exist"
```

we can reject the request without querying Redis or the database.

If it says:

```text
"Maybe exists"
```

we continue normally.

### Important Property

Bloom filters can produce **false positives**, but they do not normally produce false negatives.

That means:

```text
Bloom Filter says NO
=> Key definitely does not exist

Bloom Filter says YES
=> Key may exist
```

Therefore, the database is still the final authority.

---

## Solution 3: Input Validation

Some invalid requests can be rejected before reaching the cache.

For example:

```http
GET /users/abc
```

If `userId` must be an integer, the application can reject it immediately.

```text
Request
   |
   v
Validation
   |
   |-- Invalid --> 400
   |
   |-- Valid
        |
        v
      Cache
```

This prevents obviously invalid requests from consuming backend resources.

---

## Solution 4: Rate Limiting

Rate limiting can prevent clients from generating huge numbers of invalid requests.

Example:

```text
Client
   |
   v
Rate Limiter
   |
   |-- Limit exceeded --> 429
   |
   |-- Allowed
        |
        v
       Cache
```

This is especially useful when penetration is caused by malicious or abusive traffic.

---

# Cache Penetration Summary

```text
Problem:

Requested key doesn't exist
        |
        v
Cache MISS
        |
        v
Database MISS
        |
        v
Nothing cached
        |
        v
Same request repeats
        |
        v
Database load increases
```

### Main Solutions

```text
1. Cache negative results
2. Bloom filters
3. Input validation
4. Rate limiting
```

---

# 2. Cache Avalanche

## What is Cache Avalanche?

**Cache avalanche occurs when a large number of cache entries expire or become unavailable at approximately the same time, causing a sudden large number of requests to hit the database.**

The key difference from cache penetration is:

- **Penetration:** requested data doesn't exist.
- **Avalanche:** requested data exists, but many cached entries disappear simultaneously.

---

## Normal Cache Behaviour

Suppose we have:

```text
Product A -> Redis
Product B -> Redis
Product C -> Redis
Product D -> Redis
```

Normally:

```text
Client
  |
  v
Redis
  |
  |-- HIT
  |
  v
Return response
```

The database receives relatively few requests.

---

## Avalanche Scenario

Suppose millions of cache entries were created with exactly the same TTL:

```text
Product A -> TTL 3600
Product B -> TTL 3600
Product C -> TTL 3600
Product D -> TTL 3600
Product E -> TTL 3600
...
```

One hour later:

```text
Product A -> EXPIRED
Product B -> EXPIRED
Product C -> EXPIRED
Product D -> EXPIRED
Product E -> EXPIRED
...
```

Now millions of requests encounter cache misses.

```text
Millions of Requests
        |
        v
      Redis
        |
        |-- MASSIVE MISS
        |
        v
    Database
        |
        v
Database overload
```

This is a **cache avalanche**.

---

# Example

Imagine an ecommerce application has:

```text
10 million products
```

and all product cache entries are configured with:

```text
TTL = 1 hour
```

If they were populated around the same time, many entries can expire around the same period.

Then:

```text
10:00 AM
Cache populated

11:00 AM
Many entries expire

11:00 AM
Large traffic arrives

Redis MISS
      |
      v
Database
      |
      v
Huge DB traffic
```

The database may suddenly become the bottleneck.

---

# Why is Cache Avalanche Dangerous?

A cache exists partly to protect the database.

Under normal circumstances:

```text
100,000 requests
       |
       v
Redis handles most
       |
       v
Only small percentage
       |
       v
Database
```

During an avalanche:

```text
100,000 requests
       |
       v
Redis MISS
       |
       v
100,000 requests
       |
       v
Database
```

The database suddenly receives traffic that the system was designed to absorb through the cache.

This can cause:

```text
DB CPU ↑
DB Connections ↑
DB Latency ↑
DB Queries ↑
Application Latency ↑
Timeouts ↑
Errors ↑
```

---

# How to Prevent Cache Avalanche

## Solution 1: Randomized TTL / TTL Jitter

Instead of giving every cache key exactly the same TTL:

```text
TTL = 3600 seconds
```

add some randomness.

For example:

```javascript
const baseTTL = 3600;
const jitter = Math.floor(Math.random() * 300);

const ttl = baseTTL + jitter;
```

Now:

```text
Product A -> 3678 sec
Product B -> 3812 sec
Product C -> 3604 sec
Product D -> 3741 sec
Product E -> 3890 sec
```

Entries expire at different times.

Instead of:

```text
             MASS EXPIRATION
                   |
                   v
              DB overload
```

we get:

```text
Expiration
 |   |      |     |   |       |
 v   v      v     v   v       v
---------------------------------> Time
```

This spreads the database load over time.

---

# Solution 2: Cache Warming

Important or frequently accessed data can be loaded into the cache before traffic reaches it.

For example:

```text
Before cache expires
        |
        v
Fetch popular products
        |
        v
Store in Redis
        |
        v
Traffic arrives
        |
        v
Cache HIT
```

This is called **cache warming**.

It is particularly useful for:

- Homepage data
- Popular products
- Trending content
- Configuration
- Frequently accessed metadata

---

# Solution 3: Persistent / Highly Available Cache

A cache infrastructure can be designed so that a single cache node failure doesn't remove the entire caching layer.

For example:

```text
Application
     |
     v
Redis Cluster
  /    |    \
Node1 Node2 Node3
```

Replication and clustering can reduce the impact of individual cache-node failures.

However, high availability does not eliminate every form of avalanche. If the underlying problem is mass expiration, TTL jitter and controlled refresh are still important.

---

# Solution 4: Graceful Degradation

When the cache is unavailable, the application should avoid allowing every request to aggressively hit the database.

Depending on the application, it may:

```text
Return stale data
Use fallback data
Serve a degraded response
Temporarily reject low-priority requests
```

For example:

```text
Redis unavailable
       |
       v
Can stale data be served?
       |
      YES
       |
       v
Return stale response
```

This prevents unnecessary pressure on the database.

---

# Solution 5: Request Rate Limiting

Rate limiting can reduce the number of requests reaching the backend during abnormal traffic.

```text
Users
  |
  v
Rate Limiter
  |
  v
Application
  |
  v
Cache
  |
  v
Database
```

It does not fix the underlying cache expiration problem, but it can reduce the impact.

---

# Cache Avalanche Summary

```text
Many cache entries
       |
       v
Expire simultaneously
       |
       v
Massive cache misses
       |
       v
Requests hit DB
       |
       v
DB overload
```

### Main Solutions

```text
1. Randomized TTL / TTL jitter
2. Cache warming
3. Highly available cache
4. Graceful degradation
5. Rate limiting
```

---

# 3. Cache Breakdown — Hot Key Problem

## What is Cache Breakdown?

**Cache breakdown occurs when a very popular cache key expires or becomes unavailable while many requests are trying to access that same key.**

This is also commonly called the **Hot Key Problem**.

The important characteristic is:

> One particular key receives extremely high traffic.

This is different from cache avalanche.

---

# Example

Imagine an ecommerce website has:

```text
product:iphone-17
```

This product is extremely popular.

Suppose:

```text
1 million users
```

are requesting the same product.

Normally:

```text
1,000,000 requests
        |
        v
      Redis
        |
        v
    Cache HIT
        |
        v
    Product data
```

The database might receive almost no traffic for this product.

---

# The Problem

Suppose the key expires:

```text
product:iphone-17
        |
        v
      EXPIRED
```

Now many requests arrive simultaneously:

```text
Request 1 ----\
Request 2 -----\
Request 3 ------\
Request 4 -------\
Request 5 --------> Redis MISS
Request 6 -------/
Request 7 ------/
Request 8 -----/
       |
       v
   Database
```

If every request independently queries the database:

```text
1,000,000 requests
        |
        v
Redis MISS
        |
        v
1,000,000 DB queries
```

The database can become overloaded.

This is the **hot-key cache breakdown problem**.

---

# Cache Breakdown vs Cache Avalanche

These two problems are easy to confuse.

## Cache Avalanche

Many different keys expire together:

```text
Key A -> expired
Key B -> expired
Key C -> expired
Key D -> expired
Key E -> expired
...
```

Result:

```text
Many keys
   |
   v
Many cache misses
   |
   v
Database overload
```

## Cache Breakdown

One extremely popular key expires:

```text
HOT KEY
   |
   v
Expired
   |
   v
Millions of requests
   |
   v
Same database query repeated
```

### Simple distinction

```text
Avalanche  = MANY keys expire
Breakdown  = ONE HOT key expires
```

---

# Why is It Called a Hot Key?

A **hot key** is a key that receives disproportionately high traffic compared with other keys.

For example:

```text
product:123      -> 50 requests/min
product:456      -> 70 requests/min
product:789      -> 40 requests/min

product:iphone   -> 500,000 requests/min
```

`product:iphone` is a hot key.

Other examples include:

```text
Celebrity profile
Trending video
Popular news article
Flash-sale product
Live match information
Popular configuration
```

---

# How to Prevent Cache Breakdown

## Solution 1: Mutex / Distributed Lock

One of the most common approaches is to allow only **one request** to rebuild the cache.

Suppose the cache expires:

```text
              Redis
                |
              MISS
                |
        +-------+-------+
        |               |
     Request A       Request B
        |
        v
    Acquire Lock
        |
        v
       YES
        |
        v
    Query DB
        |
        v
    Update Redis
        |
        v
     Release Lock
```

Other requests wait for the cache to be rebuilt.

After the first request populates Redis:

```text
Request B -> Redis HIT
Request C -> Redis HIT
Request D -> Redis HIT
Request E -> Redis HIT
```

Instead of:

```text
100,000 requests
      |
      v
100,000 DB queries
```

we get approximately:

```text
100,000 requests
      |
      v
Redis MISS
      |
      v
1 request rebuilds cache
      |
      v
Redis HIT
      |
      v
Remaining requests
```

---

# Example Using Redis Lock

A simplified approach can use:

```text
SET lock:product:123 1 NX EX 10
```

The important Redis options are:

```text
NX
```

means:

> Set the key only if it doesn't already exist.

And:

```text
EX 10
```

gives the lock a TTL.

Conceptually:

```javascript
const cached = await redis.get(key);

if (cached) {
    return JSON.parse(cached);
}

const lockAcquired = await redis.set(
    `lock:${key}`,
    "1",
    "NX",
    "EX",
    10
);

if (lockAcquired) {
    try {
        const data = await database.getProduct(productId);

        await redis.set(
            key,
            JSON.stringify(data),
            "EX",
            300
        );

        return data;
    } finally {
        await redis.del(`lock:${key}`);
    }
}
```

In production, lock ownership and safe release should be handled carefully so one process does not accidentally delete another process's lock.

---

# Solution 2: Never Let the Hot Key Expire

For extremely important hot data, one strategy is to avoid normal expiration.

Instead of:

```text
Cache
  |
  v
TTL expires
  |
  v
MISS
```

the application can keep the key available and refresh it periodically.

```text
             Redis
               |
        Hot Key remains
               |
               v
       Background refresh
               |
               v
          Database
```

This is useful when the data is:

- Extremely popular
- Relatively stable
- Expensive to rebuild

However, permanent caching requires an invalidation/update strategy.

---

# Solution 3: Logical Expiration

Instead of letting Redis physically delete the data when its TTL expires, the application can store an expiration timestamp inside the cached value.

Example:

```json
{
  "data": {
    "id": 123,
    "name": "iPhone"
  },
  "expireAt": 1790000000
}
```

When the application reads the cache:

```text
Cache HIT
   |
   v
Is data logically expired?
   |
   +---- NO ----> Return data
   |
   +---- YES
           |
           v
     Refresh in background
           |
           v
     Return old/stale data
```

This prevents users from waiting for the database every time a popular key becomes stale.

---

# Solution 4: Stale-While-Revalidate

Another useful pattern is:

> Return slightly stale data while refreshing the cache in the background.

Example:

```text
User Request
     |
     v
Redis
     |
     v
Stale data?
     |
     +---- YES
     |      |
     |      +----> Return stale data immediately
     |                     |
     |                     v
     |              Background refresh
     |
     +---- NO
            |
            v
       Return data
```

This is particularly useful when:

```text
Availability > absolute freshness
```

For example, a product description can often tolerate a few seconds of staleness, while a bank balance generally cannot.

---

# Solution 5: Hot-Key Replication

For extremely high traffic, the same logical data can be replicated across multiple cache keys or cache nodes.

For example:

```text
product:123:1
product:123:2
product:123:3
product:123:4
```

Requests can be distributed:

```text
Requests
   |
   +----> product:123:1
   |
   +----> product:123:2
   |
   +----> product:123:3
   |
   +----> product:123:4
```

This can distribute read traffic instead of forcing every request toward one hot cache entry.

This technique is generally more useful at very large scale and adds complexity, especially around keeping replicas consistent.

---

# Cache Breakdown Flow

Without protection:

```text
                 HOT KEY
                    |
                    v
                 EXPIRED
                    |
                    v
             Redis MISS
                    |
       +------------+------------+
       |            |            |
       v            v            v
      DB           DB           DB
       |            |            |
       +------------+------------+
                    |
                    v
              DB OVERLOAD
```

With a lock:

```text
                 HOT KEY
                    |
                    v
                 EXPIRED
                    |
                    v
                Redis MISS
                    |
                    v
              Acquire Lock
                    |
                    v
               One Request
                    |
                    v
                Database
                    |
                    v
              Update Redis
                    |
                    v
                Cache HIT
              /    |    \
             /     |     \
           Req    Req    Req
```

---

# Interview Comparison

| Problem | What happens? | Typical Cause | Main Protection |
|---|---|---|---|
| **Cache Penetration** | Request asks for data that doesn't exist | Invalid/non-existent keys | Negative caching, Bloom filter |
| **Cache Avalanche** | Many cache keys become unavailable together | Same/similar expiration or cache failure | TTL jitter, cache warming, HA |
| **Cache Breakdown** | One extremely popular key becomes unavailable | Hot key expiration | Mutex/lock, logical expiration, stale-while-revalidate |

---

# Easy Way to Remember

Think about the **number of keys involved**.

```text
CACHE PENETRATION
       |
       v
Key doesn't exist
       |
       v
Cache + DB MISS


CACHE AVALANCHE
       |
       v
MANY keys expire
       |
       v
Many DB requests


CACHE BREAKDOWN
       |
       v
ONE HOT key expires
       |
       v
Huge traffic for same key
```

### One-Line Interview Answers

**Cache Penetration:**

> Cache penetration happens when requests repeatedly query data that doesn't exist in the cache or database, causing unnecessary database traffic. Negative caching and Bloom filters are common solutions.

**Cache Avalanche:**

> Cache avalanche happens when many cache entries expire or become unavailable at the same time, causing a large number of cache misses and a sudden spike in database traffic. TTL jitter and cache warming are common solutions.

**Cache Breakdown / Hot Key:**

> Cache breakdown happens when a highly popular cache key expires or becomes unavailable, causing many concurrent requests for that same key to hit the database. Distributed locking, logical expiration, and stale-while-revalidate can prevent this.