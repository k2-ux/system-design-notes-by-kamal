# What Interviewers Are REALLY Evaluating in System Design Interviews

> Advice from someone who has sat on both sides of the table. This is what gets written down, what kills candidates, and what makes you stand out.

---

## Section 1: The Scoring Rubric (What They Actually Write Down)

Most top companies (Google, Meta, Amazon, etc.) score candidates on 4–5 dimensions. The interviewer has a form. Every answer you give maps to one of these boxes.

### Dimension 1: Problem Scoping

Did you ask the right questions before diving in? Did you define the problem correctly?

| Good | Bad |
|---|---|
| Asks about scale: "How many DAU? Read vs write ratio?" | Jumps straight into drawing boxes |
| Clarifies what "the system" means: "Just the backend, or including client?" | Assumes what the system needs to do |
| Defines success criteria: "Is latency the priority, or consistency?" | Treats all requirements as equally important |
| Writes down constraints before starting | Never revisits requirements |

### Dimension 2: Technical Depth

Do you know HOW things actually work — not just what they're called?

| Good | Bad |
|---|---|
| "A B-tree index is fast for range queries but slows writes due to rebalancing" | "I'll add an index to make it faster" |
| "Redis is single-threaded, so it handles 100k ops/sec on a single node" | "I'll use Redis for caching" (no further detail) |
| Knows why Kafka uses sequential disk writes for throughput | Names Kafka without knowing what a partition is |
| Can explain what happens at the network layer under load | Only thinks about the application layer |

### Dimension 3: Scalability Thinking

Can you find where the system breaks and propose real solutions?

| Good | Bad |
|---|---|
| "At 10M users, the DB becomes the write bottleneck — here's how we'd shard" | "We just scale it up" (no specifics) |
| Identifies specific bottlenecks: CPU, memory, network, disk I/O | Treats "slow" as a single problem |
| Thinks in phases: "At 1M users this works; at 100M we'd need to change X" | Designs for infinite scale from day one |
| Knows the difference between vertical and horizontal scaling tradeoffs | Says "scale horizontally" without explaining how |

### Dimension 4: Communication

Do you think out loud? Can a teammate follow your reasoning?

| Good | Bad |
|---|---|
| Narrates decisions: "I'm choosing X over Y because in this case..." | Silent for 2 minutes then presents a finished diagram |
| Draws as they talk, labels everything | Draws a mess and explains it afterward |
| Checks in: "Does this make sense? Should I go deeper here?" | Never engages the interviewer |
| Admits uncertainty directly: "I'm not 100% sure of the internal implementation, but logically..." | Bluffs confidently on things they don't know |

### Dimension 5: Practicality

Would this system actually work? Or is it a whiteboard fantasy?

| Good | Bad |
|---|---|
| Starts simple: "I'd begin with a monolith and extract services only when needed" | Proposes 12 microservices for a todo app |
| Considers operational complexity: "Cassandra is powerful but the ops burden is high" | Picks the most impressive-sounding tech |
| Acknowledges what they're NOT solving: "I'm not handling auth today" | Tries to solve every problem in 45 minutes |
| Sizes the system: "At this write rate, one Postgres instance handles it fine" | Can't estimate whether their design is necessary |

---

## Section 2: The Tradeoff Rule

This is the single most important thing to internalize.

**Never say "I'll use X" without saying "The downside of X is Y, but we accept it because Z."**

If you say "I'll use X" and stop there, you sound like you read a list of technologies. If you complete the tradeoff sentence, you sound like an engineer.

### Tradeoff Templates in Practice

**Caching:**
> "I'll use Redis for caching. The tradeoff is eventual consistency — reads may return stale data for up to 60 seconds. That's acceptable for a social feed where freshness isn't critical, but would NOT be acceptable for a banking balance."

**Sharding:**
> "I'd shard by user_id. This means cross-shard queries become expensive — anything that aggregates across users requires scatter-gather. But since 95% of our queries are single-user lookups, that's a tradeoff we accept."

**Async vs Sync:**
> "I'd make the notification system async via a message queue. The downside is delivery isn't guaranteed to be immediate, and we need to handle retries and dead-letter queues. But this decouples the critical write path from a non-critical notification path."

**Consistency model:**
> "I'd use eventual consistency for the follower count. The exact number being off by a few for a few seconds is fine. If this were inventory for an e-commerce checkout, I'd need strong consistency instead."

**SQL vs NoSQL:**
> "PostgreSQL for now — it gives us ACID guarantees and makes complex queries easy. If we hit write throughput limits at scale, I'd consider moving the high-write tables to Cassandra, but that migration has real operational cost."

### The Habit

Every time you name a technology or make a design decision, complete this sentence mentally:

> "I choose [X]. The downside is [Y]. We accept this because [Z relevant to our requirements]."

If you can't complete it, you don't know the technology well enough to use it in the interview.

---

## Section 3: The 10 Most Common Mistakes (And How to Fix Them)

### Mistake 1: Jumping into design without clarifying requirements

**What it looks like:** The interviewer says "Design Twitter" and within 30 seconds you're drawing a load balancer.

**Why it's bad:** You might solve the wrong problem. Twitter for 100 users is architecturally different from Twitter for 100M users. The interviewer is testing whether you ask.

**The fix:** Force yourself to ask exactly 3 questions before touching the whiteboard.
- What's the scale? (DAU, requests/sec, data volume)
- What's the priority? (latency vs consistency vs availability)
- What's in scope? (which features, which clients)

Write the answers visibly before drawing anything.

---

### Mistake 2: Getting stuck in a rabbit hole on one component

**What it looks like:** You spend 25 of 45 minutes designing the database schema in perfect detail. The interviewer never sees the overall architecture.

**Why it's bad:** Interviewers want breadth-then-depth. They can't score your scalability thinking if you never got past the data model.

**The fix:** Timebox explicitly. Tell the interviewer your plan:
> "I'll spend 5 minutes on requirements, 10 on the high-level design, then we can go deep on whatever component you want."

If you catch yourself going deep too early, say: "I could go much deeper here, but let me finish the full design first and come back."

---

### Mistake 3: Designing for perfect consistency everywhere

**What it looks like:** Read replicas with synchronous replication for a user's "last seen" status. Distributed transactions for a like count.

**Why it's bad:** Strong consistency is expensive in every dimension: latency, throughput, availability. Demanding it where it's not needed is a signal that you don't understand the tradeoffs.

**The fix:** For every piece of data, ask: "What is the actual cost of this being slightly stale or slightly wrong?"
- User's bank balance → strong consistency
- Number of likes on a tweet → eventual consistency is fine
- Shopping cart → probably eventual consistency with conflict resolution
- Inventory count at checkout → strong consistency

---

### Mistake 4: Never mentioning failure modes

**What it looks like:** Your design has a single primary database and you never once ask what happens if it goes down.

**Why it's bad:** Production systems fail constantly. Senior engineers think about failure as a first-class concern, not an afterthought.

**The fix:** After designing each major component, ask yourself out loud: "What if this goes down?" Force yourself to address:
- Primary DB failure → replica failover, how long does it take?
- Cache failure → do reads fall through to DB? Can the DB handle that load?
- Message queue failure → do producers block? Do we lose messages?

You don't need to solve every failure. Naming them and showing you've considered them is often enough.

---

### Mistake 5: Using buzzwords without explanation

**What it looks like:** "I'd use Kafka here." Full stop. No explanation of why, what problem it solves, or what the tradeoffs are.

**Why it's bad:** Interviewers have heard every technology name hundreds of times. Naming something you can't explain is worse than not naming it — it invites follow-up questions that expose the gap.

**The fix:** Never name a technology without completing: "I'd use [X] because [specific problem it solves] — the tradeoff is [Y]."

If you can't complete that sentence, don't say the technology name. Describe what you need instead: "I'd use a message queue to decouple the write path from the notification system." The interviewer will accept that even if you can't name the specific tool.

---

### Mistake 6: Over-engineering a simple problem

**What it looks like:** The question is "Design a URL shortener for 10,000 DAU" and you propose multi-region active-active with a distributed ID generation service and event sourcing.

**Why it's bad:** It shows you can't calibrate to the actual requirements. In a real job, this person would be expensive and slow to ship.

**The fix:** Start with the simplest design that works for the given scale. Then explicitly say: "At 10x this scale, I'd change X. At 100x, I'd need to change Y." Show that you can scale incrementally.

The rule of thumb: If one Postgres instance handles the load, use one Postgres instance. Add complexity only when you've demonstrated it's necessary.

---

### Mistake 7: Ignoring the read:write ratio

**What it looks like:** You design the same architecture for both reads and writes without asking which dominates.

**Why it's bad:** A read-heavy system (100:1 read:write) calls for heavy caching and read replicas. A write-heavy system (1:100) calls for write optimization, async processing, and different indexing strategies. Ignoring this leads to wrong architectural decisions.

**The fix:** Always estimate and state the read:write ratio during requirements gathering.
> "For a social feed, I'd estimate this is heavily read-dominant — maybe 100:1 reads to writes. That means we should optimize heavily for reads, and I'd add aggressive caching in front of the DB."

---

### Mistake 8: Treating the database as the only bottleneck

**What it looks like:** Every scaling problem you identify is "the DB can't handle it" and every solution is "add more DB capacity."

**Why it's bad:** Real systems have multiple bottleneck layers. A senior engineer thinks about network bandwidth (especially for video), CPU (for computation-heavy operations), memory (for working sets that don't fit), and disk I/O separately.

**The fix:** Before blaming the DB, go through the stack:
- Network: Can the servers physically receive that much data?
- Application servers: Is the CPU pegged? Are we doing too much serialization?
- Cache: Are we hitting the DB for things that could be cached?
- DB CPU vs DB disk: Is it query complexity (CPU) or data volume (I/O)?

---

### Mistake 9: Going silent while thinking

**What it looks like:** Interviewer asks a hard question. You stare at the whiteboard for 90 seconds in silence. The interviewer is now worried and starts to intervene.

**Why it's bad:** Silence gives the interviewer nothing to score. They can't give you credit for thinking they can't see. It also makes the interviewer anxious, which colors their perception of your performance.

**The fix:** Always narrate your thinking, even when uncertain.

Instead of silence, say:
> "I'm thinking about two options here. Option one is [X] — the concern there is [Y]. Option two is [Z]. Let me think through which fits our requirements better..."

You sound smart when you reason out loud. You sound stuck when you're silent.

---

### Mistake 10: Not knowing the core SQL vs NoSQL tradeoffs

**What it looks like:** "I'd use NoSQL because it's more scalable" — no further justification. Or always defaulting to SQL because it's familiar.

**Why it's bad:** This comes up in nearly every interview. Getting it wrong signals a gap in fundamentals.

**The fix:** Memorize the three core criteria (see Section 6 below). When asked, run through them explicitly.

---

## Section 4: Phrases That Impress Interviewers

Use these exact phrases. They signal engineering maturity.

1. **"The tradeoff here is..."**
   — Use this every time you make a technology or architecture decision.

2. **"At 10x this traffic, this component becomes the bottleneck because..."**
   — Shows you're thinking about scale trajectory, not just the current state.

3. **"We could go with X or Y here. I'd lean toward X because in our specific case [specific reason tied to requirements]..."**
   — Shows you understand that the right answer depends on context.

4. **"Let me handle the happy path first, then we can layer in edge cases and failure modes."**
   — Shows structured thinking and prevents rabbit holes.

5. **"To keep this operationally simple initially, I'd start with [X], and only introduce [Y] when we actually hit [specific scale or problem]."**
   — Shows pragmatism and production awareness.

6. **"I want to validate my assumptions here — is the read:write ratio roughly what I estimated?"**
   — Engages the interviewer and shows you know what matters.

7. **"This design has a single point of failure at [X]. For now that's acceptable, but in production I'd address it by [Y]."**
   — Shows failure mode awareness without getting derailed.

8. **"I'm estimating roughly [X] requests/second at our stated scale, so [single instance / small cluster / large distributed system] should handle it."**
   — Shows you can back your design decisions with numbers.

9. **"I'd instrument [X] with metrics on [specific metric] so we'd know when to scale before it became a problem."**
   — Shows operational maturity. Rare in interviews. Very impressive.

10. **"I'm simplifying here — in a real system I'd also need to think about [auth, rate limiting, monitoring], but I'll park those unless you want to go there."**
    — Shows awareness of what you're omitting. Much better than missing things silently.

---

## Section 5: How to Handle "I Don't Know"

You will get asked something you don't know. This is guaranteed. How you handle it separates senior from junior candidates.

### What NOT to do

- Go silent and freeze
- Make something up and get caught bluffing
- Say "I don't know" and stop there — this gives the interviewer nothing to work with

### What to do instead

**Step 1: Acknowledge the gap directly.**
> "I'm not certain of the exact implementation details here."

**Step 2: Reason from first principles.**
> "But logically, the problem we're trying to solve is [X]. That means we need a solution that can do [Y]. My intuition is that it probably works by [Z]..."

**Step 3: Offer to go deeper with the interviewer's help.**
> "Is this an area you want me to explore? I can walk through my thinking even if I'm not 100% certain of the exact answer."

### Why this works

Interviewers know you don't have perfect recall. What they're testing is: do you have engineering judgment? Can you reason under uncertainty? A candidate who says "I don't know the exact internals of Paxos, but the problem it's solving is reaching agreement among nodes that might fail, so the algorithm probably needs to handle [X], [Y], [Z]..." is more impressive than one who has Paxos memorized but can't reason from first principles.

### The first-principles skeleton

When you don't know something, ask yourself:
- What problem does this solve?
- What constraints would any solution to this problem have?
- What properties would a correct solution need?
- What approaches exist for similar problems?

Reasoning out loud through these four questions will always produce a credible answer, even without prior knowledge.

---

## Section 6: The SQL vs NoSQL Decision

Interviewers ask this in some form in virtually every interview. Have a crisp, opinionated answer ready.

### When to use SQL (relational)

- You need ACID transactions (financial systems, inventory, anything where partial writes are catastrophic)
- You have complex queries involving JOINs across multiple entities
- Your data is relational by nature (users → orders → products)
- Your schema is relatively stable and well-understood
- You're at a scale where a single well-tuned Postgres instance handles your load

**Default to PostgreSQL unless you have a specific reason not to.**

### When to use NoSQL

- You need to store petabytes of data that must be horizontally sharded across hundreds of nodes
- Your access patterns are simple (lookup by key, scan by range) — no complex JOINs
- Your schema changes frequently or varies per record (product catalog with different attributes per type)
- You need to tune for extreme write throughput (Cassandra handles millions of writes/sec)
- You're storing documents, graphs, or time-series data where a document or graph model fits naturally

### The interview-ready answer

When asked "SQL or NoSQL?", say this:

> "I'd start with PostgreSQL. It gives us strong consistency, easy JOINs for our relational data, and handles [your estimated scale] comfortably. The only reason I'd move to NoSQL is if we hit [specific bottleneck — write throughput, dataset size, schema flexibility]. At that point I'd consider [Cassandra for write-heavy time-series, DynamoDB for key-value at scale, MongoDB if schema flexibility is the main driver]."

This answer is almost always correct, shows pragmatism, and demonstrates you know the specific conditions that would change your decision.

---

## Section 7: Red Flags That Kill Your Chances

These are the things that make interviewers write "No Hire" regardless of how much else went well.

**"I'd use microservices"** — said for any system without knowing why. If you can't explain what problem microservices solve for THIS specific system, don't say it. Microservices introduce real complexity: network latency, distributed transactions, operational overhead. Recommending them reflexively signals you're cargo-culting.

**Not knowing what happens when your primary database goes down.** This is a non-negotiable. Every system design interview ends at some point with "what if the DB goes down?" The answer must include: replica failover, detection time, data loss window (RPO), recovery time (RTO), and what clients see during the gap.

**Designing without considering the client.** Mobile and web clients have fundamentally different constraints. Mobile clients deal with unreliable networks, battery usage, and limited bandwidth. This affects whether you push or poll, how large your payloads are, and how you handle offline state. Ignoring the client in a client-facing system is a gap.

**Saying "just scale horizontally" without explaining HOW.** What does horizontal scaling mean for your database? It means sharding — which means picking a shard key, which has tradeoffs. For your application servers, it means statelessness and a load balancer. For your cache, it means consistent hashing. "Just scale" is not an answer.

**Not knowing what a CDN does.** At minimum: a CDN caches static assets (images, JS, CSS, videos) at edge locations geographically close to users, reducing latency and offloading traffic from your origin servers. If you're designing any system with media, you need to know this.

---

## Pre-Interview Checklist

Review these the night before any system design interview.

- [ ] **Know your numbers.** Single Postgres instance: ~10k QPS reads. Redis: ~100k ops/sec. Typical network latency within a datacenter: <1ms. Disk read: ~100 microseconds. Memory read: ~100 nanoseconds. Order of magnitude matters.

- [ ] **Practice the 3-question opener.** Before any design, ask: What's the scale (DAU/QPS)? What's the priority (latency vs consistency vs availability)? What's in scope? Do this until it's automatic.

- [ ] **Refresh the CAP theorem.** In a partition (network split), you choose consistency OR availability. Know which each major database prioritizes: Postgres leans CP, Cassandra leans AP, DynamoDB is configurable.

- [ ] **Know your failure modes.** For every component you use (DB, cache, queue, load balancer), know what happens when it fails and what the mitigation is.

- [ ] **Practice thinking out loud.** Literally talk to yourself or a partner while designing. Going silent is the most common fixable mistake.

- [ ] **Refresh the SQL vs NoSQL criteria.** You will be asked. Have the opinionated answer ready (start with Postgres, here's when I'd change).

- [ ] **Know one caching strategy end-to-end.** Cache-aside (lazy loading) is the default. Know the tradeoffs: cache miss on cold start, stale data on writes, TTL strategy.

- [ ] **Know what a message queue solves.** Decoupling producers and consumers, handling traffic spikes, enabling async processing, guaranteeing at-least-once delivery. Know when NOT to use one: when you need synchronous responses.

- [ ] **Review your tradeoff sentence template.** "I choose [X]. The downside is [Y]. We accept this because [Z]." Practice saying this for 5 different technologies until it's natural.

- [ ] **Time yourself.** Do one full mock design in 45 minutes. If you can't get through requirements + high-level + one deep dive in 45 minutes, you're spending too long in one area.

---

*The candidates who get offers are not the ones who know the most technologies. They're the ones who think clearly under pressure, communicate their reasoning, and know why they're making each decision. Every skill on this list is learnable with deliberate practice.*
