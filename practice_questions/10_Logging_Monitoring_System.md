# Design a Logging and Monitoring System
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

**Q: What does a logging and monitoring system do?**

> It watches over all your other systems so you know when something breaks — before users complain.
>
> Three pillars:
> 1. **Logs** — raw text records of everything that happens (`ERROR: DB connection failed at 3:42pm`)
> 2. **Metrics** — numbers over time (CPU %, requests/sec, error rate, latency p99)
> 3. **Alerts** — automated rules that fire when metrics cross a threshold (`if error_rate > 5% → page the on-call engineer`)
>
> Without this, you're flying blind. You only find out your system is down when users complain on Twitter.

---

## Step 1 — Requirements

**Q: What are the functional requirements?**

> 1. Collect logs from all services and servers in real-time
> 2. Store and search logs (find all errors from the last hour)
> 3. Track metrics over time (CPU, memory, latency, error rate)
> 4. Set threshold-based alert rules
> 5. Alert engineers automatically when rules fire (PagerDuty, Slack, SMS)
> 6. Visualize metrics on dashboards (Grafana)

**Q: What are the non-functional requirements?**

> - **High write throughput** — 100,000+ log lines/second from hundreds of servers
> - **Low latency ingestion** — logs must be searchable within seconds of being written
> - **Durability** — logs must never be lost (needed for debugging and compliance)
> - **Long retention** — hot logs: 30 days fast; cold logs: 1 year archived cheaply
> - **High availability** — the monitoring system must stay up even when everything else is down

---

## Step 2 — Capacity Estimation

**Q: Walk through the write volume for a logging system.**

> **Assumptions:**
> - 100 servers
> - Each writes 1,000 log lines/second
> - Average log line = 1 KB
>
> **Write volume:**
> ```
> 100 servers × 1,000 lines/sec = 100,000 lines/second
> 100,000 × 1 KB = 100 MB/second incoming data
> ```
>
> **Storage:**
> ```
> 100 MB/sec × 86,400 sec/day = ~8.6 TB/day
> × 30 days hot retention = 258 TB in Elasticsearch
> × 1 year cold = ~3 PB in S3 (compressed)
> ```
>
> **Conclusion:** No database handles 100 MB/second direct writes. Need Kafka as a buffer.

---

## Step 3 — The Hard Problems

### Problem 1: Ingestion at 100 MB/second

**Q: 100,000 log lines per second. You can't write directly to a database — it collapses. What do you use?**

> **Kafka as a buffer (shock absorber):**
>
> ```
> 100 servers
>     ↓ (each streams logs)
> [Kafka]  ← absorbs 100,000 lines/sec easily
>     ↓
> [Log Processors / Workers]
>     ↓              ↓
> [Elasticsearch]  [S3]
> (search recent   (archive cold
>  logs fast)       logs cheaply)
> ```
>
> Kafka decouples producers (servers) from consumers (storage). Workers pull at their own pace. No storage system gets hammered directly.
>
> Hot logs → Elasticsearch (fast search, expensive, keep 30 days)
> Cold logs → S3 (cheap, compressed, keep 1 year)

### Problem 2: Automated Alerting

**Q: CPU on Server 5 spikes to 99% at 3am. Does a human need to be watching a dashboard to catch this?**

> **No — threshold-based alerting fires automatically:**
>
> ```
> Alert Rules (configured by engineers):
>   IF cpu_usage > 90% for 5 minutes    → page on-call engineer
>   IF error_rate > 5%                  → send Slack alert
>   IF latency_p99 > 2 seconds          → send email warning
>   IF disk_usage > 80%                 → send warning
>
> Server 5: CPU = 99%
>       ↓
> Metrics DB: stores "server5.cpu = 99% at 3:42am"
>       ↓
> Alert Engine: checks rules every 10 seconds
>       ↓
> Rule matches: cpu > 90% → FIRE ALERT
>       ↓
> PagerDuty → wakes up on-call engineer 📱
> ```
>
> Real tools: **Prometheus** (metrics collection) + **Grafana** (dashboards) + **PagerDuty** (alert routing).

### Problem 3: Log Search at Scale

**Q: An engineer needs to find all 500 errors from the last hour across 100 servers. How?**

> **Elasticsearch with structured logs:**
>
> ```
> Bad log (unstructured):
>   "ERROR DB connection failed for user 123 at 3:42pm"
>   → Full text search only, slow
>
> Good log (structured JSON):
>   { "level": "ERROR", "service": "db", "user_id": 123,
>     "message": "connection failed", "timestamp": "2024-01-15T03:42:00Z" }
>   → Query by any field in milliseconds
>
> Elasticsearch query:
>   level=ERROR AND timestamp >= "last 1 hour"
>   → Returns 500 matching logs in milliseconds ⚡
> ```
>
> Structured logging is the difference between "search 8.6 TB of text" and "query an indexed field."

---

## Step 4 — Architecture

**Q: What are the core components?**

> ```
> 1. Log Agents (on every server)
>    → Lightweight process that tails log files
>    → Streams log lines to Kafka in real-time
>    → Tools: Fluentd, Logstash, Filebeat
>
> 2. Kafka (Ingestion Buffer)
>    → Absorbs massive write spikes
>    → Decouples servers from storage
>    → Retains logs for 24h as safety buffer
>
> 3. Log Processors (Workers)
>    → Pull from Kafka
>    → Parse and structure logs
>    → Route to Elasticsearch (hot) and S3 (cold)
>
> 4. Elasticsearch (Hot Storage)
>    → Full-text and structured search
>    → Last 30 days of logs
>    → Powers the search UI
>
> 5. S3 (Cold Storage)
>    → Compressed archives older than 30 days
>    → Cheap, slow — for compliance and rare deep dives
>
> 6. Metrics DB (Time-Series DB)
>    → Stores CPU, memory, latency numbers over time
>    → Tools: Prometheus, InfluxDB
>
> 7. Alert Engine
>    → Evaluates threshold rules every 10 seconds
>    → Fires alerts to PagerDuty / Slack / SMS
>
> 8. Dashboard (Grafana)
>    → Visualizes metrics from Prometheus
>    → Shows live graphs, trends, anomalies
> ```

---

## Full System Diagram

```
[Server 1] [Server 2] ... [Server 100]
     ↓           ↓               ↓
  [Log Agent] [Log Agent]  [Log Agent]
  (Fluentd)   (Fluentd)   (Fluentd)
         ↓         ↓        ↓
              [Kafka]
           ↓           ↓
    [Log Processors] [Metrics Collectors]
      ↓        ↓           ↓
[Elasticsearch] [S3]  [Prometheus]
(hot search)  (cold    (time-series
              archive)  metrics)
                            ↓
                      [Alert Engine]
                            ↓
                  [PagerDuty / Slack / SMS]
                            ↓
                        [Grafana]
                      (dashboards)
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| Log ingestion | Kafka | Buffer for 100 MB/sec — no DB handles direct writes at this rate |
| Hot log storage | Elasticsearch | Full-text + structured search, fast for recent logs |
| Cold log storage | S3 (compressed) | Cheap archival — rarely queried, must keep for compliance |
| Metrics storage | Prometheus (time-series DB) | Built for numbers over time, efficient compression |
| Alerting | Threshold rules + PagerDuty | Automated — no human watches dashboards 24/7 |
| Dashboards | Grafana | Visualizes Prometheus metrics in real-time |
| Structured logs | JSON format | Enables fast field-level queries vs slow full-text scan |
| Log agents | Fluentd / Filebeat | Lightweight, runs on every server, tails log files |

### Three things to mention proactively in an interview
> 1. **Kafka as ingestion buffer** — shows you know 100 MB/sec kills databases and need a shock absorber
> 2. **Hot vs cold storage split** — shows you understand cost optimization (Elasticsearch is expensive, S3 is cheap)
> 3. **Structured JSON logs** — shows you know unstructured text doesn't scale for search
