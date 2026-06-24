# Design a Notification System
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

**Q: What does a notification system do and why is it a separate system?**

> A notification system sends alerts to users across multiple channels — push, email, SMS. It's a separate system because:
> 1. **Scale** — sending 10M notifications is a background job, not something you do synchronously
> 2. **Multiple channels** — each channel has a different provider (APNs, Twilio, SendGrid) with different APIs
> 3. **User preferences** — each user has different opt-in/opt-out settings per channel and quiet hours
>
> Almost every product (Instagram, Amazon, WhatsApp) uses a shared notification system rather than rebuilding it per product.

---

## Step 1 — Requirements

**Q: What are the 3 types of notifications a system must support?**

> 1. **Push Notifications** — alerts on phone lock screen (APNs for iPhone, FCM for Android)
> 2. **Email** — via providers like SendGrid or AWS SES
> 3. **SMS** — text messages via Twilio or similar

**Q: What are the functional requirements?**

> 1. Send push, email, and SMS notifications
> 2. Support user preferences — opt-out per channel, quiet hours
> 3. Guarantee delivery — retry on failure
> 4. Support scheduled and delayed notifications
> 5. Track delivery status (sent, delivered, failed)

**Q: What are the non-functional requirements?**

> - **High throughput** — 10M+ notifications per event (viral posts, marketing blasts)
> - **Low latency** — push notifications must feel instant (< 1 second from trigger)
> - **Reliability** — notifications must not be lost (at-least-once delivery)
> - **Scalability** — horizontal scaling of workers as volume grows

---

## Step 2 — Capacity Estimation

**Q: Walk through notification volume for a system like Instagram.**

> **Assumptions:**
> - 500M users
> - Average user follows 200 accounts
> - Top creators post 5 times/day
>
> **Peak notification burst:**
> ```
> Ronaldo (500M followers) posts
> → 500M push notifications in seconds
> → At 10,000 notifications/sec per worker → need 50,000 workers
> → Kafka distributes the load across worker pool
> ```
>
> **Daily volume:**
> ```
> 500M users × 10 notifications/day = 5B notifications/day
> 5B / 86,400 = ~58,000 notifications/second average
> ```

---

## Step 3 — The Hard Problems

### Problem 1: Fan-out at Scale

**Q: Ronaldo posts. Instagram must notify 500M followers. Do you send 500M notifications synchronously before returning a response to Ronaldo's app?**

> **No — async fan-out via Kafka:**
>
> ```
> Ronaldo posts
>       ↓
> App Server: "notify 500M followers"
>       ↓
> Publish ONE event to Kafka → return response instantly ✅
>       ↓
> Notification Workers pull from Kafka:
>   Worker 1 → sends to followers 1-100,000
>   Worker 2 → sends to followers 100,001-200,000
>   Worker 3 → ...
>       ↓
> Each worker calls the right provider:
>   Push  → APNs (Apple) / FCM (Google)
>   Email → SendGrid
>   SMS   → Twilio
> ```
>
> Ronaldo's app responds in milliseconds. 500M notifications go out in the background over a few minutes.

### Problem 2: User Preferences and Quiet Hours

**Q: A user has disabled SMS and set quiet hours 10pm-8am. How does the system respect this?**

> **Workers check preferences BEFORE calling any provider:**
>
> ```
> Kafka event: "notify user_123"
>       ↓
> Worker pulls event
>       ↓
> Worker checks DB: user_123 preferences
>   → push:  enabled
>   → email: disabled
>   → SMS:   disabled
>   → quiet hours: 10pm-8am (it's currently 11pm)
>       ↓
> Push is enabled BUT it's quiet hours → SKIP ❌
>       ↓
> Re-queue with delay: send at 8am instead
> ```
>
> External providers (APNs, Twilio) only get called if the worker decides to send. They have no access to your user database.

### Problem 3: Retry on Failure

**Q: APNs returns an error — the push notification failed. What do you do?**

> **Retry with exponential backoff:**
>
> ```
> Attempt 1: fails → wait 1 second → retry
> Attempt 2: fails → wait 2 seconds → retry
> Attempt 3: fails → wait 4 seconds → retry
> Attempt 4: fails → mark as failed, log for monitoring
> ```
>
> After max retries, log the failure. Never drop silently. Monitoring alerts if failure rate spikes (e.g., APNs is down).

---

## Step 4 — Architecture

**Q: What are the core components?**

> ```
> 1. Notification Service (API)
>    → Accepts "send notification" requests from other services
>    → Validates request, publishes to Kafka
>
> 2. Kafka (Message Queue)
>    → Decouples senders from workers
>    → Buffers 500M notification bursts
>    → One topic per channel: push-topic, email-topic, sms-topic
>
> 3. Notification Workers (Consumer Pool)
>    → Pull from Kafka
>    → Check user preferences in DB
>    → Check quiet hours
>    → Call appropriate provider
>    → Retry on failure
>
> 4. Providers (External)
>    → APNs  — Apple push notifications
>    → FCM   — Android push notifications
>    → SendGrid — email delivery
>    → Twilio   — SMS delivery
>
> 5. User Preference DB (PostgreSQL)
>    → channel opt-in/out per user
>    → quiet hours settings
>    → device tokens (needed for push)
>
> 6. Notification Log (Cassandra)
>    → Every notification sent/failed/retried
>    → Time-ordered, high write volume
> ```

**Q: What is a device token and why do you need it for push notifications?**

> When a user installs your app, APNs/FCM gives their device a unique token:
> ```
> iPhone installs Instagram
>   → APNs generates token: "abc123xyz..."
>   → Instagram stores: user_123 → device_token: "abc123xyz..."
>
> When sending push:
>   → Worker fetches user's device token from DB
>   → Calls APNs: send to token "abc123xyz..."
>   → APNs delivers to that specific device
> ```
> Without the device token, you can't target a specific phone.

---

## Full System Diagram

```
[Instagram / Amazon / Any Service]
           ↓
  [Notification Service API]
           ↓
        [Kafka]
    ┌──────┼──────┐
    ↓      ↓      ↓
[Push   [Email  [SMS
Workers] Workers] Workers]
    │      │      │
    ↓      ↓      ↓
 [APNs] [SendGrid] [Twilio]
 [FCM]
    │
[User Preference DB]  ← workers check before sending
[Notification Log]    ← workers write after sending
```

---

## Quick Revision

| Decision | Choice | Reason |
|----------|--------|--------|
| Fan-out | Kafka + worker pool | Async — don't block the caller for 500M sends |
| Channel routing | Separate Kafka topic per channel | Push/email/SMS have different failure modes and speeds |
| User preferences | PostgreSQL | Checked by worker before every send |
| Retry | Exponential backoff (1s, 2s, 4s) | Provider outages are temporary — retry before giving up |
| Delivery log | Cassandra | High write volume, time-ordered, simple queries |
| Push delivery | APNs (iOS) + FCM (Android) | You don't deliver to devices yourself — providers do it |
| Quiet hours | Re-queue with delay | Don't drop — schedule for the next allowed window |

### Three things to mention proactively in an interview
> 1. **Kafka for async fan-out** — shows you know 500M synchronous sends is impossible
> 2. **Worker checks preferences before calling provider** — shows you understand the gatekeeper pattern
> 3. **Separate channels with retry** — shows you know each provider fails differently and needs independent retry logic
