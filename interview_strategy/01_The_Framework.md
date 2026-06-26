# The System Design Interview Framework

**Use this every single time. No exceptions.**

The interviewer is not evaluating whether you know the "right" answer. They are evaluating whether you think like a senior engineer — someone who clarifies before building, estimates before architecting, and proactively identifies risk. This framework makes that visible.

---

## The 5-Step Framework

| Step | Time | What You're Doing |
|------|------|-------------------|
| 1. Clarify Requirements | 3–5 min | Understand the problem before solving it |
| 2. Estimate Scale | 3–5 min | Quantify the constraints |
| 3. High-Level Design | 10–15 min | Draw the system, walk the happy path |
| 4. Deep Dive | 10–15 min | Go deep on 2–3 critical components |
| 5. Bottlenecks & Tradeoffs | 5 min | Identify failure points proactively |

---

## Step 1: Clarify Requirements (3–5 min)

### What You're Actually Doing

You are separating what the system **does** (functional) from how well it must **perform** (non-functional). Most candidates skip this and start drawing boxes immediately. That's how you spend 20 minutes designing the wrong thing.

**Functional Requirements** — What actions can users take?
- "Users can post tweets"
- "Users can follow other users"
- "Users can see a feed of tweets from people they follow"

**Non-Functional Requirements** — What are the performance constraints?
- Latency: "Feed must load in under 200ms"
- Availability: "99.99% uptime"
- Consistency: "Eventual consistency is acceptable for feed"
- Scale: "100M DAU, read-heavy"

### The Exact Questions to Ask Every Time

Do not improvise. Ask these four questions in every interview:

1. **"How many users are we designing for — DAU and total?"**
2. **"Is this read-heavy or write-heavy?"**
3. **"Do we need global availability, or is this a single region?"**
4. **"What's the acceptable latency for the core user action?"**

Depending on the system, also consider:
- "Does this need strong consistency, or is eventual consistency okay?"
- "Do we need to support real-time features like notifications or live updates?"
- "Are there any compliance or data retention requirements?"

### What to Say Out Loud

> "Before I start designing, I want to make sure I understand the requirements. Can I ask a few clarifying questions?"
>
> "What's the expected scale — roughly how many daily active users are we targeting?"
>
> "Is this more read-heavy or write-heavy? For example, for a Twitter-like system I'd expect far more reads than writes."
>
> "Do we need to support global users, or is this a single-region deployment to start?"
>
> "What's the latency requirement for loading a user's feed?"

After they answer, summarize back:

> "Okay, so I'm designing for 100M DAU, the system is read-heavy at roughly 10:1 read-to-write ratio, we need global availability, and the feed should load in under 500ms. I'll keep these constraints in mind throughout."

### What NOT to Do

- Do not skip this step to "look fast." Starting without requirements signals poor judgment.
- Do not ask more than 5–6 questions. You're clarifying, not interrogating.
- Do not ask about things you can reasonably assume (don't ask "should it support HTTP?" for a web API).
- Do not write code or name databases yet. You don't know enough.

### What the Interviewer Is Evaluating

**Can this person identify what they need to know before building?** Senior engineers don't code from ambiguous specs. They ask targeted questions, confirm understanding, and align before investing effort. The interviewer is checking whether you do the same.

---

## Step 2: Estimate Scale (3–5 min)

### What You're Actually Doing

You are deriving numbers that will drive every architectural decision. The database choice, the caching strategy, whether you need horizontal scaling — all of it flows from scale estimates.

### The Core Formulas

**QPS (Queries Per Second):**
```
QPS = DAU × avg_requests_per_user_per_day / 86400
```

**Storage per year:**
```
Storage = DAU × avg_data_written_per_user_per_day × 365
```

**Numbers to memorize:**

| DAU | QPS (rough) |
|-----|-------------|
| 1M | ~12 QPS |
| 10M | ~120 QPS |
| 100M | ~1,200 QPS |
| 1B | ~12,000 QPS |

Peak traffic is typically 2–3x average. So 100M DAU = ~1,200 avg QPS = ~3,000 peak QPS.

### Worked Example: Twitter Scale

**Given:** 100M DAU. Users average 2 tweets/day written, and read 50 tweets/day.

**Write QPS:**
```
100M × 2 / 86,400 ≈ 2,300 writes/sec
```

**Read QPS:**
```
100M × 50 / 86,400 ≈ 58,000 reads/sec
```

Read-to-write ratio: ~25:1. This tells you immediately that you need aggressive caching on the read path.

**Storage for tweets (1 year):**
```
100M users × 2 tweets/day × 280 bytes/tweet × 365 days
≈ 100M × 2 × 300 × 365
≈ 21.9 TB/year
```

Media (photos/videos) would add orders of magnitude more — handle separately.

### What to Say Out Loud

> "Let me do some quick back-of-envelope math to understand the scale we're dealing with."
>
> "With 100M DAU and users making about 50 read requests per day on average, that's roughly 100M × 50 / 86,400 — so about 58,000 read QPS. With 2x peak factor, call it 120,000 peak read QPS. That's significant."
>
> "For storage, assuming average tweet size of 300 bytes, we're writing about 2,300 tweets per second, or roughly 20TB of text per year. Media is separate."
>
> "These numbers tell me a few things: the read load is enormous, so caching is critical. The write load is manageable with a single primary DB plus replicas to start."

### What NOT to Do

- Do not skip estimation entirely. Even rough numbers matter.
- Do not get precise. You're not doing accounting. Round aggressively.
- Do not let the math derail you. Spend 3–4 minutes max, state the key numbers, and move on.
- Do not forget to state what the numbers imply. Estimation without insight is useless.

### What the Interviewer Is Evaluating

**Can this person reason quantitatively?** The interviewer wants to see that you use numbers to make decisions, not just to show math. The insight ("this means we need caching") is more important than the exact calculation.

---

## Step 3: High-Level Design (10–15 min)

### What You're Actually Doing

You are drawing the system at the 10,000-foot view. Every major component gets a box and a label. No implementation details yet. No code. Just boxes, arrows, and the happy path.

### The Order to Draw Boxes

Always go in this direction: **Client → API Layer → Services → Storage**

1. **Client** — web browser, mobile app, third-party API consumer
2. **API Gateway / Load Balancer** — single entry point, routes requests
3. **Service(s)** — the business logic layer (one box per domain: User Service, Feed Service, etc.)
4. **Cache** — where applicable, between services and storage
5. **Database(s)** — the storage layer (SQL, NoSQL, object store, etc.)
6. **Async components** — message queues, workers, CDN (add these after the core path)

Name every box. "Service" is not a name. "Feed Generation Service" is a name.

### Walk the Happy Path

After drawing the diagram, walk through what happens when a user performs the core action. Narrate it component by component.

> "Let me walk through the happy path for a user loading their Twitter feed."
>
> "The user's mobile app sends a GET /v1/feed request. It hits the load balancer, which routes it to one of the Feed Service instances. The Feed Service first checks the Redis cache for this user's pre-computed feed. If it's a cache hit — which it will be for most active users — it returns immediately. Cache miss, it falls back to the database, runs the fan-out query, populates the cache, and returns."

### What to Say Out Loud

> "Let me sketch out the high-level architecture first. I'll start with the client and work my way to storage."
>
> "On the left, we have clients — web and mobile. Requests go through a load balancer into our API gateway, which handles auth and routes to the appropriate service. For this system, I see at least three services: a User Service for profile/follow data, a Tweet Service for creating and storing tweets, and a Feed Service for generating the timeline."
>
> "For storage, I'm thinking a relational DB for user and follow relationships, and a wide-column store like Cassandra for tweet storage and fan-out at scale. I'll add a Redis cache in front of the feed read path."

### What NOT to Do

- Do not start with the database. Always start with the client.
- Do not design one giant monolithic "Backend" box. Name the services.
- Do not get into database schemas yet. That's Step 4.
- Do not spend more than 15 minutes here. The interviewer wants you to go deeper.
- Do not stay silent while drawing. Narrate everything.

### What the Interviewer Is Evaluating

**Can this person communicate a system design clearly?** The diagram alone is not enough. The interviewer wants to hear your reasoning — why each component exists, how data flows through the system. Silence while drawing is a red flag.

---

## Step 4: Deep Dive (10–15 min)

### What You're Actually Doing

You are going deep on the components that matter most. The interviewer will often guide this, but you should be prepared to drive it if they don't.

### How to Transition

Do not wait silently. Either let the interviewer guide you, or propose a direction:

> "I'd like to go deeper on the feed generation and delivery — that seems like the most interesting and challenging part of this system. Is that okay, or would you prefer to focus somewhere else?"

If the interviewer redirects, follow them immediately. They're telling you what they care about.

### Common Deep Dive Areas

**Database Design**
- Schema: tables, columns, indexes, foreign keys
- Access patterns: what queries run most? What indexes support them?
- Sharding strategy: shard by user ID? Tweet ID? Time?

**Caching Strategy**
- What to cache: hot user feeds, popular tweet data, session tokens
- Cache invalidation: when a user writes a new tweet, what cache entries are stale?
- Cache policy: LRU? TTL? Write-through vs write-behind?

**API Design**
- Endpoint signatures: `GET /v1/feed?user_id=&limit=&cursor=`
- Pagination: cursor-based (preferred) vs offset-based
- Rate limiting: per-user, per-IP, sliding window vs token bucket

**Real-Time Features**
- WebSockets for live notifications
- Long polling as a fallback
- Fan-out on write vs fan-out on read for feed generation

### How to Show Depth Without Showing Off

Signal that you know multiple approaches and can reason about tradeoffs:

> "For feed generation, there are two main approaches: fan-out on write and fan-out on read."
>
> "Fan-out on write: when a user posts a tweet, you immediately push it to all their followers' feed caches. Reads are instant. But if you have a celebrity with 10M followers, a single write triggers 10M cache writes — that's a write amplification problem."
>
> "Fan-out on read: you compute the feed at read time by querying who the user follows and fetching their recent tweets. No write amplification, but reads become expensive at scale."
>
> "A hybrid approach — which is what Twitter actually uses — does fan-out on write for regular users but fan-out on read for celebrity accounts with follower counts above a threshold. That's what I'd recommend here."

### What NOT to Do

- Do not cover everything at the same depth. Go deep on 2–3 areas, not shallow on everything.
- Do not just name technologies. Explain why you chose them.
- Do not ignore the interviewer's cues. If they keep asking follow-up questions on one topic, that's where they want you.
- Do not pretend there's only one solution. Always present alternatives before picking one.

### What the Interviewer Is Evaluating

**Does this person understand the real complexity?** Anyone can name technologies. The interviewer wants to see that you understand the tradeoffs between approaches and can make a reasoned recommendation. The phrase "we could use X or Y — X is better here because..." is what they're listening for.

---

## Step 5: Bottlenecks & Tradeoffs (5 min)

### What You're Actually Doing

You are proactively identifying what breaks — before the interviewer asks. This is the mark of a senior engineer. Junior engineers wait to be told something is wrong. Senior engineers find it themselves.

### What to Cover

**Single Points of Failure**
- What happens if your primary database goes down?
- What happens if your cache cluster loses data?
- What happens if your message queue fills up?

**Scale Bottlenecks**
- At 10x current traffic, which component fails first?
- At 100x, what's the next constraint?

**Tradeoffs You Made**
- Every architectural decision has a cost. Name it.
- "We chose eventual consistency for the feed, which means a user might see a tweet up to 5 seconds after it's posted. That's an acceptable tradeoff for this system."

### The Phrases That Impress

> "At 10x traffic, the database will be the bottleneck because all feed queries are hitting a single primary. The solution is read replicas for the read path and potentially sharding the write path by user ID."

> "The tradeoff here is consistency vs availability. We chose availability — during a partition, users will see a slightly stale feed rather than getting an error. That's the right call for a social media product."

> "The single point of failure in this design is the message queue. If Kafka goes down, tweet delivery stops. Mitigations: multi-broker cluster, replication factor of 3, consumer group retry logic."

> "We could get faster feed loads by pre-computing feeds for all users, but that costs significant storage and cache memory. Given our read QPS, I think it's worth it for the top 10% of active users."

### What to Say Out Loud

> "Before we wrap up, I want to flag a few potential bottlenecks and the tradeoffs I made."
>
> "The biggest risk at scale is fan-out for high-follower accounts. We handled it with the hybrid approach, but it adds operational complexity — we need logic to decide which users get fan-out on write vs read."
>
> "The database is a potential bottleneck. Right now I have a single primary with read replicas. At 10x traffic, we'd need to shard. I'd shard the tweet table by tweet_id using consistent hashing, and the user table by user_id."
>
> "Finally, the cache is a single region. For global availability, we'd want a distributed cache with region-local replicas, accepting that cross-region data will be eventually consistent."

### What NOT to Do

- Do not wait for the interviewer to point out problems. Find them yourself.
- Do not only mention the problem without a solution. At minimum, name the direction of the fix.
- Do not be defensive about your design. "One weakness of this design is..." is a sign of confidence, not failure.

### What the Interviewer Is Evaluating

**Does this person understand production systems?** Textbook architectures fail at real scale. The interviewer wants to know that you've thought about failure modes, not just the happy path. Proactively identifying bottlenecks signals production experience or strong systems thinking.

---

## The Cheat Sheet

| Step | Time | Goal | Key Phrase to Use | Red Flag to Avoid |
|------|------|------|-------------------|-------------------|
| 1. Clarify Requirements | 3–5 min | Align on what you're building | "Just to confirm, I'm designing for..." | Jumping straight to drawing boxes |
| 2. Estimate Scale | 3–5 min | Quantify constraints that drive architecture | "These numbers tell me that..." | Skipping estimation or not drawing conclusions from it |
| 3. High-Level Design | 10–15 min | Draw every component, walk the happy path | "Let me walk through what happens when a user..." | Drawing a monolith, staying silent while drawing |
| 4. Deep Dive | 10–15 min | Go deep on critical components with tradeoffs | "We could use X or Y — X is better here because..." | Covering everything at surface level, naming tech without reasoning |
| 5. Bottlenecks & Tradeoffs | 5 min | Proactively surface failure modes | "The tradeoff here is..." / "At 10x traffic, X will fail because..." | Waiting for the interviewer to find the problems for you |

---

## The Meta-Rule

**Talk the entire time.** The interviewer cannot evaluate your thinking if it stays in your head. When you're drawing, narrate. When you're considering options, say them out loud. When you're unsure, say "I'm deciding between X and Y" and reason through it aloud.

Silence is the single most common reason candidates fail system design interviews — not lack of knowledge.
