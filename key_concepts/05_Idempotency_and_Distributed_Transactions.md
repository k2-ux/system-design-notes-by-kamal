# Idempotency + Distributed Transactions

> "The network is reliable." — The first lie of distributed systems.

---

## Part 1: Idempotency

---

### Q1. What even IS idempotency? Give me the plain-English version.

**Short answer:** Do it once, do it a hundred times — same result either way.

The math version: `f(f(x)) = f(x)`

That fancy formula just means: applying an operation more than once produces the exact same outcome as applying it once.

**Real-world analogy:** Pressing the elevator button. Press it once — elevator comes. Press it 50 times out of impatience — elevator still comes exactly once. The button is idempotent. Pressing it more doesn't summon more elevators.

**The opposite:** Imagine a vending machine that gives you a snack every time you press a button. That's NOT idempotent. Press 50 times, get 50 snacks (and a stomachache).

**HTTP verbs — the cheat sheet:**

| Method | Idempotent? | Safe? | Notes |
|--------|-------------|-------|-------|
| GET | YES | YES | Fetching data never changes it |
| PUT | YES | NO | Replace resource — same result on repeat |
| DELETE | YES | NO | Delete twice = still deleted (404 on second, but state is the same) |
| POST | NO | NO | Creating a new resource each time |
| PATCH | SOMETIMES | NO | Depends on implementation (relative vs absolute updates) |

```
GET /orders/123         → idempotent (just reads)
PUT /orders/123 {qty:5} → idempotent (sets qty to 5, always)
DELETE /orders/123      → idempotent (deleted is deleted)
POST /orders {qty:5}    → NOT idempotent (creates new order each call)
```

**Why should you care?** Because distributed systems have exactly ONE superpower and ONE weakness:
- Superpower: You can retry failed operations
- Weakness: You can't tell if a failure happened before or after the operation executed

If your operations aren't idempotent, retrying is gambling.

---

### Q2. Why do network retries cause duplicate operations? Walk me through the disaster.

**The failure modes nobody tells you about:**

```
Client                    Network                   Server
  |                          |                         |
  |------- POST /charge ---->|                         |
  |                          |-------- arrives ------->|
  |                          |                   [CHARGED $100]
  |                          |                   [DB write OK]
  |                          |                         |
  |                          |<-- response lost! ----X |
  |                          |                         |
  | (timeout after 30s)      |                         |
  |                          |                         |
  | RETRY!                   |                         |
  |------- POST /charge ---->|                         |
  |                          |-------- arrives ------->|
  |                          |                   [CHARGED $100 AGAIN]
  |                                                     |
  CUSTOMER NOW CHARGED $200 FOR ONE ORDER
```

The client never got confirmation. From the client's perspective: the request failed. From the server's perspective: it succeeded perfectly. This is the **dual failure problem** — you can't know which side of the fence you're on.

**Uber's 2015 Double-Charge Bug (Real Incident)**

In 2015, Uber had a nasty bug where riders were being charged twice for the same trip. The root cause: their payment service would retry failed payment API calls without any idempotency mechanism. Here's what happened:

1. Rider completes trip
2. Uber backend calls payment processor: `POST /charge {amount: $23.50}`
3. Payment processor charges the card and returns 200 OK
4. Response gets lost in the network (or there's a timeout)
5. Uber's backend sees a timeout → assumes failure → retries
6. Payment processor charges the card AGAIN: `POST /charge {amount: $23.50}`
7. Rider gets charged $47.00 for a $23.50 trip

The fix? Idempotency keys. Every charge attempt carries a unique identifier, and the payment processor remembers "I've already processed this exact request."

**Amazon AWS Billing (2012)**

AWS experienced incidents where retry storms (cascading timeouts during a service disruption) caused multiple billing charges for the same API calls. The retry logic was correct and necessary — the missing piece was idempotency at the billing layer. AWS has since made this a core principle: every billable API operation must be idempotent.

**The network failure taxonomy:**

```
Three possible outcomes when you send a request:

1. Request never arrived     → Server did nothing   → Safe to retry
2. Request arrived, FAILED   → Server did nothing   → Safe to retry  
3. Request arrived, SUCCEEDED→ Server did the thing → DANGEROUS to retry
                               but response was lost
                               
You CANNOT distinguish case 1/2 from case 3 on the client side.
This is why you need idempotency keys.
```

---

### Q3. How do idempotency keys actually work? Show me the mechanism.

**The concept:** Client generates a unique key per logical operation. Server stores the result of that operation keyed by that ID. On retry, server returns the stored result without re-executing.

```
FIRST REQUEST:
Client                                        Server
  |                                              |
  | POST /charges                                |
  | Idempotency-Key: uuid-abc-123               |
  | {amount: 2999, card: tok_xxx}               |
  |--------------------------------------------->|
  |                                    Check DB: |
  |                                    key not found → process|
  |                                    Charge card: SUCCESS   |
  |                                    Store result in DB:    |
  |                                    {key: uuid-abc-123,    |
  |                                     result: {id: ch_1,   |
  |                                     status: "succeeded"}} |
  |<----------- 200 {id: ch_1, status: "succeeded"} ---------|


RETRY (network hiccup, client didn't get response):
Client                                        Server
  |                                              |
  | POST /charges                                |
  | Idempotency-Key: uuid-abc-123  ← SAME KEY  |
  | {amount: 2999, card: tok_xxx}               |
  |--------------------------------------------->|
  |                                    Check DB: |
  |                                    key FOUND! Return cached result |
  |                                    NO charge executed              |
  |<----------- 200 {id: ch_1, status: "succeeded"} ---------|
  |                                    (SAME response, no duplicate!)
```

**Key properties of idempotency keys:**
- Generated by the **client** (not the server)
- Should be a UUID (random, globally unique)
- Scoped to a specific operation type (don't reuse keys across different operations)
- Should expire after some window (e.g., 24 hours, 7 days)

**Stripe's Idempotency Key System (The Gold Standard)**

Stripe built this into their entire API because double-charging users is existentially dangerous for a payments company. Their implementation:

1. `Idempotency-Key` header on every POST request
2. Keys are stored with a 30-day expiry
3. If the same key arrives during an in-flight request, they return `409 Conflict` (don't process twice in parallel)
4. After completion, always return the stored result
5. Key + endpoint + request body must match — if they differ, return `422` (prevent key reuse for different operations)

From Stripe's engineering blog: *"Storing the result of operations keyed by idempotency keys is one of the most important things you can do to make your payments integration robust."*

---

### Q4. How does idempotency work in the database layer?

**The naive problem:**

```sql
-- NOT idempotent — creates duplicate rows
INSERT INTO orders (user_id, product_id, amount)
VALUES (42, 99, 2999);

-- Run this twice = two order rows. Nightmare.
```

**Solution 1: UPSERT (UPDATE + INSERT)**

```sql
-- PostgreSQL: idempotent with ON CONFLICT
INSERT INTO orders (order_id, user_id, product_id, amount)
VALUES ('ord-uuid-123', 42, 99, 2999)
ON CONFLICT (order_id) DO NOTHING;

-- MySQL equivalent
INSERT IGNORE INTO orders (order_id, user_id, product_id, amount)
VALUES ('ord-uuid-123', 42, 99, 2999);

-- Or UPDATE on conflict:
INSERT INTO inventory (product_id, quantity)
VALUES (99, 100)
ON CONFLICT (product_id) 
DO UPDATE SET quantity = EXCLUDED.quantity;
-- This sets quantity to 100, not adds 100. Idempotent!
```

**Solution 2: Idempotency table approach**

```sql
-- Create a separate table to track processed operations
CREATE TABLE idempotency_keys (
    key         VARCHAR(255) PRIMARY KEY,
    endpoint    VARCHAR(255) NOT NULL,
    result      JSONB,
    created_at  TIMESTAMP DEFAULT NOW(),
    expires_at  TIMESTAMP
);

-- In a transaction: check key, process, store result atomically
BEGIN;
  SELECT result FROM idempotency_keys 
  WHERE key = 'uuid-abc-123' 
  FOR UPDATE;  -- lock this row
  
  -- If found: return stored result, ROLLBACK
  -- If not found: process operation, INSERT key+result, COMMIT
COMMIT;
```

**Idempotent vs Non-Idempotent operations:**

```
IDEMPOTENT:
  SET user.balance = 100        (absolute set)
  UPDATE status = 'cancelled'   (setting a state)
  DELETE WHERE id = 42          (same result on repeat)
  INSERT ... ON CONFLICT IGNORE (deduplicated)

NOT IDEMPOTENT:
  UPDATE balance = balance - 50    (relative update - dangerous!)
  INSERT INTO ... (no conflict handling)
  balance += amount                (accumulates on retry)
```

**The trap with relative updates:**

```sql
-- This is a disaster if retried:
UPDATE accounts SET balance = balance - 100 WHERE id = 42;

-- Run twice → subtract $100 twice. Customer loses $200.

-- Fix: use versioning or idempotency key check
UPDATE accounts 
SET balance = balance - 100, version = version + 1
WHERE id = 42 AND version = 5;  -- only succeeds once (optimistic locking)
```

---

### Q5. How does idempotency work in message queues? (Kafka/SQS deduplication)

**The problem:** Message queues guarantee **at-least-once delivery**. That means the same message CAN arrive more than once. Your consumer must handle duplicates.

```
SQS Queue                    Consumer
    |                            |
    |--- Message: "charge $50" ->|
    |                            | [processing...]
    |                            | [charged $50]
    |                            | [visibility timeout expired before ACK]
    |--- Message: "charge $50" ->|  (SAME MESSAGE REDELIVERED!)
    |                            | [charges $50 AGAIN]
    |                            | BAD!
```

**Solution 1: SQS Message Deduplication IDs (FIFO queues)**

```python
# Producer: always set MessageDeduplicationId
sqs.send_message(
    QueueUrl='https://sqs.us-east-1.amazonaws.com/123/MyQueue.fifo',
    MessageBody=json.dumps({"user_id": 42, "amount": 50}),
    MessageGroupId='payments',
    MessageDeduplicationId='order-uuid-abc-123'  # SQS deduplicates within 5 min
)
```

**Solution 2: Consumer-side deduplication**

```python
# Redis-based deduplication on the consumer
import redis
import json

r = redis.Redis()

def process_message(message):
    message_id = message['MessageId']
    dedup_key = f"processed:{message_id}"
    
    # SET NX = Set if Not eXists — atomic check-and-set
    was_new = r.set(dedup_key, '1', nx=True, ex=86400)  # 24hr TTL
    
    if not was_new:
        print(f"Duplicate message {message_id}, skipping")
        return  # already processed, skip safely
    
    # Process the message only if we set the key
    charge_user(message['user_id'], message['amount'])

def consume():
    while True:
        messages = sqs.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=10
        )
        for msg in messages.get('Messages', []):
            process_message(json.loads(msg['Body']))
            sqs.delete_message(
                QueueUrl=QUEUE_URL,
                ReceiptHandle=msg['ReceiptHandle']
            )
```

**Kafka deduplication with idempotent producers:**

```python
# Kafka producer — enable idempotence
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    enable_idempotence=True,  # producer-side: exactly-once delivery per partition
    acks='all'
)

# Consumer side: track processed offsets in your DB
def process_kafka_message(msg):
    offset_key = f"{msg.topic}:{msg.partition}:{msg.offset}"
    
    with db.transaction():
        # Check if already processed
        if db.exists("processed_offsets", offset_key):
            return
        
        # Process + mark as done, atomically
        do_the_work(msg.value)
        db.insert("processed_offsets", offset_key)
```

---

### Q6. Show me a complete idempotent payment endpoint implementation.

```python
# FastAPI idempotent payment endpoint
import uuid
import json
from datetime import datetime, timedelta
from fastapi import FastAPI, Header, HTTPException
from sqlalchemy import create_engine, text

app = FastAPI()
engine = create_engine("postgresql://localhost/payments")

@app.post("/v1/charges")
async def create_charge(
    request: ChargeRequest,
    idempotency_key: str = Header(None, alias="Idempotency-Key")
):
    # 1. Require idempotency key
    if not idempotency_key:
        raise HTTPException(400, "Idempotency-Key header required")
    
    # 2. Validate key format
    try:
        uuid.UUID(idempotency_key)
    except ValueError:
        raise HTTPException(400, "Idempotency-Key must be a valid UUID")
    
    with engine.begin() as conn:
        # 3. Check for existing result — with SELECT FOR UPDATE to prevent races
        existing = conn.execute(text("""
            SELECT result, status, endpoint, request_hash
            FROM idempotency_keys
            WHERE key = :key
            FOR UPDATE
        """), {"key": idempotency_key}).fetchone()
        
        if existing:
            # 4a. Key exists — validate it's the same request
            current_hash = hash_request(request)
            if existing.endpoint != "/v1/charges" or existing.request_hash != current_hash:
                raise HTTPException(422, 
                    "Idempotency key reused with different request parameters")
            
            # 4b. If in-flight (another request is processing), return 409
            if existing.status == "processing":
                raise HTTPException(409, "Request is already being processed")
            
            # 4c. Return cached result
            return json.loads(existing.result)
        
        # 5. Mark as in-flight
        conn.execute(text("""
            INSERT INTO idempotency_keys 
                (key, endpoint, request_hash, status, created_at, expires_at)
            VALUES 
                (:key, '/v1/charges', :hash, 'processing', NOW(), NOW() + INTERVAL '7 days')
        """), {"key": idempotency_key, "hash": hash_request(request)})
    
    # 6. Execute the actual charge (outside the first transaction)
    try:
        charge_result = payment_processor.charge(
            amount=request.amount,
            currency=request.currency,
            card_token=request.card_token
        )
        result = {"id": charge_result.id, "status": "succeeded", "amount": request.amount}
        status = "succeeded"
    except PaymentDeclinedError as e:
        result = {"error": str(e), "status": "failed"}
        status = "failed"
    
    # 7. Store the result — now it's safe to return on retries
    with engine.begin() as conn:
        conn.execute(text("""
            UPDATE idempotency_keys
            SET result = :result, status = :status
            WHERE key = :key
        """), {
            "result": json.dumps(result),
            "status": status,
            "key": idempotency_key
        })
    
    return result


def hash_request(request: ChargeRequest) -> str:
    """Deterministic hash of request body to detect key reuse for different requests."""
    import hashlib
    content = json.dumps(request.dict(), sort_keys=True)
    return hashlib.sha256(content.encode()).hexdigest()
```

```sql
-- The idempotency_keys table
CREATE TABLE idempotency_keys (
    key          VARCHAR(255) PRIMARY KEY,
    endpoint     VARCHAR(255) NOT NULL,
    request_hash VARCHAR(64)  NOT NULL,
    status       VARCHAR(20)  NOT NULL DEFAULT 'processing',
    result       JSONB,
    created_at   TIMESTAMP    NOT NULL DEFAULT NOW(),
    expires_at   TIMESTAMP    NOT NULL
);

-- Index for cleanup job
CREATE INDEX idx_idempotency_keys_expires ON idempotency_keys(expires_at);

-- Periodic cleanup
DELETE FROM idempotency_keys WHERE expires_at < NOW();
```

---

## Part 2: Distributed Transactions

---

### Q7. What is ACID and why does it fall apart across microservices?

**ACID — the four guarantees of a single database:**

| Letter | Property | What it means |
|--------|----------|---------------|
| **A** | Atomicity | All or nothing — the transaction either fully commits or fully rolls back |
| **C** | Consistency | DB moves from one valid state to another — constraints never violated |
| **I** | Isolation | Concurrent transactions don't interfere with each other |
| **D** | Durability | Once committed, data survives crashes |

**Within one database: trivially achieved.**

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- debit
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- credit
COMMIT;  -- both happen or neither happens
```

**Across microservices: completely falls apart.**

```
Monolith (one DB):                Microservices (separate DBs):
                                  
[Order Service] ─── single ───>   [Order Service]    [Payment Service]   [Inventory Service]
[Pay Service  ]     DB txn        [  Orders DB   ]   [  Payments DB   ]  [  Inventory DB  ]
[Inventory    ]                   
                                  
All in one ACID transaction.      Three separate databases.
Easy.                             No shared transaction coordinator.
                                  How do you make this atomic???
```

**The distributed transaction problem in concrete terms:**

```
Scenario: User places an order for $50

Step 1: Create order in Order Service DB     ✓ (succeeded)
Step 2: Charge $50 in Payment Service DB     ✓ (succeeded)  
Step 3: Decrement inventory in Stock DB      ✗ (server crashed!)

Now what?
- Order exists
- Money is charged
- Inventory NOT decremented
- System is in an INCONSISTENT state
- Customer paid for something that might not ship correctly
```

This is the core problem. You cannot use a single database transaction across services with separate databases. The question becomes: how do you achieve consistency WITHOUT traditional ACID transactions?

---

### Q8. What is Two-Phase Commit (2PC) and why do modern systems avoid it?

**How 2PC works:** A coordinator orchestrates a two-round protocol.

```
Phase 1 — PREPARE (Voting Phase):

    Coordinator
        |
        |-------- PREPARE? -------> Service A (Order DB)
        |-------- PREPARE? -------> Service B (Payment DB)
        |-------- PREPARE? -------> Service C (Inventory DB)
        |
        |<------- VOTE: YES -------- Service A (locks resources, logs intent)
        |<------- VOTE: YES -------- Service B (locks resources, logs intent)
        |<------- VOTE: YES -------- Service C (locks resources, logs intent)

Phase 2 — COMMIT (Decision Phase):

    Coordinator (all votes = YES → send COMMIT)
        |
        |-------- COMMIT ----------> Service A → releases locks, commits
        |-------- COMMIT ----------> Service B → releases locks, commits
        |-------- COMMIT ----------> Service C → releases locks, commits
        |
    Done! (or ABORT if any vote was NO)
```

**Why 2PC is problematic — the coordinator failure problem:**

```
Phase 1: All services vote YES and lock their resources.

              COORDINATOR CRASHES HERE
              ↓
    Service A: holding locks... waiting...
    Service B: holding locks... waiting...
    Service C: holding locks... waiting...

Nobody knows if coordinator decided to COMMIT or ABORT.
All services are BLOCKED, holding locks indefinitely.
The system is frozen until coordinator recovers.
```

**The problems with 2PC:**

1. **Blocking protocol** — if coordinator dies after Phase 1 but before Phase 2, all participants are stuck holding locks. This can freeze the entire system.

2. **Single point of failure** — the coordinator is a bottleneck and failure domain.

3. **Latency** — requires 2 round trips across the network for every operation. In microservices with dozens of services, this compounds badly.

4. **Doesn't scale** — holding database locks while waiting for network round trips kills throughput.

5. **Not cloud-friendly** — cloud databases (DynamoDB, CockroachDB, etc.) don't support XA transactions (the standard 2PC protocol for distributed systems).

**The XA problem in practice:**

The XA standard (X/Open XA) is the spec for 2PC across heterogeneous databases. It exists. It theoretically works. In practice:

- MySQL's XA support has had bugs for years
- Most ORMs don't support it well
- Cloud-native databases don't support it at all
- The blocking behavior under failure is unacceptable for high-traffic systems
- Netflix, Uber, Stripe, Amazon — none of them use 2PC for their core flows

As Martin Kleppmann put it in *Designing Data-Intensive Applications*: 2PC is "a protocol for creating doubt."

---

### Q9. What is the Saga Pattern? Explain choreography vs orchestration.

**The Saga:** Instead of one giant atomic transaction, break it into a sequence of local transactions. Each local transaction updates ONE service/database and publishes an event or message. If a step fails, run compensating transactions to undo previous steps.

**Named after the 1987 paper by Hector Garcia-Molina and Kenneth Salem** — yes, this idea is older than most of us.

**Two flavors of Saga:**

---

#### Choreography (Event-Driven) — "Everyone dances to the music"

Services react to events. No central controller.

```
Order Service       Payment Service      Inventory Service     Notification Service
     |                    |                     |                      |
 [Create Order]           |                     |                      |
     |                    |                     |                      |
     |-- OrderCreated --->|                     |                      |
     |   (event)          |                     |                      |
     |               [Process Payment]          |                      |
     |                    |                     |                      |
     |                    |-- PaymentCharged -->|                      |
     |                       (event)            |                      |
     |                                    [Reserve Stock]              |
     |                                          |                      |
     |                                          |-- StockReserved ---->|
     |                                             (event)             |
     |                                                           [Send Confirmation]
     |                                                                  |
                                                                   DONE!
```

**Failure in choreography** — compensating events flow backwards:

```
Order Service       Payment Service      Inventory Service
     |                    |                     |
 [Create Order]           |                     |
     |-- OrderCreated --->|                     |
     |               [Process Payment]          |
     |                    |-- PaymentCharged -->|
     |                                    [Reserve Stock]
     |                                    FAILED! (out of stock)
     |                                          |
     |                                          |-- StockFailed ------>|
     |                                             (event)       [Refund Payment]
     |<------------------------------------------|-- PaymentRefunded --|
     |   (event)                                 
 [Cancel Order]
     |
  Order cancelled, payment refunded. Consistency restored.
```

**Choreography pros/cons:**

```
Pros:
  + Simple to implement (just emit events and react)
  + No single point of failure
  + Services are loosely coupled
  + Scales well — no bottleneck

Cons:
  - Hard to track overall transaction progress ("where is my order?")
  - Event chains are hard to debug (need distributed tracing)
  - Cyclic dependencies can sneak in between services
  - Hard to reason about business logic spread across services
```

---

#### Orchestration — "Everyone follows the conductor"

A central **Saga Orchestrator** (or Saga Coordinator) tells each service what to do.

```
                    Saga Orchestrator
                         |
          ┌──────────────┼──────────────┐
          |              |              |
          v              v              v
   [Order Service] [Payment Service] [Inventory Service]

Flow:
Orchestrator → Order Service:    "Create order"    → OK
Orchestrator → Payment Service:  "Charge card"     → OK
Orchestrator → Inventory Service:"Reserve stock"   → FAILED

Orchestrator (on failure):
Orchestrator → Payment Service:  "Refund payment"  → OK  (compensate)
Orchestrator → Order Service:    "Cancel order"    → OK  (compensate)
Done. Consistent state.
```

**Orchestration pros/cons:**

```
Pros:
  + Easy to visualize and track saga state
  + Business logic lives in one place (the orchestrator)
  + Easy to add new steps or handle complex branching
  + Clear audit trail of what happened

Cons:
  - Orchestrator can become a bottleneck
  - Orchestrator is a single point of failure (needs HA)
  - Tighter coupling between orchestrator and services
  - Orchestrator can grow into a "God Service"
```

---

**Netflix's Saga Pattern (Real Incident)**

Netflix uses the Saga pattern extensively across their microservices for video publishing workflows. When a new video is uploaded, a saga coordinates:
1. Transcoding (video processing service)
2. Content validation (compliance service)
3. CDN distribution (content delivery service)
4. Catalog update (metadata service)
5. Notification (recommendation engine update)

Each step can fail independently. Netflix uses orchestrated sagas with their **Conductor** workflow engine (open-sourced in 2016) to manage state and compensations. If CDN distribution fails, the saga rolls back — removing the catalog entry, skipping notifications, etc. The stateful orchestrator tracks every step, making it easy to retry exactly the failed step.

**Shopify's Black Friday Saga**

Shopify's inventory system during Black Friday is a masterclass in saga design. With millions of concurrent transactions:

1. `ReserveInventory` — soft-reserve items (saga step 1)
2. `ProcessPayment` — charge card (saga step 2)
3. `ConfirmInventory` — convert soft-reserve to hard deduction (saga step 3)
4. `FulfillOrder` — send to warehouse (saga step 4)

If payment fails after inventory is reserved → `CancelReservation` compensating transaction runs. This two-phase reservation (soft-reserve → confirm) prevents overselling while keeping the system eventually consistent. During Black Friday 2022, Shopify processed billions of dollars of orders using this approach — no 2PC in sight.

---

### Q10. What are compensating transactions? Give me examples.

**The concept:** A compensating transaction is the semantic "undo" of a previous saga step. Unlike database ROLLBACK, compensations can't always perfectly undo — they create new records that effectively cancel out the previous action.

```
FORWARD TRANSACTION          COMPENSATING TRANSACTION
──────────────────           ────────────────────────
Create order         ←──→   Cancel order
Charge payment       ←──→   Refund payment
Reserve inventory    ←──→   Release inventory reservation
Send email           ←──→   (can't unsend!) Send "order cancelled" email
Ship package         ←──→   (can't unship!) Create return label + process return
```

Notice the last two — some actions can't be "undone." You can't unsend an email. You can only send a corrective action. This is why sagas are described as achieving **eventual consistency**, not strict atomicity.

**Key insight:** Compensating transactions must be:
1. **Idempotent** (they might be retried)
2. **Always succeeding** (if compensation fails, you're in a really bad spot)
3. **Semantic undos** (not literal rollbacks)

**State machine for a saga:**

```
                    ┌─────────┐
              ┌─────┤  START  ├──────┐
              │     └─────────┘      │
              v                      v
       [Step 1: OK]           [Step 1: FAIL]
              │                      │
              v                      v
       [Step 2: OK]           ┌──────────────┐
              │               │  SAGA FAILED  │
              v               └──────────────┘
       [Step 3: FAIL]
              │
              v
    [Compensate Step 2]
              │
              v
    [Compensate Step 1]
              │
              v
       ┌─────────────┐
       │ COMPENSATED │ ← eventual consistency restored
       └─────────────┘
```

**Code example — saga with compensation:**

```python
class OrderSaga:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.completed_steps = []
    
    def execute(self, user_id: str, items: list, payment_token: str):
        try:
            # Step 1: Create order
            order = order_service.create_order(self.order_id, user_id, items)
            self.completed_steps.append(("create_order", order.id))
            
            # Step 2: Reserve inventory
            reservation = inventory_service.reserve(self.order_id, items)
            self.completed_steps.append(("reserve_inventory", reservation.id))
            
            # Step 3: Process payment
            charge = payment_service.charge(
                order_id=self.order_id, 
                token=payment_token, 
                amount=order.total
            )
            self.completed_steps.append(("charge_payment", charge.id))
            
            # Step 4: Confirm order
            order_service.confirm(self.order_id)
            
        except Exception as e:
            # Compensate in reverse order
            self.compensate()
            raise SagaFailedException(f"Saga failed: {e}")
    
    def compensate(self):
        # Undo in reverse order
        for step_name, step_id in reversed(self.completed_steps):
            try:
                if step_name == "charge_payment":
                    payment_service.refund(step_id)   # issue refund
                elif step_name == "reserve_inventory":
                    inventory_service.release(step_id) # release reservation
                elif step_name == "create_order":
                    order_service.cancel(step_id)      # cancel order
            except Exception as e:
                # Log and alert — compensation failure needs human intervention
                alerting.critical(f"Compensation failed for {step_name}: {e}")
                # Could retry, or send to dead-letter queue for manual handling
```

---

### Q11. What is the Outbox Pattern? How does it solve the dual-write problem?

**The problem:** You want to update your database AND publish an event to a message queue. These are two different systems. How do you do both atomically?

```
THE DUAL-WRITE PROBLEM:

Option A — DB first, then publish:
  1. Write to DB     ← succeeds
  2. Publish event   ← CRASHES
  Result: DB updated, event never sent. Consumers never know. DATA LOSS.

Option B — Publish first, then DB:
  1. Publish event   ← succeeds  
  2. Write to DB     ← CRASHES
  Result: Event sent, DB not updated. Consumers process phantom event. GHOST DATA.

Both options have failure windows. Neither is safe.
```

**The Outbox Pattern — atomic dual write:**

```
┌─────────────────────────────────────────────────────┐
│                   Your Database                      │
│                                                      │
│  ┌──────────────┐      ┌─────────────────────────┐  │
│  │ orders table │      │   outbox table           │  │
│  │              │      │                          │  │
│  │  id: 123     │      │  id: abc                 │  │
│  │  status: new │      │  event: OrderCreated     │  │
│  │  total: 50   │      │  payload: {order_id:123} │  │
│  └──────────────┘      │  published: false        │  │
│                        └─────────────────────────┘  │
│                                                      │
│         Both written in ONE database transaction     │
└─────────────────────────────────────────────────────┘
                              │
                              │ Outbox Relay (separate process)
                              │ polls for unpublished events
                              v
                    ┌──────────────────┐
                    │  Message Broker  │
                    │  (Kafka / SQS)   │
                    └──────────────────┘
```

**How it works:**

1. In a single DB transaction: write your business data + write event to `outbox` table
2. A separate **relay process** (or Debezium CDC) reads unpublished outbox events
3. Relay publishes to Kafka/SQS
4. On successful publish, relay marks outbox row as `published = true`
5. The relay uses idempotency (see Part 1!) to handle its own retries

```python
# Application code — single atomic transaction
def create_order(order_data: dict) -> Order:
    with db.transaction() as txn:
        # Business write
        order = txn.execute("""
            INSERT INTO orders (id, user_id, total, status)
            VALUES (:id, :user_id, :total, 'pending')
            RETURNING *
        """, order_data)
        
        # Outbox write (SAME transaction!)
        txn.execute("""
            INSERT INTO outbox (id, aggregate_type, aggregate_id, event_type, payload)
            VALUES (:event_id, 'Order', :order_id, 'OrderCreated', :payload)
        """, {
            "event_id": str(uuid.uuid4()),
            "order_id": order.id,
            "payload": json.dumps({"order_id": order.id, "total": order.total})
        })
        
    return order  # Transaction committed — both writes guaranteed atomic


# Outbox Relay — runs as a separate process/service
def relay_loop():
    while True:
        with db.transaction() as txn:
            # Fetch unpublished events (SELECT FOR UPDATE SKIP LOCKED for multiple replicas)
            events = txn.execute("""
                SELECT * FROM outbox
                WHERE published = false
                ORDER BY created_at ASC
                LIMIT 100
                FOR UPDATE SKIP LOCKED
            """)
            
            for event in events:
                kafka.produce(
                    topic=event.event_type,
                    key=event.aggregate_id,
                    value=event.payload,
                    headers={"event_id": event.id}  # for consumer deduplication
                )
                
                txn.execute("""
                    UPDATE outbox SET published = true, published_at = NOW()
                    WHERE id = :id
                """, {"id": event.id})
        
        time.sleep(0.1)  # poll interval
```

**Alternative: Change Data Capture (CDC) with Debezium**

Instead of polling, use CDC to stream the outbox table's WAL (Write-Ahead Log) changes directly to Kafka:

```
PostgreSQL WAL → Debezium → Kafka
     ↑
(outbox table rows appear here as they're inserted)
```

This eliminates the polling overhead and gives near-real-time event propagation.

---

### Q12. When do you accept eventual consistency vs insisting on strong consistency?

**The fundamental tradeoff:** Strong consistency requires coordination (slow, blocking). Eventual consistency allows independence (fast, scalable) but allows temporary inconsistency.

**Questions to ask:**

```
1. What's the cost of a brief inconsistency?
   
   LOW COST (use eventual consistency):
   - User's follower count is temporarily wrong by 1
   - Product search results don't include an item added 2 seconds ago
   - Dashboard metrics lag by 30 seconds
   
   HIGH COST (use strong consistency):
   - User's bank balance is shown incorrectly
   - User is charged twice
   - Two users book the same airline seat
   - Inventory shows stock when it's actually 0

2. Is this a financial/legal operation?
   → Strongly lean toward strong consistency or at minimum idempotency

3. Can the business tolerate "read your own writes" lag?
   → If a user creates something and immediately can't see it: bad UX

4. Is this an audit log / compliance record?
   → Needs to be durable and consistent, never "eventually" correct
```

**Decision matrix:**

| Scenario | Recommended Approach | Why |
|----------|---------------------|-----|
| Social media likes | Eventual consistency | Off-by-one for 100ms is fine |
| Payment processing | Idempotent + synchronous | Money is serious |
| Inventory reservation | 2-phase (soft reserve) | Overselling is bad |
| User profile updates | Eventual consistency | Lag is acceptable |
| Seat/ticket booking | Synchronous + pessimistic locking | Exact count matters |
| Email delivery | At-least-once (idempotent consumer) | Can't lose emails |
| Analytics dashboard | Eventual consistency | Slight lag is fine |
| Account balance display | Read from primary DB (strong) | User sees wrong balance = support tickets |

**The CAP Theorem reminder:**

```
CAP Theorem: In a distributed system, you can have at most 2 of:
  C — Consistency (every read sees the latest write)
  A — Availability (every request gets a response)
  P — Partition tolerance (system works despite network splits)

Since network partitions WILL happen, you choose:
  CP — Consistent + Partition tolerant → may refuse requests (ZooKeeper, HBase)
  AP — Available + Partition tolerant  → may serve stale data (DynamoDB, Cassandra)
```

---

## Quick Revision Table

| Concept | One-Line Definition | Key Mechanism | When to Use |
|---------|--------------------|-----------------|-|
| **Idempotency** | Same result no matter how many times you call it | UUID key stored in DB | Any operation that might be retried |
| **Idempotency Key** | Client-generated UUID that deduplicates requests | Server caches result by key | Payment APIs, any POST endpoint |
| **UPSERT** | Insert if new, update if existing, never duplicate | `ON CONFLICT DO NOTHING` | DB-level dedup |
| **Message Deduplication** | Consumer ignores already-processed messages | Redis SET NX / SQS dedup ID | Queue consumers |
| **ACID** | Atomic + Consistent + Isolated + Durable | Single DB transaction | Within one service/DB |
| **2PC (Two-Phase Commit)** | Coordinator asks all participants to vote, then commits/aborts | Prepare → Commit rounds | Rarely — avoid in modern systems |
| **Saga Pattern** | Sequence of local transactions with compensations | Events or orchestrator | Cross-service operations |
| **Choreography** | Services react to each other's events | Event bus, no central controller | Simple, loosely-coupled flows |
| **Orchestration** | Central coordinator tells each service what to do | Saga orchestrator / workflow engine | Complex flows needing visibility |
| **Compensating Transaction** | Semantic "undo" of a completed saga step | New record that cancels the effect | Saga rollback |
| **Outbox Pattern** | Write event to DB table atomically, relay publishes it | DB transaction + poller/CDC | Reliable event publishing |
| **Eventual Consistency** | System becomes consistent over time (not immediately) | Async propagation | Non-critical reads, high-scale writes |
| **Strong Consistency** | Every read reflects the latest write | Synchronous coordination | Money, bookings, inventory |

---

## Cheat Sheet: Which Pattern for Which Problem?

```
PROBLEM                                    SOLUTION
──────────────────────────────────────────────────────────────
Client retries causing duplicate charges   Idempotency keys
DB insert creating duplicate rows          UPSERT / ON CONFLICT
Message queue replaying messages           Consumer-side dedup (Redis SET NX)
Cross-service atomic operation             Saga pattern
Saga needs visibility + complex logic      Orchestrated saga (Conductor/Temporal)
Simple event-driven pipeline               Choreographed saga
DB write + event publish atomically        Outbox pattern
Rollback a completed saga step             Compensating transaction
Need all-or-nothing across DBs             Consider redesigning! (avoid 2PC)
Read performance vs consistency            Eventual (cache) vs Strong (primary DB)
```

---

## The Golden Rules

1. **Any operation that crosses a network must be idempotent.** Networks are unreliable. Retries are inevitable. Duplicate charges are catastrophic.

2. **Sagas, not 2PC.** Two-phase commit blocks under failure. Sagas are eventually consistent but never blocking. The industry has voted — sagas win.

3. **The Outbox Pattern is your friend.** Never do a dual-write without it. One transaction, one system, one truth.

4. **Compensations must always succeed.** If your compensation can fail, you need a human escalation path. Alert, dead-letter queue, manual intervention.

5. **Money is always strong consistency.** For financial operations: idempotency keys, synchronous confirmation, audit logs. Eventual consistency is not for your billing system.

> "Eventual consistency is not an excuse for lazy design — it's a deliberate engineering tradeoff that requires more careful implementation, not less." — Werner Vogels, AWS CTO
