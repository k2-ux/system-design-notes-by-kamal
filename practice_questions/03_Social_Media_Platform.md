# Design a Social Media Platform like Instagram
> Complete interview preparation guide — Q&A format, beginner to advanced.

---

## Table of Contents
1. [What Is It?](#what-is-it)
2. [Step 1 — Requirements](#step-1--requirements)
3. [Step 2 — Capacity Estimation](#step-2--capacity-estimation)
4. [Step 3 — High-Level Design](#step-3--high-level-design)
5. [Step 4 — The Hard Problem: News Feed Generation](#step-4--the-hard-problem-news-feed-generation)
6. [Step 5 — Database Design](#step-5--database-design)
7. [Step 6 — Media Storage and Durability](#step-6--media-storage-and-durability)
8. [Step 7 — Failure Scenarios](#step-7--failure-scenarios)
9. [Full System Diagram](#full-system-diagram)
10. [Quick Revision](#quick-revision)

---

## What Is It?

**Q: What makes Instagram architecturally interesting compared to simpler systems?**

> Two things make Instagram hard:
> 1. **Extreme read dominance** (1000:1) — people scroll far more than they post, so reads must be near-instant at massive scale
> 2. **News feed generation** — showing each user a personalized, sorted feed from hundreds of people they follow, in milliseconds, for 500M users simultaneously, is a genuinely hard distributed systems problem

---

## Step 1 — Requirements

**Q: What are the most important clarifying questions for Instagram?**

> 1. "What is the core feature scope — photo/video sharing, Stories, Reels, DMs?" (narrow scope)
> 2. "How many users and what is the read/write ratio?" (determines architecture tier)
> 3. "Does the news feed need to be real-time or is a slight delay acceptable?" (determines feed architecture)

**Q: What are the functional requirements?**

> 1. Users can upload and share photos and videos
> 2. Users can follow and unfollow other users
> 3. Users can like and comment on posts
> 4. Generate and display a personalized news feed — posts from people the user follows
> 5. Support tagging other users in posts and comments

**Q: What are the non-functional requirements?**

> - **High availability** — 99.9% uptime
> - **Low latency** — news feed must load in milliseconds
> - **High scalability** — 500M daily active users, millions concurrent
> - **High durability** — uploaded photos and videos must NEVER be lost
> - **Eventual consistency** — if a user doesn't see a new post for a few seconds, that's acceptable

**Q: Is Instagram read-heavy or write-heavy? Why does this matter?**

> Extremely **read-heavy** — approximately **1000:1** reads to writes. People scroll feeds constantly but post occasionally.
>
> This directly drives two architecture decisions:
> 1. **Redis cache** — absorb the massive read traffic without hitting the DB
> 2. **CDN** — serve photos/videos from edge nodes near each user (Instagram is an IMAGE platform — media is the majority of traffic)

---

## Step 2 — Capacity Estimation

**Q: Walk through capacity estimation for Instagram.**

> **Assumptions:**
> - 500M daily active users
> - Average user posts 1 photo every 2 days → 250M photo uploads/day
> - Average user views 100 posts/day (feed scrolling)
>
> **Traffic:**
> ```
> Writes: 250M / 86,400 ≈ 2,900 uploads/second
> Reads:  500M × 100 / 86,400 ≈ 578,000 reads/second
> ```
>
> **Storage:**
> ```
> Average photo = 3 MB (after compression)
> 250M photos/day × 3 MB = 750 TB/day of new media
> → S3 object storage is the ONLY viable option
> → CDN sits in front of S3 to serve photos fast globally
> ```

**Q: What does 578,000 reads/second immediately tell you?**

> The database cannot handle this directly. 578,000 reads/second means:
> - Cache (Redis) must absorb 99%+ of read traffic
> - CDN must serve all media without touching origin servers
> - DB is only hit on true cache misses

---

## Step 3 — High-Level Design

**Q: What are the two most important components for a 1000:1 read-heavy image platform?**

> 1. **Redis cache** — stores pre-computed feeds, post metadata, like counts. Hot data served in microseconds.
> 2. **CDN** — serves all photos and videos from edge nodes close to each user. Without CDN, 500M users downloading from one origin = instant failure.
>
> Note: Sharding handles storage scale, but cache and CDN specifically handle READ dominance.

**Q: What does the high-level architecture look like?**

> ```
> [Users]
>     ↓
> [Load Balancer]
>     ↓
> [App Servers]
>  │        │          │
> [S3+CDN] [SQL DB]  [Cassandra]
> (photos)  (users,   (posts, comments,
>            follows)  like counts)
>     ↓
> [Feed Service]
>  │        │
> [Redis]  [Kafka]
> (feeds,  (fanout
>  counts)  events)
> ```

---

## Step 4 — The Hard Problem: News Feed Generation

**Q: What is the naive approach to generating a news feed and why does it fail at scale?**

> ```
> User opens feed → query DB:
> "Get all posts from all 500 people I follow, sorted by time"
> → 500 queries or one massive JOIN
> → 578,000 users doing this simultaneously
> → 289 BILLION database operations per second
> → Every database on Earth dies 💀
> ```
>
> Computing the feed fresh from the DB on every open does not scale.

**Q: What are the two approaches to feed generation?**

> **Option A: Fanout on Write (Push model)**
> ```
> User posts photo
>     ↓
> System immediately writes that post_id into
> the pre-computed feed list of ALL followers
>     ↓
> Follower opens app → feed already ready ⚡ (just read from Redis)
> ```
>
> **Option B: Fanout on Read (Pull model)**
> ```
> User posts photo → just stored in DB
>     ↓
> Follower opens app → NOW pull posts from all followed accounts
> → compute and sort feed on demand
> ```

**Q: What is the Celebrity Problem and why does it break Fanout on Write?**

> ```
> Ronaldo (500M followers) posts ONE photo
>     ↓
> System must write to 500M feed lists simultaneously
>     ↓
> 500,000,000 write operations in seconds
>     ↓
> Servers, queues, and databases catch fire 🔥
> ```
> Fanout on Write is catastrophic for accounts with massive follower counts.

**Q: What is Instagram's actual solution?**

> **Hybrid approach — based on follower count:**
> ```
> Regular users (< 1M followers):  Fanout on WRITE
> → Pre-compute feed for all followers immediately via Kafka
> → Fast feed load — Redis already has the list
>
> Celebrities (Ronaldo, Messi):    Fanout on READ
> → Don't pre-compute — too expensive
> → When you open your feed, their latest posts are
>   pulled and merged in real-time
> ```
>
> **Feed load flow:**
> ```
> [Pre-computed feed from Redis]  +  [Live pull from 3 celebrities you follow]
>         ↓                                       ↓
>   Already ready ⚡                    Small targeted real-time query
>                     MERGE → show final feed
> ```
> Fast for everyone. No celebrity meltdown.

**Q: Where are pre-computed feed lists stored — Redis or Cassandra?**

> **Redis.**
>
> A pre-computed feed is just a list of post IDs:
> ```
> Redis key:   "feed:user_123"
> Redis value: [post_456, post_789, post_234, post_901...]
> ```
> It is small, ephemeral (regeneratable if lost), and must be read in microseconds.
>
> **Why not Cassandra?**
> - Cassandra hits disk — millisecond reads
> - Redis is in-memory — microsecond reads
> - Feed lists are temporary; if Redis loses them, just recompute
> - Cassandra is for durable long-term data that must never be lost
>
> **Rule:** Pre-computed, ephemeral, speed-critical data → Redis. Permanent, durable data → Cassandra/SQL.

---

## Step 5 — Database Design

**Q: What data needs to be stored and in which database?**

> ```
> Users table (PostgreSQL):
> ┌────────┬──────────┬───────────────┬──────┬─────────────────┐
> │ id     │ username │ email         │ bio  │ profile_pic_url │
> └────────┴──────────┴───────────────┴──────┴─────────────────┘
>
> Posts table (Cassandra):
> ┌────────┬─────────┬───────────────────────┬───────────┬─────────────┐
> │ id     │ user_id │ image_url             │ caption   │ created_at  │
> └────────┴─────────┴───────────────────────┴───────────┴─────────────┘
>
> Follows table (PostgreSQL):
> ┌─────────────┬─────────────┐
> │ follower_id │ followee_id │   ← INDEX on both columns
> └─────────────┴─────────────┘
>
> Likes table (Cassandra):
> ┌─────────┬─────────┬─────────────┐
> │ post_id │ user_id │ created_at  │
> └─────────┴─────────┴─────────────┘
>
> Comments table (Cassandra):
> ┌────────┬─────────┬─────────┬──────────┬─────────────┐
> │ id     │ post_id │ user_id │ text     │ created_at  │
> └────────┴─────────┴─────────┴──────────┴─────────────┘
> ```

**Q: The Follows table is a many-to-many relationship — should you use a Graph database?**

> No — SQL is sufficient here. The queries are simple:
> - "Who does Kamal follow?" → `SELECT followee_id WHERE follower_id = Kamal`
> - "Who follows Ronaldo?" → `SELECT follower_id WHERE followee_id = Ronaldo`
>
> Two indexes handle both queries perfectly.
>
> **Graph DB** shines for multi-hop traversal: "find friends of friends of friends." Instagram doesn't need that — it's just one hop.

---

## Step 6 — Media Storage and Durability

**Q: Should photos be stored in the database?**

> **Never.** Average photo = 3 MB. 250M photos/day = 750 TB/day. Databases are not built for blob storage at this scale — it would destroy performance and cost a fortune.

**Q: How does Instagram store photos and ensure they are never lost?**

> **S3 (object storage) + geo-replication:**
> ```
> User uploads photo
>     ↓
> S3 stores it AND automatically copies to multiple data centers:
>     ↓
> [US East] ──copy──► [EU West]
>           ──copy──► [Asia Pacific]
>
> US East datacenter burns down 🔥
> → Photo still exists in EU West and Asia Pacific
> → Zero data loss
> ```
> Amazon guarantees **99.999999999% (11 nines) durability** on S3. You get this for free — you don't build it yourself.
>
> **CDN sits in front of S3:**
> ```
> User in Karachi requests photo
>     ↓
> CDN checks nearest edge node (e.g. Dubai)
>     ↓
> Cache HIT  → served from Dubai in milliseconds ⚡
> Cache MISS → fetched from S3, cached at Dubai for next user
> ```

---

## Step 7 — Failure Scenarios

**Q: Redis (feed cache) goes down. What happens?**

> Feed requests fall through to recompute from DB — slower but functional.
> **Prevention:** Run Redis Cluster (3+ nodes). One node down → consistent hashing routes its keys to other nodes.

**Q: Kafka goes down. What happens?**

> New posts are still saved to DB and S3. But feed fanout stops — followers' pre-computed feeds stop updating. Feeds become stale.
> **Prevention:** Kafka runs as a cluster (3+ brokers). One broker failure doesn't affect the cluster.

**Q: How do you handle a suddenly viral post (millions of likes in minutes)?**

> Like counts stored in Redis as a counter:
> ```
> INCR "likes:post_456"  → atomic increment, no DB write per like
> ```
> Periodically flush Redis counters to DB in batches. This handles millions of likes/second without DB pressure.

**Q: How do you scale the feed system to 10x users?**

> ```
> App Servers       → add more instances (horizontal scaling)
> Redis             → Redis Cluster with more nodes
> Kafka             → add more partitions per topic
> Cassandra         → add more nodes (auto-rebalances)
> S3 + CDN          → already infinitely scalable
> Feed Service      → add more consumer instances reading from Kafka
> ```

---

## Full System Diagram

```
[Users]
    ↓
[Load Balancer]
    ↓
[App Servers]
 │         │              │
 ↓         ↓              ↓
[S3]    [PostgreSQL]   [Cassandra]
[CDN]   (users,        (posts, likes,
         follows)       comments)
             ↓
         [Kafka]
             ↓
       [Feed Service]
             ↓
         [Redis]
    (pre-computed feeds,
     like counts,
     online status)
```

**Request flows:**
```
UPLOAD POST:
User → App Server → save photo to S3 → save metadata to Cassandra
     → publish "new_post" event to Kafka
     → Feed Service: fanout post_id to all followers' Redis feed lists
       (skip celebrities — use fanout on read instead)

LOAD FEED:
User → App Server → fetch feed list from Redis ⚡
     → fetch post metadata from Cassandra (or Redis if cached)
     → fetch photos from CDN
     → merge celebrity posts (real-time pull)
     → return ranked feed

LIKE POST:
User → App Server → INCR Redis counter "likes:post_id"
     → async batch write to Cassandra periodically
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| Read optimization | Redis cache + CDN | 1000:1 read ratio — DB cannot handle direct reads |
| Photo storage | S3 + CDN | 750 TB/day — databases not built for blob storage |
| Photo durability | S3 geo-replication | 11 nines durability across multiple datacenters |
| Message storage | Cassandra | High write volume, time-ordered queries, no JOINs |
| User/follow data | PostgreSQL | Relational data with complex queries |
| Feed generation | Hybrid fanout | Write for regular users, read for celebrities |
| Pre-computed feeds | Redis | Ephemeral, in-memory, microsecond reads |
| Feed distribution | Kafka | Async fanout to millions of followers |
| Like counts | Redis counters + batch flush | Atomic increments without DB write per like |
| Follow relationship | SQL with two indexes | Simple one-hop queries, no graph traversal needed |

### Three things to mention proactively in an interview
> 1. **The Celebrity Problem** — shows you understand fanout on write breaks for large accounts
> 2. **Hybrid feed approach** — shows you know the practical solution used in production
> 3. **Redis counters for likes** — shows you know how to handle high-frequency write operations without DB pressure
