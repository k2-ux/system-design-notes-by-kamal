# System Design Interview Strategy
> A progressive Q&A revision guide — from beginner to advanced. Use this to drill interview readiness.

---

## Table of Contents
1. [The 7-Step Framework](#the-7-step-framework)
2. [Beginner Q&A — Framework Basics](#beginner-qa--framework-basics)
3. [Intermediate Q&A — Applying the Framework](#intermediate-qa--applying-the-framework)
4. [Advanced Q&A — Deep Dives & Tricky Questions](#advanced-qa--deep-dives--tricky-questions)
5. [The Cheat Sheet — Quick Reference](#the-cheat-sheet--quick-reference)

---

## The 7-Step Framework

Use this for EVERY system design question, regardless of what they ask.

```
Step 1: Clarify Requirements        → Ask before you draw ANYTHING
Step 2: Capacity Estimation         → Get the numbers, justify your choices
Step 3: High-Level Design           → Block diagram of major components
Step 4: Database Design             → Schema, storage choice, indexing
Step 5: API Design                  → Endpoints, protocols, communication
Step 6: Dive Deep into Components   → Focus where the interviewer guides you
Step 7: Address Key Concerns        → Scalability, reliability, failure modes
```

> The interviewer evaluates your **reasoning process**, not just your final answer. Think out loud at every step.

---

## Beginner Q&A — Framework Basics

*These are questions to make sure you understand what each step means before you walk into an interview.*

---

**Q: Why do you clarify requirements before designing anything?**

> Because the same system can mean completely different things depending on scale, features, and constraints. "Design a chat app" for 100 users is a monolith. For 2 billion users it's a distributed microservices system with WebSockets, sharding, and CDNs. Without requirements, you're designing the wrong thing.

---

**Q: What is the difference between functional and non-functional requirements?**

> **Functional** = WHAT the system does. Features a user can see and use.
> Examples: "users can send messages", "users can upload photos", "generate a short URL"
>
> **Non-Functional** = HOW WELL the system performs. Qualities the user feels.
> Examples: latency, availability (uptime), scalability, consistency, durability

---

**Q: What are the 5 most important non-functional requirements to ask about?**

> 1. **Scale** — how many users/requests per second?
> 2. **Availability** — can the system have downtime? (99.9% vs 99.999%)
> 3. **Latency** — are there real-time requirements?
> 4. **Consistency** — is stale data acceptable?
> 5. **Read/Write ratio** — is it read-heavy or write-heavy?

---

**Q: What is capacity estimation and why does it matter?**

> Capacity estimation is calculating the approximate scale of the system before designing it — number of users, requests per second, storage needed, and bandwidth required.
>
> It matters because **the numbers justify your architecture decisions**. Without them, you're guessing. With them, you can say: "58,000 messages/sec means we need horizontal scaling and sharding — one server can't handle that."

---

**Q: What 5 things should you estimate during capacity estimation?**

> 1. **Users** — daily/monthly active users, peak concurrent users
> 2. **Traffic** — read/write requests per second
> 3. **Storage** — data generated per day/year
> 4. **Memory** — cache size needed for hot data
> 5. **Bandwidth** — network throughput at peak load

---

**Q: What goes in a high-level design block diagram?**

> The major system components:
> ```
> [Clients] → [Load Balancer] → [App Servers] → [Cache]
>                                     ↓               ↓ miss
>                              [Message Queue]   [Database]
>                                     ↓               ↓
>                             [Worker Services] [Object Storage]
> ```
> Components to consider: clients, load balancer, app servers, cache, database, object storage (S3), message queues, CDN, worker services, notification service.

---

**Q: What is the difference between SQL and NoSQL and when do you use each?**

> **SQL** — structured data, predefined schema, ACID transactions, complex queries with JOINs.
> Use when: data has clear relationships, transactions matter (banking, orders), data structure is stable.
> Examples: MySQL, PostgreSQL
>
> **NoSQL** — flexible schema, horizontally scalable, eventual consistency.
> Use when: data is unstructured or semi-structured, massive scale, high write throughput, schema changes frequently.
> Examples: MongoDB (document), Cassandra (wide-column), Redis (key-value)

---

**Q: What is an API and what are the three main API styles?**

> An API defines how components of a system communicate with each other and with external clients.
>
> Three styles:
> 1. **REST** — resource-based, uses HTTP methods (GET/POST/PUT/DELETE), returns JSON. Best for public APIs.
> 2. **RPC/gRPC** — action-based, calls remote functions directly. Best for internal microservice communication.
> 3. **GraphQL** — client specifies exactly what data it needs. Best when clients have varied data needs.

---

## Intermediate Q&A — Applying the Framework

*These questions test whether you can apply concepts to real scenarios.*

---

**Q: You're designing Twitter. At Step 1, what are your 3 most important clarifying questions?**

> 1. "Should we support both posting tweets AND reading the home feed?" (defines core features)
> 2. "What scale are we targeting — millions or billions of users?" (defines architecture tier)
> 3. "Is the feed real-time (show tweets instantly) or is a slight delay acceptable?" (defines consistency requirement — AP system, eventual consistency is fine)

---

**Q: You're told 100M daily active users each send 10 tweets/day. Walk me through a quick capacity estimation.**

> ```
> Writes: 100M users × 10 tweets/day = 1 billion tweets/day
>       = 1,000,000,000 / 86,400 seconds ≈ 11,500 writes/second
>
> Reads: Twitter is ~100x read-heavy vs writes
>       = ~1,150,000 reads/second
>
> Storage: avg tweet = 300 bytes
>        = 1B × 300 bytes = 300GB/day new data
>        = ~110TB/year
> ```
> **Conclusion:** This needs horizontal scaling, read replicas, aggressive caching, and sharding. A single server handles maybe 10,000 QPS — we're at 1.15M reads/sec.

---

**Q: For a read-heavy system like Twitter, what component do you add to handle the read load and why?**

> A **cache layer** (Redis or Memcached) between the app servers and the database.
>
> Home feed data (tweets from people you follow) rarely changes second-to-second. Cache the pre-computed feed per user. A cache hit returns data in microseconds without touching the database. At 1.15M reads/second, this saves your database from being hammered.

---

**Q: How would you design the database schema for a URL shortener?**

> ```
> URLs table:
> ┌──────────┬──────────────────────┬─────────────┬────────────┐
> │ short_id │ long_url             │ created_at  │ expires_at │
> ├──────────┼──────────────────────┼─────────────┼────────────┤
> │ abc123   │ https://long-url...  │ 2024-01-01  │ 2025-01-01 │
> └──────────┴──────────────────────┴─────────────┴────────────┘
>
> Index on: short_id (every redirect lookup hits this column)
> ```
> Short IDs generated with **Base62 encoding** of an auto-incrementing ID (a-z, A-Z, 0-9 = 62 chars, 6 chars = 56 billion combinations).

---

**Q: A service goes down. How do you design for this in Step 7?**

> 1. **Eliminate single points of failure** — replicate every critical component (multiple load balancers, DB primary + replicas, multiple app servers)
> 2. **Circuit breakers** — if Service A keeps failing, stop calling it and return a fallback response rather than cascading failures across the system
> 3. **Retry with exponential backoff** — retry failed requests but wait 1s, 2s, 4s, 8s... to avoid hammering a recovering service
> 4. **Graceful degradation** — if recommendations service is down, show popular content instead of crashing the entire page
> 5. **Health checks + auto-recovery** — heartbeats detect failures, load balancer stops routing to dead servers automatically

---

**Q: When should you use a message queue in your design?**

> Use a message queue when:
> - **Decoupling** — producer and consumer don't need to be online simultaneously
> - **Traffic spikes** — buffer requests during peak load instead of overwhelming downstream services (Black Friday orders)
> - **Async processing** — tasks that don't need an immediate response (send email, resize image, process payment)
> - **Fan-out** — one event needs to trigger multiple independent services (order placed → charge card + update inventory + send confirmation)

---

**Q: Your system needs to handle real-time chat. What communication protocol do you use and why?**

> **WebSockets** — establish a persistent, bidirectional connection between client and server.
>
> Regular HTTP is request-response: client asks, server answers. For chat, the server needs to PUSH messages to clients without them asking. WebSockets keep one connection open permanently, allowing instant message delivery in both directions.
>
> Long Polling (client repeatedly asks "any new messages?") is the fallback if WebSockets aren't supported, but it creates much higher server load.

---

**Q: How do you handle database scaling when a single database can't handle the load?**

> Three progressive approaches:
>
> **Step 1: Read Replicas** — add replica databases that serve all read traffic. Primary handles writes only. Works until write volume exceeds primary capacity.
>
> **Step 2: Caching** — add Redis in front of the database. Cache hot data. Reduces DB reads by 90%+.
>
> **Step 3: Sharding** — split the database horizontally. User IDs 1-1M on shard 1, 1M-2M on shard 2, etc. Each shard is an independent database. Last resort — adds enormous complexity (no cross-shard JOINs, re-sharding is painful).

---

## Advanced Q&A — Deep Dives & Tricky Questions

*These are the questions that separate good candidates from great ones.*

---

**Q: What is the CAP theorem and how does it affect your design decisions?**

> CAP theorem states that in a distributed system, you can only guarantee TWO of three properties simultaneously:
> - **C**onsistency — every read gets the most recent write
> - **A**vailability — every request gets a response (not necessarily the latest data)
> - **P**artition Tolerance — system continues operating when network partitions occur
>
> Partition tolerance is non-negotiable in any real distributed system — networks DO partition. So the real choice is **CP vs AP**:
>
> | Choose CP | Choose AP |
> |-----------|-----------|
> | Banking (can't show stale balance) | Social media feeds |
> | Inventory (can't oversell) | DNS resolution |
> | Medical records | Shopping recommendations |
>
> In interviews, state your choice and justify it based on the system's tolerance for stale data.

---

**Q: Explain consistent hashing and why you'd use it over simple modulo hashing.**

> **Modulo hashing:** `server = hash(key) % N` where N = number of servers. Problem: when you add/remove a server, N changes, and almost every key remaps to a different server. In a cache cluster this means a cache miss storm — all cached data is effectively lost.
>
> **Consistent hashing:** Place both servers and keys on a circular hash ring. Each key is served by the first server clockwise from it. When a server is added/removed, only ~1/N of keys need remapping.
>
> Use it for: cache clusters, distributed databases, load balancing with session affinity.

---

**Q: How would you handle the "thundering herd" problem in a cache?**

> The thundering herd happens when a popular cached item expires. Thousands of requests simultaneously miss the cache and hammer the database at once.
>
> Solutions:
> 1. **Cache lock** — only the first request fetches from DB and repopulates cache. Other requests wait.
> 2. **Probabilistic early expiration** — slightly before TTL expires, start refreshing proactively rather than waiting for expiry.
> 3. **Staggered TTLs** — add random jitter to TTL values so not everything expires at the same second.
> 4. **Write-through cache** — data is always in cache (written on every DB write), so it never expires unexpectedly.

---

**Q: Design a rate limiter. What algorithm would you use and why?**

> For most production systems: **Token Bucket algorithm**.
>
> ```
> Each user has a bucket with capacity N tokens.
> Tokens refill at rate R per second.
> Each request consumes 1 token.
> If bucket is empty → request is rejected (429 Too Many Requests).
> ```
>
> Stored in Redis with atomic operations:
> ```
> key = "ratelimit:{user_id}"
> value = {tokens: 95, last_refill: timestamp}
> ```
>
> **Why Token Bucket over others:**
> - Allows controlled bursts (user can use saved-up tokens)
> - Smooth average rate (tokens refill gradually)
> - Memory efficient (one Redis entry per user)
> - Leaky Bucket for strict constant-rate processing (payment processors)
> - Fixed/Sliding Window for simpler quota enforcement (API plans)

---

**Q: How do you ensure exactly-once delivery in a message queue system?**

> This is one of the hardest problems in distributed systems. Three delivery guarantees:
>
> - **At-most-once** — fire and forget. Message may be lost. (logs, metrics)
> - **At-least-once** — retry until acknowledged. Message may be duplicated. (most common)
> - **Exactly-once** — hardest. Requires idempotency keys.
>
> **Practical solution:** Use **at-least-once delivery + idempotent consumers**.
>
> Each message has a unique ID. Consumer tracks processed IDs. If a duplicate arrives, it's a no-op.
> ```
> if message.id in processed_ids:
>     return  # already handled
> process(message)
> processed_ids.add(message.id)
> ```
> This is how payment systems handle duplicate charge attempts.

---

**Q: Your database is becoming a bottleneck. Walk me through every option in order of complexity.**

> In order from simplest to most complex:
>
> 1. **Add indexes** — index frequently queried columns. O(n) → O(log n). Free performance.
> 2. **Query optimization** — rewrite slow queries, avoid N+1 problems, use EXPLAIN.
> 3. **Add caching** — Redis in front of DB. Eliminate repeated reads. 90%+ hit rate achievable.
> 4. **Read replicas** — route all reads to replica DBs, writes to primary only.
> 5. **Vertical scaling** — give DB server more RAM/CPU. Temporary fix, has ceiling.
> 6. **Denormalization** — eliminate expensive JOINs by duplicating data.
> 7. **Sharding** — split data across multiple DB servers. Last resort. Massive complexity.

---

**Q: How would you design for global availability with low latency?**

> Three-layer approach:
>
> 1. **CDN** — static content (images, videos, JS, CSS) served from edge nodes close to users. Karachi user gets content from a nearby CDN node, not a US server.
>
> 2. **Multi-region deployment** — deploy your entire stack (app servers + databases) in multiple geographic regions (US-East, EU-West, Asia-Pacific). Route users to nearest region.
>
> 3. **Data replication strategy** — choose between:
>    - **Active-Active** — all regions serve reads AND writes, sync via replication. Risk: write conflicts.
>    - **Active-Passive** — one primary region handles writes, others replicate and serve reads.
>
> Trade-off: multi-region adds massive operational complexity. Justify it only when latency requirements demand it.

---

**Q: An interviewer asks "how would you handle 10x the traffic you just designed for?" What do you say?**

> Walk through the bottlenecks layer by layer:
>
> **App servers:** Already horizontally scaled. Just add more servers behind the load balancer. Auto-scaling handles this automatically.
>
> **Cache:** Add more Redis nodes. Use consistent hashing so existing cached data doesn't all invalidate.
>
> **Database reads:** Add more read replicas. Cache hit rate should absorb most of the extra read traffic first.
>
> **Database writes:** This is the hard part. Options: optimize write path, batch writes, introduce write-ahead queue, or eventually shard.
>
> **Message queues:** Add more consumer workers to drain queues faster.
>
> **CDN:** Already globally distributed — inherently handles traffic spikes without changes.
>
> Key insight: **most systems are read-heavy**. 10x traffic usually means 10x reads which caching handles well. The write path is where the real scaling challenge lives.

---

## The Cheat Sheet — Quick Reference

### Before You Start (Step 1 questions to always ask)

| Question | Why It Matters |
|----------|---------------|
| How many users / requests per second? | Determines if you need distributed architecture |
| Read-heavy or write-heavy? | Determines where to optimize (cache vs write path) |
| Strong or eventual consistency needed? | CP vs AP — affects DB and replication choices |
| Highly available (no downtime tolerated)? | Determines redundancy requirements |
| Global users or single region? | Determines CDN and multi-region needs |

### Component Selection Quick-Guide

| Need | Component | Why |
|------|-----------|-----|
| Distribute traffic | Load Balancer | Horizontal scaling, no single point of failure |
| Fast reads | Cache (Redis) | Microsecond reads vs milliseconds from DB |
| Async tasks | Message Queue (Kafka/SQS) | Decouple producers from consumers |
| Serve static content fast globally | CDN | Edge nodes near users |
| Store files/images/video | Object Storage (S3) | Cheap, scalable, durable |
| Real-time push to clients | WebSockets | Persistent bidirectional connection |
| Microservices find each other | Service Discovery (Consul) | Dynamic IP resolution |
| Fast "have I seen this?" check | Bloom Filter | Probabilistic, memory-efficient |
| Spread state across 1000 nodes | Gossip Protocol | No master, decentralized |

### Failure Mode Checklist (Step 7)

- [ ] Is there a single point of failure? (replicate it)
- [ ] What happens when the database goes down? (read replicas, failover)
- [ ] What happens during a traffic spike? (auto-scaling, queue buffering)
- [ ] What happens when a downstream service is slow/down? (circuit breaker, timeout, fallback)
- [ ] How do you detect failures? (heartbeats, health checks, monitoring)
- [ ] How do you recover data if storage fails? (replication, backups, checksums)

### Numbers Every Engineer Should Know

| Operation | Approximate Latency |
|-----------|-------------------|
| L1 cache read | 0.5 ns |
| RAM read | 100 ns |
| SSD read | 100 µs |
| HDD read | 10 ms |
| Network round trip (same DC) | 0.5 ms |
| Network round trip (cross-continent) | 150 ms |
| Redis cache read | ~1 ms |
| DB query (indexed, warm) | ~5 ms |

| Unit | Value |
|------|-------|
| 1 KB | 1,000 bytes |
| 1 MB | 1,000 KB |
| 1 GB | 1,000 MB |
| 1 TB | 1,000 GB |
| Seconds in a day | ~86,400 |
| Seconds in a year | ~31.5 million |
