# Rate Limiting + Message Queues & Backpressure

> "Our API was getting hammered by a single misconfigured client sending 10,000 requests per second. We had no rate limiting. That was a bad day."
> — Every startup engineer, approximately six months after launch

Two topics. One theme: **protecting your system from being destroyed by traffic it wasn't designed to handle** — whether that traffic is external (rate limiting) or internal (backpressure in message queues).

Let's learn from companies that did it wrong, then did it right.

---

# PART 1: Rate Limiting

---

## Q1: What is rate limiting and why should I care?

**Short answer:** Rate limiting is a policy that caps how many requests a client can make in a given time window.

**Long answer with a real incident:**

In 2012, a bug in a single client library caused thousands of apps using the Twitter API to flood Twitter's servers simultaneously. Twitter had rate limits, but they weren't aggressive enough. The result: cascading failures, degraded performance for everyone, and emergency ops across the company.

Rate limiting protects you from three different threats:

**1. External Abuse (DDoS / scraping)**
```
Attacker              Your API
   |                     |
   |--- 100,000 req/s -->|  <- system starts melting
   |                     |  <- legitimate users get 503s
   |                     |  <- on-call engineer gets paged at 2am
```

**2. Accidental Abuse (buggy clients)**
```
while True:
    response = requests.get("https://api.yoursite.com/data")
    # forgot the sleep() call — oops
```
This is more common than actual attacks. A misconfigured retry loop, a missing `sleep()`, a runaway cron job — and suddenly one customer is consuming 99% of your capacity.

**3. Cost Control (third-party APIs)**
If your app calls OpenAI, Stripe, Twilio, etc. — every request costs money. Rate limiting your own internal callers prevents a bug from generating a $50,000 AWS bill overnight.

**The three main reasons to rate limit:**
- **DDoS protection** — stop malicious flood attacks
- **Fair use enforcement** — one customer can't starve others
- **Cost control** — both infrastructure and third-party API costs

---

## Q2: Token Bucket algorithm — explain it like I'm five

**The mental model:** You have a bucket that holds tokens. Every API request costs one token. Tokens refill at a constant rate. If the bucket is empty, the request is rejected.

```
                    +-----------+
  REFILL RATE       |  [token]  |
  (e.g., 10/sec) -->|  [token]  |  <-- CAPACITY: 100 tokens
                    |  [token]  |
                    |    ...    |
                    +-----------+
                         |
                         v
              Each request consumes 1 token
              
              No tokens? --> HTTP 429 Too Many Requests
```

**ASCII flow:**

```
Time: 0s   Bucket: [##########] 10 tokens
           Client sends 3 requests -> [#######] 7 tokens

Time: 1s   Bucket refills 10 -> [##########+#######] but capped at 10
           Bucket: [##########] 10 tokens

Time: 2s   Client sends 15 requests -> 10 succeed, 5 rejected (429)
           Bucket: [          ] 0 tokens
```

**Key property: bursting is allowed.** If a client has been quiet for a while, they accumulate tokens and can send a burst. This is often desirable (real users have bursty behavior).

**Pseudocode:**

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity        # max tokens
        self.tokens = capacity          # current tokens
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.now()

    def allow_request(self):
        # First, refill based on elapsed time
        now = time.now()
        elapsed = now - self.last_refill
        self.tokens = min(
            self.capacity,
            self.tokens + elapsed * self.refill_rate
        )
        self.last_refill = now

        # Then check if we have a token to spend
        if self.tokens >= 1:
            self.tokens -= 1
            return True   # allow request
        return False      # reject with 429
```

**Real-world use:** AWS API Gateway uses token bucket. Most cloud providers do.

---

## Q3: Leaky Bucket — how is it different from Token Bucket?

**The mental model:** Requests go INTO a bucket (queue). The bucket "leaks" requests OUT at a constant rate. If the bucket overflows (queue full), new requests are dropped.

```
                 Requests IN (variable rate)
                      |   |  |
                      v   v  v
                 +----------+
                 |  request |
                 |  request |  <- BUCKET (fixed size queue)
                 |  request |
                 |          |
                 +----------+
                      |
                      | (constant drip rate)
                      v
                 Requests OUT (fixed rate)
```

**Key difference from Token Bucket:**

| Property | Token Bucket | Leaky Bucket |
|---|---|---|
| Allows bursts? | YES (if tokens saved up) | NO (constant output rate) |
| Output rate | Variable | Constant |
| Good for | APIs, general rate limiting | Traffic shaping, network QoS |
| Burst behavior | Absorbs bursts | Smooths bursts into steady flow |

**The intuition:**
- Token Bucket: "You can go fast if you've been patient."
- Leaky Bucket: "No matter how fast you arrive, you leave one at a time."

Leaky bucket is used more in **network equipment** (like routers) where you need perfectly smooth output. Token bucket is more common in **API rate limiting** because it's friendlier to legitimate users with bursty patterns.

---

## Q4: Fixed Window Counter — simple but has a nasty bug

**The approach:** Divide time into fixed windows (e.g., each minute). Count requests per window. Reset at window boundary.

```
Window 1 [12:00:00 - 12:00:59]   Window 2 [12:01:00 - 12:01:59]
+-----------------------------+   +-----------------------------+
|  Limit: 100 requests        |   |  Limit: 100 requests        |
|  Count: 75                  |   |  Count: 0 (just reset)      |
+-----------------------------+   +-----------------------------+
```

**The edge case bug — the boundary attack:**

```
Window 1 ends at 12:00:59, Window 2 starts at 12:01:00

12:00:58 ----> 100 requests (fills Window 1 completely)
12:01:00 ----> 100 requests (Window 2 just reset, all allowed)

Result: 200 requests in 2 seconds — DOUBLE your intended limit!
```

An attacker who knows your window boundary can send 2x your rate limit by timing requests to straddle two windows. This is called the **boundary problem** or **thundering herd at reset**.

**Pseudocode:**

```python
def allow_request(user_id, limit):
    window_key = f"{user_id}:{current_minute()}"
    count = redis.incr(window_key)
    if count == 1:
        redis.expire(window_key, 60)  # expires in 60s
    return count <= limit
```

Simple, cheap, but vulnerable. Use it only when the boundary attack doesn't matter (internal tools, generous limits).

---

## Q5: Sliding Window Counter — the Stripe way

This is the algorithm Stripe described in their famous engineering blog post. It fixes the boundary problem while keeping the implementation practical.

**The insight:** Instead of hard resets, blend the current window with the previous window using a weighted average.

```
Timeline:
|--- Window N-1 ---|--- Window N ---|--> time
                   ^
                   | You are here, 30% into Window N

Effective count = (Window N count) + (Window N-1 count * 0.70)
                                                        ^
                                     70% of previous window still "counts"
                                     because 70% of a 60s window = 42s ago
```

**Full example:**

```
Window N-1 (previous minute): 80 requests
Window N   (current minute):  30 requests (and we're 40% through it)

Sliding window count = 30 + 80 * (1 - 0.40)
                     = 30 + 80 * 0.60
                     = 30 + 48
                     = 78 requests

If limit is 100: request is ALLOWED
```

**Why this is elegant:** There's no hard reset. The count smoothly transitions between windows, so the boundary attack doesn't work.

**Pseudocode (Redis-backed):**

```python
def allow_request_sliding(user_id, limit, window_seconds=60):
    now = time.now()
    window_start = floor(now / window_seconds) * window_seconds
    prev_window_start = window_start - window_seconds

    # How far through the current window are we? (0.0 to 1.0)
    elapsed_fraction = (now - window_start) / window_seconds

    # Get counts for current and previous windows
    curr_key = f"{user_id}:{window_start}"
    prev_key = f"{user_id}:{prev_window_start}"

    curr_count = int(redis.get(curr_key) or 0)
    prev_count = int(redis.get(prev_key) or 0)

    # Weighted blend
    weight = 1.0 - elapsed_fraction
    effective_count = curr_count + (prev_count * weight)

    if effective_count < limit:
        redis.incr(curr_key)
        redis.expire(curr_key, window_seconds * 2)
        return True  # allow
    return False  # reject 429
```

**Stripe's actual approach** used Redis sorted sets (ZSET) to store individual request timestamps per user, allowing perfect sliding window accuracy — but the weighted counter approach above is a common, cheaper approximation used in production.

---

## Q6: Distributed rate limiting with Redis — how do you share state across API servers?

**The problem:** You have 10 API servers behind a load balancer. If each server tracks its own counter, a client can send 10x your limit by distributing requests across servers.

```
                    [Load Balancer]
                   /    |    |    \
               [S1]   [S2]  [S3]  [S4]
               cnt:25 cnt:25 cnt:25 cnt:25
                 
User sent 100 requests — 25 to each server.
Each server thinks: "I've only seen 25, limit is 100, allow it."
Real total: 100 requests, but ZERO servers rejected anything.
```

**Solution: centralized Redis as the rate limit store:**

```
                    [Load Balancer]
                   /    |    |    \
               [S1]   [S2]  [S3]  [S4]
                \      |      |    /
                 \     |      |   /
                  +----+------+--+
                         |
                    [ REDIS ]
                    key: "user:42:2024010112"
                    value: 87  (requests this minute)
```

**Redis INCR is atomic** — this is the key insight. Redis is single-threaded and processes commands atomically, so there's no race condition when multiple servers increment the same counter.

**The Twitter 429 story:** Twitter's API rate limiting is built exactly this way. Developers who hit the limit get a `HTTP 429 Too Many Requests` response with headers like:
```
X-Rate-Limit-Limit: 900
X-Rate-Limit-Remaining: 0
X-Rate-Limit-Reset: 1623456789  (unix timestamp when it resets)
```

Developers simultaneously hate and need these headers. They hate them because their apps break. They need them because without them, they'd have no idea when to retry.

**Cloudflare's related lesson (2019):** Cloudflare's infamous 2019 outage wasn't caused by rate limiting failures, but by resource exhaustion — a single overly-complex WAF rule caused 100% CPU usage across all their servers worldwide, knocking out 18% of the internet for ~27 minutes. The lesson is the same: **systems need to protect themselves from resource exhaustion**, whether the source is external clients or internal logic. Rate limiting is one layer; CPU and memory limits are another.

**Redis Lua script for atomic sliding window (the production pattern):**

```lua
-- Atomic sliding window check + increment
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local now = tonumber(ARGV[2])
local window = tonumber(ARGV[3])

-- Remove old entries outside the window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Count entries in current window
local count = redis.call('ZCARD', key)

if count < limit then
    -- Add this request's timestamp
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window)
    return 1  -- allowed
else
    return 0  -- rejected
end
```

---

# PART 2: Message Queues & Backpressure

---

## Q7: What is a message queue and why does it decouple systems?

**The problem without a queue:**

Imagine your e-commerce site: when an order is placed, you need to:
1. Charge the credit card
2. Send a confirmation email
3. Notify the warehouse
4. Update inventory
5. Log analytics

**Without a queue (synchronous):**
```
User clicks "Buy"
      |
      v
[API Server]
      |---> [Payment Service]    (50ms)
      |---> [Email Service]      (200ms)  <- SLOW
      |---> [Warehouse Service]  (100ms)
      |---> [Inventory Service]  (30ms)
      |---> [Analytics]          (80ms)
      |
      v
User waits 460ms total... IF nothing fails.
If Email Service is down, the entire order fails.
```

**With a queue:**
```
User clicks "Buy"
      |
      v
[API Server]
      |
      |-- publish --> [Order Queue]
      |
      v
User gets response in 5ms. Order is "accepted."

Meanwhile, workers process from the queue asynchronously:
[Order Queue] --> [Payment Worker]    (whenever it's ready)
[Order Queue] --> [Email Worker]      (even if it's slow, it catches up)
[Order Queue] --> [Warehouse Worker]  (independent, can retry on failure)
```

**The three superpowers of message queues:**
- **Decoupling**: Producer doesn't know or care about consumers
- **Buffering**: Queue absorbs traffic spikes; workers process at their own pace
- **Resilience**: If a consumer crashes, messages wait in the queue (not lost)

---

## Q8: Backpressure — what happens when consumers can't keep up?

**The LinkedIn origin story:**

In 2010-2011, LinkedIn's internal data infrastructure was a mess. They had dozens of pipelines where data producers (activity events, profile updates, messages) generated data faster than consumers (Hadoop jobs, search indexers, analytics) could process it. The queues grew unbounded. Systems fell over. This pain directly motivated Jay Kreps and colleagues to design **Apache Kafka** in 2011.

**What backpressure looks like:**

```
Producer                  Queue               Consumer
   |                        |                    |
   |-- 1000 msg/sec ------->|                    |
                            |<--- 100 msg/sec ---|
                            |
                   Queue grows 900 msg/sec
                            |
                   [after 1 minute: 54,000 messages backlogged]
                   [after 10 minutes: queue hits memory limit]
                   [SYSTEM CRASHES or starts dropping messages]
```

**Three strategies for handling backpressure:**

**1. Drop (Lossy)**
```
if queue.size() > MAX_SIZE:
    drop(message)  # just throw it away
    
# Use when: metrics, logs, non-critical events
# Never use when: financial transactions, orders
```

**2. Block (Apply Pressure Upstream)**
```
# Producer is forced to wait when queue is full
queue.put(message, block=True)  # producer blocks here

# Use when: you can slow the producer safely
# Risk: cascading slowdowns all the way to the user
```

**3. Reject with Signal (HTTP 429 / backpressure signal)**
```
# Consumer signals "I'm overwhelmed" to the producer
HTTP 503 Service Unavailable
Retry-After: 30

# Producer backs off and retries later
# Use when: producer and consumer are separate services
```

**The Robinhood 2021 incident:**

During the meme stock frenzy (GameStop, AMC), Robinhood's trading systems got hit with unprecedented order volume. Their internal message queue that handled order processing grew unbounded — they had no proper backpressure mechanism. The queue size exploded, consuming memory, slowing down processing, and eventually contributing to the system instability that forced them to restrict trading. The irony: the restriction that caused public outrage was partly an emergency backpressure mechanism applied at the wrong layer (the user interface instead of the infrastructure).

---

## Q9: At-least-once vs At-most-once vs Exactly-once delivery

This is one of the most important (and most confused) concepts in distributed systems.

**At-most-once (fire and forget):**
```
Producer                    Queue              Consumer
    |                         |                   |
    |-- send message -------->|                   |
    |                         |-- deliver ------->|
    |                         |                   |
    |                         X  (queue crashes)  |
    
Message is lost. Consumer processed it 0 times.
Producer doesn't know. Nobody retries.

USE WHEN: Metrics, logging, non-critical analytics.
         "If we miss a few pageview events, who cares."
```

**At-least-once (most common in practice):**
```
Producer                    Queue              Consumer
    |                         |                   |
    |-- send message -------->|                   |
    |                         |-- deliver ------->|
    |                         |   (consumer crashes before ACK)
    |                         |                   X
    |                         |-- redeliver ----->|
    |                         |                   |-- process (AGAIN)
    
Message is processed 1 or more times.
Consumer must be IDEMPOTENT (safe to process twice).

USE WHEN: Most business events — emails, notifications, webhooks.
REQUIREMENT: Make your consumer idempotent!
```

**Idempotency example:**
```python
def send_welcome_email(user_id, message_id):
    # Check if we already sent this email
    if email_log.exists(message_id):
        return  # already done, skip
    
    send_email(user_id, "Welcome!")
    email_log.record(message_id)  # mark as sent
```

**Exactly-once (holy grail, very expensive):**
```
Requires distributed transactions or idempotent consumers + dedup IDs.

The "exactly-once" claim from Kafka is technically 
"effectively-once within a single Kafka cluster" — 
which is still impressive engineering but has caveats.

USE WHEN: Financial transactions, inventory deductions.
COST: Significant coordination overhead, slower, complex.
```

**Quick cheat sheet:**
```
At-most-once  = fast, simple, lossy      -> metrics, logs
At-least-once = reliable, needs idempotency -> most business events  
Exactly-once  = expensive, complex       -> money, inventory
```

---

## Q10: Dead Letter Queue (DLQ) — where messages go to die (and be investigated)

**The problem:** What happens when a consumer fails to process a message? It retries. And retries. And retries. This is called a **poison pill message** — one bad message can block the entire queue.

```
Queue: [msg1][msg2][BAD_MSG][msg4][msg5]
                      |
                      v
Consumer tries to process BAD_MSG
    -> Fails (exception thrown)
    -> Retried 3 times
    -> Still fails
    -> NOW WHAT?
    
Option 1: Keep retrying forever -> queue is stuck
Option 2: Drop it -> data loss, nobody knows
Option 3: Move to Dead Letter Queue -> CORRECT
```

**Dead Letter Queue flow:**

```
Normal Queue                    DLQ
+------------------+           +------------------+
| [msg1]           |           |                  |
| [msg2]           |           |                  |
| [BAD_MSG] -------+---------> | [BAD_MSG]        |
| [msg4]           |  after    |   (timestamp)    |
| [msg5]           |  N fails  |   (error reason) |
+------------------+           |   (retry count)  |
                               +------------------+
                                        |
                               Alert fires to on-call
                               Engineer investigates
                               Fix the bug
                               Replay message
```

**DLQ configuration (AWS SQS example):**
```json
{
  "RedrivePolicy": {
    "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123:my-dlq",
    "maxReceiveCount": 3
  }
}
```

After 3 failed delivery attempts, the message automatically moves to the DLQ.

**Why DLQs are critical for operations:**
- Bad messages don't block good ones
- Engineers can investigate failures without losing data
- You can replay fixed messages after deploying a fix
- Alerting on DLQ depth tells you when something is broken

---

## Q11: Kafka specifically — what makes it different from RabbitMQ?

**The LinkedIn origin (revisited):**

When LinkedIn engineers built Kafka in 2011, they had a specific insight: existing message queues (ActiveMQ, RabbitMQ) were built for **job queues** — tasks that get consumed once and deleted. LinkedIn needed a **log** — a durable, replayable stream of events that multiple systems could read independently.

**Kafka's architecture:**

```
                        KAFKA CLUSTER
                        
Topic: "user-events"
                        
Partition 0: [msg1][msg2][msg5][msg8]----> offset 3
Partition 1: [msg3][msg6][msg9]----------> offset 2  
Partition 2: [msg4][msg7][msg10]----------> offset 2

                        ^
              Messages are APPENDED (like a log)
              Never deleted immediately
              Stored on disk (not just in memory)
```

**Consumer Groups — the magic:**

```
Topic: "user-events" (3 partitions)

Consumer Group A (Search Indexer):
  Worker A1 --> reads Partition 0 (offset: 847)
  Worker A2 --> reads Partition 1 (offset: 1203)
  Worker A3 --> reads Partition 2 (offset: 956)

Consumer Group B (Analytics):
  Worker B1 --> reads ALL partitions (offset: 200 - it's behind!)
  
Both groups read the SAME messages independently.
Search Indexer doesn't know Analytics exists.
Analytics can be days behind without affecting Search.
```

**This is the fundamental difference from RabbitMQ:**

| Feature | RabbitMQ | Kafka |
|---|---|---|
| Message storage | In-memory queue | Durable log on disk |
| After consumption | Message deleted | Message retained (configurable) |
| Multiple consumers | Competing workers (each message to ONE) | Each consumer group reads independently |
| Replay messages | No (gone after consume) | YES (rewind to any offset) |
| Throughput | Good | Extreme (millions/sec per partition) |
| Use case | Task queues, RPC | Event streaming, audit log, data pipelines |

**When to use Kafka:**
- Multiple different services need to react to the same event
- You need to replay historical events (e.g., rebuild a search index)
- You need extremely high throughput
- You want an audit log of everything that happened

**When to use RabbitMQ:**
- Simple task distribution (process these 10,000 image thumbnails)
- Request-response patterns
- You need complex routing (exchange, binding rules)
- You don't need replay

**Offset management matters:**

```python
# Consumer manually commits offset after successful processing
consumer = KafkaConsumer('user-events', group_id='analytics')

for message in consumer:
    try:
        process(message.value)
        consumer.commit()  # <- only commit AFTER success
    except Exception as e:
        log_error(e)
        # Don't commit -> message will be redelivered (at-least-once)
```

If you commit offset BEFORE processing, you get at-most-once (messages can be lost if you crash). If you commit AFTER, you get at-least-once (may reprocess on crash). Exactly-once requires Kafka transactions.

---

## Q12: When should you use a message queue vs a direct API call?

**Use a direct API call when:**
- You need an immediate response to proceed
- The operation is fast and reliable
- Failure should bubble up to the user immediately
- Example: "Is this credit card valid?" → need the answer NOW

**Use a message queue when:**

```
1. ASYNC IS ACCEPTABLE
   "Send welcome email after signup"
   User doesn't need to wait for the email to be sent
   
2. SPIKE ABSORPTION
   Black Friday: 100x normal order volume hits at once
   Queue absorbs the spike; workers process at safe rate
   
3. DECOUPLING SERVICES
   Payment service shouldn't care if Email service is down
   
4. FAN-OUT
   One order event triggers: warehouse + email + analytics + loyalty points
   One publish, many independent consumers
   
5. RETRY SEMANTICS
   "Resize this uploaded image to 5 different sizes"
   If it fails, retry automatically; user doesn't need to re-upload
```

**The decision tree:**

```
Does the caller need the result immediately?
    YES --> Direct API call
    NO  --> Consider a queue
    
Can the caller tolerate the service being down?
    YES --> Queue (absorbs failures)
    NO  --> Direct API call (fail fast)
    
Will multiple services react to this event?
    YES --> Queue with multiple consumer groups
    NO  --> Direct API call
    
Is this a high-volume event (>1000/sec)?
    YES --> Queue (batch processing more efficient)
    NO  --> Either works
```

---

# Quick Revision Table

## Rate Limiting Algorithms

| Algorithm | How it works | Burst friendly? | Complexity | Best for |
|---|---|---|---|---|
| Fixed Window | Count per time window, reset at boundary | No | Simple | Internal tools, generous limits |
| Token Bucket | Tokens refill at constant rate, requests cost tokens | YES | Medium | Most API rate limiting |
| Leaky Bucket | Requests queue up, processed at constant rate | No | Medium | Network traffic shaping |
| Sliding Window | Weighted blend of current + previous window | Partial | Medium-High | Public APIs (Stripe's approach) |

## Message Queue Concepts

| Concept | One-liner | When it matters |
|---|---|---|
| At-most-once | Fast, may lose messages | Metrics, logs |
| At-least-once | Reliable, may duplicate | Most business events |
| Exactly-once | No duplicates, no loss | Payments, inventory |
| DLQ | Quarantine for failed messages | Always configure this |
| Backpressure | Mechanism to slow producers when consumers lag | High-throughput systems |
| Consumer Group | Multiple independent readers of same Kafka topic | Kafka specifically |

## Kafka vs RabbitMQ at a Glance

| | Kafka | RabbitMQ |
|---|---|---|
| Mental model | Distributed commit log | Message broker / task queue |
| Message after consume | Kept (until retention period) | Deleted |
| Replay possible | YES | No |
| Multiple consumer types | YES (consumer groups) | Limited |
| Throughput | Extreme | High |
| Operational complexity | Higher | Lower |
| Sweet spot | Event streaming, audit logs | Task distribution, RPC |

## Real Incidents Summary

| Company | What happened | Lesson |
|---|---|---|
| Twitter (2012) | Buggy client library flooded API, rate limits insufficient | Rate limits must be aggressive enough AND clearly communicated |
| Cloudflare (2019) | Complex WAF regex caused 100% CPU, 18% of internet down 27 min | Resource exhaustion is rate limiting's cousin; protect compute too |
| LinkedIn (2011) | Data pipeline backpressure caused cascading failures | Led to Kafka's creation; backpressure needs a first-class solution |
| Robinhood (2021) | Order queue grew unbounded during meme stock frenzy | Unbounded queues are a liability; set limits and design for overflow |
| Stripe (ongoing) | Pioneered Redis sliding window rate limiting | Published approach became industry standard |

---

> **The meta-lesson:** Both rate limiting and message queues are about the same thing — **respecting the finite capacity of your system**. Water flows at the speed the pipe can handle. Design your systems to shape traffic, not just absorb it.
