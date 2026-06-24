# System Design Fundamentals
> Complete study notes from the System Design Interview Handbook
> Covers all 17 core concepts with examples, analogies, and interview tips

---

## Table of Contents

### System Design Fundamentals
1. [Scalability](#1-scalability)
2. [Availability](#2-availability)
3. [Latency vs Throughput](#3-latency-vs-throughput)
4. [CAP Theorem](#4-cap-theorem)
5. [Load Balancers](#5-load-balancers)
6. [Databases](#6-databases)
7. [CDN](#7-cdn-content-delivery-network)
8. [Message Queues](#8-message-queues)
9. [Rate Limiting](#9-rate-limiting)
10. [Database Indexes](#10-database-indexes)
11. [Caching](#11-caching)
12. [Consistent Hashing](#12-consistent-hashing)
13. [Database Sharding](#13-database-sharding)
14. [Consensus Algorithms](#14-consensus-algorithms)
15. [Proxy Servers](#15-proxy-servers)
16. [Heartbeats](#16-heartbeats)
17. [Checksums](#17-checksums)
18. [Service Discovery](#18-service-discovery)
19. [Bloom Filters](#19-bloom-filters)
20. [Gossip Protocol](#20-gossip-protocol)

### System Design Trade-Offs
- [Vertical vs Horizontal Scaling](#trade-off-1-vertical-vs-horizontal-scaling)
- [Strong vs Eventual Consistency](#trade-off-2-strong-vs-eventual-consistency)
- [Stateful vs Stateless Design](#trade-off-3-stateful-vs-stateless-design)
- [REST vs RPC](#trade-off-4-rest-vs-rpc)
- [Batch vs Stream Processing](#trade-off-5-batch-vs-stream-processing)
- [Long Polling vs WebSockets](#trade-off-6-long-polling-vs-websockets)
- [Normalization vs Denormalization](#trade-off-7-normalization-vs-denormalization)
- [TCP vs UDP](#trade-off-8-tcp-vs-udp)

### Architectural Patterns
- [Client-Server](#pattern-1-client-server-architecture)
- [Microservices](#pattern-2-microservices-architecture)
- [Serverless](#pattern-3-serverless-architecture)
- [Event-Driven](#pattern-4-event-driven-architecture)
- [Peer-to-Peer (P2P)](#pattern-5-peer-to-peer-p2p-architecture)

### Interview Framework
- [7-Step Interview Template](#7-step-system-design-interview-template)

---

## 1. Scalability

> **Definition:** Scalability is the property of a system to handle a growing amount of load by adding resources to the system.

A system that can continuously evolve to support growing work is called **scalable**.

### Two Types of Scaling

| Type | Also Called | How | Limit | Example |
|---|---|---|---|---|
| **Vertical Scaling** | Scale UP | Bigger CPU / RAM / Disk | Hardware ceiling | Upgrade 1 server to 128GB RAM |
| **Horizontal Scaling** | Scale OUT | Add more servers | Nearly unlimited | Add 1000 more servers |

### Why You Can't Scale Vertically Forever
Hardware has a **physical ceiling** — the most powerful single machine money can buy still can't handle Google-scale traffic. No such machine exists. That's why companies like Netflix, Amazon, and Google run **thousands of ordinary servers** horizontally.

### Real World
- **Twitter** handles 400M users/day → horizontal scaling only. No single server on Earth handles that load.
- **Vertical scaling** is fine at small scale — but plan for horizontal from day one.

### Interview Tip
> When asked "how would you scale X?" — always clarify whether it's a read-heavy or write-heavy system first. This determines WHERE you scale.

---

## 2. Availability

> **Definition:** Availability refers to the proportion of time a system is operational and accessible when required.

### Formula
```
Availability = Uptime / (Uptime + Downtime)
```

- **Uptime** = period the system is functional and accessible
- **Downtime** = period the system is unavailable (failures, maintenance, etc.)

### Availability Tiers ("The Nines")

| Availability % | Downtime/Year | Called |
|---|---|---|
| 99% | 3.65 days | "Two nines" |
| 99.9% | 8.76 hours | "Three nines" |
| 99.99% | 52.56 minutes | "Four nines" |
| 99.999% | 5.26 minutes | "Five nines" |
| 99.9999% | 31.5 seconds | "Six nines" |

> **Trick:** Just count the 9s! "Four Nines" = 99.99%

### Key Insight: The Nines Are NOT Linear
Going from Three → Four Nines isn't just "one more nine" — it's the difference between **hours and minutes** of downtime. Each extra nine costs exponentially more in engineering effort and money.

### Real World Application
Always match availability to **business criticality**:
- 🏥 Hospital patient monitor → Five/Six Nines (lives at stake)
- 🏦 Banking system → Four/Five Nines (money at stake)
- 🎬 Movie review website → Three Nines (nobody dies if it's down)

### Interview Question
> **Q:** Your payment system was down for 2 hours in a year. What "nines" is that?
> **A:** Three Nines (99.9%) — which allows up to 8.76 hours. Two hours fits in that bucket. Four Nines only allows ~52 minutes.

---

## 3. Latency vs Throughput

### Definitions

> **Latency** = the time it takes for a single request/operation to complete (measured in ms)

> **Throughput** = the amount of work done or data processed in a given time period (measured in RPS or TPS)

- **RPS** = Requests Per Second
- **TPS** = Transactions Per Second

### The Drive-Through Analogy 🍔
- **Latency** = how long YOU wait for YOUR burger
- **Throughput** = how many burgers the kitchen makes per hour

### They Are INDEPENDENT
High throughput does NOT guarantee low latency, and vice versa.

| | Low Latency | High Latency |
|---|---|---|
| **High Throughput** | Dream system | Possible (bulk processing) |
| **Low Throughput** | Possible (single fast ops) | Terrible system |

```
Highway (High Throughput, High Latency):
→ Moves thousands of cars/hour BUT each car takes 6 hours to cross

Ferrari on empty road (Low Throughput, Low Latency):
→ One car crosses in 2 minutes BUT only one car at a time
```

### Why Real World Latency is Worse Than Physics
```
Light speed in fiber optic cables = ~200,000 km/s
Karachi → California = ~12,000 km
Theoretical minimum latency = 60ms one way = 120ms round trip

BUT real latency = 200-400ms because:
→ Data hops through dozens of routers (each adds delay)
→ Undersea cables have physical limitations
→ Large files require multiple round trips
→ Network congestion
```

### When to Optimize Which
| Use Case | Optimize For | Why |
|---|---|---|
| 🎬 Video streaming (YouTube) | **Throughput** | Need massive continuous data flow |
| 📞 Video calls (Zoom) | **Latency** | Delay = awkward conversation |
| 🎮 Online gaming | **Latency** | 100ms delay = you're dead |
| 📧 Email delivery | **Throughput** | Volume matters, not speed |

> YouTube buffers because it's trading a tiny bit of latency for sustained high throughput — smart tradeoff!

---

## 4. CAP Theorem

> **Definition:** It is impossible for a distributed data store to simultaneously provide all three of: Consistency, Availability, and Partition Tolerance.

```
    Consistency  ●  ●  Availability
                  ●
           Partition Tolerance
```

### The Three Properties

**Consistency (C)**
> Every READ receives the **most recent write** or an error. No stale data — ever.

**Availability (A)**
> Every request receives a **non-error response**, but without guarantee it contains the most recent write. Always responds, possibly with stale data.

**Partition Tolerance (P)**
> The system continues to operate despite an arbitrary number of messages being **dropped or delayed** by the network between nodes.

### Why P is Non-Negotiable
Network failures WILL happen. Cables get cut, routers crash, data centers flood. You can't say "I won't tolerate partitions" — reality will force them on you.

```
[Server NYC] ----network---- [Server LA]
                ✂️ CUT!
[Server NYC]  💀  [Server LA]  ← can't talk to each other
```

This is called a **network partition**. Since P is mandatory, your real choice is:

### The Real Trade-off: CP vs AP

**CP System** (Consistency + Partition Tolerance):
```
Partition occurs → system refuses to serve → returns ERROR
"I'd rather be unavailable than give you wrong data"
```

**AP System** (Availability + Partition Tolerance):
```
Partition occurs → system serves last known data → possibly stale
"I'd rather give you something than nothing"
```

### CP vs AP Cheat Sheet

| System | Type | Why |
|---|---|---|
| 🏦 Banking / Payments | CP | Wrong balance = disaster |
| 📦 Inventory management | CP | Overselling = chaos |
| 🔐 Authentication | CP | Security can't be stale |
| 🐦 Twitter / Social feeds | AP | Seeing tweet 2 seconds late = fine |
| 🛒 Amazon product reviews | AP | Slightly stale reviews = acceptable |
| 📺 YouTube view counts | AP | "301 views" meme exists for a reason |
| 🌍 DNS servers | AP | Always respond, eventually consistent |

**Golden Rule:**
> Money / Security / Accuracy → **CP**
> Speed / Scale / User Experience → **AP**

### Bonus: Eventual Consistency
Google Docs is neither pure CP nor AP — it's **eventually consistent**:
- Lets you keep typing even when offline (AP-ish)
- Syncs and resolves conflicts when reconnected
- Data becomes consistent *eventually* after partition heals

> That's why you sometimes see two cursors fighting over the same word in Google Docs! 😂

### Interview Tip
> When given a system design question, one of your first questions should be: **"What's the cost of downtime vs the cost of serving stale data?"** — that determines CP vs AP.

---

## 5. Load Balancers

> **Definition:** A Load Balancer distributes incoming network traffic across multiple servers to ensure that no single server is overwhelmed.

```
                    → Server 1
Client → [LB]      → Server 2
                    → Server 3
```

Think of it as the **bouncer at a nightclub** — directing people to whichever room has space.

### 5 Load Balancing Algorithms

| Algorithm | How It Works | Best For |
|---|---|---|
| **Round Robin** | Server 1 → 2 → 3 → 1 → 2... circular | Identical servers, random traffic |
| **Weighted Round Robin** | Powerful server gets more requests | Servers with different capacities |
| **Least Connections** | Send to server with fewest active connections | Long-lived connections |
| **Least Response Time** | Send to fastest-responding server | Latency-critical systems (trading) |
| **IP Hash** | Hash client IP → always same server | Session persistence needed |

### Algorithm Deep Dive

**Round Robin** — Like dealing cards at a poker table. Everyone gets one equally.

**Weighted Round Robin** — Server A (16 cores) gets 4x requests vs Server C (4 cores). Match work to capacity.

**Least Connections** — Like joining the shortest checkout queue at Walmart. 🛒

**Least Response Time** — Like calling 3 taxis and going with whoever arrives first. 🚕

**IP Hash** — Like having your "regular table" at a restaurant. Same server every time. 🍽️

### When to Use Which
| Scenario | Algorithm | Reason |
|---|---|---|
| 🎮 Online gaming (session must persist) | IP Hash | Same player IP → same server → session intact |
| 🖥️ Mixed server capacities | Weighted Round Robin | Don't crush the weak server |
| 💹 Stock trading platform | Least Response Time | Milliseconds matter |
| 🌐 Static website, identical servers | Round Robin | Simple, fair distribution |
| 🏥 Long doctor consultations | Least Connections | Don't overload a busy doctor |

---

## 6. Databases

> **Definition:** A database is an organized collection of structured or unstructured data that can be easily accessed, managed, and updated.

### 6 Types of Databases

#### 1. Relational (RDBMS) 📊
- **Structure:** Tables with rows & columns. Strict schema. Supports SQL.
- **Examples:** MySQL, PostgreSQL, SQLite
- **Use when:** Structured data, complex queries, ACID transactions
- **Superpower:** JOINs across tables, foreign keys, constraints

#### 2. Key-Value (NoSQL) 🔑
- **Structure:** Simple hashmap — `key → value`. Like a dictionary.
- **Examples:** Redis, DynamoDB, Memcached
- **Use when:** Caching, sessions, leaderboards, real-time lookups
- **Superpower:** Blazing fast O(1) reads/writes

#### 3. Document 📄
- **Structure:** JSON-like documents. Flexible, no fixed schema.
- **Examples:** MongoDB, CouchDB, Firestore
- **Use when:** User profiles, product catalogs, content with varying fields
- **Superpower:** Each document can have different fields

#### 4. Graph 🕸️
- **Structure:** Nodes (entities) + Edges (relationships)
- **Examples:** Neo4j, Amazon Neptune
- **Use when:** Social networks, fraud detection, recommendation engines
- **Superpower:** Querying complex relationships at scale

#### 5. Time Series ⏰
- **Structure:** Data points indexed by timestamp
- **Examples:** InfluxDB, TimescaleDB, Prometheus
- **Use when:** Metrics, IoT sensors, stock prices, server monitoring
- **Superpower:** Extremely efficient storage and querying of time-ordered data

#### 6. Spatial 📍
- **Structure:** Geographic/location data with spatial indexes
- **Examples:** PostGIS (extension on PostgreSQL)
- **Use when:** Maps, "find nearby" features, delivery routing
- **Superpower:** Distance calculations and geographic queries

### System → Database Cheat Sheet

| System / Feature | Database Type | Why |
|---|---|---|
| 🏦 Banking / Payments | Relational | ACID transactions, strict rules |
| 👤 User authentication | Relational | Structured, relationships |
| 📦 Product catalog | Document | Flexible fields per product |
| 👤 User profiles | Document | JSON-like, flexible schema |
| 🎮 Game sessions / tokens | Key-Value | Blazing fast lookups |
| 🛒 Shopping cart | Key-Value | Temporary, fast access |
| 🔄 Caching layer | Key-Value | Redis — speed is everything |
| 🐦 Social follows / friends | Graph | Relationships between users |
| 🕵️ Fraud detection | Graph | Detect suspicious connection patterns |
| 📈 CPU / server metrics | Time Series | Timestamped data every second |
| 📉 Stock prices | Time Series | Time-ordered financial data |
| 🌡️ IoT sensor data | Time Series | Continuous time-stamped readings |
| 🗺️ Find nearby drivers | Spatial | GPS coordinates, distance queries |
| 📍 Delivery routing | Spatial | Geographic calculations |
| 📸 Instagram posts | Document | Flexible media + metadata |
| 🎬 Netflix watch history | Document | Flexible user activity |

### Golden Rules
```
Strict rules + money      → Relational   🏦
Fast + temporary          → Key-Value    ⚡
Flexible + JSON           → Document     📄
Who knows who             → Graph        🕸️
Time + metrics            → Time Series  ⏰
Location + maps           → Spatial      📍
```

---

## 7. CDN (Content Delivery Network)

> **Definition:** A CDN is a geographically distributed network of servers that work together to deliver web content to users based on their geographic location.

The primary purpose: deliver content with **high availability and performance** by reducing the physical distance between server and user.

```
Without CDN:
You (Karachi) --------12,000km-------- Server (California) 😫 ~300ms latency

With CDN:
You (Karachi) ---200km--- CDN Node (Dubai) 😎 ~20ms latency
```

### How It Works
When you request content, the CDN redirects you to the nearest server in its network (called an **edge node** or **PoP — Point of Presence**).

### What Belongs on a CDN?

| CDN = YES ✅ | CDN = NO ❌ |
|---|---|
| Same content for everyone | Personalized content |
| Rarely changes | Changes frequently |
| Public content | Sensitive / private |
| Large files (videos, images) | Small dynamic data |
| Static assets (JS, CSS, logos) | User-specific API responses |

**Examples:**
- 🎬 YouTube video → **YES** (same video for everyone, huge file)
- 🖼️ Facebook's logo → **YES** (same image for billions of users)
- 💬 Your Twitter DMs → **NO** (personalized, dynamic, private)
- 🏦 Your bank balance → **NO** (sensitive, changes constantly)

### Why CDNs Also Improve Availability
If one CDN node goes down, traffic automatically reroutes to the next nearest node — **no single point of failure**. Contrast that with one origin server going down = entire service dead. 💀

### Interview Tip
> CDNs are almost always the right answer when asked "how do you reduce latency for global users?" — mention it early in your design.

---

## 8. Message Queues

> **Definition:** A message queue is a communication mechanism that enables different parts of a system to send and receive messages **asynchronously**.

Producers can send messages and **move on** without waiting for consumers to process them.

### The Restaurant Analogy 🍕
- **Synchronous:** Waiter takes your order → stands at your table staring → doesn't move until food is ready → 50 other customers waiting 😤
- **Asynchronous:** Waiter takes your order → drops ticket in kitchen window 🎫 → goes to serve others → kitchen processes at own pace

**The ticket window = Message Queue**

### 3 Key Players

```
[Producer] → [Queue/Broker] → [Consumer]
```

| Player | Role |
|---|---|
| **Producer** | Creates and sends messages to the queue 📤 |
| **Queue (Broker)** | Holds messages until they're processed — the waiting room 📨 |
| **Consumer** | Picks up and processes messages from the queue 📥 |

### Real World Example — Uber Ride Request
```
[Uber App]  →  [Queue]  →  [Matching Service]
 Producer                    Consumer

1. You tap "Request Ride" → Producer sends message 📱
2. Message sits in queue 📨
3. Matching service picks it up → finds you a driver 🚗
4. Uber app already moved on — it didn't wait! ⚡
```

### Why Message Queues Are a Superpower — The Black Friday Problem
```
Without queue:
10,000 orders/sec → Server (handles 1,000/sec) → 💥 CRASH

With queue:
10,000 orders/sec → Queue 📨📨📨📨 → Server processes 1,000/sec calmly
                    (queue fills up during spike, drains as spike subsides)
```
The queue acts as a **shock absorber** for traffic spikes!

### WhatsApp Architecture Insight
```
You send "hey" → hits queue ✅ (single tick)
Queue delivers to friend → ✅✅ (double tick)
Friend reads it → 🔵✅✅ (blue ticks)
```
Each tick is a different stage of async message processing. You never knew you were looking at a message queue diagram! 😂

### Popular Message Queue Tools
- **RabbitMQ** — traditional, feature-rich
- **Apache Kafka** — high-throughput, distributed, durable
- **Amazon SQS** — managed cloud queue
- **Redis Pub/Sub** — lightweight, fast

### Message Queue vs Rate Limiting — Key Difference
| | Message Queue | Rate Limiting |
|---|---|---|
| **Purpose** | Buffer legitimate traffic spikes | Protect against abuse/overload |
| **Excess requests** | Waits in queue | REJECTED 🚫 |
| **Who's sending** | Trusted internal services | Anyone — including attackers |

---

## 9. Rate Limiting

> **Definition:** Rate limiting protects services from being overwhelmed by too many requests from a single user or client.

Think of it as the **bouncer with a guest list** — you get X entries per hour, no more.

### Why Rate Limiting Matters
- 🔐 Prevents brute force attacks (10,000 password attempts/second)
- 🤖 Blocks spam bots (mass tweeting, mass friending)
- 💳 Prevents API abuse (scraping, DDoS)
- 💰 Controls costs (pay-per-use APIs)

### 5 Rate Limiting Algorithms

#### 1. Token Bucket 🪣
```
Bucket capacity: 10 tokens | Refill rate: 2 tokens/second

9:00:00 → 10 tokens available
9:00:00 → Send 8 requests → 2 tokens left
9:00:01 → 2 tokens refill → 4 tokens
9:00:01 → Send 4 requests → 0 tokens
9:00:01 → Send 1 more → ❌ REJECTED!
```
- ✅ **Allows SHORT bursts** within overall rate limit
- 📱 Like a prepaid SIM card — spend freely, refills slowly
- **Best for:** Twitter API (burst of tweets when inspired)

#### 2. Leaky Bucket 💧
```
No matter how fast requests come in:
OUTPUT is always exactly 5 requests/second

9:00:00 → 100 requests flood in simultaneously
→ Queue holds them
→ Drip... drip... drip... 5/sec out
→ Smooth constant flow regardless of input!
```
- ✅ **Smooth, constant traffic flow** — no bursts allowed
- 🌊 Like a garden hose — pour as much as you want, same drip rate out
- **Best for:** Video processing pipelines, payment processors

#### 3. Fixed Window Counter 🪟
```
Limit: 10 requests/minute | Window resets every 60 seconds

9:00:00 → 10 requests → ALLOWED ✅
9:00:59 → counter = 10 → BLOCKED 🚫
9:01:00 → RESET! counter = 0
9:01:00 → 10 more requests → ALLOWED ✅

⚠️ BOUNDARY ATTACK WEAKNESS:
9:00:58 → 10 requests ✅ (end of window 1)
9:01:00 → 10 requests ✅ (start of window 2)
= 20 requests in 2 seconds! 😈
```
- ✅ **Simple to implement**
- ❌ Vulnerable to boundary attacks
- **Best for:** Simple use cases where precision isn't critical

#### 4. Sliding Window Log 📜
```
Limit: 5 requests/minute (rolling)

9:00:10 → ✅ log: [9:00:10]
9:00:20 → ✅ log: [9:00:10, 9:00:20]
9:00:30 → ✅ log: [9:00:10, 9:00:20, 9:00:30]
9:00:40 → ✅ log: [..., 9:00:40]
9:00:50 → ✅ log: [..., 9:00:50]
9:00:55 → 6th request → checks last 60s → 5 in window → ❌ REJECTED!
9:01:11 → 9:00:10 expires! → now 4 in window → ✅ ALLOWED!
```
- ✅ **Most accurate** — no boundary exploits
- ❌ Uses the most memory (stores every timestamp)
- **Best for:** Financial APIs where precision is critical

#### 5. Sliding Window Counter 🔢
```
Limit: 10 requests/minute
Current window: 9:01:00–9:02:00 (40 seconds in)
Previous window had: 8 requests

Weight of previous window = (60-40)/60 = 0.33
Estimated count = (8 × 0.33) + current_requests

current = 5 → (8 × 0.33) + 5 = 7.64 → ✅ ALLOWED
current = 8 → (8 × 0.33) + 8 = 10.64 → ❌ REJECTED!
```
- ✅ **Best of both worlds** — accurate + memory efficient
- **Best for:** General production use cases

### Algorithm Comparison

| Algorithm | Allows Bursts? | Smooth Flow? | Accurate? | Complexity |
|---|---|---|---|---|
| Token Bucket 🪣 | ✅ Yes | ❌ No | ✅ Good | 🟡 Medium |
| Leaky Bucket 💧 | ❌ No | ✅ Yes | ✅ Good | 🟡 Medium |
| Fixed Window 🪟 | ✅ Yes | ❌ No | ❌ Boundary bug | 🟢 Simple |
| Sliding Log 📜 | ❌ No | ✅ Yes | ✅ Best | 🔴 Complex |
| Sliding Counter 🔢 | 🟡 Sort of | ✅ Yes | ✅ Very good | 🟡 Medium |

### Memory Trick
```
Want bursts allowed?         → Token Bucket   🪣
Want constant smooth flow?   → Leaky Bucket   💧
Want simple but exploitable? → Fixed Window   🪟
Want most precise?           → Sliding Log    📜
Want best of both worlds?    → Sliding Counter 🔢
```

---

## 10. Database Indexes

> **Definition:** A database index is a super-efficient lookup table that allows a database to find data much faster. It holds indexed column values along with pointers to corresponding rows.

### The Problem Without Indexes
```
Table has 1,000,000 rows:

SELECT * FROM users WHERE email = 'kamal@gmail.com'

→ Check row 1... nope
→ Check row 2... nope
→ ... check all 999,999 rows... 😵
→ Found at row 999,999! 🐌

This is O(n) — "linear scan" — painfully slow at scale!
```

### The Solution: B-Tree Index
Database indexes are implemented as a **B-Tree (Balanced Tree)**:

```
INDEX (sorted alphabetically by email):

              [m___]
             /       \
        [a-l]         [n-z]
        /    \         /    \
     [a-f] [g-l]   [n-r]  [s-z]
                      |
               kamal@gmail.com → Row 999,999 ✅

SELECT * WHERE email = 'kamal@gmail.com'
→ 'k' < 'm' → go left
→ 'k' > 'g' → go right
→ Found! 20 comparisons instead of 999,999! ⚡

This is O(log n) — blazing fast!
```

### What an Index Looks Like
```
ACTUAL TABLE                      EMAIL INDEX (B-Tree)
┌────┬──────────────┬─────┐      ┌──────────────────────┬─────────┐
│ id │ email        │ age │      │ email (sorted)        │ row ptr │
├────┼──────────────┼─────┤      ├──────────────────────┼─────────┤
│ 1  │ alice@...    │ 25  │ ←────│ alice@gmail.com       │ row 1   │
│ 2  │ bob@...      │ 30  │ ←────│ bob@gmail.com         │ row 2   │
│ 3  │ carol@...    │ 28  │ ←────│ carol@gmail.com       │ row 3   │
│ .. │ ...          │ ..  │      │ ...                   │ ...     │
│999 │ kamal@...    │ 22  │ ←────│ kamal@gmail.com       │ row 999 │
└────┴──────────────┴─────┘      └──────────────────────┴─────────┘

Index = a SEPARATE structure stored alongside the table
      = sorted copy of ONE column + pointers to real rows
```

### The Hidden Cost of Indexes
Every INSERT/UPDATE/DELETE must **also update all indexes**:
```
You update kamal@gmail.com → newkamal@gmail.com:

Step 1: Update actual table row ✍️
Step 2: Find old entry in B-Tree 🔍
Step 3: Remove it 🗑️
Step 4: Insert new entry in sorted position ✍️
Step 5: Rebalance the B-Tree 🌳

= WAY more work than just updating the table!
```

### The Trade-off

| | No Index | With Index |
|---|---|---|
| **Read speed** | 🐌 O(n) full scan | ⚡ O(log n) tree |
| **Write speed** | ⚡ Fast | 🐌 Slower (update index too) |
| **Storage** | 💚 Small | 🔴 Larger |

### When to Add an Index
✅ **Index these:**
- Columns frequently used in `WHERE` clauses
- Columns used in `JOIN` conditions
- Columns used in `ORDER BY` / `GROUP BY`
- High-cardinality columns (many unique values like `email`, `user_id`)

❌ **Don't index these:**
- Columns rarely searched (e.g., `delivery_notes`, `middle_name`)
- Low-cardinality columns (e.g., `gender` with only 2-3 values — index barely helps)
- Tables with very heavy write load and few reads

### Interview Tip
> "Your query is slow" → first answer is always **"add an index on the WHERE/JOIN column"**. If already indexed → look at query plan, consider composite indexes or caching.

---

## 11. Caching

> **Definition:** Caching is a technique to temporarily store copies of data in high-speed storage layers to reduce the time taken to access data.

**Primary goals:** Reduce latency + offload the main database + faster data retrieval.

### Cache Hit vs Cache Miss
```
WITHOUT cache:
Every request → Database 🐌 → response

WITH cache:
Request → Cache ⚡ (found!)   → response         [Cache HIT!]
Request → Cache ❌ (missing!) → Database → Cache → response  [Cache MISS]
```

### 4 Caching Strategies

#### 1. Read-Through Cache 📖
```
App asks cache → HIT? → return data ✅
                MISS? → cache fetches from DB automatically
                      → stores in cache
                      → returns to app

App NEVER talks to DB directly — cache is always the middleman
```
- 🤝 Like a personal assistant — you never go to the filing cabinet yourself
- **Good for:** Read-heavy workloads where cache does all the work

#### 2. Write-Through Cache ✍️
```
App writes data:
→ Writes to CACHE first
→ IMMEDIATELY writes to DB too (synchronously)
→ Both always in sync! 🔄

Write: App → Cache + DB simultaneously
Read:  App → Cache ⚡
```
- 📓 Like writing in your notebook AND Google Docs at the same time
- **Good for:** Systems where consistency between cache and DB is critical

#### 3. Write-Back Cache ⏰
```
App writes data:
→ Writes to CACHE only ⚡ (super fast response!)
→ DB updated LATER in batches (asynchronously)

Write: App → Cache (instant ack!)
Sync:  Cache → DB (delayed, periodic)
```
- 📧 Like drafting an email — saved locally instantly, syncs to server later
- ⚠️ **RISK:** If cache crashes before sync → data lost! 💀
- **Good for:** Write-heavy workloads where speed matters more than durability

#### 4. Cache-Aside (Lazy Loading) 🤲
```
App checks cache manually:
→ HIT?  → use cached data ✅
→ MISS? → App fetches from DB itself
        → App manually stores in cache
        → returns data

App is fully in control of what goes in/out of cache
```
- 🍳 Like cooking yourself instead of ordering food — more control, more work
- ✅ **Most popular strategy in production** (Redis is used this way)
- **Good for:** When you want fine-grained control over cache population

### Strategy Comparison

| Strategy | Who fetches on miss? | When is DB written? | Popular? |
|---|---|---|---|
| Read-Through 📖 | Cache automatically | On reads only | Medium |
| Write-Through ✍️ | Cache automatically | Immediately with cache | Medium |
| Write-Back ⏰ | Cache automatically | Later / delayed | Less (risky) |
| Cache-Aside 🤲 | Application itself | App decides | **Most popular** |

### 4 Cache Eviction Policies

When cache is full and new data must come in — something gets **kicked out**!

#### LRU — Least Recently Used ⏰
```
Cache: [A(10min ago), B(2min ago), C(30sec ago)]
New item D needs space!
→ Kick out A (accessed longest ago) 🗑️
→ Add D ✅

"If you haven't used it in a while, you probably don't need it"
```
- 📄 Like cleaning your desk — throw out papers untouched for months
- ✅ **Most popular eviction policy in real systems**

#### LFU — Least Frequently Used 📊
```
Cache: [A(accessed 100x), B(accessed 2x), C(accessed 50x)]
New item needs space!
→ Kick out B (only accessed 2 times) 🗑️

"If nobody uses it much, it's probably not important"
```
- 🎬 Like Netflix removing shows nobody watches
- **Best for:** Popularity-based caching (music, trending content)

#### FIFO — First In First Out 📦
```
Cache entry order: [A→B→C→D→E]
New item F needs space!
→ Kick out A (entered first) 🗑️
→ Even if A is accessed every second!

"Oldest item leaves, no questions asked"
```
- 🍔 Like a McDonald's queue — first in, first out
- ⚠️ Not smart — might kick out heavily used data!
- **Best for:** Simple queues, low-overhead scenarios

#### TTL — Time to Live ⏱️
```
cache.set("user_123_session", data, TTL=3600)  // expires in 1 hour

After 3600 seconds → AUTOMATICALLY deleted 🗑️
Regardless of how often it was accessed!

"Data expires like milk — out after the date"
```
- 🥛 Like food expiry dates — gone after the time, no matter what
- **Best for:** Auth tokens, session data, time-sensitive content

### Eviction Policy Summary

| Policy | Kicks Out | Best For |
|---|---|---|
| LRU ⏰ | Longest unused | General purpose **(most common)** |
| LFU 📊 | Least accessed overall | Popularity-based data |
| FIFO 📦 | Oldest entry | Simple, low overhead |
| TTL ⏱️ | Expired items | Time-sensitive / security data |

### Match the Policy
- 🔐 Auth tokens expiring after 1 hour → **TTL**
- 📰 News articles (recent = popular) → **LRU**
- 🎵 Bohemian Rhapsody vs obscure songs → **LFU**

---

## 12. Consistent Hashing

### The Problem with Simple Hashing
```
3 servers, formula: user_id % 3

User 1 → Server 1 | User 2 → Server 2 | User 3 → Server 3 ✅

ADD a 4th server — formula becomes user_id % 4:

User 1 → Server 1 ✅ (same)
User 2 → Server 2 ✅ (same)
User 3 → Server 4 ❌ (MOVED!)
User 4 → Server 1 ❌ (MOVED!)
User 5 → Server 2 ❌ (MOVED!)

~75% of all data remaps to different servers 💀
= ALL cached data invalidated = DB gets hammered = CRASH
```

### The Solution: Hash Ring

A circular space from **0 to 2^n-1** where both servers AND data are placed using a hash function.

```
         Server A (pos 10)
              ↑
    ──────────A────────────
   /                        \
  |           RING 🔄        |
   \                        /
    ──────C────────────B────
         ↑            ↑
    Server C       Server B
    (pos 80)       (pos 50)
```

**Rule:** To find which server owns a piece of data → move **clockwise** from data's position until you hit a server.

```
Data X (pos 15) → clockwise → first server hit = B (pos 50) → stored on B
Data Y (pos 55) → clockwise → first server hit = C (pos 80) → stored on C
Data Z (pos 85) → clockwise → first server hit = A (pos 10) → stored on A
```

### Adding a New Server
```
Add Server D at position 30:

Data X (pos 15) → clockwise → hits D (pos 30) first now → moves to D 📦
Data Y (pos 55) → clockwise → still hits C (pos 80) → STAYS ✅
Data Z (pos 85) → clockwise → still hits A (pos 10) → STAYS ✅

ONLY data between A(10) and D(30) moved — everything else untouched!
```

### The Magic Formula
```
Simple Hashing:    Add 1 server → ~100% data remaps 💀
Consistent Hashing: Add 1 server → only ~1/n data remaps 🎉

With 3 servers → add 1 → only ~25% moves
With 100 servers → add 1 → only ~1% moves! 🔥
```

### Server Removal
```
Server B crashes:
Data that WAS going to B → now goes to next clockwise server (C)
Only B's data is affected — A and C's data untouched ✅
```

### Virtual Nodes — Handling Unequal Servers
Problem: Basic consistent hashing gives all servers equal data, ignoring their capacity.

Solution: **Virtual Nodes** — powerful servers get MORE positions on the ring!
```
Server A (16 cores) → positions: 10, 40, 70     (3 virtual nodes)
Server B (8 cores)  → positions: 25, 60         (2 virtual nodes)
Server C (4 cores)  → position:  85             (1 virtual node)

More positions = more data assigned = load proportional to capacity ⚖️
```

### Interview Tip
> Consistent hashing is the answer whenever you're asked "how do you distribute data across servers and handle servers being added/removed without disrupting everything?"

---

## 13. Database Sharding

> **Definition:** Database sharding is a horizontal scaling technique used to split a large database into smaller, independent pieces called **shards**, distributed across multiple servers.

### The Problem
```
1 database, 5 billion rows, 500,000 queries/sec:

→ Running out of storage 💾
→ CPU maxed out 🔥
→ Queries taking 10+ seconds 🐌
→ Vertical scaling maxed out — no bigger server exists!
```

### The Solution: Split Into Shards
```
ONE database (5 billion rows) 😵
┌─────────────────────────┐
│ ... 5 billion rows ...  │
└─────────────────────────┘
              ↓ SHARD IT!
Shard 1          Shard 2          Shard 3
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Users    │     │ Users    │     │ Users    │
│ 1-1B     │     │ 1B-2B    │     │ 2B-3B    │
└──────────┘     └──────────┘     └──────────┘

Before: 500,000 queries/sec → 1 server 💀
After:  100,000 queries/sec → each of 5 shards 😎
```

### 3 Sharding Strategies

#### 1. Range-Based Sharding 📊
```
Shard 1 → User IDs 1 to 1,000,000
Shard 2 → User IDs 1,000,001 to 2,000,000
Shard 3 → User IDs 2,000,001 to 3,000,000
```
- ✅ Simple to understand and implement
- ❌ **Hot Shard problem:** If most users have IDs in shard 1, it gets overloaded

#### 2. Hash-Based Sharding 🔢
```
shard = hash(user_id) % number_of_shards

User 1 → hash(1) % 3 = 0 → Shard 1
User 2 → hash(2) % 3 = 1 → Shard 2
User 3 → hash(3) % 3 = 2 → Shard 3
```
- ✅ Even distribution — no hot shards
- ❌ Adding a shard = massive remapping (solve this with **Consistent Hashing** from Ch. 12!)

#### 3. Geographic Sharding 🌍
```
Shard 1 → Users in Asia 🌏
Shard 2 → Users in Europe 🌍
Shard 3 → Users in Americas 🌎
```
- ✅ Data lives near users = low latency
- ✅ GDPR compliance — EU data stays in EU 🇪🇺
- ❌ Uneven if one region has far more users

### The Painful Problem: Cross-Shard JOINs
```
Before sharding (easy):
SELECT * FROM users JOIN orders ON users.id = orders.user_id
→ Everything in one DB → simple JOIN ✅

After sharding (painful):
Users table → Shard 1 (Server 1)
Orders table → Shard 3 (Server 3)

→ CAN'T JOIN across different servers! 💀
→ Must do 2 separate queries in application code
→ Fetch users from Shard 1
→ Fetch orders from Shard 3
→ Merge manually in app
= More complex code + slower performance 😫
```

### Sharding Trade-offs

| | Before Sharding | After Sharding |
|---|---|---|
| **Storage** | 💀 Limited | ✅ Unlimited |
| **Query Speed** | 🐌 Slow | ⚡ Fast |
| **Cross-table JOINs** | ✅ Easy | ❌ Painful |
| **Code Complexity** | 😎 Simple | 😵 Complex |

### How Everything Connects
```
Sharding challenge              → Solution from earlier chapters
────────────────────────────────────────────────────────────────
Hash-based remapping on resize  → Consistent Hashing (Ch. 12) ✅
Slow queries per shard          → Database Indexes (Ch. 10) ✅
Shard gets overwhelmed          → Load Balancer (Ch. 5) ✅
```

### Golden Rule
> **Don't shard until you absolutely have to!** Sharding solves scale problems but introduces massive complexity. Most apps never need it — only at Instagram/Twitter/Amazon scale.

---

## 14. Consensus Algorithms

> **Definition:** Consensus algorithms ensure that all participating nodes in a distributed system agree on the same state or sequence of events, even when some nodes fail or act maliciously.

### The Problem: Split Brain
```
5-node cluster. Network partition occurs.

Group A: [Node 1, Node 2] — think Node 1 is leader
Group B: [Node 3, Node 4, Node 5] — think Node 3 is leader

Node 1: "Write X=100" 👑
Node 3: "Write X=200" 👑

= Two conflicting leaders = data corruption = DISASTER 💀
```
This is called the **Split Brain** problem.

### The Leader's Job
Once ONE leader is elected, all writes go through it:
```
Client writes X=100
        ↓
  [Leader Node] ← only node that accepts writes!
   /    |    \
  ↓     ↓     ↓
Node2 Node3 Node4  ← leader replicates to all followers
X=100 X=100 X=100  ← all nodes agree! ✅
```

### What is a Node?
> A **node** is simply any individual machine/server in a distributed system. When you have multiple servers working together, each server = a node.

### 2 Popular Consensus Algorithms

#### Paxos 🏛️
```
1. A node says "I want to be leader!"
   → sends PREPARE message to all nodes
2. Other nodes PROMISE to support it
3. Leader PROPOSES a value
4. Majority accepts → value COMMITTED ✅

Like a politician campaigning:
"Vote for me!" → majority votes → leads → makes decisions 🗳️
```
- Used by: Google Chubby, Apache ZooKeeper
- ⚠️ Very complex to implement correctly — even Google engineers find it hard!

#### Raft 🚣
```
1. All nodes start as FOLLOWERS 👥
2. No heartbeat from leader? → one node becomes CANDIDATE
3. Candidate asks "Vote for me!"
4. Majority votes → becomes LEADER 👑
5. Leader sends HEARTBEATS every few seconds
6. No heartbeat received → new election! 🗳️

Like a classroom: teacher disappears →
student volunteers → class votes →
new teacher manages everything 📚
```
- Used by: etcd, CockroachDB, TiKV
- ✅ Designed to be **understandable** — Raft paper literally titled *"In Search of an Understandable Consensus Algorithm"*

### Paxos vs Raft

| | Paxos 🏛️ | Raft 🚣 |
|---|---|---|
| **Complexity** | Very complex 😵 | Simpler 😎 |
| **Designed for** | Correctness | Understandability |
| **Used by** | Google, ZooKeeper | etcd, CockroachDB |
| **Learn first?** | ❌ | ✅ |

### The Majority (Quorum) Rule
Both algorithms require **majority** to make decisions:
```
3 nodes → need 2 votes (majority)
5 nodes → need 3 votes (majority)
7 nodes → need 4 votes (majority)
```

> ⚠️ Always use **ODD numbers** of nodes! (3, 5, 7)
> Even numbers can result in a tie — nobody wins, system hangs!

**Example:** 5-node cluster, 3 nodes crash → only 2 alive → 2 < 3 (majority) → **cannot elect leader → system goes down**. This is a SAFETY feature — prevents split brain!

---

## 15. Proxy Servers

> **Definition:** A proxy server acts as a **gateway** between you and the internet — an intermediary server separating end users from the websites they browse.

### 2 Types of Proxy Servers

#### Forward Proxy 🔀
```
[Client] → [Forward Proxy] → Internet/Server
```
- Sits in front of the **CLIENT**
- Forwards requests to the internet **on behalf of the client**
- **Hides the client's identity** from the server

**Real world uses:**
- 🔒 **VPN** — your IP is hidden, server sees VPN's IP
- 🏢 **Corporate proxy** — company monitors/restricts employee internet access
- 🌍 **Geo-bypass** — appear to be in a different country

#### Reverse Proxy 🔄
```
Client → Internet → [Reverse Proxy] → [Server(s)]
```
- Sits in front of the **SERVER**
- Forwards requests from clients to backend servers
- **Hides the server's identity** from the client

**Real world uses:**
- ⚖️ **Load Balancer** — distributes requests across servers (yes, LBs are reverse proxies!)
- 🛡️ **Security** — hides how many servers you have, absorbs attacks
- 🗜️ **SSL Termination** — handles HTTPS encryption so servers don't have to
- 📦 **Caching** — serves cached responses without hitting backend

### Memory Trick
```
Forward  = protects/hides the CLIENT  👤  (you use VPN → you're hidden)
Reverse  = protects/hides the SERVER  🖥️  (Netflix → you don't know how many servers they have)
```

### Matching Scenarios
| Scenario | Proxy Type | Why |
|---|---|---|
| 🏢 Company monitors employee traffic | Forward | Sits in front of clients |
| 🌐 Netflix hides its server count | Reverse | Sits in front of servers |
| 🔒 User bypasses geo-restrictions | Forward | Hides user's real location |
| ⚖️ Load balancing across servers | Reverse | Routes to backend servers |

---

## 16. Heartbeats

> **Definition:** A heartbeat is a **periodic message** sent from one component to another to monitor each other's health and status.

### The ICU Analogy 🏥
You don't wait for a patient to tell you they're dying — you monitor their heartbeat continuously. No heartbeat detected → EMERGENCY! Same logic applies to servers.

### How It Works
```
Client          Server
  |──HeartBeat──→|  🟢 Online
  |──HeartBeat──→|  🟢 Online  (5s later)
  |──HeartBeat──→|  🟢 Online  (5s later)
  |              |
  |   10s gap... |  🔴 Offline!! 🚨 → trigger failover!
```

### Why Heartbeats Are Critical
```
WITHOUT heartbeat:
Server 5 crashes at 9:00am
System doesn't notice → user reports error at 9:45am 😱
= 45 MINUTES of silent downtime!

WITH heartbeat (every 5 seconds):
Server 5 crashes at 9:00am
No heartbeat at 9:00:05am → 🚨 ALERT!
System reroutes traffic automatically
= 5 SECONDS of downtime ✅
```

Without heartbeats:
- 🐌 **Delayed** fault detection and recovery
- 📈 **Increased** downtime and errors
- 📉 **Decreased** overall system reliability

### Connection to Consensus (Ch. 14)
In Raft, the **leader sends heartbeats** to all followers to signal:
> *"I'm still alive and I'm still your leader — no election needed!"*

```
Leader → ❤️ heartbeat → Follower 1: "Leader alive, stay calm" ✅
Leader → ❤️ heartbeat → Follower 2: "Leader alive, stay calm" ✅

Leader crashes 💀
No heartbeat for 10s...
Follower 1: "Leader dead! Starting election!" 🗳️
```

**Heartbeat = "I'm alive"**
**No heartbeat = "Something's wrong, react!"**

---

## 17. Checksums

> **Definition:** A checksum is a **unique fingerprint** attached to data before it's transmitted. When data arrives, the fingerprint is recalculated to ensure it matches the original.

### How It Works
```
SENDER:
Data: "Hello Kamal"
Run through hash function → Checksum: "a3f8c2"
Send both: data + checksum

RECEIVER:
Received: "Hello Kamal"
Recalculate: hash("Hello Kamal") → "a3f8c2"
Compare: "a3f8c2" == "a3f8c2" ✅ → Data is safe!

CORRUPTION CASE:
Received: "Hello Kamel"  ← bit flipped in transit
Recalculate: hash("Hello Kamel") → "x9k2p7"
Compare: "a3f8c2" ≠ "x9k2p7" ❌ → CORRUPTION DETECTED! → Retransmit!
```

### Real World Examples
```
📦 File downloads:
Download Ubuntu ISO (4GB)
Website shows: MD5 = "abc123"
Your computer calculates: MD5 = "abc123" ✅ → Perfect download!
Mismatch → corrupted file → re-download!

🌐 Network packets (TCP):
Every TCP packet carries a checksum
Router receives → recalculates → mismatch → REQUEST RETRANSMISSION 🔄

💾 Database storage:
Data written to disk with checksum
Data read back → checksum verified
Silent disk corruption caught! 🚨 (happens more than you'd think!)

🔗 Database replication (Raft/Paxos):
Leader sends checksum of full data first
Data sent in packets (each with own checksum)
Follower verifies each packet checksum ✅
Final data checksum calculated and matched against leader's initial checksum
MATCH ✅ → "Replication verified!"
NO MATCH ❌ → "Corruption detected, resend!" 🔄
```

### How to Hash 10GB of Data?
You can't load 10GB into memory at once — that would kill your RAM! 😵

**Solution: Streaming / Chunking**
```
10GB file → split into 1MB chunks:

Chunk 1 (1MB) → hash function → partial hash
Chunk 2 (1MB) → hash function → partial hash
...
Chunk 10,000 → hash function → partial hash
Combine all → FINAL checksum ✅

Like counting $1,000,000 — count $1,000 at a time, keep running total 💵
```

### Popular Checksum Algorithms

| Algorithm | Speed | Security | Use Case |
|---|---|---|---|
| **CRC32** | ⚡⚡ Fastest | Low | Network packets, file integrity |
| **MD5** | ⚡ Very fast | Medium | File downloads, checksums |
| **SHA-256** | 🟡 Medium | High | Security, Bitcoin transactions! |
| **SHA-512** | 🔴 Slower | Very high | Passwords, sensitive data |

> SHA-256 is what **Bitcoin** uses to verify every single transaction — you've been learning blockchain fundamentals this whole time! 🪙😂

### Why Catch Corruption Early?
Silent corruption is the **worst kind of bug** — your system thinks everything is fine but data is wrong. A checksum mismatch gives you a **loud, immediate failure** instead of a silent wrong answer spreading through your system.

---

## 18. Service Discovery

> **Definition:** Service Discovery is a mechanism that allows services in a distributed system to **find and communicate** with each other dynamically — without hardcoding addresses.

### The Problem It Solves

In a microservices system (e.g. Netflix with 500 services), every service needs to know WHERE other services are. IP addresses change constantly as servers restart and scale. Hardcoding `billing.service = 192.168.1.45` breaks the moment that server restarts with a new IP.

### How It Works

```
                    SERVICE REGISTRY
                   ┌──────────────────────┐
                   │ billing  → 10.0.1.5  │
                   │ search   → 10.0.2.3  │
                   │ stream   → 10.0.3.8  │
                   └──────────────────────┘
                    ↑ Register          ↓ Lookup
           [billing service]    [recommendations service]
                                        ↓
                                   3. CALL billing
```

1. **Register** — on startup, every service registers its address: "I'm billing at 10.0.1.5"
2. **Lookup** — when service A needs service B, it queries the registry
3. **Call** — now it has the address, it calls directly

### How the Registry Stays Fresh

Two mechanisms keep entries up-to-date:
- **Re-registration on restart** — when a service restarts with a new IP, it immediately registers the new address
- **Heartbeats** — services continuously ping the registry ("still alive at 10.0.1.5"). No heartbeat for 30s → registry removes the entry automatically

### Key Concern: The Registry is a Single Point of Failure

If the registry goes down, no service can find any other service — the entire system collapses even though every individual service is healthy.

**Fix:** Run the registry as a **cluster** (multiple replicas with consensus). Real-world tools: **Consul, Eureka, Zookeeper** — all run as clusters, never as single servers.

### When to Use It

| Needs Service Discovery | Doesn't Need It |
|------------------------|-----------------|
| 200 microservices in a cloud that auto-scales | Small app with 1 server + 1 DB |
| Services with dynamic, changing IPs | Mobile app calling a single fixed API |
| Kubernetes/Docker environments | Static infrastructure with hardcoded IPs |

### Interview Tip
> Service Discovery + Heartbeats + Consensus are the trio that make microservices self-healing. If an interviewer asks how 500 microservices coordinate, this is your answer.

---

## 19. Bloom Filters

> **Definition:** A Bloom filter is a **probabilistic data structure** that answers "is this element in the set?" — using a fraction of the memory a real set would require. It can have **false positives** but NEVER false negatives.

### The Problem It Solves

You're building a web crawler that visits billions of URLs. Before crawling any URL, you check: "Have I visited this before?" Storing billions of URLs in a database for lookup is expensive. Bloom filter does it in kilobytes.

### How It Works

```
Setup: bit array of 0s, k hash functions
[0][0][0][0][0][0][0]

Add "google.com" → 3 hash functions → positions 1, 3, 5 → set to 1
[0][1][0][1][0][1][0]

Add "twitter.com" → 3 hash functions → positions 0, 3, 6 → set to 1
[1][1][0][1][0][1][1]

Query "google.com" → hash → positions 1, 3, 5 → ALL are 1 → PROBABLY in set ✅
Query "amazon.com" → hash → positions 2, 4, 6 → position 2 is 0 → DEFINITELY NOT in set ❌
```

### The Magic Rule

> **If ANY bit is 0 → element is DEFINITELY NOT in the set.**
> **If ALL bits are 1 → element is PROBABLY in the set (could be a false positive).**

### Why False Positives Happen

Multiple different elements can accidentally set the same bit positions to 1. A new element's hash positions might all already be 1 — not because it was added, but because other elements happened to use those same positions.

```
"google.com"  sets bits → positions [1, 3, 5]
"youtube.com" sets bits → positions [2, 3, 6]

Query "amazon.com" → hashes to [1, 3, 6]
All three bits are 1! But we never added amazon.com! → FALSE POSITIVE 😱
```

### False Positive vs False Negative

| | Bloom Filter Behaviour |
|--|----------------------|
| **False Positive** | Says "probably seen" when actually new — CAN HAPPEN |
| **False Negative** | Says "definitely not seen" when actually seen — IMPOSSIBLE |

### When False Positives Are OK

For a web crawler, skipping one URL that hasn't been visited is a minor annoyance. For a banking system, a false positive ("you already withdrew this") is catastrophic. Use bloom filters only when false positives are tolerable.

### Real-World Uses

- **Google Chrome** — checks if a URL is malicious before doing a full DB lookup
- **Databases** — avoids expensive disk reads for keys that don't exist
- **Spam filters** — fast first-pass check before detailed analysis
- **Username availability** — pre-check before hitting the DB

### Interview Tip
> Memory trick: **Bloom filter can LIE about yes, but NEVER lies about no.**
> Use when: large dataset + memory is precious + occasional false positives are acceptable.

---

## 20. Gossip Protocol

> **Definition:** Gossip Protocol is a **decentralized communication protocol** used in distributed systems to spread information across all nodes — without a central coordinator. Inspired by how office gossip spreads by word-of-mouth.

### The Problem It Solves

In a large cluster (e.g. Cassandra with 1000 nodes), every node needs to know the state of every other node. Heartbeats from a central leader to 999 nodes every second = massive network overhead that doesn't scale.

### How It Works

```
Round 1: Node A knows info → randomly tells B and C
Round 2: B tells D, E  /  C tells F, G
Round 3: D, E, F, G each randomly tell 2 more nodes...
Eventually: ALL 1000 nodes know! 📢
```

4 steps:
1. **Initialization** — one node starts with a piece of information (the "gossip")
2. **Gossip Exchange** — at regular intervals, each node randomly selects another node and shares its current state
3. **Propagation** — the receiving node merges the received info with its own, then spreads further
4. **Convergence** — eventually ALL nodes have consistent information

### Gossip vs Heartbeat at Scale

```
Heartbeats (centralized):      Gossip (decentralized):
Leader → node1                 A → B, C
Leader → node2                 B → D, E
Leader → node3                 C → F, G
Leader → node999               D,E,F,G → ...
[1 node does ALL the work]     [work is shared by everyone]
```

### Key Properties

| Property | Gossip Protocol |
|----------|----------------|
| **Decentralized** | No single leader required |
| **Fault tolerant** | Node crashes don't stop info spreading — routes around failures |
| **Scalable** | Each node only talks to a FEW neighbors, but info spreads exponentially |
| **Eventual consistency** | All nodes converge to same state — but not instantly |

### Real-World Uses

- **Apache Cassandra** — uses gossip so every node knows the state of every other node with no master
- **Amazon DynamoDB** — node membership and failure detection
- **Redis Cluster** — node state propagation

### Interview Tip
> When asked "how do 1000 nodes stay aware of each other without a master?" — Gossip Protocol is the answer. It's the backbone of leaderless distributed databases.

---

# System Design Trade-Offs

> Trade-offs are the heart of system design interviews. There is no "correct" answer — only justified choices.

---

## Trade-Off 1: Vertical vs Horizontal Scaling

*(Already covered in depth in [Chapter 1](#1-scalability))*

| | Vertical (Scale Up) | Horizontal (Scale Out) |
|--|---------------------|----------------------|
| How | Bigger CPU/RAM/Disk | Add more servers |
| Limit | Hardware ceiling | Nearly unlimited |
| Complexity | Simple | Requires load balancer + distributed system thinking |
| Use when | Early stage, small traffic | Large scale, must not have single point of failure |

---

## Trade-Off 2: Strong vs Eventual Consistency

*(Also connected to [CAP Theorem](#4-cap-theorem))*

**Strong Consistency** — any read returns the most recent write. Once a write is acknowledged, ALL subsequent reads reflect it everywhere immediately.

**Eventual Consistency** — all nodes will EVENTUALLY converge to the same value, but there's no guarantee of WHEN. Temporarily stale reads are possible.

```
Strong:                          Eventual:
Write "balance = $500"           Write "balance = $500"
Node A reads → $500 ✅           Node A reads → $500 ✅
Node B reads → $500 ✅           Node B reads → $450 (stale!) ⏳
                                 Node B reads 2s later → $500 ✅ (converged)
```

| Use Strong Consistency | Use Eventual Consistency |
|----------------------|-------------------------|
| Banking, financial transactions | Social media feeds |
| Inventory systems (can't oversell) | Shopping cart suggestions |
| Medical records | DNS propagation |

---

## Trade-Off 3: Stateful vs Stateless Design

**Stateful** — the server remembers client data between requests (stores session in server memory).

**Stateless** — the server remembers NOTHING. Each request contains ALL information needed to process it. State lives in a shared store (database/cache), not on the server.

```
STATEFUL (problematic with load balancers):
Request 1 → Server A → "User has 3 items in cart" (in Server A's memory)
Request 2 → Server B → "Who are you? I don't know you!" 😱 CART GONE!

STATELESS (works with any server):
Request 1 → Server A → reads token from request → processes → done
Request 2 → Server B → reads same token → processes → done ✅
```

| | Stateful | Stateless |
|--|---------|-----------|
| **Horizontal scaling** | Hard (sticky sessions needed) | Easy (any server handles any request) |
| **Server crash** | User loses session | No problem, next server takes over |
| **Speed** | Fast (state in memory) | Slightly slower (state in DB/cache) |
| **Best for** | Gaming sessions, long connections | REST APIs, microservices |

> **Rule:** Stateless = scalable. Stateful = sticky. REST APIs are stateless by design — that's why they scale so well.

---

## Trade-Off 4: REST vs RPC

Both are ways for services to communicate. REST thinks in **things (nouns)**. RPC thinks in **actions (verbs)**.

```
REST (resource-based):             RPC (action-based):
GET    /users/123                  getUser(123)
POST   /orders                     createOrder(data)
DELETE /users/123/posts/5          deletePost(userId=123, postId=5)
                                   chargeCard(userId, amount, token)
```

| | REST | RPC |
|--|------|-----|
| **Style** | Resource-based (nouns) | Action-based (verbs) |
| **Protocol** | HTTP only | HTTP, TCP, gRPC, etc. |
| **Best for** | Public APIs, CRUD operations | Internal microservice communication |
| **Data format** | JSON/XML | Custom (often binary with gRPC) |
| **Real world** | Twitter API, GitHub API | Google internal services (gRPC) |

> **Rule:** Public API? Use REST. Internal high-performance microservice calls? Use RPC (specifically gRPC).

---

## Trade-Off 5: Batch vs Stream Processing

Two ways to process data. The key difference is **when** processing happens.

```
Batch Processing:                   Stream Processing:
data accumulates → process all      data arrives → process immediately
at once (scheduled intervals)       (milliseconds latency)

Like: doing laundry once a week     Like: washing each item as it gets dirty
```

| | Batch | Stream |
|--|-------|--------|
| **Latency** | High (hours to days) | Low (milliseconds) |
| **Cost** | Cheap (runs once) | Expensive (always running) |
| **Complexity** | Simple | Complex |
| **Use case** | Reports, payroll, ETL, backups | Fraud detection, live dashboards, IoT |

**Real world:**
- Netflix uses **batch** for "Top 10 in your country" (daily update is fine)
- Netflix uses **stream** for detecting if servers are down RIGHT NOW

---

## Trade-Off 6: Long Polling vs WebSockets

Both solve the same problem: how does the **server push updates to the client** without the client constantly asking?

```
LONG POLLING:
Client → "Any new messages?" → Server waits... → "YES! Here's one" → Client disconnects
Client → "Any new messages?" → Server waits... → timeout → Client tries again
[Repeated HTTP requests — high overhead]

WEBSOCKETS:
Client → Handshake → Connection opened
Both sides send messages freely at any time
Connection stays open until closed
[Like a phone call — one connection, bidirectional]
```

| | Long Polling | WebSockets |
|--|-------------|-----------|
| **Connection** | New HTTP request each time | One persistent connection |
| **Latency** | Higher (request overhead) | Lower (connection already open) |
| **Server load** | Higher (many connections) | Lower (one connection per client) |
| **Use when** | Occasional updates (notifications) | Real-time: chat, gaming, live collaboration |

> **Rule:** WebSockets for anything real-time. Long polling as a fallback when WebSockets aren't supported.

---

## Trade-Off 7: Normalization vs Denormalization

**Normalization** — split data into related tables, store each fact once. Reduces redundancy, protects integrity.

**Denormalization** — combine data into fewer tables, duplicate data. Improves read performance by eliminating JOINs.

```
NORMALIZED (2 tables):              DENORMALIZED (1 table):
CUSTOMER: id, name, email           CUSTOMERORDER: id, name, email,
ORDER: id, customer_id, date, amt   date, quantity, price
→ Need JOIN to get customer name    → Everything in one query, no JOIN
→ Update email in 1 place ✅        → Update email in 500 order rows 😱
```

| | Normalized | Denormalized |
|--|-----------|-------------|
| **Storage** | Efficient (no duplicates) | Wasteful (lots of duplication) |
| **Write speed** | Fast (update one place) | Slow (update everywhere) |
| **Read speed** | Slower (JOINs are expensive) | Fast (no JOINs) |
| **Data integrity** | High | Risk of inconsistency |
| **Best for** | Write-heavy, financial systems | Read-heavy, analytics |

**Real world:**
- **Bank account** → normalized (data integrity is everything)
- **Twitter feed** → denormalized (1 billion reads/day, JOINs would kill it)

---

## Trade-Off 8: TCP vs UDP

**TCP (Transmission Control Protocol)** — connection-oriented, reliable, ordered delivery. Every packet is guaranteed to arrive in order. Lost packets are retransmitted automatically.

**UDP (User Datagram Protocol)** — connectionless, fires packets and forgets. No handshake, no confirmation, no retransmission. Fast but unreliable.

```
TCP:                                    UDP:
Client: "Hello?" (SYN)                  Client: *just starts blasting packets*
Server: "Hello back!" (SYN-ACK)         Server: *catches whatever arrives*
Client: "OK, let's go" (ACK)            [no ceremony, just data]
[THEN data flows]
```

| | TCP | UDP |
|--|-----|-----|
| **Reliability** | Guaranteed delivery | Packets may be lost |
| **Order** | In-order delivery | May arrive out of order |
| **Speed** | Slower (overhead) | Faster (no overhead) |
| **Use when** | Every byte matters | Speed matters more than perfection |
| **Examples** | File download, email, banking | Video calls, gaming, DNS, live streaming |

**Why Zoom uses UDP, not TCP:**
> If a video packet from 2 seconds ago is lost, TCP freezes and retransmits it. By the time it arrives, it's stale — the video stutters. UDP just drops that old frame and shows the next one. A tiny visual glitch beats a freeze every time.

> **Memory trick:** TCP = phone call (establish connection, then talk). UDP = shouting across the room (just shout, hope they hear).

---

# Architectural Patterns

> These are the high-level blueprints for how entire systems are structured.

---

## Pattern 1: Client-Server Architecture

The most fundamental pattern. The system is split into two roles:

- **Client** — user-facing (browser, mobile app). Sends requests, displays results.
- **Server** — processes requests, manages data, sends responses back.

```
[Clients]  ──── request ────►  [Server + DB]
[Clients]  ◄─── response ────  [Server + DB]

Browser, Mobile App              Backend + Database
```

**When to use:** Almost always as the base layer. Every web app, mobile app, and API uses client-server at its core.

---

## Pattern 2: Microservices Architecture

Split a large application into small, **independently deployable services** — each owning a single business capability and its own database.

```
[Auth Service]  [Todo Service]  [Notification Service]  [File Service]
    own DB          own DB            own DB                own DB
        ↕               ↕                 ↕                    ↕
                    [Message Queue / REST APIs]
```

| | Monolith | Microservices |
|--|---------|--------------|
| **Build complexity** | Simple | Complex |
| **Fault isolation** | One crash = all down | One service down, rest keep running |
| **Independent scaling** | Scale everything | Scale only what's needed |
| **Independent deployment** | Redeploy whole app | Deploy only changed service |
| **Best for** | Early-stage startups | Large teams, large scale |

> **Rule:** Start with a monolith. Break into microservices when the pain of NOT doing so is greater than the complexity of doing so.

---

## Pattern 3: Serverless Architecture

No server management. Write a **function**, upload it, the cloud provider runs it only when triggered — and you pay only for execution time.

```
Traditional Server:                 Serverless:
Rent server 24/7                    Write a function
Install dependencies                Upload to AWS Lambda
Configure auto-scaling              Cloud handles everything
Pay even when idle 💸               Pay ONLY for ~33 mins/day used ✅
```

**When to use serverless:**
- Event-driven tasks (resize image on upload, send email on signup)
- Infrequent workloads (not running 24/7)
- Unpredictable traffic spikes

**When NOT to use serverless:**
- Constant heavy traffic (dedicated server becomes cheaper)
- Long-running tasks (15 min time limit on most platforms)
- Ultra-low latency required (cold start adds delay)

---

## Pattern 4: Event-Driven Architecture (EDA)

Services communicate by **emitting and consuming events** through an event channel (message queue). No service calls another directly.

```
User places order
       ↓
[Order Service] emits → "ORDER_PLACED" event
                                ↓
        ┌───────────────────────┼────────────────────┐
[Payment Service]       [Inventory Service]    [Email Service]
reacts → charge         reacts → reserve       reacts → confirm
```

**Three superpowers:**

| Benefit | What it means |
|---------|--------------|
| **Loose coupling** | Services don't need each other online simultaneously — events queue up |
| **Easy to extend** | Add a new consumer without changing any existing service |
| **Independent scaling** | Each consumer scales based on its own load |

**Real world:** Every Amazon order triggers payment, warehouse, delivery tracking, and confirmation email — all independently, all in parallel, via events.

---

## Pattern 5: Peer-to-Peer (P2P) Architecture

No central server. Every node acts as both a **client AND a server** simultaneously, sharing resources directly with other peers.

```
Client-Server:                  P2P:
[Server] ← 1M users             Each peer shares with neighbors
Server crashes = gone            1 peer crashes = nobody notices
More users = server struggles    More users = network gets STRONGER
```

| | P2P | Client-Server |
|--|-----|--------------|
| **Single point of failure** | None | The central server |
| **Infrastructure cost** | Near zero | Expensive servers |
| **Security/control** | Hard to control | Centralized control |
| **Scalability** | Gets stronger with more peers | Server must scale to handle load |
| **Real world** | BitTorrent, Blockchain, early Skype | Netflix, Google, Instagram |

---

# 7-Step System Design Interview Template

> Use this framework for EVERY system design question. It shows structured thinking, which is what interviewers evaluate.

## Step 1: Clarify Requirements

Before drawing a single box, ask the interviewer questions in two categories:

**Functional Requirements** (WHAT the system does):
- What are the core features?
- Who are the users and how do they interact with the system?
- What data types does the system handle (text, images, video)?
- Are there any third-party integrations needed?

**Non-Functional Requirements** (HOW WELL it performs):
- Is the system read-heavy or write-heavy?
- Can the system have downtime, or must it be highly available?
- What are the latency requirements?
- How critical is data consistency?
- Should we rate-limit users?

> **Interview tip:** Never skip this step. Jumping straight to design without clarifying is the #1 mistake candidates make.

## Step 2: Capacity Estimation

Estimate scale to understand what you're building. For WhatsApp:
```
2 billion users, ~100M active daily
Each user sends ~50 messages/day
→ 5 billion messages/day = ~58,000 messages/second

Storage: avg message = 100 bytes
→ 5B × 100 bytes = 500GB/day of new data
```

This tells you: you need horizontal scaling, sharding, and CDNs. Without numbers, you're guessing.

**Estimate these 5 things:**
1. Daily/monthly active users + peak concurrent users
2. Read/write requests per second
3. Storage needed per day/year
4. Cache memory needed for hot data
5. Network bandwidth at peak load

> *Note: Check with the interviewer whether capacity estimation is required — some skip it.*

## Step 3: High-Level Design

Sketch a block diagram of the major components:

```
[Clients: mobile/web]
        ↓
[Load Balancer]
        ↓
[Application Servers] ←→ [Cache (Redis)]
        ↓
[Message Queue]    [Database (sharded)]
        ↓                    ↓
[Worker Services]    [Object Storage (S3)]
        ↓
[Notification Service]
```

Major components to consider:
1. Clients (mobile, web, API consumers)
2. Load Balancers (distribute traffic)
3. Application Servers (process requests)
4. Databases (structured data)
5. Cache (fast read layer)
6. Object Storage (files, images, videos)
7. Message Queues (async tasks)
8. External Services (payment gateways, email)

## Step 4: Database Design

1. **Data Modeling** — identify entities, relationships, attributes, unique identifiers
2. **Choose the right storage:**
   - Relational (MySQL/PostgreSQL) → structured data, complex queries, ACID transactions
   - NoSQL (MongoDB/Cassandra) → unstructured data, high scalability, eventual consistency
   - Key-Value (Redis) → sessions, caching, leaderboards
   - Time Series → metrics, logs, IoT data
3. **Apply normalization or denormalization** based on read/write ratio
4. **Plan for indexing** on frequently queried columns

## Step 5: API Design

Define how components communicate with external clients:

1. List the APIs you'll expose (based on functional requirements)
2. Choose an API style: RESTful, GraphQL, or RPC
3. Choose communication protocols:
   - **HTTPS** → RESTful APIs, standard web communication
   - **WebSockets** → real-time bidirectional (chat, gaming)
   - **gRPC** → efficient internal microservice communication
   - **AMQP/MQTT** → async messaging with message queues

## Step 6: Dive Deep into Key Components

The interviewer will guide you to focus on specific areas. Common deep-dives:

- **Databases** → how do you handle massive data growth? (sharding, replication, read replicas)
- **Application Servers** → how do you add more capacity? (horizontal scaling behind load balancer)
- **Caching** → where do you add cache? how do you handle cache invalidation?
- **Real-time features** → how do messages get pushed to clients? (WebSockets, Gossip)
- **Unique ID generation** → how do you generate IDs at massive scale? (UUID, Snowflake ID, Base62)

## Step 7: Address Key Concerns

**Scalability:**
- Horizontal scaling + load balancer
- Cache to reduce DB load
- Database indexes for fast queries
- Sharding for massive datasets
- Denormalize for read-heavy workloads

**Reliability:**
- Eliminate single points of failure (replicate everything)
- Geographical redundancy (multi-region deployment)
- Circuit breakers (prevent cascading failures)
- Retry with exponential backoff
- Comprehensive monitoring and alerting

> **The golden rule:** Always think out loud. The interviewer cares MORE about your reasoning than your final answer.

---

## Quick Revision — Everything in One Place

| Concept | One-Line Summary | Key Interview Answer |
|---|---|---|
| **Scalability** | Handle more load by adding resources | Horizontal > Vertical at scale |
| **Availability** | % of time system is up | 99.99% = Four Nines = 52min downtime/year |
| **Latency** | Time for 1 request to complete | Optimize for real-time (games, calls) |
| **Throughput** | Work done per second (RPS/TPS) | Optimize for bulk (streaming, ETL) |
| **CAP Theorem** | Can't have C+A+P simultaneously | Real choice is CP vs AP |
| **Load Balancer** | Distribute traffic across servers | 5 algorithms — match to use case |
| **Databases** | 6 types for different data shapes | Match data shape to DB type |
| **CDN** | Serve content from nearest server | Static, public content only |
| **Message Queue** | Async communication via buffer | Producer → Queue → Consumer |
| **Rate Limiting** | Protect from request floods | Token Bucket for bursts, Leaky for smooth |
| **DB Indexes** | B-Tree lookup table for fast reads | O(n) → O(log n), costs write speed |
| **Caching** | Fast temporary data layer | Cache-Aside most popular (Redis) |
| **Consistent Hashing** | Hash ring for smooth rebalancing | Only 1/n data moves when server added |
| **Sharding** | Split DB horizontally into shards | Last resort — adds huge complexity |
| **Consensus** | Elect one leader, avoid split brain | Raft > Paxos for understandability |
| **Proxy Servers** | Intermediary between client/server | Forward hides client, Reverse hides server |
| **Heartbeats** | Periodic health ping | No heartbeat = server dead → react! |
| **Checksums** | Fingerprint to detect corruption | Recalculate and compare on arrival |
| **Service Discovery** | Services find each other via a registry | Registry + Heartbeats = always up-to-date |
| **Bloom Filters** | Probabilistic "have I seen this?" check | Can lie about YES, never lies about NO |
| **Gossip Protocol** | Decentralized info-spreading between nodes | No master needed — scales to 1000s of nodes |

### Trade-Offs at a Glance

| Trade-Off | Pick Left When... | Pick Right When... |
|-----------|------------------|--------------------|
| **Vertical vs Horizontal** | Small scale, simplicity | Large scale, no single point of failure |
| **Strong vs Eventual Consistency** | Banking, inventory | Social feeds, DNS |
| **Stateful vs Stateless** | Long connections, gaming | REST APIs, microservices |
| **REST vs RPC** | Public APIs, CRUD | Internal services, high performance |
| **Batch vs Stream** | Reports, ETL, backups | Fraud detection, real-time analytics |
| **Long Polling vs WebSockets** | Occasional updates | Real-time chat, gaming |
| **Normalized vs Denormalized** | Write-heavy, integrity critical | Read-heavy, speed critical |
| **TCP vs UDP** | Every byte matters | Speed over perfection (video, gaming) |

### Architectural Patterns at a Glance

| Pattern | Core Idea | Real World |
|---------|-----------|-----------|
| **Client-Server** | Client requests, server responds | Every web/mobile app |
| **Microservices** | Small independent services, own DB each | Netflix, Amazon, Uber |
| **Serverless** | Functions run only when triggered | AWS Lambda, image resizing |
| **Event-Driven** | Services react to events, loosely coupled | Amazon orders, Kafka pipelines |
| **P2P** | Every node is client AND server | BitTorrent, Blockchain |
