# CAP Theorem + Replication

> "The network is reliable." — First Fallacy of Distributed Computing (it is not)

---

## Part 1: CAP Theorem

---

### Q1: What the heck is CAP? Explain each letter like I'm not a distributed systems PhD.

**CAP** stands for three properties a distributed system *wishes* it could have all at once. Spoiler: it can't.

---

**C — Consistency**

Every read gets the most recent write. If you write `x = 5` on Node A, any node you ask next returns `5`. Not the old value. Not "maybe 5". Definitely 5.

```
Client writes x=5 to Node A
        |
        v
   [Node A: x=5] ----sync----> [Node B: x=5]

Client reads x from Node B
        |
        v
     Returns: 5  <-- guaranteed fresh
```

Think of it like a bank ledger. If you deposit $1000, you expect every ATM to show the updated balance instantly. That's consistency.

---

**A — Availability**

Every request gets *a response* — success or failure, but never "I'm busy, come back later." The system stays up and responsive even if some nodes are sick.

Key word: *every request gets a response*. The response might be slightly stale, but you get one. No timeouts. No errors. Just an answer.

---

**P — Partition Tolerance**

The system keeps working even when the network splits and some nodes can't talk to each other. A **partition** is when Node A and Node B lose their network link — they're both alive, but isolated.

```
     [Node A]           [Node B]
         \                 /
          \               /
           X -- SPLIT -- X
          (network partition)

Both nodes are alive, but can't talk to each other.
What do they do?
```

---

### Q2: Why can you only pick 2 of 3? Why is P always forced on you?

Here's the triangle everyone draws:

```
              Consistency (C)
                    /\
                   /  \
                  /    \
                 /  ???  \
                /----------\
  Availability(A)          Partition
                          Tolerance(P)

  CA systems: Traditional single-node DBs (MySQL on one machine)
  CP systems: ZooKeeper, HBase, Postgres (with sync replication)
  AP systems: Cassandra, CouchDB, DynamoDB
```

**Why you can't have all three:**

Imagine a partition happens (Node A can't talk to Node B). Now a write comes in to Node A.

You have exactly two choices:

1. **Reject the write** (stay consistent, sacrifice availability) — "Sorry, I can't confirm this replicated safely." That's CP.
2. **Accept the write** (stay available, sacrifice consistency) — "Sure, I'll take it. Node B will catch up later." That's AP.

There is no third option. You *must* choose.

**And here's the dirty secret: you always need P.**

In a single-machine system, you can have CA (no partitions possible). But the moment you have more than one machine — and you need to for any real scale — partitions *will* happen. Network cables get cut. AWS regions go down. A misconfigured firewall drops packets.

The Amazon Dynamo paper (2007) put it bluntly: partition tolerance is non-negotiable for a system that has to run across data centers. So the *real* choice is always **CP vs AP**.

---

### Q3: What's a CP system? Give me real examples and what happens during a partition.

**CP = Consistency + Partition Tolerance.** During a network partition, the system refuses to serve stale data — it would rather go unavailable than lie to you.

**Real systems:** ZooKeeper, HBase, Etcd, Postgres with synchronous replication.

**What happens during a partition:**

```
  Client --> [Primary Node A]  ~~~X~~~  [Replica Node B]
                  |
                  | (partition! A can't confirm B got the write)
                  |
              Returns: ERROR 503
              "Sorry, can't guarantee consistency right now."
```

ZooKeeper literally stops accepting writes if it loses quorum (majority of nodes). It would rather say "no" than say "yes" and be wrong.

---

**The GitHub 2012 Incident — CP in the wild:**

In October 2012, GitHub had a MySQL failover incident. Their primary MySQL database became unavailable. They had to decide: do we fail over to the replica immediately and risk data inconsistency (some writes might not have replicated yet), or do we wait and keep the site degraded until we're sure the replica is caught up?

GitHub chose **CP behavior** — they kept the site partially down, investigated the replication lag, and only failed over once they were confident in data integrity. Downtime was painful but the alternative was serving inconsistent data to millions of developers (imagine git pushes getting silently lost or corrupted). Data integrity > uptime.

The lesson: for a source control system where losing a commit is *catastrophic*, CP is the right call.

---

### Q4: What's an AP system? How did Netflix and Amazon make peace with "wrong" data?

**AP = Availability + Partition Tolerance.** During a partition, the system keeps answering requests — but the answers might be stale or inconsistent across nodes.

**Real systems:** Cassandra, CouchDB, DynamoDB, Riak.

**What happens during a partition:**

```
  Client A --> [Node A: x=5]  ~~~X~~~  [Node B: x=3]
  Client B --> [Node B: x=3]

  Both get responses! But they get different values.
  The system is "split-brained" until the partition heals.
```

---

**Netflix — "Stale is fine, down is not":**

Netflix's recommendation system runs on AP principles. When you load Netflix, it shows you recommendations. If there's a network partition or a cache node is unreachable, Netflix doesn't show you an error page — it shows you *slightly old* recommendations.

Did you get the "perfect" recommendation set? Maybe not. Did you get *something* that lets you watch TV? Yes. Netflix made the explicit product decision: **stale movie recommendations are infinitely better than a blank screen and an error.** No one is harmed by seeing yesterday's "Top Picks for You."

This is the core insight of AP systems: not all data needs to be perfectly fresh. The cost of staleness varies by use case.

---

**Amazon Dynamo (2007) — The paper that changed everything:**

Amazon's 2007 Dynamo paper is one of the most important papers in distributed systems. Amazon engineers were sick of their shopping cart being unavailable due to consistency-related failures.

The paper explicitly states their design philosophy: **"an 'always writeable' data store."** Amazon decided that it was better for a shopping cart to show slightly stale items than to fail to let a customer add something to their cart (which directly loses money).

Dynamo uses **eventual consistency** + **vector clocks** to handle conflicts, and it lets *multiple* nodes accept writes during a partition. Conflicts are resolved later — sometimes by the application, sometimes by "last write wins."

The key quote from the paper: *"Availability trumps consistency in many scenarios we face."* This became the philosophical foundation for a generation of NoSQL databases.

---

### Q5: Eventual consistency — what does "eventually" actually mean in practice?

"Eventually consistent" sounds vague because it is — on purpose. Here's the deal:

**The promise:** If no new writes come in, all replicas will *eventually* converge to the same value. There's no guarantee on *when*.

In practice, "eventually" is usually **milliseconds to seconds** in a healthy system. But under heavy load, network issues, or geographic distribution, it could be longer.

```
  t=0: Client writes x=10 to Node A
       [Node A: x=10]  [Node B: x=7]  <-- stale!

  t=0.05s: Replication message in flight
       [Node A: x=10]  [Node B: x=7]  <-- still stale

  t=0.1s: Node B receives and applies the update
       [Node A: x=10]  [Node B: x=10]  <-- converged!

  "Eventually" was 100ms in this case.
```

**Techniques used to achieve eventual consistency:**

- **Anti-entropy / Gossip protocols:** Nodes periodically gossip their state to each other. Cassandra does this.
- **Read repair:** When you read from multiple replicas and one is stale, the system fixes it in the background.
- **Hinted handoff:** If a node is down, another node temporarily holds the writes for it and delivers them when it comes back.

---

**The Cassandra / Facebook Inbox Search Postmortem:**

Facebook used Cassandra for inbox search — a use case where you search your messages. Cassandra's AP nature meant that under certain conditions, search indexes could be inconsistent. A message you just sent might not appear in search for a moment. Messages that were deleted might still briefly appear.

Facebook eventually abandoned Cassandra for inbox search and rebuilt on HBase (a CP system), partly because the eventual consistency model created too many edge cases that were hard to reason about and created a poor user experience. The lesson: **eventual consistency is powerful, but it shifts complexity onto the application layer.** You have to handle conflicts, stale reads, and divergence explicitly.

For some use cases, the simplicity of CP systems is worth the availability tradeoff.

---

### Q6: PACELC — Why CAP is only half the story

CAP only talks about what happens *during a partition*. But partitions are rare. What about the 99.9% of the time when everything is fine?

**PACELC** extends CAP:

```
  If there's a Partition (P):
      choose between Availability (A) or Consistency (C)
  Else (E — no partition):
      choose between Latency (L) or Consistency (C)
```

The insight: **even without a partition, you face a latency vs. consistency tradeoff.** If you want every read to be perfectly consistent, you have to synchronously replicate writes and wait for confirmation — that adds latency. If you want low latency, you accept that reads might be slightly stale.

```
  PACELC Classification of Common Systems:

  System        | Partition behavior | Normal behavior
  --------------|-------------------|----------------
  DynamoDB      | PA (available)    | EL (low latency)
  Cassandra     | PA (available)    | EL (low latency)
  ZooKeeper     | PC (consistent)   | EC (consistent)
  MySQL (sync)  | PC (consistent)   | EC (consistent)
  MongoDB       | PA or PC*         | EL or EC*
  
  * depends on configuration (write concern, read preference)
```

**The MongoDB 2013 write concern incident** is a perfect PACELC illustration. MongoDB's default write concern at the time was `w:0` — fire and forget. Write a document, don't wait for acknowledgment. Maximum throughput, minimum latency. But if the server crashed between the write and the flush to disk, your data was gone. Users *assumed* MongoDB was CP (it has a primary/secondary model), but with default settings it was sacrificing consistency for latency in a dangerous way. Several high-profile data loss incidents followed. MongoDB later changed their defaults to be safer, but the lesson stands: **defaults matter enormously, and "looks CP" ≠ "is CP."**

---

## Part 2: Replication

---

### Q7: What is replication and why do we need it?

**Replication** = keeping copies of your data on multiple machines.

Why? Three reasons:

1. **Fault tolerance** — if one machine dies, others have the data
2. **Read scaling** — spread read traffic across multiple machines
3. **Low latency** — put copies geographically close to users

The classic setup is **Primary/Replica** (historically called Master/Slave):

```
                    WRITES
                      |
                      v
              +---------------+
              |   PRIMARY     |
              |  (Leader)     |
              +-------+-------+
                      |
          +-----------+-----------+
          |                       |
          v                       v
  +---------------+       +---------------+
  |   REPLICA 1   |       |   REPLICA 2   |
  |  (Follower)   |       |  (Follower)   |
  +---------------+       +---------------+
          |                       |
          v                       v
        READS                   READS

  Writes go to Primary only.
  Reads can go to Primary OR Replicas.
```

The primary receives all writes and replicates them to replicas. Replicas are read-only copies (in the standard setup).

---

### Q8: Synchronous vs Asynchronous replication — what's the difference and when do you use each?

This is one of the most important tradeoffs in database configuration.

---

**Synchronous Replication:**

```
  Client
    |
    | WRITE "x=5"
    v
  [Primary]
    |
    |------ replicate ------> [Replica]
    |                              |
    |<---- "ack received" ---------|
    |
    | Returns "write successful" to client

  Client only gets success AFTER replica confirms.
```

**Pros:** Zero data loss. If the primary dies right after a write, the replica has it.
**Cons:** Latency is now limited by the slowest replica. If a replica is slow or network is laggy, every write slows down.

---

**Asynchronous Replication:**

```
  Client
    |
    | WRITE "x=5"
    v
  [Primary]
    |
    | Returns "write successful" to client IMMEDIATELY
    |
    |------ replicate (eventually) ------> [Replica]

  Client doesn't wait for replica confirmation.
```

**Pros:** Low write latency. Primary isn't blocked by slow replicas.
**Cons:** If the primary crashes between the write and the replication, data is lost. The replica doesn't know about the write yet.

---

**Semi-Synchronous (the compromise):**

Most production systems use a hybrid. For example, MySQL's semi-sync replication waits for *at least one* replica to acknowledge, then returns success. You get durability without waiting for *every* replica.

PostgreSQL's synchronous replication lets you configure `synchronous_standby_names` to specify exactly which replicas must confirm before a write succeeds.

The GitHub 2012 incident touched this directly — their MySQL replica had **replication lag** and wasn't fully caught up when the primary went down. With fully synchronous replication, that data loss risk wouldn't exist. The tradeoff: their write performance would have been worse during normal operation.

---

### Q9: Read replicas — how do you scale reads without scaling writes?

**The problem:** Your database is getting hammered. Most traffic is reads (say, 90% reads, 10% writes). Adding more primary nodes doesn't help because you'd need to coordinate writes across all of them.

**The solution:** Add read replicas. Direct all reads there.

```
  Application Layer
    |          |
    | writes   | reads
    v          v
  [Primary]  [Replica 1]  [Replica 2]  [Replica 3]
      |          ^              ^             ^
      |          |              |             |
      +----async replication----+-------------+

  Write throughput: limited to Primary's capacity
  Read throughput: scales horizontally with replicas
```

This is how most high-traffic relational database setups work. Instagram, Twitter, and most large web apps use this pattern extensively.

**How it works in practice:**
- Your application connection pool has a "write connection" pointing to the primary
- A "read connection" (or pool) pointing to one or more replicas
- An ORM like ActiveRecord (Rails) or Sequelize (Node) can route queries automatically

```python
# Pseudocode showing read/write splitting
class UserRepository:
    def create_user(self, data):
        return self.write_db.insert("users", data)   # goes to primary

    def get_user(self, user_id):
        return self.read_db.select("users", user_id) # goes to replica
```

**Caveat:** After a write, immediately reading your own write from a replica might return stale data (due to replication lag). Applications need to handle this — e.g., "read your own writes" consistency, where post-write reads are directed to the primary for a short window.

---

### Q10: Replication lag — what goes wrong when a replica falls behind?

**Replication lag** is the delay between a write hitting the primary and that write appearing on the replica. In a healthy system this is milliseconds. When things go wrong, it can be seconds, minutes, or... days (yes, really).

**What causes lag:**
- Network congestion between primary and replica
- Replica hardware is slower than primary
- Large transactions take a long time to apply on the replica
- Replica is under heavy read load and can't keep up with applying writes

```
  t=0:    Primary processes 1000 writes/sec
  t=5s:   Replica is processing writes from t=3s  <-- 2 second lag

  A user writes their profile at t=4s.
  At t=5s they refresh the page, hit the replica.
  The replica doesn't have their t=4s write yet.
  They see their OLD profile.

  This is the "read your own writes" problem.
```

**Real-world consequences of replication lag:**

1. **Phantom reads:** User updates their email, then "Edit Profile" page still shows the old email (loaded from lagged replica)
2. **Double-posting:** User thinks their post failed (stale replica didn't show it), submits again
3. **Broken foreign keys:** Record on replica references a parent row that hasn't replicated yet (consistency violation)

**Mitigation strategies:**

- **Sticky sessions / read-your-own-writes:** Route the same user's reads to the primary (or the same replica) for a short window after a write
- **Replica lag monitoring:** Alert if lag exceeds threshold (5 seconds, 30 seconds, etc.)
- **Synchronous replication for critical data:** Accept the latency cost for important writes
- **Causal consistency:** Track version tokens and only serve reads that are "at least as fresh as" the user's last write

---

### Q11: Multi-master replication — when you need to write to multiple nodes

**The setup:** Multiple nodes can accept writes. Great for geographic distribution — write to the nearest data center, don't take the round-trip latency hit.

```
  User in US East          User in EU West
        |                        |
        | write                  | write
        v                        v
  [Master US-East] <--sync--> [Master EU-West]
        |                        |
     Reads                    Reads
```

**The problem: write conflicts.**

What happens when two users write to the same record at the same time, hitting different masters?

```
  t=0: User A (on US master): updates profile photo to "cat.jpg"
  t=0: User B (on EU master): updates SAME profile photo to "dog.jpg"

  US master: photo = "cat.jpg"
  EU master: photo = "dog.jpg"

  Which one wins?
```

**Conflict resolution strategies:**

1. **Last Write Wins (LWW):** Timestamp-based. Whoever has the higher timestamp wins. Problem: clocks are not perfectly synchronized across machines (clock skew). Also, you silently discard data.

2. **First Write Wins:** Reject any write to a record that's already been modified since the version you read. Safer but more rejections.

3. **Application-level resolution:** Store both conflicting values, let the application (or user) decide. This is what Amazon Dynamo does with shopping carts — merge the carts from conflicting versions.

4. **CRDTs (Conflict-free Replicated Data Types):** Special data structures designed so that concurrent writes can always be merged deterministically. Works for specific data types (counters, sets, etc.), not general purpose.

**The tradeoff:** Multi-master is complex. Most systems avoid it unless they genuinely need geographic write performance. Single-primary is much easier to reason about.

---

### Q12: What happens when the primary dies? Leader election explained.

The primary is down. Chaos. What now?

```
  Normal state:
  [Primary] --> [Replica 1]
             --> [Replica 2]

  Primary dies:
  [   DEAD  ]    [Replica 1]  <-- who's in charge now?
                 [Replica 2]
```

**Manual failover (old-school):**
A human operator decides which replica to promote. Safe but slow — can take minutes or hours. GitHub's 2012 incident involved significant manual intervention time because they wanted to verify replica lag before promoting.

**Automatic failover:**
Systems like MySQL Group Replication, Postgres Patroni, or MongoDB's replica sets detect primary failure and automatically elect a new primary. Faster but requires consensus algorithms.

---

**Raft and Paxos — how do replicas agree on who's the new boss?**

When the primary goes down, replicas need to elect a new leader without talking to the old primary (it's dead). But they also can't just shout "I'm the leader!" at the same time — that creates split-brain (two nodes both think they're primary, both accept writes, chaos).

**Raft** (the readable one) works like this:

```
  1. Replica detects primary is gone (no heartbeat)
  2. It starts an election: "Vote for me! I'm Candidate for term #5"
  3. Other replicas vote (they'll vote for whichever candidate they hear from first,
     IF that candidate's log is at least as up-to-date as theirs)
  4. Candidate needs MAJORITY of votes (quorum)
  5. Winner becomes the new primary and tells everyone

  With 5 nodes: need 3 votes to win (majority)
  If network splits into 2+3: only the group of 3 can elect a leader
  (The 2-node group can't reach majority — no split brain!)
```

**Paxos** is the original algorithm (older, more complex, equivalent guarantees). Both ensure that in any network split, at most one node can be elected leader, because you need a majority and you can only have one majority.

**ZooKeeper** uses a Paxos variant. **Etcd** and **CockroachDB** use Raft. **MongoDB** uses its own Raft-like protocol.

**The tradeoff:** To elect a leader, you need quorum (majority). If you have 3 nodes and 2 die, you can't elect a new leader — the system halts. This is the CP choice: prefer being unavailable over electing a potentially stale leader.

---

## Quick Revision Table

### CAP Theorem at a Glance

| Concept | Plain English | What You Sacrifice | Example Systems |
|---|---|---|---|
| **Consistency (C)** | Every read gets the latest write | Potentially: availability or latency | |
| **Availability (A)** | Every request gets a response (no errors/timeouts) | Potentially: freshness of data | |
| **Partition Tolerance (P)** | Works even when network splits | Must always have this in distributed systems | |
| **CP System** | Refuses requests during partition rather than serve stale data | Availability during partition | ZooKeeper, HBase, Etcd, Postgres (sync) |
| **AP System** | Serves requests during partition even if data is stale | Consistency during partition | Cassandra, DynamoDB, CouchDB, Riak |
| **Eventual Consistency** | All nodes will agree... eventually | Immediate consistency | Cassandra, DynamoDB, DNS |
| **PACELC** | CAP + Latency vs Consistency even without partitions | Latency OR Consistency always | Whole classification system |

---

### Replication at a Glance

| Concept | What it is | Key Tradeoff |
|---|---|---|
| **Primary/Replica** | One node takes writes; copies data to read-only replicas | Single write bottleneck; read scalability |
| **Sync Replication** | Write confirmed only after replica acknowledges | Zero data loss vs. higher write latency |
| **Async Replication** | Write confirmed immediately; replica catches up later | Low latency vs. potential data loss on crash |
| **Read Replica** | Route read queries to replica nodes | Scales reads horizontally; stale reads possible |
| **Replication Lag** | Replica is behind the primary by N seconds | Stale reads; "read your own writes" bugs |
| **Multi-Master** | Multiple nodes accept writes | Write conflicts; complex conflict resolution |
| **Leader Election** | Surviving replicas pick a new primary via quorum | Safety (no split-brain) vs. availability during election |
| **Raft/Paxos** | Consensus algorithms ensuring at-most-one leader | Halt if majority unavailable (CP behavior) |

---

### The Real-World Cheat Sheet

| Company / System | Choice | Reasoning |
|---|---|---|
| **Amazon Dynamo** | AP | Shopping cart availability > perfect consistency |
| **Netflix recommendations** | AP | Stale recommendations are fine; downtime is not |
| **GitHub MySQL 2012** | CP (manual) | Losing commits is catastrophic; downtime acceptable |
| **Facebook inbox search on Cassandra** | AP → eventually abandoned | Eventual consistency too complex for search correctness |
| **MongoDB 2013 defaults** | Accidentally neither | Default w:0 write concern meant data loss on crash |
| **ZooKeeper** | CP | Distributed coordination requires strong guarantees |

---

### The Three Questions to Ask for Any System

1. **What's the cost of stale data?**
   - Life-critical (banking, medical): CP, pay the availability cost
   - User-facing but not critical (feeds, recommendations): AP, staleness is fine

2. **What's the cost of downtime?**
   - Revenue-generating (shopping carts, checkout): AP, stay up no matter what
   - Internal tooling: CP, correctness over availability

3. **What's the write pattern?**
   - Mostly reads: Add read replicas
   - Geographically distributed writes: Consider multi-master (carefully)
   - Single-region, mixed load: Primary/replica with async replication is usually fine

---

*Next up: Consistent Hashing and Sharding — how to distribute data across multiple nodes without rebuilding everything when you add/remove a node.*
