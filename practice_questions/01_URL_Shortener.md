# Design a URL Shortener like TinyURL
> Complete interview preparation guide — Q&A format, beginner to advanced.

---

## Table of Contents
1. [What Is It?](#what-is-it)
2. [Step 1 — Requirements](#step-1--requirements)
3. [Step 2 — Capacity Estimation](#step-2--capacity-estimation)
4. [Step 3 — High-Level Design](#step-3--high-level-design)
5. [Step 4 — Core Algorithm: Generating Short URLs](#step-4--core-algorithm-generating-short-urls)
6. [Step 5 — Database Design](#step-5--database-design)
7. [Step 6 — API Design](#step-6--api-design)
8. [Step 7 — Deep Dives](#step-7--deep-dives)
9. [Step 8 — Failure Scenarios](#step-8--failure-scenarios)
10. [Full System Diagram](#full-system-diagram)
11. [Quick Revision](#quick-revision)

---

## What Is It?

**Q: What does a URL shortener actually do?**

> A URL shortener maps a long URL to a short, unique code. When a user visits the short URL, the server looks up the code and redirects them to the original URL.
>
> ```
> Input:   https://www.some-website.com/articles/how-to-study?ref=twitter&campaign=2024
> Output:  tinyurl.com/x7k2pq
>
> User visits tinyurl.com/x7k2pq
>     → server looks up "x7k2pq"
>     → finds original URL
>     → sends redirect response → user lands on original page
> ```

**Q: Why do people use URL shorteners?**

> - Social media character limits (a long URL eats up your entire tweet)
> - Cleaner, shareable links in print/presentations
> - Track click analytics (how many people clicked, from where, on what device)
> - Custom branded links (`company.co/sale` instead of a messy long URL)
> - Hide affiliate tracking parameters from view

---

## Step 1 — Requirements

**Q: What are the functional requirements?**

> 1. **Shorten a URL** — given a long URL, generate and return a unique short URL
> 2. **Redirect** — when a user visits the short URL, redirect them to the original long URL
> 3. **Link expiration** — short URLs should stop working after a configured time period
> 4. *(Optional)* **Custom alias** — user can choose their own short code (e.g. `tinyurl.com/my-portfolio`)
> 5. *(Optional)* **Analytics** — track click count, location, and device type per short URL

**Q: What are the non-functional requirements?**

> - **High availability** — if TinyURL goes down, every short link on the internet breaks. Must be 99.99%+ uptime.
> - **Low latency** — redirects must happen in milliseconds. User should not notice any delay.
> - **Scalability** — must handle millions of URL creations and billions of redirects per day.
> - **Durability** — shortened URLs must persist for years. Data must not be lost.
> - **Security** — prevent phishing links and spam abuse.

**Q: Is this system read-heavy or write-heavy?**

> Massively **read-heavy**. Users create short URLs occasionally, but click on them constantly.
>
> Typical ratio: **100:1 reads to writes** — for every 1 URL created, it gets clicked ~100 times.
>
> This single fact drives the architecture: **optimize aggressively for reads** → caching is essential.

---

## Step 2 — Capacity Estimation

**Q: Walk me through capacity estimation for TinyURL.**

> **Assumptions:**
> - 100 million new URLs created per day
> - 100:1 read/write ratio → 10 billion redirects per day
>
> **Traffic per second:**
> ```
> Writes: 100,000,000 / 86,400 seconds ≈ 1,160 writes/second
> Reads:  10,000,000,000 / 86,400 seconds ≈ 116,000 reads/second
> ```
>
> **Storage over 10 years:**
> ```
> Each URL record ≈ 500 bytes (short_id + long_url + timestamps + metadata)
> 100M URLs/day × 365 days × 10 years = 365 billion records
> 365 billion × 500 bytes ≈ 182 TB
> ```
>
> **Cache memory needed:**
> ```
> 80% of clicks hit 20% of URLs (Pareto principle — viral URLs dominate)
> Cache hot 20%: 100M URLs/day × 20% × 500 bytes ≈ 10 GB/day
> → Very manageable Redis cache size
> ```

**Q: What does this estimation tell you about your architecture choices?**

> - **116,000 reads/second** → a single database server cannot handle this → need caching layer + read replicas
> - **1,160 writes/second** → manageable on a single primary DB to start with
> - **182 TB over 10 years** → sharding will eventually be required; won't fit on one machine
> - **10 GB cache** → Redis can hold all hot URLs comfortably in memory

---

## Step 3 — High-Level Design

**Q: Draw the high-level architecture.**

> ```
>                       [Users / Clients]
>                               │
>                      [Load Balancer]
>                       │            │
>              [Write Service]    [Read Service x N]
>              (URL creation)     (redirects)
>                    │                  │
>            [Primary DB]         [Redis Cache]
>            (writes only)        (LRU, 10 GB)
>                    │                  │ miss
>                    │ replicates  [Read Replica DB x N]
>                    └────────────────►(reads only)
>                    │
>            [ID Generator]
>            [Cleanup Cron Job]
>            (nightly, deletes expired rows)
> ```

**Q: Why separate Write Service and Read Service?**

> They have completely different scaling needs:
> - **Read Service** handles 116,000 req/sec → needs many instances, heavy caching, read replicas
> - **Write Service** handles 1,160 req/sec → needs far fewer instances, only writes to primary DB
>
> Separating them lets you scale each independently and avoids coupling unrelated concerns.

---

## Step 4 — Core Algorithm: Generating Short URLs

**Q: What are the two essential properties a short code must have?**

> 1. **Short** — ideally 6-8 characters for readability and shareability
> 2. **Unique** — two different long URLs must never get the same short code

**Q: What are the approaches to generating short codes and which is best?**

> **Option A: Hashing (MD5)**
> ```
> MD5("https://very-long-url.com") → "a9b8c7d6e5f4..."
> Take first 6 characters → "a9b8c7"
> ```
> Problem: **hash collisions** — two different URLs could produce the same first 6 characters. You need collision detection + retry logic, adding DB lookups on every write.
>
> **Option B: Base62 Encoding (Recommended)**
> ```
> Characters available: a-z (26) + A-Z (26) + 0-9 (10) = 62 characters
>
> 62^6 = 56,800,235,584  →  over 56 billion unique codes from just 6 characters
> 62^7 = 3.5 trillion unique codes from 7 characters
>
> How it works:
> 1. Maintain a global auto-incrementing counter (1, 2, 3, 4...)
> 2. Encode that integer in Base62
>    ID = 1       → "000001"
>    ID = 100     → "000001s"
>    ID = 1000000 → "4c92"
> 3. Use encoded value as the short code
> ```
> **Guaranteed unique** (tied to a unique integer), no collision risk, no extra DB lookups.

**Q: What is the problem with a single global auto-increment counter at scale?**

> At 1,160 writes/second, every write must fetch-and-increment the same counter — it becomes a **bottleneck and a single point of failure**.
>
> **Fix — Range-based ID generation:**
> ```
> Coordinator assigns each Write Service instance a range:
> Server 1 → IDs 1 to 1,000,000
> Server 2 → IDs 1,000,001 to 2,000,000
> Server 3 → IDs 2,000,001 to 3,000,000
>
> Each server generates IDs from its own range independently.
> When range is exhausted → fetch next range from coordinator.
> ```
> Or use **Twitter Snowflake IDs** — 64-bit IDs composed of timestamp + machine ID + sequence number. Globally unique, generated independently per machine, no central coordinator needed.

**Q: Why not just use a random string?**

> You could, but you'd have to check the database on every write to ensure no collision — an expensive lookup at 1,160 writes/second. Auto-increment Base62 gives guaranteed uniqueness with zero DB lookups during generation.

---

## Step 5 — Database Design

**Q: What does the database schema look like?**

> ```
> URLs table:
> ┌────────────┬───────────────────────────────┬─────────┬─────────────┬────────────┐
> │ short_id   │ long_url                      │ user_id │ created_at  │ expires_at │
> ├────────────┼───────────────────────────────┼─────────┼─────────────┼────────────┤
> │ x7k2pq     │ https://very-long-url.com/... │ 1042    │ 2024-01-01  │ 2025-01-01 │
> │ abc123     │ https://another-url.com/...   │ NULL    │ 2024-01-02  │ NULL       │
> └────────────┴───────────────────────────────┴─────────┴─────────────┴────────────┘
>
> Indexes:
>   PRIMARY KEY on short_id  → every redirect query hits this column
>   INDEX on user_id         → for "show me all my URLs" feature
> ```

**Q: SQL or NoSQL — which do you choose and why?**

> **NoSQL (Cassandra or DynamoDB)** is the stronger fit:
> - The access pattern is purely key-value: look up `long_url` by `short_id` — no JOINs needed
> - 116,000 reads/second → NoSQL scales horizontally with ease
> - Simple schema that won't need complex relational queries
>
> **SQL (PostgreSQL/MySQL)** is also acceptable — simpler to reason about, fine at medium scale.
>
> Either answer is valid in interviews. The key is justifying your choice based on access patterns and scale.

**Q: How do you handle URL expiration — both for users and for database hygiene?**

> **Two-part strategy:**
>
> **1. Lazy deletion — instant user feedback:**
> ```sql
> SELECT long_url FROM urls
> WHERE short_id = 'x7k2pq'
>   AND (expires_at IS NULL OR expires_at > NOW())
> ```
> If expired → return HTTP 410 Gone. Row stays in DB temporarily.
>
> **2. Background batch cleanup — keep DB clean:**
> ```sql
> -- Cron job runs nightly at 2am:
> DELETE FROM urls WHERE expires_at < NOW()
> ```
> Deletes all expired rows in bulk. Use batch sizes (e.g. 1000 rows at a time) to avoid locking the table.
>
> Both together: user gets immediate feedback AND the database doesn't bloat with dead rows.

---

## Step 6 — API Design

**Q: What are the two core APIs?**

> **1. Create Short URL**
> ```
> POST /api/v1/shorten
>
> Request body:
> {
>   "long_url":     "https://very-long-url.com/...",
>   "custom_alias": "my-link",      // optional
>   "expires_at":   "2025-01-01"    // optional
> }
>
> Response: 201 Created
> {
>   "short_url":  "tinyurl.com/x7k2pq",
>   "short_id":   "x7k2pq",
>   "expires_at": "2025-01-01"
> }
> ```
>
> **2. Redirect**
> ```
> GET /{short_id}
> Example: GET /x7k2pq
>
> Response: 302 Found
> Location: https://very-long-url.com/...
> ```

**Q: Should the redirect return HTTP 301 or 302? (Classic trick question)**

> - **301 Permanent Redirect** → browser caches it forever. Next click goes DIRECTLY to the destination without touching TinyURL's server.
>   - Pro: reduces server load significantly
>   - Con: you lose all analytics — requests bypass your server entirely
>
> - **302 Temporary Redirect** → browser never caches it. Every click goes through your server.
>   - Pro: you capture every click for analytics
>   - Con: more server load per click
>
> **Answer:** Use **302** when analytics matter (which they almost always do). Use 301 only when you explicitly want to minimize server load and analytics are not required.

---

## Step 7 — Deep Dives

### Caching

**Q: Walk through the exact redirect flow with caching.**

> ```
> User clicks tinyurl.com/x7k2pq
>         ↓
> Read Service receives: GET /x7k2pq
>         ↓
> Check Redis cache for key "x7k2pq"
>         ↓
>   ┌─── Cache HIT ────────────────────────────────────────────┐
>   │  Return long_url instantly ⚡ (microseconds, no DB hit)  │
>   └──────────────────────────────────────────────────────────┘
>         ↓
>   ┌─── Cache MISS ───────────────────────────────────────────┐
>   │  Query Read Replica DB: SELECT long_url WHERE short_id=? │
>   │  Check expiry: if expires_at < NOW() → return 410 Gone   │
>   │  Store result in Redis with TTL                          │
>   │  Return long_url                                         │
>   └──────────────────────────────────────────────────────────┘
> ```
> With 10 GB of hot URLs in Redis, ~80% of requests return from cache without touching the DB.

**Q: What Redis eviction policy and TTL should you use?**

> - **Eviction policy: LRU (Least Recently Used)** — evicts URLs that haven't been clicked recently. Hot/viral URLs stay in cache automatically. Perfect match for URL access patterns.
>
> - **TTL strategy:**
>   - Non-expiring URL → cache with 24h TTL, refresh on access
>   - Expiring URL → `cache_ttl = min(24h, expires_at - now)` — never cache past the URL's own expiry

---

### Database Replication

**Q: The primary database is a single point of failure. Fix it.**

> **Primary-Replica replication:**
> ```
> Write Service ──► Primary DB (all writes)
>                        │
>                        │ replicates continuously
>                        ↓
> Read Service  ──► Read Replica 1  ◄─ (reads load-balanced across replicas)
>               ──► Read Replica 2
>               ──► Read Replica 3
> ```
> - Primary fails → replica promoted to primary automatically (30-60 seconds)
> - Add more replicas for more read capacity
> - No single point of failure on the read path

---

### Scalability — 10x Traffic

**Q: The system grows 10x. What breaks first and how do you fix it?**

> Walk through layer by layer:
>
> 1. **Cache memory fills up** → upgrade Redis memory OR switch to Redis Cluster (distributes keys across multiple Redis nodes using consistent hashing)
>
> 2. **Read Replicas overwhelmed** → add more replicas. Route reads via load balancer across them.
>
> 3. **Write throughput exceeds primary DB** → introduce **database sharding**
>    ```
>    Shard by short_id:
>    short_ids a-f → Shard 1
>    short_ids g-l → Shard 2
>    short_ids m-r → Shard 3
>    short_ids s-z → Shard 4
>    ```
>
> 4. **ID generation is a bottleneck** → switch to range-based allocation or Snowflake IDs (distributed generation, no central coordinator)

**Q: How do you handle a suddenly viral short URL (millions of hits in minutes)?**

> A single hot URL can overwhelm even a cached system:
>
> 1. **Local in-memory cache on app servers** — top 100 most-requested URLs cached in each app server's own memory. Zero network hop, even faster than Redis.
> 2. **Rate limiting per short_id** — prevent any single URL from consuming more than X% of system capacity
> 3. **Pre-warming cache** — if your analytics detect a URL going viral, proactively push it to all cache layers before the spike hits

---

## Step 8 — Failure Scenarios

**Q: Redis goes down. What happens?**

> All traffic falls through to the Read Replica DBs. System still works — just slower.
> 116,000 req/sec now hitting the DB directly may overwhelm replicas.
>
> **Prevention:** Run Redis as a **Redis Cluster** (3+ nodes). If one node fails, consistent hashing routes its keys to other nodes. Only that node's cached entries temporarily miss.

**Q: Primary DB goes down. What happens?**

> - Read traffic continues fine (replicas keep serving redirects)
> - Write traffic (new URL creation) temporarily fails
> - Automated failover promotes one replica to primary (30-60 seconds)
> - Write Service detects new primary endpoint and reconnects
>
> Use managed DB with automated Multi-AZ failover (e.g. AWS RDS Multi-AZ) to minimize downtime.

**Q: ID Generator service goes down. What happens?**

> New URL creation fails. Existing redirect traffic is completely unaffected.
>
> **Prevention:** Run multiple ID Generator instances, each with their own pre-allocated ID range. No single point of failure.

**Q: How do you prevent abuse (spammers creating millions of links)?**

> 1. **Rate limiting** — max 100 new URLs/day per IP for anonymous users, higher limits for authenticated users
> 2. **Authentication required** — require account creation to reduce anonymous abuse
> 3. **URL malware scanning** — check `long_url` against known phishing/malware databases before shortening
> 4. **CAPTCHA** — on unauthenticated creation requests

---

## Full System Diagram

```
                         [Users / Clients]
                                │
                       [Load Balancer]
                        │             │
             [Write Service]      [Read Service x N]
                   │                    │
          [ID Generator]          [Redis Cache]
          (Range/Snowflake)       (LRU, 10 GB)
                   │                    │ miss
          [Primary DB] ──replicates──► [Read Replicas x N]
          (writes only)                (reads only)
                   │
          [Cleanup Cron Job]
          (nightly: DELETE expired rows)
```

**Request flows:**

```
CREATE:  Client → LB → Write Service → ID Generator → Primary DB → return short URL
REDIRECT: Client → LB → Read Service → Redis HIT → 302 redirect
                                     → Redis MISS → Read Replica → cache it → 302 redirect
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| Short code algorithm | Base62 of auto-increment ID | Guaranteed unique, no collision checks |
| Short code length | 6 characters | 56 billion combinations |
| Database type | NoSQL (Cassandra) or SQL | Simple key-value access, massive scale |
| Read optimization | Redis cache (LRU) | 80% of traffic hits 20% of URLs |
| Cache TTL | 24h / min(24h, expires_at) | Never serve expired links from cache |
| Redirect type | HTTP 302 | Captures analytics; 301 bypasses server |
| Expiry handling | Lazy check + nightly batch DELETE | Fast user feedback + clean database |
| DB high availability | Primary + Read Replicas | No single point of failure |
| ID generation at scale | Range-based or Snowflake | No central bottleneck |
| Abuse prevention | Rate limiting + URL blacklist | Block spam and phishing |

### Three things to mention proactively in an interview
> 1. **301 vs 302** — shows you understand the analytics trade-off
> 2. **ID generation strategy** — shows you know single counters don't scale
> 3. **Lazy deletion + cron cleanup** — shows you thought about operational hygiene, not just the happy path
