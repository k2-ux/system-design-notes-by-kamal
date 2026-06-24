# DSA for System Design
> Every data structure and algorithm you learned in DSA has a real home in a production system. This document maps each one to where and why it's used in system design interviews.

---

## Table of Contents

### Data Structures
1. [Hash Map / Hash Set](#1-hash-map--hash-set)
2. [Queue](#2-queue)
3. [Priority Queue (Heap)](#3-priority-queue-heap)
4. [Linked List](#4-linked-list)
5. [Tree (B-Tree)](#5-tree-b-tree)
6. [Graph](#6-graph)
7. [Bloom Filter](#7-bloom-filter)
8. [Consistent Hashing Ring](#8-consistent-hashing-ring)
9. [Trie](#9-trie)
10. [Ring Buffer](#10-ring-buffer)

### Algorithms
11. [Hashing](#11-hashing)
12. [Binary Search](#12-binary-search)
13. [Dijkstra's Shortest Path](#13-dijkstras-shortest-path)
14. [Consistent Hashing Algorithm](#14-consistent-hashing-algorithm)
15. [LRU Eviction Algorithm](#15-lru-eviction-algorithm)
16. [MapReduce](#16-mapreduce)
17. [Exponential Backoff](#17-exponential-backoff)
18. [Leaky Bucket / Token Bucket](#18-leaky-bucket--token-bucket)

### [Quick Reference Table](#quick-reference-table)
### [The Meta-Lesson](#the-meta-lesson)

---

# DATA STRUCTURES

---

## 1. Hash Map / Hash Set

**What it is:**
> Key → Value lookup in O(1). Hash Set is just a HashMap where you only care about the key existing, not the value.

**Where it lives in system design:**

> **Redis** is essentially a distributed HashMap:
> ```
> "session:user_123"  → "{name: kamal, role: admin}"   O(1) lookup
> "rate:ip_1.2.3.4"   → "47"                            O(1) increment
> "online:alice"      → "true" TTL:30s                  O(1) check
> ```
>
> **URL Shortener:**
> ```
> HashMap: short_code → original_url
> "abc123" → "https://very-long-url.com/article/..."
> O(1) redirect — no DB query needed if cached
> ```
>
> **Deduplication:**
> ```
> visited = HashSet()
> Before crawling URL: if url in visited → skip  O(1) check
> ```

**The system design insight:**
> Any time you see "fast lookup by key" — that's a HashMap. Redis IS a HashMap. Cache IS a HashMap. The difference between a slow system and a fast one is often just "did you put a HashMap in front of your database."

---

## 2. Queue

**What it is:**
> FIFO — First In, First Out. Items processed in the order they arrive.

**Where it lives in system design:**

> **Kafka** is a distributed, durable, scalable Queue:
> ```
> Producer → [Kafka Queue] → Consumer
>
> Instagram: Ronaldo posts → "notify followers" event enters Kafka
>            Workers consume events, send notifications one by one
>            Kafka buffers the spike — workers process at their own pace
> ```
>
> **Video transcoding:**
> ```
> Video uploaded → [Queue] → Transcoding workers pick up jobs
>   Worker 1: transcode to 1080p
>   Worker 2: transcode to 720p
>   Worker 3: transcode to 480p
> ```
>
> **Why queue instead of direct call?**
> ```
> Direct call:  Upload Service → Transcoding Service (synchronous, fragile)
> Queue:        Upload Service → [Kafka] → Transcoding Service (async, resilient)
>
> If Transcoding Service crashes:
>   Direct call → Upload fails too ❌
>   Queue       → Jobs wait in Kafka, resume when service recovers ✅
> ```

**The system design insight:**
> Queues decouple producers from consumers. Any time you have a spike in work or two services that shouldn't depend on each other's uptime — use a queue.

---

## 3. Priority Queue (Heap)

**What it is:**
> Like a queue, but items have a priority score. Highest priority item always comes out first. O(log n) insert and remove.

**Where it lives in system design:**

> **Web Crawler — URL prioritization:**
> ```
> Priority Queue:
>   [bbc.com/news,    priority: 95]  ← dequeued first
>   [wikipedia.org,   priority: 87]
>   [medium.com/post, priority: 43]
>   [random-blog.xyz, priority: 2]   ← dequeued last
> ```
>
> **Notification scheduler — delayed sends:**
> ```
> Priority Queue ordered by send_time:
>   [send email at 8:00am]   ← runs first
>   [send SMS at 8:05am]
>   [generate report at 9am]
> ```
>
> **Rate Limiter — sliding window expiry:**
> ```
> Priority Queue ordered by timestamp:
>   [request at T=100ms]  ← expired first, popped out
>   [request at T=200ms]
>   [request at T=900ms]
> Pop all requests older than 1 second to maintain the window
> ```

**The system design insight:**
> Whenever work items are NOT equal — some more urgent, some scheduled for later — you need a Priority Queue, not a plain Queue.

---

## 4. Linked List

**What it is:**
> Nodes where each points to the next. O(1) insert/delete at known position. Most famously used as a Doubly Linked List for LRU caches.

**Where it lives in system design:**

> **LRU Cache — the most famous system design use:**
> ```
> Cache capacity: 3 items
> Access order (head = most recent): [C] ↔ [B] ↔ [A]  (A is least recent)
>
> New item D arrives, cache is full:
>   → Evict A (tail = least recently used)
>   → Add D at head
>   → [D] ↔ [C] ↔ [B]
>
> Access B:
>   → Move B to head
>   → [B] ↔ [D] ↔ [C]
> ```
>
> **Doubly Linked List + HashMap = O(1) LRU:**
> ```
> HashMap:       key → node pointer  (O(1) find any node)
> Linked List:   node order = recency (O(1) move to front on access)
>
> Redis implements LRU eviction this way internally.
> CDN cache eviction uses the same pattern.
> ```

**The system design insight:**
> You won't implement a linked list in a system design interview. But LRU cache comes up constantly — knowing it's backed by a doubly linked list + HashMap (for O(1) everything) shows real depth.

---

## 5. Tree (B-Tree)

**What it is:**
> Hierarchical structure. B-Tree is the database-optimized variant — wide, shallow, designed to minimize disk reads.

**Where it lives in system design:**

> **Every database index is a B-Tree:**
> ```
> Without index:
>   SELECT * FROM users WHERE email = 'kamal@gmail.com'
>   → Scan all 100M rows → minutes ❌
>
> With B-Tree index on email:
>   → Tree traversal: O(log 100M) ≈ 27 comparisons → milliseconds ✅
> ```
>
> **Why B-Tree and not binary tree?**
> ```
> Binary tree: 2 children per node → very deep → many disk reads
> B-Tree:      100s of children per node → very shallow → few disk reads
>
> Disk reads are 1000x slower than RAM reads.
> B-Tree minimizes disk reads by maximizing children per node.
> ```
>
> **File system directories:**
> ```
> /home/kamal/documents/file.txt
> → Tree traversal: O(depth) not O(total files)
> ```

**The system design insight:**
> When you say "add an index" in a system design interview, you're saying "build a B-Tree on this column." Indexes speed reads but slow writes — every write must also update the B-Tree.

---

## 6. Graph

**What it is:**
> Nodes connected by edges. Directed or undirected, weighted or unweighted.

**Where it lives in system design:**

> **Social networks:**
> ```
> Directed Graph:
>   Kamal → follows → Ronaldo
>   Ronaldo → follows → Kamal  (mutual)
>
> BUT — Instagram uses SQL, not a graph DB. Why?
>   Simple query: "who follows Kamal?" → SQL with index handles perfectly
>   Graph DB needed only for multi-hop: "friends of friends of friends"
> ```
>
> **When to use a Graph DB (Neo4j):**
> ```
> LinkedIn "People You May Know" → friends of friends → multi-hop traversal
> Fraud detection → "this card connected to accounts linked to fraud"
> Google Maps → shortest route on road network
> ```
>
> **Google Maps — weighted directed graph:**
> ```
> Nodes: intersections
> Edges: roads (weight = travel time in seconds)
> Algorithm: Dijkstra's → finds shortest path
> ```

**The system design insight:**
> Don't reach for a graph DB just because data has relationships. SQL handles one-hop relationships fine. Graph DBs shine for multi-hop traversal only.

---

## 7. Bloom Filter

**What it is:**
> Probabilistic data structure. Answers "have I seen this?" using a bit array + multiple hash functions. 98% less memory than HashSet. May have false positives. Never has false negatives.

**Where it lives in system design:**

> **Web Crawler — URL deduplication:**
> ```
> 50 billion URLs to track:
> HashSet:      50B × 100 bytes = 5 TB RAM ❌
> Bloom Filter: 50B × 10 bits  = 60 GB RAM ✅
>
> False positive: skips a valid URL → minor annoyance ✅
> False negative: impossible → no infinite crawl loops ✅
> ```
>
> **Cassandra uses Bloom Filters internally:**
> ```
> Cassandra data spreads across many SSTable files on disk
> Before reading each file to find a key:
>   Check that file's Bloom Filter
>   If NO  → definitely not in this file, skip it (saves disk I/O) ✅
>   If YES → read the file to confirm
> This is why Cassandra reads are fast despite massive data spread
> ```
>
> **Database query optimization:**
> ```
> Before hitting the DB:
>   Bloom Filter says NO → definitely not in DB, return 404 instantly ✅
>   Bloom Filter says YES → probably in DB, query to confirm
> Saves majority of unnecessary DB queries
> ```

**The system design insight:**
> Bloom Filters appear whenever you need "have I seen this?" at massive scale and can tolerate rare false positives. False positives OK. False negatives impossible. Cassandra, Redis, and BigTable all use them internally.

---

## 8. Consistent Hashing Ring

**What it is:**
> A technique for distributing data across servers so that when a server is added or removed, only a small fraction of keys need to remapped — not all of them.

**Where it lives in system design:**

> **Database sharding:**
> ```
> Naive: user_id % 3 servers
>   Add 4th server → user_id % 4 → ALL keys remap → massive migration ❌
>
> Consistent Hashing Ring:
>   Add 4th server → only ~25% of keys move ✅
>   Remove server  → only its keys move to its neighbor ✅
> ```
>
> **Redis Cluster:**
> ```
> Ring with 3 nodes: [Node A]───[Node B]───[Node C]───back to A
>
> Key "user_123" hashes to position on ring
> → Assigned to nearest node clockwise
>
> Add Node D between B and C:
> → Only keys between B and D move to D
> → A and B completely untouched
> ```
>
> **CDN routing:**
> ```
> User's IP hashes to a position on ring
> → Served by nearest CDN edge node
> New edge node added → only nearby traffic remaps to it
> ```

**The system design insight:**
> Whenever you say "sharding" in an interview, follow up with "and we'd use consistent hashing so adding/removing nodes doesn't cause a full data reshuffle." This separates a good answer from a great one.

---

## 9. Trie

**What it is:**
> A tree where each node represents a character. Used for prefix-based lookups in O(L) where L = length of the query string.

**Where it lives in system design:**

> **Search autocomplete:**
> ```
> Trie stores: ["apple", "app", "application", "apply"]
>
> User types "app":
>   → Traverse: root → a → p → p
>   → Return all words under this node: ["app", "apple", "application", "apply"]
>   → Milliseconds, regardless of dictionary size
> ```
>
> **IP routing (longest prefix match):**
> ```
> Router's routing table is a Trie of IP prefixes:
>   192.168.0.0/16  → Network A
>   192.168.1.0/24  → Network B (more specific)
>
> Packet for 192.168.1.50:
>   → Trie finds longest match: 192.168.1.0/24 → Network B
> ```
>
> **DNS resolution:**
> ```
> "mail.google.com" → Trie: com → google → mail → IP
> ```

**The system design insight:**
> If an interview asks you to design search autocomplete, Trie is the core data structure. For production at scale, also mention Redis Sorted Sets as a simpler alternative (store word + popularity score, query by prefix range).

---

## 10. Ring Buffer

**What it is:**
> Fixed-size buffer where new data overwrites the oldest. O(1) write. Automatically evicts old data without cleanup jobs.

**Where it lives in system design:**

> **Rate Limiter — sliding window:**
> ```
> Ring Buffer of size 100 stores request timestamps
> New request: overwrite oldest slot with current timestamp
> Count requests in last 60 seconds → if > limit → reject
> O(1) write, no cleanup needed
> ```
>
> **Metrics/monitoring:**
> ```
> Prometheus stores: [T=0: cpu=45%] [T=10: cpu=67%] [T=20: cpu=89%]...
> Ring buffer of last N data points per metric
> Old data auto-evicted or compressed to cold storage
> ```
>
> **Recent message cache:**
> ```
> "Show last 20 messages in this chat"
> Ring buffer of size 20 per chat room
> New message → push, oldest automatically dropped
> ```

**The system design insight:**
> Time-series databases (Prometheus, InfluxDB) are optimized ring buffers with compression. When designing a monitoring system, mention that metrics are time-series data with automatic retention windows — shows you understand the access pattern.

---

# ALGORITHMS

---

## 11. Hashing

**What it is:**
> A function that converts any input into a fixed-size output (hash). Same input always gives same output. Different inputs (usually) give different outputs.

**Where it lives in system design:**

> **Deduplication — content hashing:**
> ```
> Google Drive: SHA-256 hash of each 4 MB file chunk
>   chunk_content → "a3f9b2c1..."  (always the same for same bytes)
>
> Two users upload same file with different names:
>   Both produce same hash → store only once → save petabytes
> ```
>
> **Sharding — which server gets this data?**
> ```
> hash(user_id) % num_servers = server_index
>
> hash("user_123") = 8472364
> 8472364 % 3 = 1 → goes to Server 1
>
> Same user always hashes to same server → consistent routing
> ```
>
> **Password storage:**
> ```
> NEVER store plain password
> Store: bcrypt(password + salt) → "x7k$2m..."
> On login: bcrypt(input + salt) → compare hashes
> Even if DB is stolen, passwords can't be reversed
> ```
>
> **Checksums — data integrity:**
> ```
> File transfer: send file + MD5 hash
> Receiver: recalculate hash → if different → file corrupted in transit
> S3 uses this on every upload/download
> ```

**The system design insight:**
> Hashing is the backbone of caching (HashMap), sharding, deduplication, and security. Anytime you need to "route to the same place consistently" or "detect if two things are identical" — hashing.

---

## 12. Binary Search

**What it is:**
> Search a sorted array in O(log n) by repeatedly halving the search space.

**Where it lives in system design:**

> **Database index lookups:**
> ```
> B-Tree index is a sorted structure
> "Find user with email kamal@gmail.com"
> → Binary search through sorted email index
> → O(log n) not O(n) — the entire reason indexes exist
> ```
>
> **Time-series data — find events in a time range:**
> ```
> "Get all logs between 3pm and 4pm"
> Logs stored sorted by timestamp
> → Binary search to find 3pm position
> → Read forward until 4pm
> → O(log n + results) instead of O(n)
> ```
>
> **Distributed systems — finding which shard owns a key:**
> ```
> Sorted list of shard ranges:
>   Shard 0: keys 0-1000
>   Shard 1: keys 1001-2000
>   Shard 2: keys 2001-3000
>
> Binary search to find which range contains key 1547
> → O(log num_shards) — instant even with 1000 shards
> ```

**The system design insight:**
> You rarely implement binary search in system design. But every time you say "index" or "sorted data" or "range query" — binary search is what makes those fast under the hood.

---

## 13. Dijkstra's Shortest Path

**What it is:**
> Algorithm that finds the shortest path from a source node to all other nodes in a weighted graph. O((V + E) log V) with a priority queue.

**Where it lives in system design:**

> **Google Maps — fastest route:**
> ```
> Graph:
>   Nodes: road intersections
>   Edges: road segments (weight = travel time in seconds)
>
> "Fastest route from Clifton to Airport"
> → Dijkstra finds minimum-weight path through the road graph
> → Considers real-time traffic as dynamic edge weights
> ```
>
> **Network routing — packets finding their way:**
> ```
> Internet routers run a variant of Dijkstra (OSPF protocol)
> Each router knows the "cost" to reach neighboring routers
> Dijkstra computes the lowest-cost path to every destination
> Your packet hops from router to router along the shortest path
> ```
>
> **CDN — which edge node to serve from:**
> ```
> Graph of CDN nodes with latency as edge weights
> User request → Dijkstra finds lowest-latency path to nearest cache
> ```

**The system design insight:**
> If your system involves routing (packets, vehicles, requests) through a network with costs — Dijkstra is the algorithm. In interviews, mentioning "we'd use a shortest-path algorithm like Dijkstra on the road graph" for a maps system shows strong depth.

---

## 14. Consistent Hashing Algorithm

**What it is:**
> Places both servers and keys on a hash ring (0 to 2³²). Each key is served by the nearest server clockwise. Adding/removing a server only affects nearby keys.

**Where it lives in system design:**

> Already covered in Data Structures section (#8). The algorithm detail:
> ```
> Step 1: Hash each server to a position on the ring
>   hash("Server_A") = 100
>   hash("Server_B") = 200
>   hash("Server_C") = 350
>
> Step 2: Hash each key to a position
>   hash("user_123") = 150 → goes to Server_B (nearest clockwise)
>   hash("user_456") = 80  → goes to Server_A
>   hash("user_789") = 280 → goes to Server_C
>
> Step 3: Add Server_D at position 250
>   → Only keys between 200-250 move from Server_C to Server_D
>   → Everything else unchanged
> ```
>
> **Virtual nodes (for uneven load):**
> ```
> Problem: with 3 servers, one might get 60% of keys by luck
> Solution: each server gets 150 virtual positions on the ring
>   Server_A → positions 45, 178, 267, 389, ...
>   → Load spreads more evenly
> ```

**The system design insight:**
> Always mention virtual nodes when discussing consistent hashing — shows you know the naive version has a load imbalance problem and you know the fix.

---

## 15. LRU Eviction Algorithm

**What it is:**
> When cache is full, evict the Least Recently Used item. Implemented with Doubly Linked List + HashMap for O(1) get and put.

**Where it lives in system design:**

> Already covered in Data Structures (#4). The algorithm:
> ```
> GET(key):
>   1. Check HashMap → O(1) find node
>   2. Move node to head of linked list (most recent)
>   3. Return value
>
> PUT(key, value):
>   1. If key exists → update value, move to head
>   2. If cache full → remove tail node (least recent), delete from HashMap
>   3. Create new node at head, add to HashMap
>
> Every operation: O(1)
> ```
>
> **Where used:**
> ```
> Redis: maxmemory-policy = allkeys-lru
>   → When Redis hits memory limit, evicts least recently used keys
>
> CDN: edge nodes evict least accessed content to make room for new content
>
> Browser: tab memory management evicts inactive tabs' resources
> ```

**The system design insight:**
> Redis's LRU eviction is why you can use it as a cache without manually managing what stays and what goes. The algorithm handles it automatically.

---

## 16. MapReduce

**What it is:**
> A programming model for processing huge datasets in parallel across many machines. Split into two phases: Map (transform each item) and Reduce (aggregate results).

**Where it lives in system design:**

> **Counting word frequency across 1 billion documents:**
> ```
> MAP phase (parallel, one worker per document):
>   Document 1: "the cat sat" → [("the",1), ("cat",1), ("sat",1)]
>   Document 2: "the dog sat" → [("the",1), ("dog",1), ("sat",1)]
>   Document 3: "cat and dog" → [("cat",1), ("and",1), ("dog",1)]
>
> Shuffle: group by key across all workers
>   "the": [1,1]
>   "cat": [1,1]
>   "sat": [1,1]
>   "dog": [1,1]
>
> REDUCE phase (parallel, one worker per key):
>   "the" → sum([1,1]) = 2
>   "cat" → sum([1,1]) = 2
>   "dog" → sum([1,1]) = 2
> ```
>
> **Generating analytics reports:**
> ```
> "Count total orders per country last month"
> Map:    each order → (country, 1)
> Reduce: sum counts per country
> → Runs across 1000 machines in parallel, finishes in minutes not days
> ```
>
> **Google's original PageRank was computed using MapReduce.**
>
> Real tools: **Apache Hadoop** (original MapReduce), **Apache Spark** (faster, in-memory MapReduce).

**The system design insight:**
> When asked "how do you process petabytes of data for analytics?" — MapReduce (or Spark). The key insight is parallelism: split the data, process independently, aggregate results. No single machine needs to see all the data.

---

## 17. Exponential Backoff

**What it is:**
> When a request fails, retry — but wait longer each time. Wait 1s, then 2s, then 4s, then 8s... Prevents hammering a struggling service.

**Where it lives in system design:**

> **Any service calling another service:**
> ```
> Notification worker → APNs (fails, APNs is temporarily down)
>
> Attempt 1: fails → wait 1 second
> Attempt 2: fails → wait 2 seconds
> Attempt 3: fails → wait 4 seconds
> Attempt 4: fails → wait 8 seconds
> Attempt 5: fails → mark as failed, alert on-call
>
> Why exponential and not fixed 1 second?
>   Fixed retry: 1000 workers × retry every 1s = 1000 req/s hammering a down service ❌
>   Exponential: retries spread out → service gets breathing room to recover ✅
> ```
>
> **With jitter (randomness):**
> ```
> Without jitter: all 1000 workers retry at exactly T=1s, T=2s → synchronized thundering herd
> With jitter:    each worker waits random(0, 2^attempt) seconds → retries spread out naturally
> ```
>
> **Used everywhere:**
> ```
> AWS SDK → exponential backoff on all API calls by default
> Kafka consumers → backoff on failed message processing
> Payment retries → backoff before retrying failed charge
> ```

**The system design insight:**
> Exponential backoff with jitter is the standard retry pattern in distributed systems. Always mention it when discussing failure handling. Never retry immediately in a tight loop — you'll turn a struggling service into a dead one.

---

## 18. Leaky Bucket / Token Bucket

**What it is:**
> Algorithms for rate limiting — controlling how many requests a user or service can make per unit of time.

**Where it lives in system design:**

> **Token Bucket (more common):**
> ```
> Each user gets a "bucket" with N tokens, refilled at rate R/second
>
> User makes request:
>   → If bucket has tokens → consume 1 token, allow request ✅
>   → If bucket empty → reject request (429 Too Many Requests) ❌
>
> Example: 100 requests/minute limit
>   Bucket capacity: 100 tokens
>   Refill rate: 100 tokens per 60 seconds (~1.67/sec)
>
>   User sends 100 requests in 1 second → all allowed (bucket had 100) ✅
>   User sends 101st request → rejected ❌
>   Wait 60 seconds → bucket refills → 100 more allowed
> ```
>
> **Leaky Bucket:**
> ```
> Requests pour into bucket at any rate
> Bucket "leaks" (processes) at fixed rate regardless of input
>
> Burst of 1000 requests → bucket fills up, overflow is dropped
> → Smooths out traffic spikes, always processes at steady rate
>
> Good for: smoothing traffic
> Token Bucket good for: allowing controlled bursts
> ```
>
> **Where used:**
> ```
> API Gateway (AWS, Kong) → rate limiting per API key
> Redis → store token count per user key, DECR atomically
> Nginx → limit_req module uses leaky bucket
> ```

**The system design insight:**
> When an interview asks "how do you prevent a single user from overwhelming your API?" — Token Bucket is the answer. Store token count in Redis (atomic DECR), refill with a background job or on-demand calculation. Mention 429 status code for rejected requests.

---

## Quick Reference Table

| DSA Concept | System Design Use Case | Real Tool / Example |
|-------------|----------------------|---------------------|
| HashMap | Cache, session store, fast key lookup | Redis |
| HashSet | URL dedup at small scale | In-memory visited set |
| Queue (FIFO) | Task processing, event streaming, decoupling | Kafka, SQS |
| Priority Queue | Crawl priority, scheduled tasks, rate limiting | Kafka + custom scoring |
| Doubly Linked List + HashMap | LRU cache eviction | Redis (internal), CDN |
| B-Tree | Database indexes, range queries | PostgreSQL, MySQL |
| Graph | Social recommendations, fraud detection, routing | Neo4j, SQL + Dijkstra |
| Bloom Filter | URL dedup at scale, DB query skip | Cassandra (internal) |
| Consistent Hashing Ring | Sharding, cache routing | Redis Cluster, Cassandra |
| Trie | Search autocomplete, IP routing, DNS | Custom, Elasticsearch |
| Ring Buffer | Rate limiting, metrics, recent activity | Prometheus, InfluxDB |
| Hashing | Deduplication, sharding, password storage | SHA-256, bcrypt, MD5 |
| Binary Search | Index lookups, range queries, shard routing | Every DB index |
| Dijkstra's | Maps routing, network routing, CDN routing | Google Maps, OSPF |
| Consistent Hashing Algo | Key distribution across shards | Redis Cluster, Cassandra |
| LRU Algorithm | Cache eviction policy | Redis maxmemory-policy |
| MapReduce | Large-scale batch analytics | Hadoop, Spark |
| Exponential Backoff | Retry on failure without hammering | AWS SDK, Kafka, payments |
| Token Bucket | API rate limiting | Redis + API Gateway |

---

## The Meta-Lesson

> In a DSA interview, you implement data structures and algorithms from scratch.
> In a system design interview, you choose WHICH data structure or algorithm (already implemented by a tool) solves your problem — and explain WHY.
>
> ```
> DSA interview:    "Implement an LRU Cache"
>                    → Write DoublyLinkedList + HashMap code
>
> System design:    "How do you cache 100M user sessions?"
>                    → "Use Redis — it implements LRU eviction internally
>                       using a doubly linked list + HashMap, giving O(1)
>                       eviction when memory is full"
> ```
>
> The knowledge is the same. The application is different.
> Understanding the underlying DSA helps you pick the right tool AND explain why it works — which is exactly what interviewers want to hear.
