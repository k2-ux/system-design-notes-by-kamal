# Back-of-Envelope Estimation: A Practical Guide

> Goal: Spend 3-4 minutes on estimation, not 15. Get to "order of magnitude correct" — within 10x is fine.

---

## Section 1: Numbers Every Engineer Must Memorize

You will look these up in real life. In an interview, you cannot. Memorize these cold.

### Powers of 2 (Data Units)

| Power | Value | Name |
|-------|-------|------|
| 2^10  | 1,024 ≈ 1,000 | 1 KB (Kilobyte) |
| 2^20  | 1,048,576 ≈ 1,000,000 | 1 MB (Megabyte) |
| 2^30  | ~1 billion | 1 GB (Gigabyte) |
| 2^40  | ~1 trillion | 1 TB (Terabyte) |
| 2^50  | ~1 quadrillion | 1 PB (Petabyte) |

Memory trick: every 10 powers of 2 = 1,000x bigger. KB → MB → GB → TB → PB.

---

### Latency Numbers (From Fast to Slow)

These are the real costs of operations. Knowing these tells you *why* certain designs are better.

| Operation | Latency | Human comparison |
|-----------|---------|-----------------|
| L1 cache reference | 0.5 ns | — |
| L2 cache reference | 7 ns | 14x slower than L1 |
| RAM access | 100 ns | 200x slower than L1 |
| SSD random read | 150 μs | 150,000 ns |
| HDD seek + read | 10 ms | 10,000,000 ns |
| Network round trip (same datacenter) | 500 μs | |
| Network round trip (cross-continent) | 150 ms | |
| Reading 1 MB from RAM | 250 μs | |
| Reading 1 MB from SSD | 1 ms | |
| Reading 1 MB from HDD | 20 ms | |

Key takeaways from these numbers:
- RAM is 100,000x faster than disk. Cache everything you can in memory.
- SSD is 66x faster than HDD. Use SSD for databases.
- A cross-continent network call costs 150ms. Minimize round trips across regions.
- An HDD seek at 10ms means 100 random reads/sec maximum — this is why HDD is almost never used for random access in modern systems.

---

### Data Sizes Per Record Type

| Data type | Size |
|-----------|------|
| ASCII character | 1 byte |
| Integer (32-bit) | 4 bytes |
| Long / timestamp | 8 bytes |
| UUID / GUID | 16 bytes |
| IPv4 address | 4 bytes |
| IPv6 address | 16 bytes |
| Average tweet | 280 chars ≈ 280 bytes ≈ **0.3 KB** |
| Typical metadata row (user record) | ~100–500 bytes |
| Thumbnail image | ~10 KB |
| Profile photo (compressed) | ~50 KB |
| Full photo (JPEG) | ~300 KB |
| 1 minute of 720p video | ~50 MB |
| 1 minute of 1080p video | ~150 MB |
| 1 hour of 720p video | ~2 GB |
| 1 hour of 1080p video | ~4 GB |

---

## Section 2: The QPS Formula

QPS = Queries Per Second. This tells you how much load your system must handle.

### The Formula

```
Average QPS = DAU × requests_per_user_per_day / 86,400

Peak QPS = Average QPS × 3
```

Use 86,400 seconds/day. In practice, round to 100,000 to make the math easier.

The 3x multiplier for peak is a standard rule of thumb — traffic is not uniform, it spikes in the morning, at lunch, after work. 3x covers most real-world spikes.

---

### Worked Example: Twitter

**Given:** 300M DAU. Each user posts 1 tweet/day and reads 50 tweets/day.

**Write QPS (tweets being created):**
```
300,000,000 × 1 / 86,400
= 300,000,000 / 100,000   (rounding 86,400 to 100,000)
= 3,000 writes/sec

Peak write QPS = 3,000 × 3 = ~9,000 writes/sec
```

**Read QPS (timeline reads):**
```
300,000,000 × 50 / 86,400
= 15,000,000,000 / 100,000
= 150,000 reads/sec

Peak read QPS = 150,000 × 3 = ~450,000 reads/sec
```

**Conclusion:** Read:Write ratio is 50:1. This system is massively read-heavy.

Design implication: You need aggressive caching (Redis, Memcached), read replicas on your database, and a CDN. Writes are cheap; reads are the bottleneck.

**Say this out loud in the interview:**
> "The read-to-write ratio is about 50:1, so I'm going to focus on optimizing the read path. We'll need heavy caching."

That single sentence shows you know how to translate numbers into design decisions — which is exactly what the interviewer is looking for.

---

## Section 3: Storage Estimation

### The Formula

```
Daily storage = DAU × data_per_user_per_day

Total storage = Daily storage × retention_period_in_days
```

Always separate text/metadata storage from media storage. They differ by orders of magnitude.

---

### Worked Example: WhatsApp

**Given:** 500M DAU. Each user sends 40 messages/day. Average text message = 100 bytes. 10% of messages contain a photo (300 KB each).

**Step 1: Text message storage**
```
Daily text storage = 500M × 40 messages × 100 bytes
                   = 500,000,000 × 40 × 100
                   = 2,000,000,000,000 bytes
                   = 2 TB/day
```

**Step 2: 5-year text storage**
```
5-year storage = 2 TB × 365 × 5
               = 2 × 1,825
               = 3,650 TB
               ≈ 3.65 PB
```

**Step 3: Photo storage**
```
Photos per day = 500M users × 40 messages × 10% with photos
              = 500M × 4 photos
              = 2,000,000,000 photos/day
              = 2B photos/day

Photo storage/day = 2B × 300 KB
                  = 600,000,000,000 KB
                  = 600,000,000 MB
                  = 600,000 GB
                  = 600 TB/day
```

**Step 4: Total**
```
Total daily storage = 2 TB (text) + 600 TB (photos) = ~602 TB/day
5-year total        = 602 TB × 1,825 days ≈ 1.1 EB (Exabytes)
```

**Conclusion:** We need petabyte-scale, probably exabyte-scale, object storage. This points directly to S3-compatible object storage (AWS S3, Google Cloud Storage), not a traditional relational database for media.

---

## Section 4: Bandwidth Estimation

Bandwidth = how much data flows in and out per second. This determines your network infrastructure.

- **Ingress** = data coming *into* your system (uploads)
- **Egress** = data going *out* of your system (downloads, streaming)

Egress almost always dominates. Users consume far more than they produce.

### The Formula

```
Ingress bandwidth = data_uploaded_per_day / 86,400

Egress bandwidth = data_consumed_per_day / 86,400
```

---

### Worked Example: YouTube

**Given:** 500 hours of video uploaded per minute. 1 billion users watch an average of 30 minutes/day at 720p.

**Ingress (uploads):**
```
Upload rate = 500 hours/minute × 1 hour of 720p ≈ 2 GB
           = 500 × 2 GB / 60 seconds
           = 1,000 GB / 60
           ≈ 16.7 GB/sec ingress

(At 1080p ≈ 4 GB/hr: 500 × 4 / 60 ≈ 33 GB/sec)
```

**Egress (streaming):**
```
Total watch time/day = 1,000,000,000 users × 30 min = 30,000,000,000 minutes/day

720p streaming rate ≈ 2 GB/hr = 2,000 MB / 3,600 sec ≈ 0.56 MB/sec per stream

Total egress = 1B users × 30 min × 0.56 MB/sec
             ... simplify differently:

Total data served/day = 1B × 30 min × (2 GB/60 min)
                      = 1,000,000,000 × 1 GB
                      = 1,000,000,000 GB
                      = 1 EB/day

Per second = 1,000,000,000 GB / 86,400
           ≈ 11,574 GB/sec
           ≈ 11.5 TB/sec egress
```

**Conclusion:** YouTube needs ~11 TB/sec of outbound bandwidth globally. This is why YouTube uses CDNs (Akamai, their own Google Global Cache) to push content close to users — you cannot serve 11 TB/sec from a single datacenter. The math justifies the architecture.

---

## Section 5: How Many Servers Do I Need?

### Rule of Thumb

```
1 commodity server handles ~10,000 requests/sec for a typical web application
```

This varies based on what the server does, but 10K RPS is a reasonable starting point for a CPU-light application server (API gateway, stateless service).

For CPU-heavy workloads (video transcoding, ML inference): divide by 10x or more.

### The Formula

```
Servers needed = Peak QPS / 10,000

Add 2× headroom for safety and rolling deployments
```

---

### Worked Example

**Given:** From the Twitter example above, peak read QPS = 450,000.

```
Servers for reads = 450,000 / 10,000 = 45 servers minimum

With 2× headroom = 90 servers
```

**Given:** Peak write QPS = 9,000.

```
Servers for writes = 9,000 / 10,000 = 1 server minimum

With 2× headroom = 2-3 servers
```

**Conclusion:** Your read fleet is ~90 servers. Your write fleet is ~3 servers. This is why read replicas and horizontal read scaling matter so much here.

Also mention in the interview: "I'd put these behind a load balancer, and for stateless services, horizontal scaling is straightforward."

---

## Section 6: Common Mistakes

### Mistake 1: Using exact numbers

Wrong approach: trying to compute 300,000,000 / 86,400 exactly in your head.

Right approach: round aggressively.
- 86,400 → 100,000 (or 10^5)
- 300,000,000 → 3 × 10^8
- Result: 3 × 10^8 / 10^5 = 3,000. Done in 5 seconds.

You are estimating. Being off by 20% is irrelevant. Being off by 10x means you rounded wrong.

---

### Mistake 2: Silent calculation

The interviewer cannot read your mind. Say every assumption out loud.

Bad: *[quietly does math for 2 minutes]*

Good: "I'm going to assume 500M DAU. I'll assume each user uploads one photo per day at roughly 300KB. Let me calculate daily storage..."

If your assumptions are wrong, the interviewer will correct you. That is fine and expected.

---

### Mistake 3: Spending 15 minutes on math

Estimation is a 3-4 minute exercise. If you're still doing math at minute 10, you have lost the interview. The interviewer wants to see your system design, not your arithmetic.

If you get stuck: make a rough guess, state it as an assumption, and move on.

"I'll assume roughly 1TB/day of storage and come back to refine if needed — let me move on to the architecture."

---

### Mistake 4: Forgetting units

Always carry your units through the calculation. Bytes vs KB vs MB is a 1000x difference.

```
500M × 40 × 100 = ?

Without units: 2,000,000,000,000 — is that bytes or KB or GB?
With units: 500M users × 40 messages × 100 bytes = 2,000,000,000,000 bytes = 2TB
```

Convert at the very end: compute in the smallest unit, then convert up.

---

### Mistake 5: MB vs MiB

In interviews, don't worry about the 1024 vs 1000 distinction. Use 1 KB = 1,000 bytes (SI units) or 1 KB = 1,024 bytes (binary) — pick one and be consistent. The difference is under 10% and irrelevant at estimation scale.

---

## Quick Reference Card

Cut this out mentally and have it ready.

### Data Units
```
1 KB = 10^3 bytes
1 MB = 10^6 bytes
1 GB = 10^9 bytes
1 TB = 10^12 bytes
1 PB = 10^15 bytes
```

### Time
```
1 minute  = 60 seconds
1 hour    = 3,600 seconds
1 day     = 86,400 seconds  → round to 10^5
1 month   = 2.6M seconds    → round to 3 × 10^6
1 year    = 31.5M seconds   → round to 3 × 10^7
```

### Latency (memorize the order of magnitude)
```
L1 cache:        ~1 ns
RAM:             ~100 ns
SSD:             ~150 μs   (150,000 ns)
HDD:             ~10 ms    (10,000,000 ns)
Network (DC):    ~500 μs
Network (WAN):   ~150 ms
```

### Record Sizes
```
char:            1 B
int:             4 B
long/timestamp:  8 B
UUID:            16 B
Tweet:           ~300 B
User metadata:   ~100-500 B
Photo (JPEG):    ~300 KB
Video (720p/min):~50 MB
Video (1080p/hr):~4 GB
```

### Key Formulas
```
QPS (avg)   = DAU × req/user/day / 86,400
QPS (peak)  = avg QPS × 3
Storage/day = DAU × data/user/day
Servers     = peak QPS / 10,000  × 2 (headroom)
Bandwidth   = data_per_day / 86,400
```

---

## Worked Estimation Template

Use this exact sequence every time. Do not skip steps.

```
STEP 1 — STATE ASSUMPTIONS (30 seconds)
  "I'll assume X DAU, Y requests per user per day, Z data per request."

STEP 2 — QPS (60 seconds)
  Avg QPS   = DAU × req/user/day / 100,000
  Peak QPS  = Avg QPS × 3
  "Read-to-write ratio is ___. This is [read/write/balanced]-heavy."

STEP 3 — STORAGE (60 seconds)
  Daily     = DAU × data/user/day
  Total     = Daily × retention days
  Separate text from media.
  "We need roughly ___ scale. This suggests [MySQL / S3 / distributed FS]."

STEP 4 — BANDWIDTH (30 seconds)
  Ingress   = uploads per day / 86,400
  Egress    = consumed per day / 86,400
  "Egress dominates, so we'll need a CDN."

STEP 5 — SERVERS (30 seconds)
  Servers   = Peak QPS / 10,000 × 2
  "We need roughly ___ servers, deployed behind a load balancer."

STEP 6 — TRANSITION (5 seconds)
  "Based on these numbers, the key constraints are [read-heavy / storage-heavy /
  bandwidth-heavy]. Let me now walk through the high-level design with that in mind."
```

Total time: under 4 minutes. The point is not to get exact numbers. The point is to show you can identify the bottleneck and let it drive your design decisions.

---

## Putting It All Together: Design Instagram (Mini-Estimation)

**Assumptions stated out loud:**
- 500M DAU
- Average user views 50 photos/day (reads)
- Average user uploads 1 photo/day (writes)
- Photo size: 300 KB. Thumbnail: 10 KB.
- Retention: 10 years

**QPS:**
```
Write QPS = 500M × 1 / 100,000 = 5,000/sec
Read QPS  = 500M × 50 / 100,000 = 250,000/sec
Peak read = 750,000/sec
Ratio: 50:1 read-heavy
```

**Storage:**
```
Daily photo storage = 500M × 300 KB = 150,000,000,000 KB = 150 TB/day
10-year storage = 150 TB × 3,650 days = 547,500 TB ≈ 550 PB
```

**Bandwidth:**
```
Egress = 500M users × 50 photos × 300 KB / 86,400
       = 500M × 15,000 KB / 86,400
       = 7,500,000,000,000 KB / 86,400
       = 86,805,555 KB/sec
       ≈ 83 GB/sec egress
```

**Servers:**
```
Read servers = 750,000 / 10,000 × 2 = 150 servers
```

**Design conclusions spoken aloud:**
- "50:1 read ratio → heavy caching layer (Redis for metadata, CDN for photos)"
- "550 PB over 10 years → S3 or similar object storage, not a traditional DB"
- "83 GB/sec egress → we cannot serve this from one region, we need a global CDN"
- "150 read servers → stateless API tier behind a load balancer, easy to autoscale"

Those four conclusions are what your entire system design should be built around. The numbers led you there.
