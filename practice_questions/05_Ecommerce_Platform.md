# Design an E-commerce Platform like Amazon
> Complete interview preparation guide — Q&A format, beginner to advanced.

---

## Table of Contents
1. [What Is It?](#what-is-it)
2. [Step 1 — Requirements](#step-1--requirements)
3. [Step 2 — Capacity Estimation](#step-2--capacity-estimation)
4. [Step 3 — The Hard Problems](#step-3--the-hard-problems)
5. [Step 4 — Database Design](#step-4--database-design)
6. [Step 5 — Search Architecture](#step-5--search-architecture)
7. [Step 6 — Payment and Order Flow](#step-6--payment-and-order-flow)
8. [Full System Diagram](#full-system-diagram)
9. [Quick Revision](#quick-revision)

---

## What Is It?

**Q: What makes e-commerce architecturally different from social media or chat?**

> **Money is involved.** This changes consistency requirements fundamentally:
> - Social media: stale data = minor annoyance
> - E-commerce: stale inventory = overselling = lawsuits
> - E-commerce: double payment = fraud = lawsuits
>
> The same system must handle **eventual consistency** for search/browsing AND **strong consistency** for inventory and payments. This dual requirement drives the entire architecture.

---

## Step 1 — Requirements

**Q: What are the functional requirements?**

> 1. Sellers list products with title, price, images, inventory count
> 2. Users search for products by keyword, category, filters
> 3. Users add products to a shopping cart
> 4. Users place orders and make payments
> 5. Users can rate and review purchased products

**Q: What are the non-functional requirements?**

> - **High scalability** — 100M daily active users, millions of concurrent sessions
> - **High availability** — 99.9% uptime (downtime = lost revenue)
> - **Low latency** — fast search, fast page loads, fast checkout
> - **Strong consistency** — inventory and payments must be exact, no exceptions
> - **High durability** — orders, payments, user data must never be lost

**Q: Which parts of Amazon require strong consistency vs eventual consistency?**

> | Feature | Consistency | Why |
> |---------|-------------|-----|
> | Product search | Eventual | Stale result = minor annoyance |
> | Product images | Eventual | CDN cached, slightly old is fine |
> | Inventory count | **Strong** | Stale count = overselling = disaster |
> | Payment processing | **Strong** | Double charge = fraud |
> | Order records | **Strong** | Must never be lost or duplicated |
> | Shopping cart | Eventual | Cart items can be slightly stale |
> | Review/ratings | Eventual | Seeing a review 1 second late is fine |

---

## Step 2 — Capacity Estimation

**Q: Walk through capacity estimation for Amazon.**

> **Assumptions:**
> - 100M daily active users
> - 1000:1 read/write ratio (people browse far more than they buy)
> - 500M products in catalog
>
> **Traffic:**
> ```
> Reads (browsing/search): ~1,150,000 reads/second
> Writes (orders/updates): ~1,150 writes/second
> ```
>
> **Storage:**
> ```
> 500M products × 10 KB metadata = 5 TB product data
> 500M products × 5 images × 500 KB = 1.25 PB images
> → Images MUST go to S3 + CDN
> ```

---

## Step 3 — The Hard Problems

### Problem 1: Inventory Race Condition

**Q: Two customers simultaneously try to buy the last item in stock. Both see "1 in stock." Both click Buy Now at the exact same millisecond. How do you prevent both from succeeding?**

> **Optimistic Locking with version numbers:**
>
> The database processes ONE transaction at a time per row. Even if both requests arrive simultaneously, one is always processed first — even by a fraction of a millisecond.
>
> ```sql
> BEGIN TRANSACTION
>
> -- Read current state:
> SELECT inventory, version FROM products
> WHERE id = 'iphone_123'
> -- Returns: inventory=1, version=42
>
> -- Update ONLY if version hasn't changed:
> UPDATE products
> SET inventory = inventory - 1,
>     version = version + 1
> WHERE id = 'iphone_123'
>   AND version = 42        ← the guard condition
>   AND inventory > 0
>
> -- Customer 1 (arrives 1ms first):
> -- version=42 matches → UPDATE affects 1 row → order placed ✅
> -- version is now 43, inventory is now 0
>
> -- Customer 2 (arrives 1ms later):
> -- version=43, NOT 42 → UPDATE affects 0 rows → SOLD OUT ❌
>
> COMMIT
> ```
>
> **How to think about it:**
> ```
> Timeline:
> 0ms: Both customers click Buy
> 1ms: Customer 1's transaction reaches DB first
>      DB: "version=42 matches, decrement inventory" ✅
>      DB: version now = 43, inventory now = 0
> 2ms: Customer 2's transaction reaches DB
>      DB: "version=42 no longer exists (it's 43 now)"
>      DB: UPDATE fails → 0 rows affected → "Sold out!" ❌
> ```
>
> It is NOT luck — it is physics. One network packet always arrives before the other. The version number ensures only the first one wins, regardless of how close together they arrive.

### Problem 2: Payment Double-Charge

**Q: Payment is charged, then the service crashes before recording the order. Customer is charged but has no order. How do you fix this?**

> **Idempotency Keys:**
>
> ```
> Step 1: Generate unique key BEFORE charging:
>         idempotency_key = "order_user123_cart456_2024_abc"
>
> Step 2: Save key to DB → status: "payment_pending"
>
> Step 3: Send payment request WITH idempotency key to Stripe:
>         charge(amount=999, key="order_user123_cart456_2024_abc")
>
> Step 4: Stripe charges card successfully
>
> Step 5: [SERVER CRASHES 💥]
>
> Recovery: Customer or system retries
>         → sends SAME idempotency key again
>         → Stripe: "I already processed this key → same success response"
>         → DB: "key exists and pending → update to complete"
>         → Order recorded ✅ No double charge ✅
> ```
>
> **Rule:** Every payment request must carry a unique idempotency key. Same key = same result, no matter how many times it's sent.

---

## Step 4 — Database Design

**Q: What databases does Amazon use and why?**

> ```
> Data Type          Database          Reason
> ─────────────────────────────────────────────────────────────
> Product catalog    Elasticsearch     Full-text search, filters
> Product images     S3 + CDN          Large files, global delivery
> Shopping cart      Redis             Temporary, fast reads/writes
> User sessions      Redis             Ephemeral, sub-ms reads
> Orders             PostgreSQL        ACID transactions, must be exact
> Inventory counts   PostgreSQL        Row-level locking, strong consistency
> Payments           PostgreSQL        ACID, audit trail required
> Reviews/ratings    Cassandra         High write volume, simple queries
> ```

**Q: Why is the shopping cart stored in Redis and not a database?**

> Shopping carts are:
> - **Temporary** — most carts are abandoned (70%+ of carts never convert to orders)
> - **Session-bound** — tied to user session, changes frequently
> - **Fast to read** — page load depends on cart rendering
> - **Regeneratable** — if lost, user just re-adds items (minor annoyance, not catastrophic)
>
> Redis is perfect: in-memory, sub-millisecond reads, TTL for auto-expiry after session ends. Storing 100M carts in PostgreSQL would waste expensive transactional DB resources on throwaway data.

---

## Step 5 — Search Architecture

**Q: Why can't Amazon use PostgreSQL `WHERE title LIKE '%iphone%'` for product search?**

> ```sql
> SELECT * FROM products WHERE title LIKE '%iphone%'
> ```
> The `%` wildcard at the start means "could start with anything" — the database CANNOT use an index. It must read every single row and check manually.
>
> ```
> 500 million products × full scan = takes MINUTES
> → Completely unusable for a search box
> ```

**Q: How does Elasticsearch solve this with an Inverted Index?**

> ```
> Normal database stores rows:
> Row 1: "Apple iPhone 15 Pro Max"
> Row 2: "Samsung Galaxy S24"
>
> Elasticsearch builds an inverted index:
> "apple"   → [product_1]
> "iphone"  → [product_1]
> "15"      → [product_1]
> "samsung" → [product_2]
> "galaxy"  → [product_2]
>
> Search "iphone":
> → Direct lookup in index: "iphone" → [product_1]
> → Returns results in milliseconds ⚡
> → No row scanning at all
> ```
>
> Like the index at the back of a textbook — you don't read every page to find "iphone", you jump straight to the right page.

**Q: How do products get into Elasticsearch?**

> Products are written to PostgreSQL first (source of truth), then synced to Elasticsearch asynchronously via Kafka:
> ```
> Seller adds product
>         ↓
> Write to PostgreSQL (durable record)
>         ↓
> Publish "PRODUCT_CREATED" event to Kafka
>         ↓
> Elasticsearch consumer picks up event
>         ↓
> Indexes product in Elasticsearch
>         ↓
> (slight delay — eventual consistency, acceptable for search)
> ```

---

## Step 6 — Payment and Order Flow

**Q: Walk through the complete order placement flow.**

> ```
> Customer clicks "Place Order"
>         ↓
> [Order Service]
> 1. Generate idempotency key
> 2. Save order with status = "pending"
> 3. Lock inventory (optimistic locking)
>         ↓
> [Payment Service]
> 4. Charge card with idempotency key
> 5. Payment succeeds
>         ↓
> [Order Service]
> 6. Update order status = "confirmed"
> 7. Decrement inventory permanently
>         ↓
> [Kafka Event: "ORDER_PLACED"]
>         ↓
>     ┌───┴────────────────────┐
> [Email Service]    [Warehouse Service]
> Send confirmation  Pick and pack order
> ```

**Q: What happens if payment fails?**

> ```
> Payment fails
>         ↓
> Inventory lock is RELEASED (transaction rolled back)
> Order status = "failed"
> User shown error: "Payment failed, please try again"
> Idempotency key discarded
> ```
> The item goes back into available inventory. No money moved, no inventory consumed.

---

## Full System Diagram

```
[Users]
    ↓
[Load Balancer]
    ↓
[App Servers]
    │           │              │              │
    ↓           ↓              ↓              ↓
[Elasticsearch] [Redis]   [PostgreSQL]    [S3 + CDN]
(product search) (cart,   (orders,        (product
                 sessions) inventory,      images)
                           payments)
                               ↓
                        [Payment Service]
                        (Stripe + idempotency)
                               ↓
                           [Kafka]
                               ↓
                  ┌────────────┼────────────┐
           [Email Service] [Warehouse]  [Analytics]
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| Product search | Elasticsearch | Inverted index — full-text search across 500M products in ms |
| Product images | S3 + CDN | 1.25 PB of images — only object storage handles this |
| Shopping cart | Redis | Temporary, fast, 70% abandoned — don't waste DB on it |
| Orders/payments | PostgreSQL | ACID transactions — money requires strong consistency |
| Inventory locking | Optimistic locking + version number | Prevents overselling without locking entire table |
| Payment safety | Idempotency keys | Prevents double-charge on retries or crashes |
| Search sync | Kafka (async) | Eventual consistency acceptable for search |
| Reviews | Cassandra | High write volume, simple time-ordered queries |

### Three things to mention proactively in an interview
> 1. **Optimistic locking for inventory** — shows you know how to prevent overselling without killing DB performance
> 2. **Idempotency keys for payments** — shows you understand exactly-once payment semantics
> 3. **Elasticsearch for search, PostgreSQL for transactions** — shows you know different databases for different consistency requirements
