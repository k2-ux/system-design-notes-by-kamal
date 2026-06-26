# Database Indexing + Caching Strategies

> "There are only two hard things in Computer Science: cache invalidation and naming things."
> — Phil Karlton, and every engineer who has ever been paged at 3am

If you've ever wondered why one SQL query returns in 2ms and another brings an entire production database to its knees, you're in the right place. This guide covers the two most powerful levers you have for making data systems fast: **indexing** and **caching**. We'll learn from engineers who did it wrong first — so you don't have to.

---

# PART 1: Database Indexing

---

## Q1: What even IS a database index? Why do I need one?

**Short answer:** An index is a separate data structure that lets the database find rows *without reading the entire table.*

**Long answer with a book analogy:**

Imagine you have a 1000-page textbook and someone asks: "Find every mention of Dijkstra." Without an index, you'd flip through every single page (a **full table scan**). With an index at the back of the book, you go directly to the right pages.

Databases work exactly the same way. Without an index, finding `WHERE user_id = 42` means scanning every row in the table — even if there are 50 million of them.

The most common index type is a **B-Tree (Balanced Tree)**:

```
                     [50]
                    /    \
               [25]        [75]
              /    \      /    \
           [10]  [30] [60]    [90]
```

Each node stores values in sorted order. To find `user_id = 60`, the DB starts at root `[50]`, goes right to `[75]`, goes left to `[60]`. Found in 3 comparisons instead of scanning all rows. For a table with 1 million rows, a B-Tree index typically needs only ~20 comparisons. This is O(log n) vs O(n) — the difference between surviving and dying.

**Creating an index in SQL:**

```sql
-- Without index: full table scan
SELECT * FROM users WHERE email = 'alice@example.com';

-- Add an index
CREATE INDEX idx_users_email ON users(email);

-- Now the same query is blazing fast
SELECT * FROM users WHERE email = 'alice@example.com';
```

---

## Q2: I've heard "indexes speed up reads but slow down writes." How bad is it really?

**The tradeoff is real, and you need to understand it before over-indexing.**

When you insert, update, or delete a row, the database has to update *every index on that table* — not just the row itself. Each index is a separate B-Tree that needs to be rebalanced.

```
INSERT INTO orders (user_id, product_id, status, created_at) VALUES (...);

What actually happens:
1. Write row to main table (heap)         ← 1 write
2. Update index on user_id                ← 1 write
3. Update index on product_id             ← 1 write
4. Update index on status                 ← 1 write
5. Update index on created_at             ← 1 write

Total: 5 writes for 1 logical insert
```

**The GitHub 2012 MySQL outage is the canonical cautionary tale here — but from the opposite direction.**

GitHub had a massive `repositories` table with millions of rows. They were doing `WHERE owner_id = X` queries without an index on `owner_id`. Every query did a full table scan. As the table grew, these scans took longer and longer, acquired locks for longer, and eventually queries started piling up. Lock waits cascaded into a complete MySQL meltdown. The fix was adding a single index — but they had to do it carefully on a live table without causing further downtime.

**The lesson:** The pain of a missing index on a read-heavy column is usually far worse than the overhead of maintaining it. But also: don't add indexes "just in case." Each one costs write performance and disk space. Index what you actually query.

**Rule of thumb:**
- Read-heavy table (analytics, user profiles, product catalog): index aggressively
- Write-heavy table (event logs, audit trails, message queues): index conservatively

---

## Q3: What are composite indexes and why does column ORDER matter so much?

**A composite index covers multiple columns. The order of columns is not arbitrary — it is everything.**

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

This index can satisfy:
- `WHERE user_id = 5`                         -- YES, leftmost prefix
- `WHERE user_id = 5 AND status = 'shipped'`  -- YES, full index
- `WHERE status = 'shipped'`                  -- NO, can't skip the first column

This is called the **leftmost prefix rule**. Think of a phone book sorted by last name, then first name. You can look up "Smith" (last name only) or "Smith, John" (both). But you can't look up everyone named "John" efficiently — you'd have to scan the whole book.

```
Index: (user_id, status)

user_id=1, status='pending'   ← sorted first by user_id
user_id=1, status='shipped'
user_id=2, status='pending'
user_id=2, status='shipped'
user_id=3, status='cancelled'
...

Query: WHERE user_id = 1                    → seek to user_id=1, scan contiguous rows ✓
Query: WHERE user_id = 1 AND status='shipped' → seek to exact position ✓
Query: WHERE status = 'shipped'             → have to scan entire index ✗
```

**How to decide column order:**

1. Put the column used in equality filters (`=`) first
2. Then put columns used in range filters (`>`, `<`, `BETWEEN`) last
3. Put high-cardinality columns earlier (more unique values = better filtering)

```sql
-- Good: equality on user_id, range on created_at
CREATE INDEX idx_orders ON orders(user_id, created_at);

-- Query this enables:
SELECT * FROM orders
WHERE user_id = 42 AND created_at > '2024-01-01';
```

---

## Q4: What's a covering index and why did Instagram specifically need it?

**A covering index is an index that contains all the columns your query needs — so the DB never has to touch the main table at all.**

Normal index lookup:

```
Query: SELECT id, username, avatar_url FROM users WHERE email = 'alice@example.com'

Step 1: Look up email in index → find row pointer (e.g., page 4821, slot 3)
Step 2: Fetch that page from the main table (heap) → get id, username, avatar_url

This second step is called a "heap fetch" or "table lookup"
```

Covering index lookup:

```sql
-- Include all columns the query needs
CREATE INDEX idx_users_email_covering
ON users(email)
INCLUDE (id, username, avatar_url);   -- PostgreSQL syntax
-- In MySQL: CREATE INDEX ... ON users(email, id, username, avatar_url)

Step 1: Look up email in index → row is RIGHT THERE in the index itself
Step 2: (nothing — we already have everything we need)
```

**The Instagram story:**

As Instagram scaled to hundreds of millions of users, their home feed query was something like:

```sql
SELECT post_id, image_url, like_count, created_at
FROM posts
WHERE user_id IN (... list of followed users ...)
ORDER BY created_at DESC
LIMIT 20;
```

This was hitting a table with billions of rows. Even with an index on `user_id`, every match required a heap fetch to get `image_url`, `like_count`, and `created_at`. At Instagram's scale, those random I/O heap fetches were killing query times.

Adding a covering index that included all four columns eliminated the heap fetches entirely. The index itself contained everything. Feed load times dropped significantly — and this was one of the optimizations that let Instagram keep their backend team famously small.

**When to use covering indexes:** When you have a query that runs millions of times per day and the columns you SELECT are predictable. The cost is more disk space for the index.

---

## Q5: When should I NOT add an index? Aren't more indexes always better?

**Absolutely not. Indexes on the wrong columns are worse than useless.**

**Trap 1: Low-cardinality columns**

Cardinality = number of distinct values. An index on a column with 2 possible values (like `is_active = true/false` or `gender`) is nearly useless.

```
Table: 10 million users
Column: is_active (values: true or false)
Distribution: 9.5 million active, 500k inactive

Query: WHERE is_active = true

Without index: full scan, finds 9.5M rows
With index: uses index, finds 9.5M pointers, then does 9.5M heap fetches
            → SLOWER than just scanning the table!
```

The DB query planner is smart enough to usually skip the index in this case, but you've still wasted the disk space and write overhead. Rule: if a column has fewer than ~100 distinct values relative to table size, think hard before indexing it.

**Trap 2: Tiny tables**

If a table has 500 rows, a full scan takes microseconds. An index adds overhead and complexity for zero benefit.

**Trap 3: Columns rarely used in WHERE/JOIN**

If you only filter by `middle_name` in one admin report that runs once a month, the constant write overhead for all your high-volume inserts is not worth it.

**Trap 4: Function calls on indexed columns**

```sql
-- This CANNOT use the index on created_at:
WHERE YEAR(created_at) = 2024

-- This CAN:
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
```

Wrapping a column in a function breaks index usage. The DB has to compute `YEAR(created_at)` for every row — right back to a full scan.

---

## Q6: How do I know if my query is actually using an index? Enter: EXPLAIN

**EXPLAIN (and EXPLAIN ANALYZE) is your X-ray machine for slow queries. Learn to use it.**

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
```

**PostgreSQL output (simplified):**

```
Index Scan using idx_orders_user_id on orders
  (cost=0.43..8.45 rows=3 width=152)
  Index Cond: (user_id = 42)
```

`Index Scan` — great, it's using the index.

Now without the index:

```
Seq Scan on orders
  (cost=0.00..45231.00 rows=1000000 width=152)
  Filter: (user_id = 42)
```

`Seq Scan` = Sequential Scan = full table scan = danger.

**EXPLAIN ANALYZE** actually *runs* the query and shows real timing:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42 AND status = 'shipped';
```

```
Index Scan using idx_orders_user_status on orders
  (cost=0.43..12.46 rows=3 width=152)
  (actual time=0.043..0.051 rows=2 loops=1)
  Index Cond: ((user_id = 42) AND (status = 'shipped'))

Planning Time: 0.3 ms
Execution Time: 0.1 ms      ← 0.1ms! Beautiful.
```

**Key things to look for:**

| Output | Meaning | Verdict |
|---|---|---|
| `Seq Scan` | Full table scan | Usually bad on large tables |
| `Index Scan` | Using index, then fetching heap | Good |
| `Index Only Scan` | Covering index, no heap fetch | Great |
| `Bitmap Heap Scan` | Multiple index matches batched | OK, depends |
| High `rows=` estimate way off from actual | Stale statistics | Run ANALYZE |

**The GitHub postmortem lesson applies here directly:** if the team had run `EXPLAIN` on their `owner_id` query, they would have seen `Seq Scan on repositories (rows=12000000)` and added the index before the outage. Make `EXPLAIN` part of your code review checklist for any query touching a table with more than ~100k rows.

---

# PART 2: Caching Strategies

---

## Q7: What's the simplest caching pattern and how does it work?

**Cache-aside (also called lazy loading) is the most common pattern. Learn this one first.**

The application code handles all the logic:

```
Read Request Flow:

   Client
     |
     v
  Application
     |
     |--- 1. Check cache for key "user:42" ----> Cache (Redis/Memcached)
     |                                                |
     |         Cache HIT: return value <------------- |  (done, fast path)
     |
     |         Cache MISS: key not found
     |
     |--- 2. Query database ----------------------> Database
     |         result = { id: 42, name: "Alice" }
     |
     |--- 3. Write result to cache
     |         SET "user:42" { id:42, name:"Alice" } EX 3600
     |
     |--- 4. Return result to client
```

```python
def get_user(user_id):
    cache_key = f"user:{user_id}"

    # Step 1: Try cache first
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache HIT - fast path

    # Step 2: Cache miss - go to DB
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)

    # Step 3: Populate cache for next time
    redis.setex(cache_key, 3600, json.dumps(user))  # TTL = 1 hour

    # Step 4: Return result
    return user
```

**Why "lazy" loading?** The cache only gets populated when data is first requested. Cold start = all misses. After a while, frequently-accessed data is hot in cache.

**Pros:**
- Simple to implement
- Cache only contains data that's actually been requested
- DB failure doesn't take down reads that are already cached

**Cons:**
- First request is always slow (cold cache miss)
- Cache can get stale if DB changes and cache isn't invalidated
- Can lead to thundering herd (more on this in Q10)

---

## Q8: What's write-through vs write-behind caching? When do I use each?

**These patterns are about what happens when you WRITE data, not just read it.**

### Write-Through

Every write goes to the cache AND the database synchronously, together:

```
Write Request Flow (Write-Through):

   Client
     |
     v
  Application
     |--- 1. Write to Cache ------------------> Cache
     |         (update "user:42")
     |
     |--- 2. Write to Database ----------------> Database
     |         (UPDATE users SET ...)
     |
     |--- 3. Confirm success to client
```

```python
def update_user(user_id, name):
    # Write to both simultaneously (or DB first, then cache)
    db.execute("UPDATE users SET name = %s WHERE id = %s", name, user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps({"id": user_id, "name": name}))
```

**Pros:** Cache is never stale. Reads after a write are always correct.
**Cons:** Every write is slower (two writes instead of one). Cache fills up with data that may never be read.

---

### Write-Behind (Write-Back)

Write to cache immediately, write to DB *asynchronously* later:

```
Write Request Flow (Write-Behind):

   Client
     |
     v
  Application
     |--- 1. Write to Cache ------------------> Cache (fast, immediate)
     |         Client gets success response
     |
     (background)
     |--- 2. Async flush to DB ----------------> Database
     |         (happens every few seconds/minutes)
```

**Pros:** Writes are extremely fast (only cache write in the hot path). Great for write-heavy workloads like counters, view counts, game scores.
**Cons:** Risk of data loss if cache crashes before DB flush. Complexity in the async flush logic.

**Real example — Stack Overflow's view counter:**

Stack Overflow receives millions of pageviews. If every view did `UPDATE posts SET view_count = view_count + 1`, their SQL Server would be hammered. Instead, they batch view counts in cache and periodically flush to the DB. The view count you see might be a few minutes stale — but the site stays fast. This is write-behind in practice.

---

## Q9: What is TTL and why is it one of the most important knobs you have?

**TTL (Time To Live) is how long a cache entry lives before it expires and is considered stale.**

```
SET user:42 {...} EX 3600    # Expires in 3600 seconds (1 hour)
```

Setting TTL is one of the hardest judgment calls in system design. There's a fundamental tension:

```
SHORT TTL                              LONG TTL
(e.g., 60 seconds)                    (e.g., 24 hours)

+ Cache stays fresh                    + Cache hit rate is high
+ Stale data risk is low               + DB load stays low
- DB gets more traffic                 - Stale data risk is high
- More cache misses                    - Updates take a long time to propagate
```

**How to pick TTL — think about the data's "staleness tolerance":**

| Data Type | Good TTL | Reason |
|---|---|---|
| User profile (name, avatar) | 1-24 hours | Changes rarely, stale is mostly fine |
| Product price | 5-15 minutes | Changes sometimes, accuracy matters |
| Stock price / sports score | 10-60 seconds | Changes constantly, freshness is the product |
| Auth tokens / sessions | Exactly until expiry | Must match auth system |
| Homepage featured content | 5 minutes | Marketing can wait a bit |

**The sneaky problem with fixed TTL:** If you cache 1 million user profiles all at 3600s TTL and you restarted your cache at the same time (e.g., a deployment), they all expire at the same moment. One hour later, you get 1 million simultaneous cache misses. This is called a **TTL thundering herd** — a variant of the problem we'll cover in Q10.

**Fix:** Add jitter (randomness) to TTL:

```python
import random

# Instead of always 3600:
ttl = 3600 + random.randint(-300, 300)  # Between 3300 and 3900 seconds
redis.setex(cache_key, ttl, value)
```

Spreading expirations across a 10-minute window turns a synchronized spike into a smooth trickle.

---

## Q10: What is a cache stampede / thundering herd? How did it almost destroy Facebook?

**This is one of the most famous cache failure modes. It caused a major Facebook incident in 2010.**

**The scenario:**

1. A popular key expires in Memcached (e.g., "front page news feed")
2. 10,000 requests arrive simultaneously, all asking for that key
3. All 10,000 see a cache miss
4. All 10,000 query the database at the same time
5. Database gets crushed under 10,000 simultaneous identical queries
6. DB response time spikes → more requests pile up → cascade failure

```
Normal flow (cache warm):

10,000 requests ──→ Cache HIT ──→ return cached value
                    (1 DB query ever)

Thundering herd (after cache expiry):

request 1 ──→ Cache MISS ──→ DB query #1
request 2 ──→ Cache MISS ──→ DB query #2
request 3 ──→ Cache MISS ──→ DB query #3
    ...
request 10000 → Cache MISS → DB query #10000
                              ↑
                         DB on fire
```

**The Facebook 2010 incident:**

Facebook's Memcached cluster was serving billions of requests per day for things like friend lists, news feed data, and profile information. During a maintenance window where some Memcached nodes went offline, the thundering herd hit their MySQL cluster with millions of simultaneous queries. The DB couldn't handle it, and Facebook experienced a cascading outage. This event became a seminal case study and led Facebook to publish their famous Memcached paper, which described how they built cache lease mechanisms at massive scale.

**Solution 1: Mutex lock / cache lock**

Only one request gets to regenerate the cache. Others wait.

```python
def get_with_lock(cache_key, fetch_fn, ttl=3600):
    value = redis.get(cache_key)
    if value:
        return json.loads(value)

    lock_key = f"lock:{cache_key}"
    lock_acquired = redis.set(lock_key, "1", nx=True, ex=10)  # nx = only if not exists

    if lock_acquired:
        # We won the lock — go fetch and populate
        try:
            value = fetch_fn()
            redis.setex(cache_key, ttl, json.dumps(value))
            return value
        finally:
            redis.delete(lock_key)
    else:
        # Someone else is fetching — wait briefly and retry
        time.sleep(0.05)
        return get_with_lock(cache_key, fetch_fn, ttl)
```

**Solution 2: Probabilistic Early Expiration (XFetch)**

Instead of waiting for the key to fully expire, start probabilistically refreshing it *before* it expires:

```python
import math
import random
import time

def get_with_early_expiration(cache_key, fetch_fn, ttl=3600, beta=1.0):
    value, expiry = redis.get_with_expiry(cache_key)  # get value + remaining TTL

    if value is None:
        # Already expired, fetch immediately
        value = fetch_fn()
        redis.setex(cache_key, ttl, json.dumps(value))
        return value

    # Should we refresh early?
    # Higher beta = more aggressive early refresh
    # As expiry approaches, probability of early refresh increases
    fetch_time_estimate = 0.1  # estimated seconds to fetch from DB

    if time.time() - fetch_time_estimate * beta * math.log(random.random()) > expiry:
        # Probabilistically refresh now, before the key expires
        value = fetch_fn()
        redis.setex(cache_key, ttl, json.dumps(value))

    return json.loads(value)
```

The math works out so that the chance of early refresh increases as the expiry time approaches. Traffic is spread over a window instead of all hitting at once.

**Solution 3: Background refresh**

Never let the key expire at all for popular content. A background job refreshes it before it expires:

```python
# Background worker
def refresh_popular_keys():
    for key in get_popular_cache_keys():
        value = fetch_from_db(key)
        redis.setex(key, 3600, json.dumps(value))

# Run every 30 minutes (well before 1hr TTL)
schedule.every(30).minutes.do(refresh_popular_keys)
```

---

## Q11: What is LRU eviction and what happens when your cache fills up?

**Caches have limited memory. When they're full, something has to go. LRU decides what.**

**LRU = Least Recently Used.** When the cache is full and a new item needs to be added, evict the item that was accessed least recently. The idea: if you haven't needed it recently, you probably won't need it soon.

```
Cache size: 4 slots

Initial state:
[ A | B | C | D ]
  ^most recent       ^least recent

Access E (not in cache, cache is full):
  → Evict D (least recently used)
  → Insert E

[ E | A | B | C ]
  ^most recent       ^least recent

Access B:
  → Cache hit, B moves to front

[ B | E | A | C ]
  ^most recent       ^least recent
```

**LRU is implemented with a doubly-linked list + hashmap:**

```
HashMap: { key → node pointer }  ← O(1) lookup
Doubly Linked List: head ↔ [B] ↔ [E] ↔ [A] ↔ [C] ↔ tail
                    (most recent)           (least recent)

On access: move node to head — O(1)
On eviction: remove tail — O(1)
```

**Redis eviction policies** (set in `redis.conf` or at runtime):

| Policy | Behavior |
|---|---|
| `noeviction` | Return error when memory full (default) |
| `allkeys-lru` | Evict least recently used key from all keys |
| `volatile-lru` | Evict LRU key only from keys with TTL set |
| `allkeys-lfu` | Evict least *frequently* used (LFU — different from LRU) |
| `allkeys-random` | Evict random key |

```bash
# Set policy in Redis
redis-cli config set maxmemory-policy allkeys-lru
redis-cli config set maxmemory 2gb
```

**LRU vs LFU:** LRU evicts what was accessed *least recently*. LFU evicts what was accessed *least frequently*. LFU is better if some items are accessed infrequently but in bursts (a viral post that gets 1M hits, then nothing). LRU would evict it after the burst dies down; LFU would keep it around because of its high access count.

---

## Q12: Cache invalidation — why is it called "the hardest problem in CS"?

**Because there is no perfect solution. Every approach has a tradeoff.**

When data in the DB changes, you need the cache to either update or expire. This sounds simple. It is not.

**The core problem:**

```
Time 0: Cache has user:42 → { name: "Alice", email: "alice@old.com" }
Time 1: User updates email to "alice@new.com" in DB
Time 2: Another request reads user:42 from CACHE
         → Gets old email "alice@old.com"
         → Shows stale data

How long until cache expires? Could be hours.
Does the user notice the stale data? Maybe. Do they email support? Definitely.
```

**Strategy 1: Delete on write (cache invalidation)**

When data changes in DB, immediately delete the cache key. Next read is a miss and fetches fresh data.

```python
def update_user_email(user_id, new_email):
    db.execute("UPDATE users SET email = %s WHERE id = %s", new_email, user_id)
    redis.delete(f"user:{user_id}")  # Nuke the cache key
```

**Why delete instead of update?** Avoid a race condition:

```
Thread A: updates DB (email = new)
Thread B: reads DB (gets email = new), writes to cache
Thread A: writes old value to cache (if update not delete)
          → Cache now has OLD value despite DB having new value!

Deleting is safer — next read simply re-fetches from DB.
```

**Strategy 2: Versioned cache keys**

```python
def get_user(user_id):
    version = redis.get(f"user:{user_id}:version") or "1"
    return redis.get(f"user:{user_id}:v{version}")

def update_user(user_id, data):
    db.execute("UPDATE users SET ...")
    redis.incr(f"user:{user_id}:version")  # Bump version, old key becomes orphan
```

**Strategy 3: Event-driven invalidation**

Use database change events (CDC — Change Data Capture) to automatically invalidate cache:

```
DB Write → Binlog/WAL → CDC stream (e.g., Debezium) → Message Queue → Cache Invalidation Service → Delete cache key
```

This is what large-scale systems like Netflix and LinkedIn do. Every DB write publishes an event; a separate service consumes events and invalidates the appropriate cache keys.

**Why it's STILL hard even with all these strategies:**

```
Race Condition Example:

T1: Request A reads key "user:42" → cache MISS
T2: Request B updates user:42 in DB → deletes cache key
T3: Request A finishes DB read → writes OLD value to cache
T4: Now cache has stale data again despite an invalidation just happening

Solution: conditional writes (only write to cache if key still doesn't exist)
          or use a version/generation number in the cache key itself
```

**The Stack Overflow "runs on little hardware" angle:**

Stack Overflow famously serves billions of pageviews with a remarkably small number of servers. One of their secrets: their cache invalidation strategy is extremely explicit and deliberate. When a question is edited, they purge the cached rendered HTML for that question. When a tag is renamed, they have a specific invalidation path for that. Instead of relying on TTL drift, they have code paths that say "this specific thing changed, invalidate these specific keys." It's more work upfront, but it means their cache hit rate stays extremely high and they rarely serve stale data — which is why they need fewer servers to handle the load.

**The iron laws of cache invalidation:**

1. There is no global perfect solution — pick your tradeoff (consistency vs. simplicity vs. performance)
2. Prefer explicit invalidation over relying solely on TTL for correctness-sensitive data
3. Design your cache keys to be invalidatable — avoid giant "blob" keys that contain many pieces of data
4. Accept that distributed cache invalidation has a window of inconsistency — the question is how wide that window is

---

# Quick Revision Table

## Indexing

| Concept | One-Line Summary | Watch Out For |
|---|---|---|
| B-Tree Index | Sorted tree structure; O(log n) lookup vs O(n) full scan | Don't index everything blindly |
| Index on writes | Every write updates all indexes — write performance degrades | Balance read vs write load |
| Composite index | Multi-column index; leftmost prefix rule applies | Column order is critical |
| Covering index | Index contains all SELECT columns; no heap fetch needed | More disk space |
| Low cardinality | Boolean/enum columns — index often ignored or harmful | Check with EXPLAIN first |
| EXPLAIN | Shows query plan: Seq Scan (bad) vs Index Scan (good) | Run on any query touching >100k rows |

## Caching

| Concept | One-Line Summary | Watch Out For |
|---|---|---|
| Cache-aside | App checks cache, on miss fetches DB and populates cache | Cold start, first request is slow |
| Write-through | Write to cache AND DB synchronously on every write | Slower writes, cache has unread data |
| Write-behind | Write to cache immediately, async flush to DB later | Data loss on cache crash |
| TTL | Expiry time for cache entries; balance freshness vs DB load | Add jitter to avoid synchronized stampede |
| LRU eviction | Evict least recently used when cache is full | LFU is better for bursty access patterns |
| Cache stampede | Simultaneous misses on popular key after expiry → DB crushed | Use mutex lock or probabilistic early expiration |
| Cache invalidation | Removing/updating stale cache entries when DB changes | Race conditions; explicit > TTL-only for correctness |

---

> **Final thought:** Database indexes and caching are complements, not alternatives. Indexes make your DB fast under load. Caching keeps load off your DB in the first place. The engineers who built systems that scaled — Stack Overflow, Instagram, Facebook — used both, tuned both, and understood the failure modes of both. Now you do too.
