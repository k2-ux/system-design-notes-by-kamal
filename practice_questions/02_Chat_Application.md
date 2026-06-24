# Design a Chat Application like WhatsApp
> Complete interview preparation guide — Q&A format, beginner to advanced.

---

## Table of Contents
1. [What Is It?](#what-is-it)
2. [Step 1 — Requirements](#step-1--requirements)
3. [Step 2 — Capacity Estimation](#step-2--capacity-estimation)
4. [Step 3 — High-Level Design](#step-3--high-level-design)
5. [Step 4 — Message Delivery Deep Dive](#step-4--message-delivery-deep-dive)
6. [Step 5 — Database Design](#step-5--database-design)
7. [Step 6 — Media Handling](#step-6--media-handling)
8. [Step 7 — API Design](#step-7--api-design)
9. [Step 8 — Failure Scenarios](#step-8--failure-scenarios)
10. [Full System Diagram](#full-system-diagram)
11. [Quick Revision](#quick-revision)

---

## What Is It?

**Q: What does a chat application actually need to do at its core?**

> A chat app enables real-time text and media messaging between users — one-on-one and in groups. The key technical challenge is that the server must **push messages to recipients instantly** without them asking, which requires persistent connections rather than standard HTTP.

---

## Step 1 — Requirements

**Q: What are your first clarifying questions?**

> 1. How many users are we designing for? (determines scale)
> 2. What is the read/write volume expected? (determines DB and caching strategy)
> 3. Text only, or also images/videos/voice notes? (determines storage architecture)

**Q: What are the functional requirements?**

> 1. One-on-one messaging between users
> 2. Group chats (up to 256 members)
> 3. Message delivery receipts — sent ✓, delivered ✓✓, read ✓✓🔵
> 4. Online/offline status indicator
> 5. Media sharing — images, videos, voice notes
> 6. Push notifications when app is closed

**Q: What are the non-functional requirements?**

> - **High availability** — 99.99% uptime. Messages must deliver 24/7, no downtime acceptable.
> - **Low latency** — messages must arrive in real-time (milliseconds).
> - **Durability** — messages must never be lost, even if recipient is offline for days.
> - **Scalability** — 2 billion users, 100M daily active.
> - **Consistency** — messages must arrive in correct order. Stale data is NOT acceptable for messages (but is acceptable for online status and read receipts).

**Q: Is WhatsApp read-heavy or write-heavy?**

> Unlike URL shortener (100:1 reads), WhatsApp is roughly **equal reads and writes** — every message sent is also received and read. In practice it is slightly **write-heavy** because each message triggers multiple writes: storing the message, updating delivery status, updating last-seen timestamps.

---

## Step 2 — Capacity Estimation

**Q: Walk through capacity estimation for WhatsApp.**

> **Assumptions:**
> - 2 billion total users, 100M daily active
> - Each active user sends 50 messages/day
>
> **Traffic:**
> ```
> Messages/day: 100M users × 50 = 5 billion messages/day
> Writes/second: 5,000,000,000 / 86,400 ≈ 58,000 writes/second
> ```
>
> **Storage:**
> ```
> Average message = 100 bytes
> 5B messages/day × 100 bytes = 500 GB/day
> × 365 days = ~180 TB/year
> ```

**Q: What does 58,000 writes/second immediately tell you about the architecture?**

> A single database server handles ~5,000-10,000 writes/second max. At 58,000 writes/second we are **6-10x beyond a single DB's capacity**. This means:
> - Database sharding is mandatory from day one
> - Multiple app server instances required
> - A distributed message queue (Kafka) is needed, not a simple queue

---

## Step 3 — High-Level Design

**Q: Why can't WhatsApp use regular HTTP request-response for messages?**

> Standard HTTP is pull-based — the client asks, the server answers. For chat, the server needs to **push messages to your phone** without you constantly asking "any new messages?". Polling would be high-latency and extremely wasteful at 2 billion users.

**Q: What protocol does WhatsApp use and how does it work?**

> **WebSockets** — a persistent, bidirectional connection between each phone and a Chat Server.
> ```
> Phone ──── WebSocket (stays open) ────► Chat Server
>
> Chat Server can push messages to Phone at any time
> Phone can send messages to Chat Server at any time
> One connection, both directions, no request needed
> ```

**Q: Alice is connected to Chat Server A. Bob is connected to Chat Server B. How does Alice's message reach Bob?**

> Via a **Message Queue (Kafka)**. Chat Servers do not talk to each other directly.
> ```
> Alice ──WebSocket──► [Chat Server A] ──► [Kafka] ──► [Chat Server B] ──WebSocket──► Bob
> ```
> 1. Alice sends message → Chat Server A receives it
> 2. Server A publishes message to Kafka topic
> 3. Chat Server B (where Bob is connected) subscribes to Kafka
> 4. Server B receives message from Kafka → pushes to Bob via WebSocket
>
> This is **Event-Driven Architecture** — Chat Servers are decoupled and communicate through events.

**Q: Why didn't your personal small chat app need Kafka?**

> At small scale, Alice and Bob can both be on the SAME single server — the server just pushes directly. Kafka becomes necessary only when you have many servers and can no longer guarantee two users are on the same one.
> ```
> Small app:  Alice ──► [1 Server] ──► Bob  ✅ direct push
> WhatsApp:   Alice ──► [Server A]  ...  [Server B] ──► Bob  → needs Kafka bridge
> ```
> **Rule:** Message queues are only needed when you have more servers than can talk directly.

---

## Step 4 — Message Delivery Deep Dive

**Q: Bob is offline. Alice sends a message. What happens?**

> ```
> Alice sends message
>         ↓
> [Chat Server A]
>         ↓                    ↓
> [Kafka - routing]    [Cassandra - permanent storage]
>         ↓
> [Chat Server B]: "Bob has no WebSocket — he's offline"
>         ↓
> Message waits in Cassandra
>         ↓
> Bob comes online → WebSocket established
>         ↓
> Server pulls pending messages from Cassandra → pushes to Bob
>         ↓
> Bob's device receives → Server sends "delivered ✓✓" back to Alice
> Bob opens chat → Server sends "read ✓✓🔵" back to Alice
> ```

**Q: What are the three delivery states and what does each mean?**

> ```
> Sent ✓        = message reached WhatsApp servers
> Delivered ✓✓  = message reached Bob's DEVICE
> Read ✓✓🔵     = Bob opened the chat and saw the message
> ```
> These are three separate events, each triggering a status update back to Alice.

**Q: What sends Bob a notification when WhatsApp is closed?**

> **Push Notifications** via APNs (Apple) or FCM (Google). When Chat Server B can't deliver via WebSocket (app closed), it hands off to the push notification service which wakes up Bob's phone.

---

## Step 5 — Database Design

**Q: Why Cassandra instead of PostgreSQL for storing messages?**

> | WhatsApp's needs | Cassandra | PostgreSQL |
> |-----------------|-----------|------------|
> | 58,000 writes/sec | ✅ Built for this | ❌ Struggles |
> | 180 TB/year | ✅ Horizontal sharding built-in | ❌ Painful to shard |
> | Query by chat_id + time | ✅ Partition + sort key design | ❌ Needs indexes + JOINs |
> | No complex JOINs needed | ✅ Simpler is better | ❌ ACID overhead wasted |
>
> WhatsApp actually uses Cassandra in production.

**Q: What does the messages table look like in Cassandra?**

> ```
> Messages table:
> ┌─────────────┬─────────────┬──────────────┬─────────┬────────┬──────────────────────────┐
> │ chat_id     │ created_at  │ message_id   │ sender  │ type   │ content                  │
> ├─────────────┼─────────────┼──────────────┼─────────┼────────┼──────────────────────────┤
> │ alice_bob   │ 2024-01-01  │ uuid-001     │ alice   │ TEXT   │ "hey!"                   │
> │ alice_bob   │ 2024-01-01  │ uuid-002     │ bob     │ TEXT   │ "hey back!"              │
> │ alice_bob   │ 2024-01-02  │ uuid-003     │ alice   │ IMAGE  │ "cdn.whatsapp.com/a.jpg" │
> └─────────────┴─────────────┴──────────────┴─────────┴────────┴──────────────────────────┘
>
> Partition key: chat_id   → all messages for a conversation sit on the same node
> Sort key:      created_at → automatically ordered by time, newest messages fast to retrieve
> ```

**Q: Where is online/offline status stored and why not in Cassandra?**

> **Redis** — not Cassandra.
>
> Online status is:
> - Read constantly (every time you open any chat)
> - Changes frequently (every login/logout/background)
> - Temporary and ephemeral (doesn't need long-term persistence)
> - Needs sub-millisecond reads
>
> ```
> Redis:
> "alice_status" → "online"   TTL: 30s (refreshed by heartbeat every 20s)
> "bob_status"   → "offline"
> "carol_status" → "online"   TTL: 30s
>
> If heartbeat stops → TTL expires → auto-becomes "offline"
> No manual logout needed!
> ```
>
> Cassandra is built for durable long-term storage — wasteful for ephemeral status flags.

---

## Step 6 — Media Handling

**Q: Should images and videos be stored in Cassandra alongside messages?**

> **No.** A text message = ~100 bytes. A photo = ~3 MB. A video = ~50 MB.
> Storing 50MB blobs in a database row destroys performance and wastes expensive DB storage.

**Q: How does WhatsApp handle media?**

> **Object Storage (S3) + CDN:**
> ```
> User sends a photo:
> 1. Phone uploads photo directly to S3
> 2. S3 returns URL: "cdn.whatsapp.com/media/abc123.jpg"
> 3. That URL is stored in Cassandra as the message content (type: IMAGE)
> 4. Recipient's phone downloads photo from CDN (closest edge node)
>
> Cassandra stores:   type=IMAGE, content="cdn.whatsapp.com/abc123.jpg"
> S3/CDN serves:      the actual 3MB photo file
> ```
>
> **Rule:** Databases store metadata. Object storage stores files.

---

## Step 7 — API Design

**Q: Which actions use REST and which use WebSockets?**

> | Action | Protocol | Why |
> |--------|----------|-----|
> | Send/receive messages | WebSocket | Real-time push, bidirectional |
> | Online status updates | WebSocket | Continuous heartbeat |
> | Post a story | REST (POST) | One-time upload, no real-time needed |
> | Fetch chat history | REST (GET) | Pull-based, one-time load |
> | Upload media | REST (POST) | File upload to S3, not real-time |
> | Register account | REST (POST) | One-time action |

**Q: What happens at the WebSocket connection level when you open WhatsApp?**

> ```
> 1. App opens → establishes WebSocket with nearest Chat Server
> 2. Server registers: "Alice is connected to me on WebSocket #4821"
> 3. Server updates Redis: alice_status = "online" (TTL: 30s)
> 4. Server pulls any pending messages from Cassandra → pushes to Alice
> 5. Every 20s → app sends heartbeat → server refreshes Redis TTL
> 6. App closes → no heartbeat → TTL expires → Alice = "offline"
> ```

---

## Step 8 — Failure Scenarios

**Q: A Chat Server crashes mid-conversation. What happens?**

> 1. WebSocket connections to that server drop
> 2. Clients detect disconnection → automatically reconnect to another Chat Server via load balancer
> 3. New Chat Server pulls pending messages from Cassandra
> 4. User experience: brief reconnection delay (1-3 seconds), then messages continue
>
> Messages are safe — they were written to Cassandra before the server crashed.

**Q: Kafka goes down. What happens?**

> Messages between users on different Chat Servers cannot be routed. New messages fail to deliver.
>
> **Prevention:** Run Kafka as a cluster (3+ brokers). One broker failing doesn't affect the cluster. Kafka is designed for this — it's inherently distributed.

**Q: The Cassandra cluster loses a node. What happens?**

> Cassandra replicates data across multiple nodes (typically replication factor = 3). Losing one node means the other two still have the data. Zero data loss, zero downtime.
>
> **Cassandra's design principle:** No single point of failure. Every node is equal, data is always replicated.

**Q: How do you scale to handle 10x message volume?**

> ```
> Current bottleneck order:
> 1. Chat Servers    → add more instances behind load balancer
> 2. Kafka           → add more partitions and brokers
> 3. Cassandra       → add more nodes (automatic rebalancing)
> 4. Redis           → switch to Redis Cluster
> 5. S3/CDN          → already infinitely scalable, no action needed
> ```

---

## Full System Diagram

```
[Alice's Phone]                              [Bob's Phone]
      │ WebSocket                                  │ WebSocket
      ▼                                            ▼
[Load Balancer]                            [Load Balancer]
      │                                            │
[Chat Server A] ──────[Kafka]────────────[Chat Server B]
      │                                            │
      │                                     (Bob offline?)
      │                                            │
      ▼                                            ▼
[Cassandra DB]                            [Push Notification]
(all messages)                            (APNs / FCM)
      │
[Redis Cache]                [S3 + CDN]
(online status,              (images, videos,
 recent messages)             voice notes)
```

**Request flows:**
```
SEND MESSAGE (Bob online):
Alice → Chat Server A → Kafka → Chat Server B → Bob ✓✓

SEND MESSAGE (Bob offline):
Alice → Chat Server A → Cassandra (stored)
Bob comes online → Chat Server B pulls from Cassandra → Bob ✓✓

SEND PHOTO:
Alice → uploads to S3 → URL stored in Cassandra as message → Bob downloads from CDN
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| Real-time protocol | WebSockets | Persistent bidirectional — server pushes without client asking |
| Cross-server routing | Kafka | Chat Servers decoupled, event-driven message routing |
| Message storage | Cassandra | 58,000 writes/sec, horizontal scaling, time-ordered queries |
| Online status | Redis | Ephemeral, sub-millisecond reads, TTL auto-expiry |
| Media storage | S3 + CDN | Files don't belong in DB rows — object storage is cheaper and scalable |
| Offline delivery | Store in Cassandra, push on reconnect | Durability guaranteed regardless of recipient status |
| Notifications (app closed) | Push (APNs/FCM) | WebSocket unavailable when app is closed |
| Delivery receipts | Sent/Delivered/Read as separate events | Each state = separate write back to sender |

### Three things to mention proactively in an interview
> 1. **WebSocket vs HTTP** — explain WHY WebSocket is needed (server push), not just that you use it
> 2. **Kafka for cross-server routing** — shows you understand that at scale, users won't share a server
> 3. **Redis for online status with TTL** — shows you understand ephemeral data doesn't belong in a persistent DB
