# Full Mock Interview Walkthroughs

Three complete system design walkthroughs from start to finish — formatted as real interview dialogues. Each walkthrough is a condensed 45-minute session showing exactly what to say, what to draw, and how to think out loud.

---

## How to Use This Document

Read each walkthrough as a script. The interviewer sets the problem. You play the candidate. The candidate lines show you what to actually say — not bullet points you have to memorize, but natural language that demonstrates structured thinking.

The goal is to internalize the *pattern*, not memorize the answers.

---

---

# Walkthrough 1: Design a URL Shortener (bit.ly)

**Difficulty:** Beginner-friendly  
**Time:** ~45 minutes  
**Why this problem:** Clean scope, clear bottlenecks, teachable ID generation — great for learning the 5-step framework.

---

## Step 1 — Requirements Clarification (5 minutes)

> **Interviewer:** "Design a URL shortening service like bit.ly."

> **You:** "Great. Before I jump in, I'd like to ask a few clarifying questions to make sure I'm solving the right problem.
>
> First — what are the core features we need? I'm assuming: shorten a long URL to a short one, redirect users when they visit the short URL, and maybe analytics like click counts. Is that right? Are there any other features like expiration dates or custom aliases?"

> **Interviewer:** "Yes, those three. Let's say custom aliases are a nice-to-have. No expiration for now."

> **You:** "Got it. What kind of scale are we targeting? How many URLs are shortened per day, and how many redirects?"

> **Interviewer:** "Assume 1 billion URLs shortened total, 100 million daily active users."

> **You:** "And is this a read-heavy or write-heavy system? I'd expect far more redirects than URL creations — maybe 100:1?"

> **Interviewer:** "Good assumption, yes."

> **You:** "Last one — does the short code need to be unpredictable, like a random hash? Or is sequential fine?"

> **Interviewer:** "Unpredictable is better. We don't want people guessing others' URLs."

> **You:** "Perfect. Let me summarize what I'm building: shorten a URL and return a short code, redirect short code to original URL, track click counts. Read-heavy (100:1 ratio), unpredictable short codes, 100M DAU. I'll also think about storage since we have 1B URLs total."

---

## Step 2 — Scale Estimation (5 minutes)

> **You:** "Let me do a quick back-of-envelope. Bear with me.
>
> If we have 100M DAU and they each shorten about 1 URL per day, that's roughly 100M writes/day. Over 10 years that's 365 billion writes — but the problem says 1B total, so let's use 1B as our target storage ceiling.
>
> For reads: if redirects are 100x writes, we're looking at 10B redirects/day. That's about 115,000 read requests per second. Write QPS is around 1,160/second.
>
> For storage: each URL record might be ~500 bytes (long URL + short code + metadata). 1B records × 500 bytes = 500 GB of URL data. That fits comfortably on a few database servers.
>
> So the main challenge is read throughput — 115K reads/second is high. We'll need a caching layer."

---

## Step 3 — High-Level Design (10 minutes)

> **You:** "Let me sketch the write path first, then the read path."

```
WRITE PATH (Create Short URL)

Client
  |
  v
[ Load Balancer ]
  |
  v
[ API Server ]
  |
  +---> [ ID Generator Service ] --> generates unique 7-char code
  |
  v
[ Database (PostgreSQL) ]
  stores: short_code -> long_url, created_at, user_id, clicks
```

```
READ PATH (Redirect)

Client visits bit.ly/aX3kP9z
  |
  v
[ Load Balancer ]
  |
  v
[ API Server ]
  |
  +---> [ Redis Cache ] --- HIT? --> return long_url --> 301 redirect
  |           |
  |           MISS
  |           |
  +---------> [ Database ] --> fetch long_url --> store in cache --> redirect
```

> **You:** "The write path is straightforward: client sends long URL, API server generates a short code, stores it in the database. The read path is where performance matters: we check Redis first. If it's a cache hit, we redirect immediately without touching the DB. If it's a miss, we fetch from DB, populate cache, then redirect.
>
> One question I want to flag for the deep dive: how do we generate that short code? That's the interesting part of this problem."

> **Interviewer:** "Yes, let's go there."

---

## Step 4 — Deep Dive: ID Generation (10 minutes)

> **You:** "There are three main approaches to generating a unique short code. Let me walk through each and explain the tradeoff."

### Option A: Auto-increment + Base62 Encoding

> **You:** "The simplest approach: use a database auto-increment ID, then encode that number in base62 (a-z, A-Z, 0-9 = 62 characters).
>
> For example, if the DB gives us ID = 125, we convert 125 to base62:
>
> - 125 / 62 = 2 remainder 1 → characters '2' and '1'
> - Result: '21' in base62
>
> A 7-character base62 string gives us 62^7 = 3.5 trillion combinations — more than enough for 1 billion URLs.
>
> The problem: it's predictable. IDs are sequential, so someone can guess other short codes. Also, a single auto-increment counter is a bottleneck if we have multiple write servers."

### Option B: MD5/SHA Hash of the Long URL

> **You:** "Take the long URL, MD5 hash it, take the first 7 characters.
>
> Problem: collision risk. Two different URLs could produce the same first 7 characters of their hash. We'd need to check the database and retry, which adds latency and complexity."

### Option C: Dedicated ID Generation Service (Recommended)

> **You:** "The cleanest approach for scale: a dedicated ID generator service — like Twitter's Snowflake — that hands out unique IDs without needing the database in the critical path.
>
> Alternatively, we can pre-generate a pool of random 7-character codes, store them in a 'keys' table marked as 'available' or 'used'. When a URL is shortened, we grab an available key, mark it used, and return it. This is the approach bit.ly actually uses — it's called a Key Generation Service (KGS)."

```
Key Generation Service (KGS)

[ Offline worker ] --> pre-generates random 7-char codes
                  --> stores in [ keys_db ] with status: 'available'

On URL shortening request:
[ API Server ] --> SELECT key WHERE status='available' LIMIT 1 FOR UPDATE
              --> UPDATE status = 'used'
              --> return that key
```

> **You:** "This decouples key generation from request handling, avoids hash collisions, and is fast because we're just doing a single DB lookup. We keep a buffer of pre-generated keys in memory on each API server so we don't hit the key DB on every request."

---

## Step 5 — Deep Dive: DB Schema + Bottleneck Analysis (10 minutes)

### Database Schema

> **You:** "The schema is simple:"

```sql
-- Main URL table
CREATE TABLE urls (
  short_code   CHAR(7)      PRIMARY KEY,
  long_url     TEXT         NOT NULL,
  user_id      BIGINT,
  created_at   TIMESTAMP    DEFAULT NOW(),
  click_count  BIGINT       DEFAULT 0
);

-- Index for reverse lookup (find short code by long URL, avoid duplicates)
CREATE INDEX idx_long_url ON urls (long_url);
```

> **You:** "The primary key is the short_code — that's what we look up on every redirect. We index long_url so we can check if a URL was already shortened and return the existing short code instead of creating a duplicate."

### Bottleneck: Read-Heavy Load

> **You:** "With 115K reads/second, the database alone can't handle this. This is where Redis becomes critical.
>
> Strategy: Redis with LRU eviction. We cache the most recently and frequently accessed short codes. If 90% of traffic hits cached URLs — which is realistic since popular links get most of the traffic — then only 11,500 requests/second actually hit the database. That's very manageable.
>
> Cache key: the short code. Cache value: the long URL. TTL: maybe 24 hours or no expiry (since URLs don't change once created). LRU eviction means if Redis gets full, it drops the least recently used entries — exactly what we want."

```
With 90% cache hit rate:

  115,000 reads/sec total
  ├── 103,500/sec → Redis (fast, in-memory)
  └──  11,500/sec → PostgreSQL (manageable)
```

> **You:** "If Redis goes down, we degrade gracefully — all traffic hits the DB. It'll be slow but not broken. We'd add Redis replication (primary + replica) to reduce that risk."

---

## Final Architecture Diagram

```
                     ┌──────────────────────────────────┐
                     │           bit.ly System           │
                     └──────────────────────────────────┘

  [Client Browser]
        │
        ▼
  [Load Balancer]
        │
        ├─────────────────────────────────────────┐
        ▼                                         ▼
  [API Server 1]                           [API Server 2]
        │                                         │
        │  (reads)                                │
        ▼                                         ▼
  [Redis Cache] ←───────── shared ──────────────→
        │ (miss)
        ▼
  [PostgreSQL DB]  ←──── writes ────── [API Servers]
        │
        ▼
  [Key Generation Service]
  (pre-generates short codes offline,
   buffers available keys in memory)
```

**Summary of decisions:**
- Short codes: KGS with pre-generated base62 keys
- Storage: PostgreSQL (simple relational, 500GB fits fine)
- Read optimization: Redis with LRU, 90%+ hit rate target
- Click counting: async (write to a queue, batch-update DB to avoid hot rows)

---
---

# Walkthrough 2: Design a Rate Limiter

**Difficulty:** Intermediate  
**Time:** ~45 minutes  
**Why this problem:** Tests algorithm knowledge (Token Bucket, sliding window), distributed systems thinking, and fault tolerance reasoning.

---

## Step 1 — Requirements Clarification (5 minutes)

> **Interviewer:** "Design a rate limiter for our API platform."

> **You:** "I'd like to clarify the scope before diving in. A few questions:
>
> Are we rate limiting per user, per IP, per API key, or some combination? And what does 'rate limit' mean here — requests per second, per minute, per hour?"

> **Interviewer:** "Per user, and per minute. Say 100 requests per minute per user."

> **You:** "Is this for a single server or a distributed system — multiple API servers handling requests?"

> **Interviewer:** "Distributed. We have dozens of API servers."

> **You:** "Got it — that's the key complexity. If it were a single server, we could just use in-memory counters. Distributed means we need shared state.
>
> Should the rate limiter be embedded in each API server, or as a separate service like an API Gateway?"

> **Interviewer:** "Good question. Either works — what would you recommend?"

> **You:** "I'd recommend middleware at the API Gateway level. It's a single enforcement point, easier to update rules, and keeps rate limiting logic out of application code. I'll design it that way.
>
> One more: what should happen when a user is rate limited — hard reject with 429, or queue the request?"

> **Interviewer:** "Hard reject with 429 Too Many Requests."

> **You:** "Great. So: distributed rate limiter, per-user per-minute limits, implemented at the API Gateway, hard reject when exceeded. Let me estimate scale and then design."

---

## Step 2 — Scale Estimation (5 minutes)

> **You:** "For scale estimation:
>
> Assume 10 million active users at peak. If each makes requests occasionally, maybe 100,000 requests/second at peak across all users. The rate limiter needs to process 100K decisions per second.
>
> Each rate limit check needs to be sub-millisecond — we can't add 50ms to every API call. So this has to be fast, in-memory lookups.
>
> Storage: for each user, we need to track their request count in a rolling time window. Let's say 16 bytes per user (user_id + count + timestamp). 10M users × 16 bytes = 160 MB. That fits entirely in Redis."

---

## Step 3 — High-Level Design (10 minutes)

> **You:** "Here's the architecture:"

```
Incoming API Request
        │
        ▼
  [ API Gateway ]
        │
        ├──→ [ Rate Limiter Middleware ]
        │           │
        │           ▼
        │     [ Redis Cluster ]
        │     (shared counter state)
        │           │
        │     ┌─────┴──────┐
        │     │ ALLOWED?   │
        │     └─────┬──────┘
        │     YES   │   NO
        │     │     │    │
        │     ▼     │    ▼
        │  forward  │  429 Too Many Requests
        │  to API   │  + Retry-After header
        │  servers  │
        ▼
  [ API Server Fleet ]
```

> **You:** "The rate limiter middleware intercepts every request. It checks Redis to see if the user has exceeded their limit. If yes, it returns 429 immediately. If no, it increments their counter and forwards the request to the API servers.
>
> The key insight is Redis is the single source of truth for all counters. All API gateway instances read and write to the same Redis cluster, so the limit is enforced globally regardless of which gateway server handles the request."

---

## Step 4 — Deep Dive: Token Bucket Algorithm (10 minutes)

> **You:** "There are several rate limiting algorithms. Let me walk through the main ones and explain why Token Bucket is often preferred."

### Algorithm 1: Fixed Window Counter

> **You:** "Simplest approach: divide time into fixed 1-minute windows. Keep a counter per user per window. Reset at the start of each window.
>
> Problem: boundary exploitation. If the window resets at :00, a user can send 100 requests at :59, the window resets, and they send 100 more at :00 — 200 requests in 2 seconds. The 'rate limit' is technically respected but the intent is violated."

```
Fixed Window Problem:

  Window 1 (0:00-1:00)     Window 2 (1:00-2:00)
  ┌─────────────────────┐  ┌─────────────────────┐
  │                ████ │  │ ████                 │
  │           100 reqs  │  │  100 reqs            │
  └─────────────────────┘  └─────────────────────┘
                   ↑ 200 requests in 2 seconds ↑
```

### Algorithm 2: Sliding Window Log

> **You:** "Store a timestamp for every request in the last 60 seconds. On each request, remove timestamps older than 60 seconds, count remaining — if under 100, allow.
>
> Accurate but memory-intensive: you're storing a timestamp for every single request, which is expensive at scale."

### Algorithm 3: Token Bucket (Recommended)

> **You:** "This is what most production systems use, including AWS API Gateway and Stripe.
>
> The concept: each user has a 'bucket' with a maximum capacity (say 100 tokens). Tokens are added at a fixed rate (say 100 per minute). Each request consumes one token. If the bucket is empty, the request is rejected.
>
> This naturally handles bursts: if a user hasn't made requests for a while, they accumulate tokens and can burst. But sustained rates are still capped."

```
Token Bucket State (stored in Redis per user):

{
  "user:123:tokens": 87,         // current tokens remaining
  "user:123:last_refill": 1719400000  // unix timestamp of last refill
}

Pseudocode for each request:

function isAllowed(userId):
  now = currentTimestamp()

  # Fetch current state from Redis atomically (Lua script)
  tokens, lastRefill = redis.hmget(userId, "tokens", "last_refill")

  # Calculate how many tokens to add since last refill
  elapsed = now - lastRefill
  newTokens = elapsed * (100 / 60)  # 100 tokens per 60 seconds
  tokens = min(tokens + newTokens, 100)  # cap at max capacity

  if tokens >= 1:
    tokens -= 1
    redis.hmset(userId, "tokens", tokens, "last_refill", now)
    return ALLOW
  else:
    return DENY
```

> **You:** "The critical word here is 'atomically'. We can't do read-modify-write in three separate Redis commands — that's a race condition. Between two requests reading 1 token and both decrementing, both would think they're allowed.
>
> We use a Redis Lua script, which executes atomically. Or we use Redis transactions (MULTI/EXEC). The Lua script approach is cleaner:"

```lua
-- Lua script (runs atomically in Redis)
local key = KEYS[1]
local now = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])   -- tokens per second
local capacity = tonumber(ARGV[3])

local data = redis.call("HMGET", key, "tokens", "last_refill")
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

local elapsed = now - last_refill
local new_tokens = math.min(capacity, tokens + elapsed * rate)

if new_tokens >= 1 then
  redis.call("HMSET", key, "tokens", new_tokens - 1, "last_refill", now)
  return 1  -- ALLOW
else
  return 0  -- DENY
end
```

---

## Step 5 — Bottleneck: Redis Single Point of Failure (10 minutes)

> **You:** "The obvious concern: Redis is now a critical dependency. If Redis goes down, what happens?"

> **Interviewer:** "Good. What do you do?"

> **You:** "Two sub-problems here: Redis failure, and Redis becoming a bottleneck."

### Problem 1: Redis Bottleneck

> **You:** "At 100K requests/second, a single Redis instance can actually handle this — Redis can do 500K-1M operations/second. But we want headroom and redundancy.
>
> Solution: Redis Cluster. We shard users across multiple Redis nodes. User 123's tokens live on node A, user 456's tokens live on node B. We can add nodes as traffic grows."

```
Redis Cluster (sharded by user_id):

  user_id % 3 == 0  →  Redis Node A
  user_id % 3 == 1  →  Redis Node B
  user_id % 3 == 2  →  Redis Node C

Each node has a replica for high availability.
```

### Problem 2: Redis Failure — Fail Open vs. Fail Closed

> **You:** "This is a classic design debate. If Redis is unreachable when a request comes in, what do we do?
>
> **Fail closed (strict):** Reject the request with 429. Safety-first — we can't verify the limit isn't exceeded, so we deny.
>
> Downside: all users get blocked when Redis is down, even if they're well under their limit. That's a bad user experience and could be catastrophic during a traffic spike when Redis is also under pressure.
>
> **Fail open (permissive):** Allow the request when Redis is unreachable. The service stays available.
>
> Downside: during Redis downtime, unlimited requests go through. A bad actor could exploit a Redis outage to bypass rate limits.
>
> My recommendation: fail open with local fallback. Each API Gateway node keeps a local in-memory counter as a backup. If Redis is down, we rate limit locally. It's not perfectly accurate across servers — a user might get 100 requests on each of 10 gateway nodes — but it prevents complete bypass while keeping the service available."

```
Decision Tree:

  Redis available? ─── YES ──→ Use Redis counter (accurate, global)
         │
         NO
         │
         ▼
  Local fallback:
  - Each gateway node has local Token Bucket in memory
  - Limits are per-node (not global)
  - Alert on-call when Redis is down
  - Acceptable degradation vs. complete failure
```

> **You:** "I'd also add: circuit breaker pattern around Redis. If Redis latency exceeds 10ms (our SLA), we automatically fail open rather than adding latency to every single API request."

---

## Final Architecture Diagram

```
                ┌─────────────────────────────────────────┐
                │         Rate Limiter Architecture         │
                └─────────────────────────────────────────┘

[Client]
   │
   ▼
[API Gateway Cluster]
  ┌──────────────────────────────────────────┐
  │  [Gateway Node 1]  [Gateway Node 2]  ... │
  │        │                  │              │
  │  Rate Limiter         Rate Limiter       │
  │  Middleware           Middleware         │
  │  (Lua Script)         (Lua Script)       │
  └──────────┬───────────────┬──────────────┘
             │               │
             ▼               ▼
        [Redis Cluster]   (sharded, replicated)
        ┌────────────────────────────────┐
        │  Node A  |  Node B  |  Node C  │
        │ (+ replicas for each)          │
        └────────────────────────────────┘
             │
    If Redis down → fallback to local in-memory counter

[API Servers] ← receives only allowed requests
```

**Summary of decisions:**
- Algorithm: Token Bucket (handles bursts, memory-efficient)
- Atomicity: Lua scripts in Redis (no race conditions)
- Distributed state: Redis Cluster sharded by user_id
- Failure mode: Fail open with local per-node fallback counters
- Monitoring: Alert on Redis latency > 10ms, 429 rate spikes

---
---

# Walkthrough 3: Design Twitter/X Feed

**Difficulty:** Advanced  
**Time:** ~45 minutes  
**Why this problem:** Tests understanding of fan-out, large-scale caching, storage layer decisions, and the classic "celebrity problem." Shows you can reason about real-world tradeoffs at 300M DAU scale.

---

## Step 1 — Requirements Clarification (5 minutes)

> **Interviewer:** "Design Twitter's feed."

> **You:** "Before I dive in, let me clarify a few things.
>
> What are the core features? I'm thinking: post a tweet, follow/unfollow users, see a home feed — say the 50 most recent tweets from people you follow. Is that the scope, or do we also need things like likes, retweets, search, notifications?"

> **Interviewer:** "Post tweets, follow users, see home feed. Let's say likes are a nice-to-have."

> **You:** "What's the scale? Twitter is big — any numbers you want me to work with?"

> **Interviewer:** "300 million daily active users."

> **You:** "And I'd guess this is very read-heavy — people scroll their feed far more than they tweet?"

> **Interviewer:** "Yes. Assume 100:1 read-to-write ratio."

> **You:** "A few more: what's the feed sorted by — reverse chronological? Or ranked by an algorithm?"

> **Interviewer:** "Reverse chronological for now. Algorithmic ranking adds complexity we can skip."

> **You:** "And should tweets include media — photos, videos?"

> **Interviewer:** "Yes, assume tweets can have images and video."

> **You:** "Perfect. Let me summarize: 300M DAU, post tweets with optional media, follow users, see 50 most recent tweets from follows in reverse chronological order, 100:1 read:write ratio. Media needs to be handled separately from text. Ready to estimate scale."

---

## Step 2 — Scale Estimation (5 minutes)

> **You:** "Let me work through the numbers.
>
> **Write QPS (tweets):** If 300M DAU and maybe 5% of users post per day, that's 15M tweets/day. 15M / 86,400 seconds ≈ 175 tweets/second. Call it 200/second with peaks.
>
> **Read QPS (feed loads):** 100:1 ratio means 20,000 feed reads/second. But each feed load fetches 50 tweets, so we're actually retrieving 1 million tweet records/second.
>
> **Storage:**
> - Tweets: 200 tweets/sec × 100 bytes (text) × 86,400 seconds × 365 days = ~630 GB/year of text
> - Media: if 20% of tweets have an image (average 1MB), that's 40 images/sec × 1MB = 40MB/sec = ~1.2 PB/year
> - Conclusion: media dominates. We need blob storage (S3), not a traditional database, for media.
>
> **Feed fan-out:** Average Twitter user follows 200 accounts. When someone tweets, we need to push it to all their followers' feeds. Average follower count ≈ 200, so 200 tweets/sec × 200 followers = 40,000 fan-out writes/second. But a celebrity with 10M followers tweeting = 10M writes from a single tweet. That's the 'celebrity problem' we'll need to handle."

---

## Step 3 — High-Level Design (10 minutes)

> **You:** "Let me sketch the major services first, then we'll go deep on the feed generation."

```
High-Level Services:

  [Client App]
       │
       ▼
  [API Gateway / Load Balancer]
       │
  ┌────┴──────────────────────────────────────┐
  │                                           │
  ▼                                           ▼
[Tweet Service]                         [Feed Service]
- POST /tweet                           - GET /feed
- validates, stores tweet               - returns user's home feed
- triggers fan-out                      - reads from feed cache
       │                                       │
       ▼                                       ▼
[Fan-out Service]                       [Feed Cache (Redis)]
- pushes tweet to followers             - pre-computed per-user feed
- handles celebrity exception
       │
       ├──→ [Message Queue (Kafka)]
       │           │
       │           ▼
       │    [Fan-out Workers]
       │
       ▼
[Storage Layer]
  ├── Tweets DB (Cassandra)
  ├── User/Follow Graph (Redis or MySQL)
  └── Media (S3 + CDN)
```

> **You:** "The Tweet Service stores tweets and puts a message on a Kafka queue. Fan-out workers consume from the queue asynchronously and push tweet IDs into followers' feed caches in Redis. The Feed Service just reads from Redis — it's fast because the work was done at write time.
>
> But this is exactly the design decision I want to dig into: is this the right approach? Let's talk about fan-out strategies."

---

## Step 4 — Deep Dive: Fan-out on Write vs. Fan-out on Read (12 minutes)

> **You:** "There are two fundamental approaches to building a user's home feed. This is the most interesting part of the problem."

### Approach 1: Fan-out on Write (Push Model — Pre-compute)

> **You:** "When User A posts a tweet, we immediately push that tweet ID into the feed cache of every one of A's followers.
>
> So when User B opens their feed, we just read B's pre-built feed list from Redis. Super fast reads — O(1) to get the feed."

```
Fan-out on Write:

  User A tweets (100 followers)
       │
       ▼
  Fan-out Worker
       │
  ┌────┴────────────────────────────────────────────┐
  │    push tweet_id to feed of:                    │
  │    User B cache, User C cache, ... (100 writes) │
  └─────────────────────────────────────────────────┘

  User B opens feed:
       │
       ▼
  Read B's pre-built feed from Redis → instant!
```

> **You:** "The problem is celebrities. Elon Musk has 100+ million followers. If he tweets, we're doing 100 million cache writes from one event. That takes minutes and creates a massive write spike."

### Approach 2: Fan-out on Read (Pull Model — Compute on demand)

> **You:** "The opposite approach: when User B opens their feed, we fetch the list of people B follows, get recent tweets from each of them, merge and sort. No pre-computation at write time.
>
> Simple writes — posting a tweet is just one DB insert. But reads are expensive: for each feed load, we're doing N queries where N is the number of people you follow."

```
Fan-out on Read:

  User A tweets:
       │
       ▼
  Store in Tweets DB (one write, done)

  User B opens feed (follows 200 people):
       │
       ▼
  Fetch recent tweets from 200 users → merge → sort → return
  (200 DB reads per feed load = slow)
```

> **You:** "This is slow for users who follow many people, and at 20,000 feed loads/second, it generates 4 million DB reads/second. That's brutal."

### Approach 3: Hybrid (What Twitter Actually Does)

> **You:** "Twitter uses a hybrid model, which is the answer I'd give in an interview.
>
> For regular users (< ~10,000 followers): use fan-out on write. Pre-compute their followers' feeds. Fast reads, manageable write fan-out.
>
> For celebrities (> ~10,000 followers): skip the fan-out. Don't push their tweets to anyone's cache. Instead, when a regular user opens their feed, we pull tweets from the celebrities they follow on-demand, and merge with their pre-computed feed from cache."

```
Hybrid Fan-out:

  Tweet posted by User X:
       │
       ├── if X.follower_count < 10,000:
       │       Fan-out to all followers' caches (push model)
       │
       └── if X.follower_count >= 10,000 (celebrity):
               Just store in Tweets DB. No fan-out.

  User B opens feed:
       │
       ├── Read B's pre-built feed from Redis (covers non-celebrity follows)
       │
       └── For each celebrity B follows:
               Fetch their recent tweets from Tweets DB (pull model)
               Merge with pre-built feed, re-sort by time
               Return top 50
```

> **You:** "This is elegant: the expensive fan-out is avoided for celebrities, and the pull-on-read cost is bounded because the number of celebrities any user follows is typically small (maybe 5-10), so we're only doing 5-10 extra queries, not 200."

> **Interviewer:** "How do you define 'celebrity' — is it a hard threshold?"

> **You:** "Good question. In practice it's a dynamic threshold that can be tuned — Twitter used ~10,000 followers historically. It doesn't have to be binary either; you could fan-out to only the first 1M followers and mark the rest as 'pull on read.' The threshold is a knob to balance write amplification versus read latency."

---

## Step 5 — Deep Dive: Storage Layer (8 minutes)

> **You:** "Let me walk through the storage choices."

### Tweet Storage: Cassandra

> **You:** "Tweets are append-only, never updated (in the base design), and are queried by user ID + recency. This is a perfect use case for Cassandra:
>
> - Write-heavy: Cassandra is optimized for writes (LSM tree storage)
> - Time-series access pattern: we always query 'recent tweets by user'
> - No complex joins: each tweet is self-contained
> - Horizontal scalability: Cassandra shards data automatically
>
> Schema:"

```sql
-- Cassandra Table
CREATE TABLE tweets (
  user_id    UUID,
  tweet_id   TIMEUUID,   -- built-in time-based UUID
  content    TEXT,
  media_urls LIST<TEXT>,
  created_at TIMESTAMP,
  PRIMARY KEY (user_id, tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);
-- Reads: SELECT * FROM tweets WHERE user_id = X LIMIT 50
-- Naturally returns most recent tweets first
```

### User/Follow Graph: Redis or MySQL

> **You:** "The follow relationship — 'who does User B follow?' — needs to be fast because we check it on every fan-out.
>
> For a primary store, MySQL works: a simple edge table (follower_id, followee_id). But for the fan-out workers that need to enumerate all followers of a user instantly, we cache the follower list in Redis as a Set.
>
> Redis Set: key = `followers:{user_id}`, value = Set of follower IDs. Fast O(1) lookup and O(N) enumeration."

### Media: S3 + CDN

> **You:** "Images and video are stored in S3 — it's cheap, durable, and scales infinitely. But S3 alone would be too slow for global users.
>
> We put a CDN (CloudFront or Fastly) in front of S3. When a user in Tokyo views a tweet with an image, the image is served from a Tokyo edge node, not from our US-East S3 bucket. This reduces latency from 150ms to <10ms and massively reduces our S3 egress costs."

```
Media Flow:

  Upload:
  Client → API Server → S3 (store blob)
                      → Return CDN URL (e.g., cdn.twitter.com/img/abc123.jpg)
                      → Store CDN URL in tweet record

  Read:
  Client requests cdn.twitter.com/img/abc123.jpg
     → CDN Edge Node (cache HIT?) → serve directly
     → CDN Edge Node (cache MISS?) → fetch from S3, cache, serve
```

---

## Bottleneck Analysis (5 minutes)

> **You:** "Let me flag the remaining bottlenecks."

### Bottleneck 1: The Celebrity Problem at Write Time

> **You:** "Already addressed with the hybrid fan-out. Celebrity tweets are pulled on read. The remaining risk: what if a celebrity tweets during a traffic spike and millions of users open the app simultaneously, all pulling that tweet from Cassandra?
>
> Solution: cache the celebrity's recent tweets in Redis with a short TTL (30 seconds). The first request populates it, all subsequent requests within 30 seconds hit cache. This bounds the Cassandra load."

### Bottleneck 2: Hot Partitions in Cassandra

> **You:** "If a celebrity's user_id is a Cassandra partition key, all reads and writes for that user land on the same Cassandra node — a hot partition.
>
> Solution: salt the partition key. Append a shard number: `(user_id, shard_number)` where shard_number = tweet_id % 10. Now that user's data is spread across 10 partitions on 10 nodes. Reads query all 10 and merge, but the load is distributed."

### Bottleneck 3: CDN for Media

> **You:** "Already addressed. S3 alone at 1.2 PB/year with global users is untenable without a CDN. CDN cache hit rates for popular media can be 99%+, meaning almost no traffic ever reaches S3."

---

## Final Architecture Diagram

```
                    ┌───────────────────────────────────────────┐
                    │           Twitter Feed System              │
                    └───────────────────────────────────────────┘

[Mobile / Web Client]
        │
        ▼
[API Gateway + Load Balancer]
        │
   ┌────┼──────────────────────────────────────┐
   │    │                                      │
   ▼    ▼                                      ▼
[Tweet      [Feed Service]            [User Service]
 Service]   GET /feed                 follow/unfollow
POST /tweet │                               │
   │        │ read from                     │
   │        ▼                               ▼
   │   [Redis Feed Cache]          [MySQL: follow graph]
   │   per-user feed list          [Redis: follower sets]
   │   (tweet IDs only)
   │
   ├──→ [Kafka Queue]
   │        │
   │        ▼
   │   [Fan-out Workers]
   │   - if regular user: push to followers' Redis caches
   │   - if celebrity: skip (pull on read)
   │
   ▼
[Cassandra: Tweets DB]
   (sharded by user_id+shard, clustered by time)

[Media Path]
Client upload → API Server → [S3 (origin)]
                               ↑
Client read ←── [CDN Edge] ────┘
                (99% cache hit rate)
```

**Summary of decisions:**

| Decision | Choice | Why |
|---|---|---|
| Feed generation | Hybrid fan-out | Avoids celebrity write amplification |
| Tweet storage | Cassandra | Write-heavy, time-series, horizontally scalable |
| Feed cache | Redis (per-user list of tweet IDs) | O(1) feed reads for pre-computed feeds |
| Follow graph | MySQL (source of truth) + Redis Sets (cache) | Fast fan-out enumeration |
| Media | S3 + CDN | Cheap durable storage, global low-latency reads |
| Celebrity tweets | Pull on read + short Redis TTL | Bounds Cassandra load without 100M fan-out |

---
---

# Cross-Walkthrough Patterns

These three walkthroughs all follow the same 5-step framework. Notice the repeated structure:

```
Step 1: Requirements (5 min)
  └── Always ask: features, scale, constraints, edge cases

Step 2: Estimation (5 min)
  └── Always derive: QPS (read + write), storage, bandwidth
  └── Always identify: is this read-heavy or write-heavy?

Step 3: High-Level Design (10 min)
  └── Draw boxes and arrows. Client → LB → App Server → DB
  └── Separate read path from write path if they differ
  └── Name the services and their responsibilities

Step 4: Deep Dive (15 min)
  └── Go deep on the hardest part of the problem
  └── URL Shortener: ID generation
  └── Rate Limiter: Token Bucket algorithm + atomicity
  └── Twitter: Fan-out strategy (push vs pull vs hybrid)

Step 5: Bottlenecks (10 min)
  └── Proactively identify what breaks at scale
  └── Cache hit rates, hot partitions, single points of failure
  └── Show you know how to make it production-grade
```

**Common themes across all three:**
- Redis appears in every system — as a cache, as a rate limiting store, as a feed store
- Kafka appears when you need async decoupling of write from downstream effects
- CDNs appear whenever media or static content is involved
- The "fail open vs fail closed" debate matters whenever a dependency can go down

These are the building blocks. Mix and match them for any system design problem.
