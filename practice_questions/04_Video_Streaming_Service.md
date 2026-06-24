# Design a Video Streaming Service like YouTube
> Complete interview preparation guide — Q&A format, beginner to advanced.

---

## Table of Contents
1. [What Is It?](#what-is-it)
2. [Step 1 — Requirements](#step-1--requirements)
3. [Step 2 — Capacity Estimation](#step-2--capacity-estimation)
4. [Step 3 — Video Upload and Transcoding](#step-3--video-upload-and-transcoding)
5. [Step 4 — Video Streaming](#step-4--video-streaming)
6. [Step 5 — CDN Strategy](#step-5--cdn-strategy)
7. [Step 6 — Database Design](#step-6--database-design)
8. [Step 7 — Failure Scenarios](#step-7--failure-scenarios)
9. [Full System Diagram](#full-system-diagram)
10. [Quick Revision](#quick-revision)
11. [Real World Comparison — YouTube vs Small Scale (Lykstage)](#real-world-comparison--youtube-vs-small-scale)

---

## What Is It?

**Q: What makes YouTube architecturally different from Instagram?**

> Instagram serves images (3 MB each). YouTube serves videos (4 GB each). This changes everything:
> - You cannot download a video before watching — requires **chunked streaming**
> - Raw uploads must be **transcoded** into multiple resolutions for different devices/connections
> - **CDN is not optional** — it is the entire streaming architecture
> - High-frequency counters (view counts) create **write hotspot** problems not seen with images

---

## Step 1 — Requirements

**Q: What are the most important clarifying questions for YouTube?**

> 1. "Live streaming or on-demand only?" (completely different architecture — scope out live for interviews)
> 2. "Any upload size limit?" (determines chunked upload strategy — YouTube has no hard limit)
> 3. "Likes, comments, view counts needed?" (additional write features)

**Q: What is the hidden critical question most candidates miss?**

> "When a user uploads a 4K video — do we store and serve that exact file?"
>
> **Answer: No.** YouTube transcodes every video into multiple resolutions (360p, 480p, 720p, 1080p, 4K) and multiple formats (MP4, WebM). This is called **Video Transcoding** and it is the heart of YouTube's upload architecture.

**Q: What are the functional requirements?**

> 1. Users can upload videos (any size)
> 2. Users can stream videos on-demand
> 3. Support video titles, descriptions, tags, thumbnails
> 4. Search for videos by keyword
> 5. Likes, dislikes, comments, view counts

**Q: What are the non-functional requirements?**

> - **High availability** — 99.99% uptime
> - **Low latency** — video streaming must be smooth, no buffering
> - **High scalability** — 500M daily active users
> - **High durability** — uploaded videos must never be lost
> - **Eventual consistency** — view counts don't need to be exact in real-time

---

## Step 2 — Capacity Estimation

**Q: Walk through capacity estimation for YouTube.**

> **Assumptions:**
> - 500M daily active users
> - 5M video uploads per day
> - Each user watches ~5 videos per day
> - Average video size = 500 MB (after transcoding all resolutions)
>
> **Traffic:**
> ```
> Uploads:       5M / 86,400 ≈ 58 uploads/second
> Video streams: 500M × 5 / 86,400 ≈ 29,000 streams/second
> ```
>
> **Storage:**
> ```
> 5M videos/day × 500 MB = 2.5 PB/day of new video data
> → S3 object storage only — no database can hold this
> → CDN is mandatory — serving 2.5 PB/day from origin is impossible
> ```

---

## Step 3 — Video Upload and Transcoding

**Q: Should the upload API wait for transcoding to finish before responding to the user?**

> **No.** Transcoding a 10GB 4K video takes 10-30 minutes. Making the user wait that long is unacceptable.
>
> The correct approach is **async processing**:
> ```
> User uploads video
>         ↓
> App Server receives file → saves raw video to S3
>         ↓
> Server responds immediately: "Upload successful! Processing..." ✅
>         ↓ (user is free, doesn't wait)
> Background: Transcoding Workers pick up the job asynchronously
> ```

**Q: How does the server notify Transcoding Workers that a new video needs processing?**

> **Kafka (Message Queue)** — same Event-Driven pattern as WhatsApp:
> ```
> App Server → publishes "VIDEO_UPLOADED" event to Kafka
>                     ↓
>              [Kafka Queue]
>                     ↓
>       [Transcoding Worker Pool]
>       (workers consume from Kafka)
>                     ↓
>   Worker 1 → 360p  chunks → S3
>   Worker 2 → 480p  chunks → S3
>   Worker 3 → 720p  chunks → S3  (all in parallel)
>   Worker 4 → 1080p chunks → S3
>   Worker 5 → 4K    chunks → S3
>                     ↓
>       DB updated: video status = "ready"
>       User notified: "Your video is live!"
> ```

**Q: Why are multiple Transcoding Workers running in parallel?**

> Each resolution is independent — there's no reason to transcode them sequentially. Running 5 workers in parallel cuts transcoding time by ~5x.
>
> This is horizontal scaling applied to a compute-intensive background job.

---

## Step 4 — Video Streaming

**Q: How does YouTube stream a 4GB video without making you download it all first?**

> **Chunking** — every video is split into small 2-10 second segments:
> ```
> video.mp4 (4 GB)
> → chunk_001.ts (5 MB) — covers 0:00 to 0:05
> → chunk_002.ts (5 MB) — covers 0:05 to 0:10
> → chunk_003.ts (5 MB) — covers 0:10 to 0:15
> → ... thousands of chunks
> ```
> Your browser downloads chunk_001 and starts playing immediately. While chunk_001 plays, chunk_002 is downloading in the background. You watch smoothly without ever downloading the full 4GB file.

**Q: What is Adaptive Bitrate Streaming (ABR)?**

> ABR allows YouTube to automatically switch video quality mid-stream based on your connection speed — without interrupting playback.
>
> ```
> Browser constantly measures download speed:
>
> Fast internet  (10 Mbps) → request 1080p chunks
> Medium         (2 Mbps)  → request 720p chunks
> Slow           (0.5 Mbps)→ request 360p chunks
>
> Connection drops mid-video?
> → Next chunk request switches to 360p folder
> → No interruption — quality drops, buffering doesn't
> ```
>
> This is only possible because transcoding created ALL resolutions upfront.

**Q: Where are video chunks stored?**

> ```
> S3 stores ALL versions of ALL chunks:
> s3://youtube/video_123/360p/chunk_001.ts
> s3://youtube/video_123/360p/chunk_002.ts
> s3://youtube/video_123/720p/chunk_001.ts
> s3://youtube/video_123/720p/chunk_002.ts
> s3://youtube/video_123/1080p/chunk_001.ts
> ...
> ```
> Browser requests chunks from CDN → CDN serves from nearest edge node → CDN miss → fetched from S3.

---

## Step 5 — CDN Strategy

**Q: Why is CDN not optional for YouTube?**

> ```
> Without CDN:
> User in Karachi watches video → request travels to US origin servers
> → 200ms+ latency PER CHUNK → constant buffering 😤
>
> With CDN:
> Video chunks cached at edge node near Karachi (e.g. Dubai)
> → 5ms latency per chunk → smooth playback ⚡
> ```
> At 29,000 streams/second globally, serving all traffic from origin servers is physically impossible.

**Q: Not all videos deserve CDN caching. How do you decide which videos to cache?**

> **Cache by popularity:**
> ```
> Trending (1M+ views)   → proactively pushed to ALL global CDN nodes
>                           (cache warming — done before viral spike hits)
> Regular (10K views)    → cached only in regions where it's being watched
> New/unpopular (<100)   → served directly from S3, not CDN cached
> ```
>
> CDN storage space is finite and expensive. Caching a video with 3 views on 200 edge nodes globally wastes resources.

---

## Step 6 — Database Design

**Q: What data needs to be stored and where?**

> ```
> Videos table (PostgreSQL or Cassandra):
> ┌────────┬─────────┬────────┬─────────────┬────────┬─────────────┐
> │ id     │ user_id │ title  │ description │ status │ created_at  │
> │        │         │        │             │(ready/ │             │
> │        │         │        │             │process)│             │
> └────────┴─────────┴────────┴─────────────┴────────┴─────────────┘
>
> Video chunks metadata (stored in a manifest file, e.g. HLS .m3u8):
> Points browser to correct chunk URLs at each resolution
>
> Comments (Cassandra):
> ┌────────┬──────────┬─────────┬──────────┬─────────────┐
> │ id     │ video_id │ user_id │ text     │ created_at  │
> └────────┴──────────┴─────────┴──────────┴─────────────┘
>
> View counts (Redis → batch flush to DB):
> key: "views:video_123"  value: 10,000,083
> ```

**Q: A viral video gets 10M views in one hour — 2,778 writes per second to ONE row. What's wrong with updating the DB directly?**

> Every update requires **locking that row**. At 2,778 simultaneous updates/second to the same row:
> ```
> Request 1: UPDATE SET count = 1000001 WHERE video_id = 123 → locks row
> Request 2: waiting...
> Request 3: waiting...
> ... 2,778 requests queuing → DB collapses 💀
> ```
> This is called a **write hotspot** — too many writes targeting the same data.

**Q: How do you fix the write hotspot problem for view counts?**

> **Redis counter + batch flush:**
> ```
> Every view → atomic Redis increment (microseconds, no locking):
> INCR "views:video_123" → 1,000,001
> INCR "views:video_123" → 1,000,002
> [2,778 increments/sec — Redis handles easily]
>
> Every 30 seconds → batch job flushes to DB:
> UPDATE videos SET view_count = 1,000,083 WHERE id = 123
> [ONE DB write every 30s instead of 2,778/sec]
> ```
>
> Same pattern as Instagram likes. Redis absorbs high-frequency writes, DB gets batched updates.
>
> **Rule:** High-frequency updates to a single value → Redis counter + periodic batch flush to DB.

---

## Step 7 — Failure Scenarios

**Q: A Transcoding Worker crashes mid-transcoding. What happens?**

> The Kafka message was not acknowledged — Kafka redelivers it to another available worker. The partially transcoded files are discarded and transcoding restarts from the raw S3 file.
>
> **Idempotency is key:** Transcoding the same raw video twice produces the same output — safe to retry.

**Q: CDN node goes down. What happens?**

> Users routed to next nearest CDN node automatically. Slightly higher latency for that region temporarily. Origin S3 serves as fallback.

**Q: How do you handle a video going viral and overwhelming CDN?**

> 1. **Cache warming** — detect viral signal early (view count spike) → proactively push to more CDN nodes before it peaks
> 2. **CDN auto-scaling** — major CDN providers (Cloudfront, Fastly) scale edge capacity automatically
> 3. **Rate limiting** — prevent any single video from consuming all bandwidth on a node

---

## Full System Diagram

```
UPLOAD FLOW:
[User] → [Load Balancer] → [Upload Service]
                                  ↓
                           [S3 raw video]
                                  ↓
                            [Kafka Queue]
                                  ↓
                    [Transcoding Worker Pool]
                    Worker 1 → 360p chunks → S3
                    Worker 2 → 720p chunks → S3
                    Worker 3 → 1080p chunks → S3
                                  ↓
                         [DB: video = ready]

STREAMING FLOW:
[User] → [Load Balancer] → [App Server]
                                  ↓
                         [DB/Redis: video metadata]
                         [manifest file: chunk URLs]
                                  ↓
              [Browser fetches chunks from CDN] ⚡
              CDN HIT  → serve chunk instantly
              CDN MISS → fetch from S3 → cache → serve
                                  ↓
              [Adaptive Bitrate: browser switches
               resolution based on connection speed]

VIEW COUNT:
[User watches] → [Redis INCR "views:video_id"]
                       ↓ (every 30 seconds)
              [Batch flush to DB]
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| Video storage | S3 + CDN | 2.5 PB/day — only object storage + CDN can handle this |
| Transcoding | Async via Kafka + Worker Pool | Can't block user for 30 min transcoding wait |
| Multiple resolutions | 360p/480p/720p/1080p/4K | Adaptive bitrate streaming needs all versions |
| Streaming method | Chunked (2-10 sec segments) | Can't download 4GB before watching |
| Adaptive quality | ABR (Adaptive Bitrate) | Auto-switch resolution based on connection speed |
| CDN caching priority | Popularity-based | Cache warming for trending, regional for regular |
| View count writes | Redis counter + batch flush | Write hotspot prevention — 2,778 writes/sec to one row kills DB |
| Comments storage | Cassandra | High write volume, time-ordered, no complex JOINs |
| Worker coordination | Kafka | Event-driven, decoupled, retry on worker crash |

### Three things to mention proactively in an interview
> 1. **Video transcoding is async** — shows you understand not to block the user for a 30-minute job
> 2. **Adaptive Bitrate Streaming** — shows you know HOW video streaming actually works, not just "use CDN"
> 3. **Redis counter for view counts** — shows you identified the write hotspot problem and know the fix

---

## Real World Comparison — YouTube vs Small Scale

> This section compares YouTube's architecture against a small-scale video platform (like Lykstage) to understand WHEN and WHY architectural decisions change with scale.

**Q: What is HLS and how does it relate to everything we designed?**

> **HLS (HTTP Live Streaming)** is Apple's standard format for video streaming over the internet. It is the implementation of chunked streaming + adaptive bitrate that we designed.
>
> ```
> HLS output after transcoding:
> video.m3u8          ← master manifest (index of all resolutions)
>     ↓ points to:
> 360p/index.m3u8     ← chunk list for 360p
> 720p/index.m3u8     ← chunk list for 720p
> 1080p/index.m3u8    ← chunk list for 1080p
>     ↓ each points to:
> chunk_001.ts  (2 seconds of video)
> chunk_002.ts  (2 seconds of video)
> chunk_003.ts  ...
> ```
>
> Browser downloads the master manifest first → reads available resolutions → measures connection speed → picks the right resolution playlist → downloads chunks one by one → plays smoothly.
>
> The `.m3u8` file IS the adaptive bitrate mechanism — the browser switches which resolution's `.m3u8` it follows based on speed.

**Q: How does video uploading differ between a small platform and YouTube?**

> | Step | Small Platform (Lykstage) | YouTube |
> |------|--------------------------|---------|
> | **Upload method** | Single HTTP POST to S3 | Multipart chunked upload |
> | **Why different** | Small files work fine as one request | 10GB cannot upload as one reliable HTTP request — connection drops lose everything |
> | **Connection drop** | Re-upload entire file | Re-upload only the failed chunk |
>
> YouTube uses **S3 multipart upload** — the video is split into parts during upload itself. Each part uploads independently. If your connection drops at 9GB into a 10GB upload, only the last chunk re-uploads.

**Q: How does transcoding differ between a small platform and YouTube?**

> | Step | Small Platform (Lykstage) | YouTube |
> |------|--------------------------|---------|
> | **Who does it** | Managed service (AWS MediaConvert / Mux) | Custom-built pipeline |
> | **How** | Send raw video URL → they return .m3u8 | Kafka → fleet of dedicated transcoding servers |
> | **Codec** | H.264 (industry standard) | AV1 / VP9 (custom, 30% smaller at same quality) |
> | **Quality levels** | 3-4 (360p, 720p, 1080p) | 8+ (144p, 240p, 360p, 480p, 720p, 1080p, 1440p, 4K, 8K) |
> | **Cost model** | Pay per minute of video transcoded | Own infrastructure — fixed cost, massive scale |
>
> **Key insight:** At small scale you BUY transcoding (AWS MediaConvert). At YouTube's scale you BUILD it — AWS would charge billions and you can do it cheaper yourself with custom codecs.

**Q: How does video streaming differ between a small platform and YouTube?**

> | Step | Small Platform (Lykstage) | YouTube |
> |------|--------------------------|---------|
> | **Streaming format** | HLS only (.m3u8) | DASH + HLS both supported |
> | **Player** | HLS.js or Video.js (open source) | Custom YouTube player |
> | **CDN** | AWS CloudFront | YouTube's own global CDN |
> | **CDN nodes** | ~100 CloudFront edge locations | 1000s of YouTube edge servers globally |
> | **Cache warming** | Not needed at small scale | Critical — pre-warms trending videos before spike |
> | **Scale** | Thousands of concurrent viewers | 500M daily active users |
>
> **DASH vs HLS:** DASH (Dynamic Adaptive Streaming over HTTP) is more flexible than HLS and works natively on all browsers without plugins. YouTube supports both — HLS for Apple devices, DASH for everything else.

**Q: What is the core architectural principle that explains ALL these differences?**

> **Small scale = use managed services. Massive scale = build your own.**
>
> ```
> Lykstage:
> Upload    → S3 (AWS manages)
> Transcode → AWS MediaConvert (AWS manages)
> Stream    → CloudFront CDN (AWS manages)
> You focus → product features, not infrastructure
>
> YouTube:
> Upload    → custom multipart pipeline
> Transcode → custom Kafka + worker fleet + AV1 codec
> Stream    → custom global CDN
> Why?      → at 500M users, AWS would cost more than building it yourself
>             + custom codecs give 30% bandwidth savings = billions saved
> ```
>
> This is a general system design principle:
> - **Startup / small scale** → managed services (faster to ship, pay per use)
> - **Hyperscale** → custom infrastructure (cheaper per unit, more control)
