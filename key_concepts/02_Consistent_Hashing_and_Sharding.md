# Consistent Hashing + Database Sharding

> "When your database starts sweating, you have two choices: scale up (buy a bigger box) or scale out (buy more boxes). Sharding is how you scale out. Consistent hashing is how you do it without losing your data in the process."
> — The hard lesson learned by every team that hit a wall at scale

If you've ever wondered how Discord handles millions of messages, how Instagram stored billions of photos, or why Twitter had a "Fail Whale" for years before going quiet — the answer traces back to the same two ideas: **consistent hashing** and **database sharding**. This guide teaches both, with the actual war stories baked in.

---

# PART 1: Consistent Hashing

---

## Q1: What's the "naive" approach to distributing data across servers, and why does it fall apart?

**The naive approach: modulo hashing.**

Suppose you have 3 servers and you want to distribute user data. The simplest thing in the world:

```
server_index = hash(user_id) % N
```

Where `N` is the number of servers. Beautiful. Simple. Works perfectly.

Until you add or remove a server. Then it catastrophically doesn't work.

**Scenario:** You have 3 servers. User `alice` has `hash("alice") = 1000`:

```
1000 % 3 = 1  →  alice lives on Server 1
```

You add a 4th server because traffic is growing. Now:

```
1000 % 4 = 0  →  alice lives on Server 0
```

Alice's data just moved! And it's not just Alice. **Almost every single key remaps to a different server** when N changes.

```
With 3 servers:
  Key 1000 → 1000 % 3 = 1  (Server 1)
  Key 1001 → 1001 % 3 = 2  (Server 2)
  Key 1002 → 1002 % 3 = 0  (Server 0)

With 4 servers (added one):
  Key 1000 → 1000 % 4 = 0  (Server 0)  ← MOVED
  Key 1001 → 1001 % 4 = 1  (Server 1)  ← MOVED
  Key 1002 → 1002 % 4 = 2  (Server 2)  ← MOVED
```

All 3 keys moved. In a real system with 1 million keys, roughly **2/3 of all data needs to move**. This means:

- Every cache in your fleet is cold simultaneously (thundering herd)
- Massive replication traffic as data migrates
- Your database, which was already stressed enough to need a new server, now gets hammered trying to move its own data

This is why Twitter's early years were chaos. They were essentially running modulo hashing variants and every scaling event was a production incident.

**The insight that fixes this: consistent hashing.**

---

## Q2: Okay, what IS consistent hashing? Explain the ring.

**Consistent hashing: instead of mapping keys to servers directly, map both keys AND servers onto a circular ring.**

The ring spans a fixed hash space — say 0 to 2^32 - 1 (about 4 billion slots). Both your servers and your keys get hashed into this same space.

```
                     0 / 2^32
                       |
             .-------- * --------.
           /     Server A (100)    \
         /                           \
  2^32*3/4 *                         * 2^32/4
  Server D   \                      / Server B
  (3000)       \                  /   (1000)
                 `------*-------'
                   Server C (2000)
                   2^32/2
```

**The rule for finding a key's server: go clockwise around the ring until you hit a server.**

```
Key X hashes to position 500:
→ Go clockwise from 500
→ Hit Server B at position 1000
→ Key X belongs to Server B
```

```
Key Y hashes to position 1500:
→ Go clockwise from 1500
→ Hit Server C at position 2000
→ Key Y belongs to Server C
```

Now watch what happens when you **add a new Server E at position 750**:

```
Before:
  Keys 100–1000 → Server B

After adding Server E at 750:
  Keys 100–750  → Server E  (NEW)
  Keys 750–1000 → Server B  (unchanged ownership, smaller range)
```

Only the keys between the new server's position and the **previous server** need to move. Everything else is untouched.

**The magic property: when a server is added or removed, only `1/N` of keys need to move** (where N is the number of servers). Compare this to mod hashing where `(N-1)/N` keys move. That's the entire difference between a smooth scale-up and a production fire.

This is exactly what Amazon's Dynamo paper described in 2007. The paper (written by engineers at Amazon's storage team) became one of the most influential distributed systems papers ever written because it showed that you could build a highly available key-value store at planetary scale. Consistent hashing was the foundation that let Dynamo add and remove nodes — for hardware failures, replacements, capacity changes — without taking the whole thing down.

---

## Q3: One point per server on the ring sounds unfair. What are virtual nodes?

**The problem with one point per server: uneven load distribution.**

If each physical server gets one position on the ring, the key ranges assigned to each server are determined entirely by random hash luck. You might end up with:

```
Server A: handles 60% of the key space
Server B: handles 25% of the key space
Server C: handles 15% of the key space
```

Server A is overwhelmed. Server C is bored. Your "distributed" system is actually not very distributed.

A second problem: when Server B fails, *all* of Server B's load shifts to Server C. Now Server C has 40% of the load, and it wasn't built for that.

**The solution: virtual nodes (vnodes).**

Instead of placing each server once on the ring, place it **many times** — each placement is a "virtual node."

```
Physical server  →  Virtual nodes on the ring

Server A        →  A1, A2, A3, A4, A5, A6  (spread around the ring)
Server B        →  B1, B2, B3, B4, B5, B6
Server C        →  C1, C2, C3, C4, C5, C6
```

```
          0 / 2^32
            *
      A6 *     * B1
    C5 *           * A1
   B5 *               * C1
   C4 *               * B2
    A5 *             * A2
      B4 *         * C2
         C3 * * * B3
               A3
```

Now each server "owns" multiple small, interleaved segments of the ring instead of one big chunk. The statistical effect of many random placements means load distributes much more evenly.

**Bonus: graceful failure.** When Server B goes down, its 6 virtual nodes (B1–B6) each hand off to a *different* neighbor on the ring. The load from dead Server B is spread across all surviving servers — not dumped entirely on one unlucky neighbor.

Amazon Dynamo used 150–200 virtual nodes per physical server. Modern systems like Apache Cassandra (which Discord eventually migrated to) use 256 vnodes per node by default.

---

## Q4: Give me a concrete example of how little data moves when a node joins.

**The math is satisfying once you see it.**

Suppose you have 10 servers, each with 100 virtual nodes, managing 1 million keys. That's 1000 vnodes total covering the ring. Each vnode owns roughly 1000 keys on average.

A new server joins with 100 vnodes. What happens?

Each new vnode lands somewhere on the ring. It steals its segment from the current owner of that position. That current owner had ~1000 keys in that segment. The new vnode takes ~500 of them (roughly half the segment).

So the new server acquires: 100 vnodes × 500 keys = **~50,000 keys**

Out of 1 million total keys, only **5% of data moved**. And crucially, that data came from 100 different existing nodes — nobody got drained.

```
1,000,000 keys, 10 servers → add 1 server

Mod hashing:    ~909,090 keys move  (91%)  ← catastrophic
Consistent hash:   ~50,000 keys move  (5%)  ← manageable
```

This is why consistent hashing is the default in every serious distributed system today.

---

## Q5: How does Redis Cluster use consistent hashing in practice?

**Redis Cluster uses a variant called "hash slots" — it's consistent hashing with a fixed number of buckets.**

Redis pre-divides the ring into exactly **16,384 hash slots** (positions 0–16383). Every key maps to a slot:

```python
slot = CRC16(key) % 16384
```

Then slots are assigned to nodes. With 3 nodes:

```
Node A: slots 0    – 5460
Node B: slots 5461 – 10922
Node C: slots 10923 – 16383
```

This is visible in Redis Cluster output:

```bash
$ redis-cli cluster nodes
a1b2... 192.168.1.1:6379 master - 0 0 connected 0-5460
c3d4... 192.168.1.2:6379 master - 0 0 connected 5461-10922
e5f6... 192.168.1.3:6379 master - 0 0 connected 10923-16383
```

**Adding a node is surgical:** you move a subset of hash slots (and the keys within them) to the new node. No full reshuffle. The cluster can serve traffic the entire time migration is happening, moving one slot at a time.

```
Add Node D:
  Move slots 0-1365    from Node A → Node D
  Move slots 5461-6826 from Node B → Node D
  Move slots 10923-12287 from Node C → Node D

Result: each node now owns ~25% of slots
```

The 16,384 number isn't magic — it's small enough that the slot mapping table (a bitfield) fits in a single gossip message between nodes, but large enough that you can redistribute granularly even with many nodes.

---

# PART 2: Database Sharding

---

## Q6: What is sharding, and how is it different from replication?

**This is one of the most commonly confused pairs in system design interviews. Get it crystal clear.**

```
Replication:        Sharding:
                    
[All Data]          [Shard 1]   [Shard 2]   [Shard 3]
     |               Users       Users       Users
  .--+--.           A-H         I-P         Q-Z
  |     |
[Copy] [Copy]       Each shard has DIFFERENT data.
                    Together they hold ALL the data.

Same data,          Different data,
different places.   same schema.
```

**Replication** = make copies of your data. Every replica has the same data. Purpose: availability and read throughput. If one node dies, others serve the data. Great for reads.

**Sharding (horizontal partitioning)** = split your data across multiple databases. Each shard holds a *different subset* of rows. Purpose: write throughput and storage capacity. No single machine holds all the data.

**They are complementary, not competing.** A production system typically does both: shard the data across N groups, and then replicate each shard for redundancy.

```
Shard 1 (Users A-H):  Primary + 2 Replicas
Shard 2 (Users I-P):  Primary + 2 Replicas
Shard 3 (Users Q-Z):  Primary + 2 Replicas
```

Instagram does exactly this. They shard by user ID and replicate each shard for high availability.

---

## Q7: Wait — horizontal vs vertical partitioning. Are those the same as sharding vs something else?

**Good catch. These are related but distinct concepts.**

**Vertical partitioning** = split a table by *columns*.

Imagine a `users` table with 50 columns: `id, name, email, password_hash, bio, profile_photo_url, last_login, created_at, preferences_json, ...`

Vertical partitioning splits this into:

```sql
-- Hot table: accessed on every request
users_core (id, name, email, password_hash, last_login)

-- Cold table: only loaded on profile page
users_profile (id, bio, profile_photo_url, preferences_json)
```

Why? Smaller rows = more rows fit per database page = better cache utilization = faster queries. The cold data doesn't slow down your hot path. Also useful for compliance: put PII columns in a separate table with stricter access controls.

**Horizontal partitioning (= sharding)** = split a table by *rows*. All shards have the same columns, but each shard has a different subset of rows.

```
users table:

Shard 1: rows where user_id % 3 = 0
Shard 2: rows where user_id % 3 = 1
Shard 3: rows where user_id % 3 = 2
```

**The interview rule of thumb:**
- "Partitioning" often refers to splitting within a single database (Postgres table partitioning, for example)
- "Sharding" usually means splitting across multiple database *instances* (different machines)

They achieve the same logical split but at different infrastructure levels.

---

## Q8: How do you pick a shard key? What makes a bad one?

**Shard key selection is where teams make or break their scaling story. A bad shard key is a gift that keeps on giving — in pain.**

The shard key is the field you use to decide which shard a row lives on. Every query that doesn't include the shard key must fan out to *every* shard (expensive). Every write goes to a specific shard based on this key.

**A bad shard key: `status` field.**

Imagine sharding an orders table by `status` (pending, processing, shipped, delivered):

```
Shard 1: status = 'pending'
Shard 2: status = 'processing'
Shard 3: status = 'shipped'
Shard 4: status = 'delivered'
```

What happens during Black Friday? Every new order hits 'pending'. **Shard 1 gets 95% of all writes.** Shards 3 and 4 are basically idle. This is a **hotspot** — one shard is a bottleneck while others cool their heels.

**A bad shard key: timestamp.**

Sharding by `created_at` date range seems logical until you realize: all your *new* traffic hits the latest shard. The "hot" shard is always the current one.

**A good shard key: user_id or a high-cardinality hash.**

```
shard = hash(user_id) % N
```

- High cardinality: millions of user IDs → traffic spreads evenly
- User data is co-located: all queries for one user hit one shard
- Monotonically increasing? Use a hash *of* the ID so sequential IDs don't pile on the same shard

**The Instagram approach (from their engineering blog):** They chose `user_id` as the shard key from day one. Any query about a user's photos, followers, or feed hits exactly one shard. They never need to fan out across shards for single-user operations.

**Twitter's fail whale era** had partial roots in poor sharding strategies. Their early MySQL setup had uneven load distribution — the celebrity user problem. An account like @Oprah, when she joined in 2009, generated traffic patterns no single shard was built for. The tweets, the follower updates, the timeline fan-outs — everything related to a viral celebrity was a hotspot on whichever shard held her data.

---

## Q9: What are the three types of sharding strategies?

**Range sharding, hash sharding, and directory sharding. Know them all.**

### Range Sharding

Split data by value ranges of the shard key:

```
Shard 1: user_id 1          – 1,000,000
Shard 2: user_id 1,000,001  – 2,000,000
Shard 3: user_id 2,000,001  – 3,000,000
```

**Pros:**
- Range queries are efficient: "get all users with ID between 500K and 600K" hits exactly one shard
- Easy to reason about data location

**Cons:**
- Hotspots from sequential inserts: new users always go to the last shard
- Uneven load if data isn't uniformly distributed across the range

**Real use case:** HBase and Cassandra use range partitioning at the storage layer. Good for time-series data where you query a time window.

---

### Hash Sharding

Apply a hash function to the shard key, take modulo:

```python
shard_id = hash(user_id) % num_shards

# Example with 4 shards:
user 1001 → hash(1001) % 4 = 1 → Shard 1
user 1002 → hash(1002) % 4 = 2 → Shard 2
user 1003 → hash(1003) % 4 = 3 → Shard 3
user 1004 → hash(1004) % 4 = 0 → Shard 0
```

**Pros:**
- Even distribution: hash functions spread data uniformly
- No hotspots from sequential keys

**Cons:**
- Range queries are terrible: "get users 1000–2000" requires hitting all shards
- Resharding is painful (mod N problem — see Part 1)
- Adding a shard remaps almost everything (unless you use consistent hashing!)

**Discord** used hash sharding on their Cassandra cluster, using the user/channel ID as the partition key. Cassandra internally uses consistent hashing so adding capacity nodes only moves a small fraction of data — the exact lesson from Part 1.

---

### Directory Sharding

Maintain a lookup table that explicitly maps keys to shards:

```
Shard Directory (stored in a fast lookup service):

user_id range    → shard
1   – 500,000    → shard-us-east-1
500,001 – 1M     → shard-us-west-2
1M+              → shard-eu-central-1
user_id 777      → shard-vip-1   ← special override!
```

**Pros:**
- Maximum flexibility: you can move specific users or ranges arbitrarily
- Handles "celebrity" hotspots: put @Oprah on a dedicated shard
- No reshuffling math — just update the directory

**Cons:**
- The directory itself is a single point of failure (must be replicated and cached aggressively)
- Extra network hop for every request to look up the shard
- Directory can become stale

**Real use case:** Pinterest uses a variant of directory sharding. They have a "sharding config" that maps objects to shards, and specific high-traffic accounts can be manually moved to dedicated shards.

---

## Q10: Cross-shard queries and JOINs — how painful is it really?

**Extremely painful. This is the core trade-off of sharding and the reason you should delay sharding as long as possible.**

In a single database, a JOIN is a single query:

```sql
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.signup_date > '2024-01-01';
```

The database handles this internally, efficiently, using indexes. Done.

In a sharded database where `users` and `orders` are on different shards:

```
Application layer has to:

1. Query shard 1, 2, 3 for users where signup_date > '2024-01-01'
   → GET /shard-1/users?filter=...
   → GET /shard-2/users?filter=...
   → GET /shard-3/users?filter=...

2. Collect all user_ids from all three responses

3. For each user_id, determine which shard holds their orders
   → Might be different shards than the users!

4. Query order shards for each user_id batch

5. Join the results in application memory

6. Return
```

This is called a **scatter-gather query** or **fan-out query**, and it is:
- Slower (multiple network round trips)
- Complex (the application now contains JOIN logic)
- Fragile (if any shard is slow, the whole query waits)
- Hard to paginate correctly across shards

**Discord's experience:** When they moved from single Postgres to Cassandra, they intentionally designed their data model to *never need cross-shard JOINs*. They denormalized aggressively — storing redundant copies of data together so that any single query only touches one partition. Their table schema is modeled around access patterns, not relational normalization. This was a deliberate trade: disk space for query simplicity.

**The mitigation strategies:**

1. **Co-locate related data on the same shard.** If you always query users and their orders together, shard both by `user_id`. Then a user's orders are always on the same shard as the user.

2. **Denormalize.** Store redundant copies. Yes, this violates database normalization. At scale, you make that trade.

3. **Global tables.** Small reference tables (countries, product categories) can be replicated to every shard. They don't shard — they clone.

4. **Dual writing.** Write to multiple shards simultaneously to support different query patterns (expensive but sometimes necessary).

5. **Accept the fan-out.** For some queries, scatter-gather is unavoidable. Make it fast with async parallel queries and set timeouts aggressively.

---

## Q11: Resharding — adding new shards after you're already in production. Is this as bad as it sounds?

**Yes. Resharding in production is one of the most stressful operations in distributed systems. It is why Instagram pre-sharded into 8192 logical shards from day one.**

**The resharding problem:**

You start with 4 shards. You chose `shard = hash(user_id) % 4`. Business is booming. 18 months later you need 8 shards. Now:

```
Old mapping (4 shards):  user 1001 → hash(1001) % 4 = 1 → Shard 1
New mapping (8 shards):  user 1001 → hash(1001) % 8 = 1 → Shard 1  (lucky!)
                         user 1003 → hash(1003) % 4 = 3 → Shard 3
                         user 1003 → hash(1003) % 8 = 7 → Shard 7  (moved!)
```

As we saw in Part 1, roughly half of all data needs to move when you double the shard count with mod hashing.

**What "moving data" means in practice:**

1. Stand up 4 new shards
2. Begin dual-writing: every new write goes to both old and new shard
3. Run a background migration: read from old shard, write to new shard, for all existing data
4. This migration might run for **days or weeks** on a large dataset
5. Monitor lag between old and new shard
6. Once lag is near zero, cut over reads to new shards
7. Stop writes to old shards
8. Verify, then decommission old shards
9. Pray nothing went wrong

During steps 1–7, you're running two versions of your storage layer simultaneously. Any bug in the migration script, any inconsistency between old and new, any partial failure — all become production incidents.

**The Instagram solution: logical pre-sharding.**

Instagram's engineering team described this in their engineering blog. When they launched (with a small user base), they sharded into **8192 logical shards from day one** — even though they only had a handful of physical database servers.

```
Instagram sharding model:

Logical shards:  0 to 8191  (8,192 total, always)
Physical servers: however many we need today

Mapping:
  Logical shards 0-511     → Physical server 1
  Logical shards 512-1023  → Physical server 2
  ...

When we need more capacity:
  Move logical shards 0-255 from server 1 → new server 5
  Update the directory
  Done. No key remapping. No migration of millions of rows.
```

**The key insight:** logical shards are the stable unit. Physical servers are just where logical shards live. To rebalance, you move logical shards between physical servers. The hash mapping (`hash(user_id) % 8192`) **never changes**. So the application always knows: user 1001 is in logical shard X. The infrastructure layer resolves logical shard X to whatever physical server it lives on today.

This is what Shopify, Airbnb, and many others do as well. Pre-shard aggressively. Pay the complexity cost once, upfront, when stakes are low. Avoid the nightmare resharding at 3am when your database is on fire.

---

# Quick Revision Table

| Term | One-liner definition |
|------|---------------------|
| **Mod hashing** | `server = hash(key) % N` — simple, breaks when N changes |
| **Consistent hashing** | Keys and servers on a ring; only 1/N data moves when N changes |
| **Hash ring** | Circular space 0–2^32; clockwise lookup for key→server mapping |
| **Virtual nodes (vnodes)** | Each physical server gets many ring positions for even load distribution |
| **Hash slots (Redis)** | Fixed 16,384 slots; Redis Cluster assigns slot ranges to nodes |
| **Sharding** | Split rows of a table across multiple databases (different data per shard) |
| **Replication** | Copy the same data to multiple databases (same data, multiple places) |
| **Vertical partitioning** | Split a table by columns (hot columns vs cold columns) |
| **Horizontal partitioning** | Split a table by rows (= sharding) |
| **Shard key** | The column used to decide which shard a row belongs to |
| **Hotspot** | One shard receiving disproportionate traffic due to bad shard key |
| **Range sharding** | Shard by value ranges (user_id 0–1M on shard 1, etc.) |
| **Hash sharding** | `shard = hash(key) % N`; even distribution, bad for range queries |
| **Directory sharding** | Lookup table maps keys to shards; most flexible, needs a fast directory |
| **Scatter-gather** | Cross-shard fan-out query; expensive, complex, unavoidable sometimes |
| **Resharding** | Moving data when shard count changes; painful, often done at 3am |
| **Logical pre-sharding** | Instagram's trick: create 8192 logical shards, map to fewer physical servers; move logical shards to rebalance, never change the hash |
| **Co-location** | Store related data on the same shard to avoid cross-shard JOINs |
| **Denormalization** | Store redundant copies of data to avoid cross-shard JOINs |

---

# Incident Recap

| Company | Problem | Solution |
|---------|---------|----------|
| **Amazon (2007)** | Needed a highly available key-value store that could add/remove nodes without downtime | Dynamo paper: consistent hashing with vnodes, the paper that taught the industry |
| **Twitter (2008–2012)** | "Fail Whale" era — MySQL couldn't scale writes; celebrity users caused hotspots | Gradually moved to sharded MySQL + eventually a mix of distributed stores; sharding by tweet ID |
| **Instagram (2011)** | Needed to scale storage before they were famous; didn't want a resharding nightmare later | Pre-sharded into 8192 logical shards from launch day; only ever moves logical shards, never remaps keys |
| **Discord (2017+)** | Hit 100M+ messages/day on Postgres; single database couldn't keep up | Migrated to Cassandra; designed data model around access patterns, eliminated cross-shard JOINs by denormalizing |

---

> **One thing to remember above all else:** sharding is a last resort. It adds enormous operational complexity. Every engineer who has done a live resharding migration will tell you: exhaust vertical scaling, read replicas, caching, query optimization, and connection pooling first. Shard when you must, pre-shard logically if you can see it coming, and design your shard key with obsessive care because you will live with that decision for years.
