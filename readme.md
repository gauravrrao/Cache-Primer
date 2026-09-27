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

# Caching: Eviction Policies & Cache Invalidation

Two important concepts in production caching systems are:

1. **Cache Eviction Policies** — What should happen when the cache is full?
2. **Cache Invalidation** — When and how should cached data be removed or updated because the underlying data has changed?

These concepts are related, but they solve **different problems**.

---

# 1. Cache Eviction Policies

## What is Cache Eviction?

A cache has limited memory.

For example:

```text
Redis
Maximum Memory = 1 GB
```

Eventually, the cache may reach its memory limit:

```text
+-------------------------------+
|          Redis Cache          |
|                               |
| Key A                         |
| Key B                         |
| Key C                         |
| Key D                         |
| ...                           |
|                               |
| Memory: 1 GB / 1 GB           |
+-------------------------------+
```

Now a new key needs to be inserted:

```text
SET product:999 ...
```

But there is no available memory.

The cache needs to decide:

> **Which existing data should be removed to make space for the new data?**

This decision is made using a **cache eviction policy**.

---

# Eviction vs Expiration

These terms are often confused.

## Expiration

Expiration means:

> A key has reached its configured TTL and is no longer considered valid.

Example:

```text
product:123
TTL = 300 seconds
```

After 300 seconds:

```text
product:123
      |
      v
   EXPIRED
```

---

## Eviction

Eviction means:

> The cache removes data to free memory according to an eviction policy.

For example:

```text
Redis is full
     |
     v
Need 100 MB
     |
     v
Eviction policy
     |
     v
Choose keys to remove
```

### Simple difference

```text
Expiration = "The data's lifetime ended."

Eviction = "The cache needs space."
```

A key can also have a TTL and be evicted **before** that TTL is reached.

---

# Why Do We Need Eviction Policies?

Imagine a cache with:

```text
Maximum capacity = 1 million keys
```

Applications continue adding:

```text
Request 1 -> cache key
Request 2 -> cache key
Request 3 -> cache key
...
Request 1,000,001 -> cache key
```

Without an eviction mechanism, memory usage could continue increasing until the process crashes or the system refuses new writes.

Therefore:

```text
Cache Full
    |
    v
Eviction Policy
    |
    v
Remove some keys
    |
    v
Free memory
    |
    v
Store new key
```

---

# Common Cache Eviction Policies

The most important policies to understand are:

1. **LRU — Least Recently Used**
2. **LFU — Least Frequently Used**
3. **FIFO — First In, First Out**
4. **Random Eviction**
5. **TTL-based / volatile eviction**
6. **No Eviction**

Redis supports several variants of these strategies.

---

# 1. LRU — Least Recently Used

## What is LRU?

LRU stands for:

> **Least Recently Used**

The cache removes the item that has not been accessed for the longest time.

Suppose we have:

```text
Cache capacity = 3
```

Current cache:

```text
A
B
C
```

Access sequence:

```text
A
B
C
A
```

The recent usage becomes:

```text
B -> older
C -> newer
A -> most recently used
```

Now we insert:

```text
D
```

The least recently used item is:

```text
B
```

Therefore:

```text
Before:

B   C   A

Insert D

After:

C   A   D
```

`B` is evicted.

---

# LRU Example

Imagine an application caching user profiles:

```text
User 1 -> accessed 10 times
User 2 -> accessed 2 times
User 3 -> accessed 20 times
```

But the important question for LRU is **when they were last accessed**, not how many times they were accessed.

For example:

```text
User 1 -> accessed 2 minutes ago
User 2 -> accessed 1 hour ago
User 3 -> accessed 5 minutes ago
```

If space is needed:

```text
User 2
```

is the LRU candidate.

---

# LRU Advantages

LRU works well when:

> Recently accessed data is likely to be accessed again.

This pattern is called **temporal locality**.

Examples:

```text
Recently viewed products
Recently accessed user profiles
Frequently visited pages
Recent API responses
```

---

# LRU Disadvantage

LRU only considers **recency**.

Suppose:

```text
Key A -> accessed 1,000 times yesterday
Key B -> accessed once 1 minute ago
```

LRU may prefer keeping `B` because it was accessed more recently.

It doesn't directly care that `A` was accessed 1,000 times.

This is where LFU can be useful.

---

# 2. LFU — Least Frequently Used

LFU stands for:

> **Least Frequently Used**

Instead of asking:

> "Which key was accessed least recently?"

LFU asks:

> "Which key has been accessed the fewest times?"

Example:

```text
Cache:

A -> 100 accesses
B -> 5 accesses
C -> 50 accesses
```

If the cache is full and we need to remove something:

```text
B
```

is the LFU candidate.

Because:

```text
B = 5 accesses
```

which is the lowest frequency.

---

# LRU vs LFU

Consider:

```text
A -> accessed 1,000 times yesterday
B -> accessed 10 times recently
```

LRU may evict:

```text
A
```

because A hasn't been accessed recently.

LFU may keep:

```text
A
```

because A has a much higher access frequency.

Therefore:

```text
LRU -> cares about RECENCY

LFU -> cares about FREQUENCY
```

---

# When LFU is Useful

LFU can be useful when certain objects consistently receive much more traffic than others.

For example:

```text
Homepage
Trending videos
Popular products
Popular posts
Frequently accessed configuration
```

If:

```text
Product A -> 1,000,000 requests
Product B -> 10 requests
```

LFU recognizes that Product A is significantly more frequently used.

---

# Problem With LFU

A simple LFU implementation can have a problem called **historical bias**.

Suppose:

```text
Product A
Used 1 million times last month
```

and today:

```text
Product A
No longer popular
```

Its historical frequency may still be very high.

Meanwhile:

```text
Product B
Recently became extremely popular
```

but has a lower total frequency.

A pure LFU algorithm may take time to adapt.

Modern implementations can use frequency decay or more sophisticated approximations.

---

# 3. FIFO — First In, First Out

FIFO means:

> The item that entered the cache first is removed first.

Example:

```text
Cache capacity = 3

A -> inserted first
B -> inserted second
C -> inserted third
```

Now:

```text
Insert D
```

The cache removes:

```text
A
```

because it arrived first.

```text
Before:

A B C

After inserting D:

B C D
```

---

# FIFO vs LRU

Suppose:

```text
A -> inserted first
B -> inserted second
C -> inserted third
```

Then we access:

```text
A
```

With FIFO:

```text
A is still the oldest
```

So A can be evicted.

With LRU:

```text
A was just accessed
```

so A is likely to remain.

Therefore:

```text
FIFO -> insertion order

LRU -> access order
```

---

# 4. Random Eviction

Random eviction simply chooses a key randomly.

Example:

```text
A
B
C
D
E
```

Cache is full.

A new key arrives:

```text
F
```

The system randomly chooses:

```text
C
```

and removes it.

```text
A B [C] D E

        ↓

A B D E F
```

This is simple and has low overhead, but it doesn't try to preserve the most useful data.

---

# 5. TTL-Based Eviction

TTL means:

> **Time To Live**

A key can be stored with an expiration time.

Example:

```text
SET user:123 {...} EX 300
```

Meaning:

```text
user:123
     |
     v
TTL = 300 seconds
     |
     v
Expiration
```

After the TTL expires, the key becomes eligible for removal.

---

# TTL Is Not Exactly an Eviction Policy

This distinction is important in interviews.

TTL primarily determines:

> **When a key becomes expired.**

An eviction policy determines:

> **Which key should be removed when memory pressure occurs.**

For example:

```text
Key A -> TTL 10 minutes
Key B -> TTL 1 hour
Key C -> TTL 1 day
```

If Redis runs out of memory, the configured eviction policy determines which keys are eligible for eviction.

---

# 6. No Eviction

A cache can also be configured not to evict keys.

When memory is full:

```text
Cache full
   |
   v
New write
   |
   v
Rejected
```

This can be useful when evicting cached data is unacceptable and the application wants to explicitly handle memory pressure.

However, the application then needs another strategy for managing capacity.

---

# Redis Eviction Policies

Redis provides several policies. The commonly discussed ones include:

```text
noeviction
allkeys-lru
volatile-lru

allkeys-lfu
volatile-lfu

allkeys-random
volatile-random

volatile-ttl
```

The names are easier to understand by splitting them.

### `allkeys`

Means:

> Consider all keys for eviction.

### `volatile`

Means:

> Consider only keys that have an expiration/TTL.

Therefore:

```text
allkeys-lru
```

means:

> Evict the least recently used key among all keys.

And:

```text
volatile-lru
```

means:

> Evict the least recently used key among keys that have TTLs.

---

# `allkeys-lru` vs `volatile-lru`

This is a common interview question.

Suppose Redis contains:

```text
A -> no TTL
B -> TTL
C -> TTL
D -> no TTL
```

With:

```text
allkeys-lru
```

all four keys can potentially be considered.

With:

```text
volatile-lru
```

only:

```text
B
C
```

are candidates.

---

# `allkeys-lfu`

This uses LFU across all keys.

```text
All Keys
   |
   v
Frequency analysis
   |
   v
Least frequently used key
   |
   v
Evict
```

---

# `volatile-lfu`

Only keys having an expiration time are considered.

```text
Keys with TTL
      |
      v
Frequency analysis
      |
      v
Least frequently used
      |
      v
Evict
```

---

# Choosing an Eviction Policy

A simple interview framework is:

```text
Does recent access matter?
        |
       YES
        |
       LRU
```

If long-term popularity matters:

```text
Frequency matters
        |
       YES
        |
       LFU
```

If insertion order is sufficient:

```text
FIFO
```

If simplicity is the priority:

```text
Random
```

For Redis specifically, the choice also depends on whether **all keys** or only **TTL-bearing keys** should be eligible.

There is no universally correct eviction policy.

---

# 2. Cache Invalidation

## What is Cache Invalidation?

Cache invalidation means:

> **Removing or updating cached data when the underlying source of truth changes.**

This is one of the most difficult problems in caching.

The reason is simple:

```text
Database
   |
   | Data changes
   v
Database = NEW DATA

Cache
   |
   v
Cache = OLD DATA
```

Now the application can return stale information.

---

# Example

Suppose the database contains:

```json
{
  "id": 123,
  "name": "Gaurav",
  "age": 27
}
```

The application caches it:

```text
Redis:

user:123
{
  "id": 123,
  "name": "Gaurav",
  "age": 27
}
```

Now the user updates their name:

```text
Database:

name = "Rahul"
```

But Redis still contains:

```text
name = "Gaurav"
```

If the next request reads Redis:

```text
Client
   |
   v
Redis
   |
   v
"Gaurav"
```

The application returns stale data.

This is a **cache consistency problem**.

---

# Why Cache Invalidation Is Difficult

Because there are now two copies of the data:

```text
              +----------+
              | Database |
              +----------+
                   |
                   |
              +----------+
              |  Redis   |
              +----------+
```

Whenever the database changes:

```text
Database changes
       |
       v
Cache must eventually reflect change
```

The challenge is ensuring that these two systems don't remain inconsistent longer than the application can tolerate.

---

# Common Cache Invalidation Strategies

The major patterns to understand are:

1. **Cache Aside / Lazy Invalidation**
2. **Write-Through Cache**
3. **Write-Behind / Write-Back Cache**
4. **Write-Around Cache**
5. **Explicit Cache Deletion**
6. **TTL-Based Invalidation**
7. **Event-Driven Invalidation**

---

# 1. Cache-Aside Pattern

Cache-aside is one of the most commonly used caching strategies.

The application controls both the cache and database.

## Read

```text
Client
  |
  v
Application
  |
  v
Cache
  |
  +---- HIT ----> Return data
  |
  +---- MISS
          |
          v
       Database
          |
          v
       Cache
          |
          v
       Return
```

---

## Write

Suppose we update a user:

```text
Client
   |
   v
Application
   |
   v
Database UPDATE
   |
   v
Delete Cache
```

For example:

```javascript
await db.users.update(
    { id: userId },
    { name: newName }
);

await redis.del(`user:${userId}`);
```

The next read causes a cache miss:

```text
Next request
    |
    v
Redis MISS
    |
    v
Database
    |
    v
New data
    |
    v
Redis
```

This is called:

> **Cache Aside with cache invalidation on write.**

---

# Why Delete Instead of Update?

You could update Redis directly:

```text
Database -> UPDATE
Redis    -> UPDATE
```

But this creates a dual-write problem.

What if:

```text
Database UPDATE succeeds
Redis UPDATE fails
```

Now:

```text
Database = new
Redis = old
```

Or:

```text
Redis UPDATE succeeds
Database UPDATE fails
```

Now the cache contains data that isn't actually committed to the database.

Deleting the cache after a successful database update is often simpler:

```text
Database
   |
   v
UPDATE successful
   |
   v
DELETE cache
```

The next read reconstructs the cache from the database.

---

# Cache-Aside Example

```javascript
async function getUser(userId) {
    const key = `user:${userId}`;

    const cached = await redis.get(key);

    if (cached) {
        return JSON.parse(cached);
    }

    const user = await db.users.findById(userId);

    if (!user) {
        return null;
    }

    await redis.set(
        key,
        JSON.stringify(user),
        "EX",
        300
    );

    return user;
}
```

Update:

```javascript
async function updateUser(userId, data) {
    const user = await db.users.update(userId, data);

    await redis.del(`user:${userId}`);

    return user;
}
```

This is a very common interview implementation.

---

# 2. Write-Through Cache

In write-through caching:

> The application writes to the cache, and the cache synchronously writes the data to the database.

Conceptually:

```text
Application
     |
     v
Cache
     |
     v
Database
```

When writing:

```text
Client
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

The cache and database are updated as part of the write path.

---

# Advantage

The cache generally contains fresh data immediately after a successful write.

```text
Write
 |
 +--> Cache updated
 |
 +--> DB updated
```

---

# Disadvantage

Every write has additional work.

```text
Application
     |
     v
Cache
     |
     v
Database
```

It can also introduce complexity around failures and transactional guarantees.

---

# 3. Write-Behind / Write-Back Cache

With write-behind:

> The application writes to the cache first, and the database is updated asynchronously later.

```text
Application
     |
     v
Cache
     |
     v
Queue
     |
     v
Database
```

Example:

```text
User updates profile
        |
        v
Cache updated immediately
        |
        v
Response returned
        |
        v
Background worker
        |
        v
Database updated
```

---

# Advantage

The write response can be very fast.

```text
Client
   |
   v
Cache
   |
   v
Response
```

Database persistence happens asynchronously.

---

# Risk

If the cache fails before the data is persisted:

```text
Cache updated
     |
     X
Cache failure
     |
     v
Database never receives update
```

Therefore, write-behind requires reliable queues, retry mechanisms, and failure handling.

---

# 4. Write-Around Cache

With write-around:

> Writes go directly to the database and do not immediately populate the cache.

```text
Write:

Application
    |
    v
Database
```

Later:

```text
Read
 |
 v
Cache MISS
 |
 v
Database
 |
 v
Cache
```

This is useful when data is written frequently but may not be read frequently.

It avoids filling the cache with data that might never be requested.

---

# 5. Explicit Cache Deletion

The simplest invalidation strategy is:

```text
UPDATE DB
   |
   v
DELETE CACHE
```

Example:

```javascript
await db.products.update(productId, {
    price: 999
});

await redis.del(`product:${productId}`);
```

Next read:

```text
Redis MISS
    |
    v
Database
    |
    v
New price
    |
    v
Redis
```

This is simple and widely used.

---

# 6. TTL-Based Invalidation

Another approach is to allow the cache to expire naturally.

Example:

```text
product:123
TTL = 300 seconds
```

After five minutes:

```text
product:123
      |
      v
Expired
```

Next request:

```text
Cache MISS
    |
    v
Database
    |
    v
Fresh data
```

---

# TTL Does NOT Guarantee Strong Consistency

Suppose:

```text
Cache TTL = 1 hour
```

Database changes at:

```text
10:05 AM
```

Cache might still contain old data until:

```text
11:00 AM
```

Therefore:

```text
TTL = maximum cache lifetime
```

but it does not necessarily mean:

```text
Data is immediately consistent
```

For data that must reflect changes quickly, explicit invalidation may be needed.

---

# 7. Event-Driven Cache Invalidation

In larger distributed systems, services can use events.

Example:

```text
User Service
     |
     v
Database UPDATE
     |
     v
Publish Event
     |
     v
Kafka / RabbitMQ
     |
     v
Cache Invalidation Consumer
     |
     v
Redis DEL
```

For example:

```text
UserUpdated
{
    "userId": 123
}
```

The cache service consumes the event:

```text
UserUpdated
    |
    v
redis.del("user:123")
```

This is useful when multiple services need to react to the same data change.

---

# Cache Invalidation Race Condition

This is an important interview topic.

Consider:

```text
Request A                 Request B

DB UPDATE
   |
   |
Redis DEL
```

At the same time:

```text
Request B
   |
Redis GET -> MISS
   |
DB GET
   |
old data
```

Depending on timing, Request B can potentially repopulate Redis with stale data.

For example:

```text
Time →

A: DB UPDATE
B: Redis MISS
B: DB READ old value
A: Redis DEL
B: Redis SET old value
```

Final state:

```text
Database = NEW
Redis    = OLD
```

This demonstrates why cache invalidation can become difficult under concurrency.

---

# Double Delete Pattern

One technique sometimes used to reduce certain cache/database race windows is **delayed double deletion**.

Conceptually:

```text
1. Delete cache
2. Update database
3. Wait briefly
4. Delete cache again
```

Example:

```javascript
await redis.del(key);

await db.update(data);

setTimeout(async () => {
    await redis.del(key);
}, 100);
```

The second deletion is intended to remove a stale value that may have been repopulated during the race.

However, the delay is not a universal guarantee of consistency. In distributed systems, stronger approaches may involve versioning, event ordering, synchronization, or carefully designed read/write protocols.

---

# Cache Invalidation With Versioning

Another approach is to associate cached data with a version.

Database:

```text
User
version = 10
```

Cache:

```text
user:123
version = 10
```

After an update:

```text
Database
version = 11
```

Now the application can detect:

```text
Cache version 10
Database version 11
```

and refresh the cache.

Versioning can be particularly useful when stale writes or ordering problems need to be detected.

---

# Cache Invalidation and Distributed Systems

In a single server:

```text
Application
    |
    +---- Database
    |
    +---- Redis
```

Invalidation is relatively straightforward.

In a distributed system:

```text
                +--> Service A
                |
Database -------+--> Service B
                |
                +--> Service C
                |
                +--> Redis
```

Now a database update may require multiple caches to be invalidated.

This is why event-driven approaches are often used:

```text
Database Change
       |
       v
   Event Bus
       |
   +---+---+---+
   |   |   |   |
   v   v   v   v
 S1  S2  S3 Redis
```

---

# Cache Invalidation Strategies Comparison

| Strategy | Where is the write performed? | Cache updated when? | Main Characteristic |
|---|---|---|---|
| **Cache Aside** | Database | On next read | Simple and common |
| **Write Through** | Cache → DB | Immediately | Cache stays fresh |
| **Write Behind** | Cache | DB updated asynchronously | Fast writes |
| **Write Around** | Database | On next cache miss | Avoids caching unused writes |
| **TTL** | Database | After expiration | Simple but can serve stale data |
| **Event Driven** | Database | After event | Useful for distributed systems |

---

# Eviction vs Invalidation

This distinction is extremely important.

## Eviction

Eviction is primarily about:

> **Memory management.**

Example:

```text
Cache full
   |
   v
Need space
   |
   v
Remove LRU key
```

The key may still represent perfectly valid data.

---

## Invalidation

Invalidation is primarily about:

> **Data correctness / freshness.**

Example:

```text
Database changed
      |
      v
Cache contains old value
      |
      v
Invalidate cache
```

The cache may have plenty of free memory.

---

# Example Showing Both

Suppose:

```text
Redis capacity = 1 GB
```

A product is cached:

```text
product:123
price = ₹500
```

Then the product price changes:

```text
Database
price = ₹600
```

We invalidate:

```text
redis.del("product:123")
```

This is **cache invalidation**.

Later Redis becomes full:

```text
Redis = 1 GB / 1 GB
```

Redis needs space and removes:

```text
product:456
```

because it is the least recently used key.

This is **cache eviction**.

So:

```text
Invalidation -> Data changed

Eviction -> Cache needs space
```

---

# Interview Cheat Sheet

## Cache Eviction

### Question:
**What happens when Redis reaches its maximum memory?**

Answer:

> Redis uses its configured eviction policy to determine which keys can be removed to free memory. Common strategies include LRU, LFU, random eviction, TTL-based policies, or no eviction.

---

### Question:
**LRU vs LFU?**

Answer:

> LRU removes the least recently accessed key, while LFU removes the least frequently accessed key. LRU is useful when recent access predicts future access, whereas LFU is useful when long-term popularity matters.

---

### Question:
**What is `allkeys-lru`?**

Answer:

> It means Redis can evict the least recently used key from the entire keyspace.

---

### Question:
**What is `volatile-lru`?**

Answer:

> It means Redis chooses the least recently used key only from keys that have an expiration time.

---

# Cache Invalidation

### Question:
**What is cache invalidation?**

Answer:

> Cache invalidation is the process of removing or updating cached data when the underlying source of truth changes, so that stale data is not served.

---

### Question:
**Why do we usually delete the cache after updating the database?**

Answer:

> Deleting the cache avoids maintaining two independent copies during the write operation. The next read gets the latest value from the database and repopulates the cache.

---

### Question:
**Does TTL solve cache invalidation?**

Answer:

> TTL provides eventual expiration, but it does not guarantee immediate freshness. If the database changes before the TTL expires, the cache can still contain stale data.

---

### Question:
**What is cache-aside?**

Answer:

> The application first checks the cache. On a miss, it reads from the database and populates the cache. On writes, the application typically updates the database and invalidates the corresponding cache key.

---

# Final Mental Model

Remember these two questions:

```text
                 CACHE
                   |
        +----------+----------+
        |                     |
        v                     v
  "Do I have space?"     "Is my data fresh?"
        |                     |
        v                     v
     EVICTION             INVALIDATION
        |                     |
        v                     v
Which key should go?    Which data should go/update?
```

### Eviction

```text
Cache is FULL
     |
     v
Choose what to REMOVE
     |
     v
LRU / LFU / FIFO / Random / etc.
```

### Invalidation

```text
Database CHANGED
     |
     v
Cache may be STALE
     |
     v
REMOVE or UPDATE cache
```

The simplest way to remember the distinction is:

> **Eviction is about cache capacity. Invalidation is about cache correctness.**