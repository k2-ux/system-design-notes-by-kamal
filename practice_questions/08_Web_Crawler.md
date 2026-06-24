# Design a Web Crawler
> Complete interview preparation guide — Q&A format, beginner to advanced.

---

## Table of Contents
1. [What Is It?](#what-is-it)
2. [Step 1 — Requirements](#step-1--requirements)
3. [Step 2 — Capacity Estimation](#step-2--capacity-estimation)
4. [Step 3 — The Hard Problems](#step-3--the-hard-problems)
5. [Step 4 — Architecture](#step-4--architecture)
6. [Full System Diagram](#full-system-diagram)
7. [Quick Revision](#quick-revision)

---

## What Is It?

**Q: What does a web crawler actually do?**

> A web crawler starts with a list of seed URLs, fetches those pages, extracts all links, adds them to a queue, and repeats — forever. Google uses this to discover and index every page on the internet.
>
> ```
> Start: [google.com, wikipedia.org]
>          ↓
> Fetch google.com → finds [gmail.com, maps.google.com, news.google.com]
>          ↓
> Add to queue → fetch each → find more links
>          ↓
> Repeat until the entire internet is crawled
> ```

---

## Step 1 — Requirements

**Q: What are the functional requirements?**

> 1. Start from a set of seed URLs and crawl the web recursively
> 2. Extract and store page content (HTML, text) for indexing
> 3. Discover new URLs from crawled pages and add to queue
> 4. Respect website crawl rules (robots.txt, crawl-delay)
> 5. Re-crawl pages periodically to detect updates

**Q: What are the non-functional requirements?**

> - **Scale** — must crawl ~50 billion pages
> - **Politeness** — must not overload any single website
> - **Robustness** — must handle dead links, timeouts, malformed HTML
> - **Extensibility** — easy to add new content types (images, PDFs)
> - **Freshness** — high-priority pages re-crawled more frequently

---

## Step 2 — Capacity Estimation

**Q: Walk through the scale of crawling the web.**

> **Assumptions:**
> - 50 billion pages on the internet
> - Average page = 500 KB
> - Crawl the entire web in 4 weeks
>
> **Crawl rate needed:**
> ```
> 50B pages / (4 weeks × 7 days × 86,400 sec) = ~20,000 pages/second
> ```
>
> **Storage:**
> ```
> 50B pages × 500 KB = 25 petabytes
> → Only distributed object storage (S3) handles this
> ```
>
> **URL dedup storage:**
> ```
> 50B URLs × 100 bytes = 5 TB just for the visited set
> → Too big for RAM → need Bloom Filter (reduces to ~60 GB)
> ```

---

## Step 3 — The Hard Problems

### Problem 1: Avoiding Infinite Loops (URL Deduplication)

**Q: The internet has circular links. cnn.com links to google.com which links back to cnn.com. How do you avoid crawling the same URL forever?**

> **Keep a "visited" record and check before crawling.**
>
> Naive approach — Hash Set:
> ```
> visited = HashSet()
>
> Before crawling any URL:
>   if url in visited → SKIP
>   else → crawl it, then visited.add(url)
> ```
> O(1) lookup. But 50B URLs × 100 bytes = **5 TB RAM** — too expensive.
>
> **Better approach — Bloom Filter:**
> ```
> Hash Set:     50B URLs × 100 bytes = 5 TB RAM ❌
> Bloom Filter: 50B URLs × ~10 bits  = 60 GB RAM ✅ (98% smaller)
>
> Tradeoff: false positives possible
> → May say "already seen" when you haven't → skip a valid URL (minor annoyance)
> → NEVER says "not seen" when you have → no infinite loops ✅
> ```
>
> False positives = occasionally miss a URL. Acceptable.
> False negatives = crawl the same page forever. Catastrophic.
> Bloom Filter is the right fit.

### Problem 2: Politeness Policy

**Q: You find 10,000 links on nytimes.com. Do you fire 10,000 simultaneous requests?**

> **No — that's a DDoS attack.** You'd overload their server and get your IP banned.
>
> ```
> Bad crawler:
> nytimes.com ← 10,000 simultaneous requests 💀 (server crashes)
>
> Polite crawler:
> nytimes.com ← 1 request every 2 seconds ✅
> ```
>
> **Implementation — per-domain queues:**
> ```
> nytimes.com → [url1, url2, url3...] — process 1 every 2 seconds
> bbc.com     → [url1, url2, url3...] — process 1 every 2 seconds
> cnn.com     → [url1, url2, url3...]
>
> Different domains crawled in parallel.
> Same domain crawled slowly and politely.
> ```
>
> Also read and obey `robots.txt`:
> ```
> nytimes.com/robots.txt:
>   Disallow: /admin/
>   Disallow: /user-data/
>   Crawl-delay: 2
> ```

### Problem 3: Prioritization

**Q: BBC breaking news vs a personal blog that hasn't updated in 2 years. Which do you crawl first?**

> **Priority Queue — higher traffic and more inbound links = higher priority:**
>
> ```
> URL Queue (sorted by priority score):
>   [bbc.com/breaking-news, score: 95]  ← crawl first
>   [wikipedia.org/article, score: 87]
>   [medium.com/post,       score: 43]
>   [random-blog.xyz,       score: 2]   ← crawl last
> ```
>
> **How priority is calculated:**
> - **PageRank** — how many other sites link to this URL? More inbound links = more important
> - **Update frequency** — BBC publishes 100 articles/day → re-crawl often. Personal blog hasn't updated in 2 years → crawl rarely
>
> Personal blogs get crawled eventually — just not urgently.

---

## Step 4 — Architecture

**Q: What are the core components of a web crawler?**

> ```
> 1. URL Frontier (Priority Queue)
>    → Holds URLs to be crawled, sorted by priority
>    → Per-domain sub-queues enforce politeness
>
> 2. Fetcher (Worker Pool)
>    → Downloads HTML from URLs in the queue
>    → Handles timeouts, redirects, errors
>    → Multiple workers for parallelism (~20,000 pages/sec)
>
> 3. Parser
>    → Extracts links from downloaded HTML
>    → Sends new links to URL Frontier
>    → Sends page content to storage
>
> 4. Bloom Filter (Dedup)
>    → Checks if URL was already crawled before adding to frontier
>
> 5. Content Store (S3)
>    → Stores raw HTML of crawled pages (25 PB)
>
> 6. Indexer (downstream)
>    → Reads from content store, builds search index (Elasticsearch)
> ```

**Q: What happens when a page returns a 404 error or times out?**

> ```
> 404 Not Found → mark URL as dead, remove from future crawl schedules
> Timeout       → retry up to 3 times with exponential backoff
>                 (wait 1s, then 2s, then 4s between retries)
> 5xx Server Error → retry later, lower priority score for that domain
> ```
> Robustness is key — the crawler runs 24/7 unattended and must handle any failure gracefully.

---

## Full System Diagram

```
[Seed URLs]
     ↓
[URL Frontier - Priority Queue]
  (per-domain sub-queues for politeness)
     ↓
[Fetcher Worker Pool] ← 20,000 pages/sec
  (handles HTTP, timeouts, robots.txt)
     ↓
[Parser]
  │              │
  ↓              ↓
[New URLs]    [Page Content]
  │              │
  ↓              ↓
[Bloom Filter] [S3 Content Store]
(dedup check)  (25 PB raw HTML)
  │              │
  ↓              ↓
[URL Frontier] [Indexer → Elasticsearch]
(add if new)   (for search)
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| URL deduplication | Bloom Filter | 5 TB HashSet → 60 GB Bloom Filter; false positives OK, false negatives not |
| Politeness | Per-domain queues with delay | Avoid DDoS'ing websites and getting banned |
| Crawl priority | Priority queue (PageRank + update frequency) | High-traffic pages indexed first |
| Page storage | S3 | 25 PB of HTML — only object storage handles this |
| Search indexing | Elasticsearch (downstream) | Inverted index for fast text search |
| Fault tolerance | Retry with exponential backoff | Crawler runs unattended 24/7, must handle any failure |

### Three things to mention proactively in an interview
> 1. **Bloom Filter for deduplication** — shows you know HashSet doesn't scale to 50B URLs and you know the right alternative
> 2. **Politeness policy with per-domain queues** — shows you understand real-world constraints (robots.txt, crawl-delay)
> 3. **Priority queue based on PageRank** — shows you understand not all pages are equal and crawling is a resource allocation problem
