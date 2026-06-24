# Design a Ride-Sharing Service like Uber
> Complete interview preparation guide — Q&A format, beginner to advanced.

---

## Table of Contents
1. [What Is It?](#what-is-it)
2. [Step 1 — Requirements](#step-1--requirements)
3. [Step 2 — Capacity Estimation](#step-2--capacity-estimation)
4. [Step 3 — The Hard Problems](#step-3--the-hard-problems)
5. [Step 4 — Database Design](#step-4--database-design)
6. [Step 5 — Real-Time Tracking](#step-5--real-time-tracking)
7. [Step 6 — Surge Pricing](#step-6--surge-pricing)
8. [Full System Diagram](#full-system-diagram)
9. [Quick Revision](#quick-revision)

---

## What Is It?

**Q: What makes Uber architecturally unique compared to other systems?**

> Two things make Uber genuinely hard:
> 1. **Massive real-time location writes** — 5 million drivers each updating their GPS every 4 seconds = 1.25 million location writes/second. Traditional databases collapse under this.
> 2. **Race condition on driver assignment** — two passengers requesting the same driver simultaneously. Without careful design, the same driver gets assigned twice.

---

## Step 1 — Requirements

**Q: What are the functional requirements?**

> 1. Riders can request a ride from their current location
> 2. System finds and assigns the nearest available driver
> 3. Rider sees driver's real-time location on a map
> 4. Driver navigates to pickup, then to destination
> 5. Trip ends, fare is calculated and charged

**Q: What are the non-functional requirements?**

> - **Low latency** — driver matching must happen in seconds
> - **High availability** — 99.99% uptime (people need rides 24/7)
> - **Strong consistency** — a driver must not be assigned to two passengers simultaneously
> - **Real-time** — live GPS tracking with sub-5-second update latency
> - **High write throughput** — 1.25M location updates/second

---

## Step 2 — Capacity Estimation

**Q: Walk through the location update math.**

> **Assumptions:**
> - 5 million active drivers globally
> - Each driver sends GPS update every 4 seconds
>
> **Location writes:**
> ```
> 5,000,000 drivers ÷ 4 seconds = 1,250,000 writes/second
> ```
>
> **What this means:**
> - A single PostgreSQL instance handles ~5,000-10,000 writes/second
> - 1.25M writes/second = 125-250x beyond a single DB's capacity
> - Traditional databases cannot handle this regardless of how many you shard
> - **Conclusion: must use in-memory storage (Redis)**

---

## Step 3 — The Hard Problems

### Problem 1: Live Driver Location Storage

**Q: 1.25 million location writes per second. What database handles this?**

> **Redis GEO** — not a traditional database.
>
> Driver locations are:
> - Updated every 4 seconds (old data is immediately stale and useless)
> - Only queried geographically ("find drivers near me")
> - Not needed permanently — only current position matters
>
> ```
> Redis GEO commands:
> GEOADD drivers 67.001 24.860 "driver_A"   ← update position (overwrites old)
> GEOADD drivers 67.025 24.861 "driver_B"
> GEOADD drivers 66.998 24.859 "driver_C"
>
> GEORADIUS drivers 67.001 24.860 2 km       ← find nearby drivers
> → returns [driver_A, driver_C] in milliseconds ⚡
> ```
>
> Redis is in-memory — each location update simply OVERWRITES the old one. No append, no history, no disk write. Handles 1.25M writes/second with ease.

### Problem 2: Driver Assignment Race Condition

**Q: Two passengers request a ride simultaneously. Both are near Driver A. How do you prevent Driver A from being assigned to both?**

> **Optimistic locking with version numbers** — same technique as Amazon inventory.
>
> ```sql
> UPDATE drivers
> SET status = 'assigned',
>     passenger_id = 'passenger_X',
>     version = version + 1
> WHERE driver_id = 'driver_A'
>   AND status = 'available'
>   AND version = 7        ← the guard condition
>
> -- Passenger 1 (arrives 1ms first):
> -- version=7 matches → UPDATE affects 1 row → driver assigned ✅
> -- version is now 8, status is now 'assigned'
>
> -- Passenger 2 (arrives 1ms later):
> -- version=7 is gone (it's 8 now) → UPDATE affects 0 rows → try next driver ❌
> ```
>
> Passenger 2 is automatically matched to the next nearest available driver. No double assignment possible.

---

## Step 4 — Database Design

**Q: What databases does Uber use and for what?**

> ```
> Data Type              Database          Reason
> ──────────────────────────────────────────────────────────────
> Live driver locations  Redis GEO         In-memory, 1.25M writes/sec, geospatial queries
> Driver/rider profiles  PostgreSQL        Relational, structured, strong consistency needed
> Driver assignment      PostgreSQL        Optimistic locking, ACID transactions
> Trip history           Cassandra         High write volume, time-ordered, no JOINs needed
> Surge multipliers      Redis             Recalculated every minute, fast reads needed
> ```

---

## Step 5 — Real-Time Tracking

**Q: How does the passenger see the driver moving on the map in real-time?**

> **WebSockets in both directions:**
>
> ```
> Driver's phone ──WebSocket──► [Uber Server] ──WebSocket──► Passenger's phone
>
> Flow:
> 1. Driver moves 50 meters
> 2. Phone sends: {driver_id: "A", lat: 24.861, lng: 67.001}
> 3. Server updates Redis GEO (driver's live position)
> 4. Server pushes new coordinates to passenger's WebSocket
> 5. Map animates the car icon moving ⚡
> ```
>
> Same concept as WhatsApp — the server pushes data to the passenger without the passenger asking (pull-based polling would be too slow and wasteful).

---

## Step 6 — Surge Pricing

**Q: How does Uber know when and where to activate surge pricing?**

> **Supply vs demand ratio per geographic zone:**
>
> ```
> Zone: DHA Karachi, 10pm Friday
>
> Available drivers:   8
> Ride requests:       94
>
> Ratio = 94/8 = 11.75x demand over supply
> → Surge multiplier = 2.1x (dynamic formula)
> → All new rides in this zone: 2.1x normal price
> ```
>
> **Implementation:**
> ```
> Redis GEO: count available drivers within zone boundary
> DB counter: count pending/active ride requests in zone
> Surge Service: if requests/drivers > threshold → apply multiplier
> ```
>
> Surge is recalculated every ~1 minute per zone. Higher price attracts more drivers to the zone → supply increases → multiplier drops. Self-correcting system.

---

## Full System Diagram

```
[Rider's Phone]              [Driver's Phone]
      │ WebSocket                  │ WebSocket (GPS every 4s)
      ▼                            ▼
[Load Balancer]            [Location Service]
      │                            │
[Matching Service]         [Redis GEO]
      │                    (1.25M writes/sec,
      │                     live positions)
      ▼
[PostgreSQL]
(driver assignment,
 optimistic locking)
      │
  [Kafka]
      ↓
┌─────┼─────────────┐
[Trip Service]  [Billing]  [Notification]
(Cassandra)    (Stripe)    (APNs/FCM)
```

**Request flows:**
```
FIND DRIVER:
Rider requests ride → GEORADIUS finds nearest available drivers
→ Try to assign first driver (optimistic lock)
→ If version conflict → try next driver
→ Assignment confirmed → notify driver via WebSocket

LIVE TRACKING:
Driver GPS update → Location Service → GEOADD to Redis
→ Push coordinates to rider's WebSocket
→ Map updates every 4 seconds

SURGE DETECTION:
Surge Service every 60s: count drivers vs requests per zone
→ ratio > threshold → set multiplier in Redis
→ Pricing Service reads multiplier at booking time
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| Live driver locations | Redis GEO | 1.25M writes/sec — in-memory only option; geospatial queries in ms |
| Driver assignment | Optimistic locking (version numbers) | Prevents two passengers getting same driver |
| Real-time map tracking | WebSockets | Server must push GPS coordinates without passenger asking |
| Trip history | Cassandra | High write volume, time-ordered, no complex JOINs |
| Surge pricing | Demand/supply ratio per zone | Auto-corrects: higher price → more drivers → multiplier drops |
| Driver/rider profiles | PostgreSQL | Structured relational data, not high-frequency writes |

### Three things to mention proactively in an interview
> 1. **Redis GEO for driver locations** — shows you know 1.25M writes/sec kills any traditional DB, and that you need both in-memory AND geospatial capability
> 2. **Optimistic locking for driver assignment** — shows you identified the race condition and know how to prevent double-assignment
> 3. **WebSockets for live tracking** — shows you understand real-time push (not polling) and why it matters for map UX
