# 🎮 The Interview Game Plan

> How to not freeze when they say "Design the YouTube app in 45 minutes."

---

### Q: So what even IS a Mobile System Design interview?

**A:** Imagine your interviewer slides a napkin across the table and says:
*"Design Instagram. Go."*

You're not writing code. You're not pixel-pushing UI. You're **thinking out loud** about how a real app works under the hood — the APIs, the data flow, what happens when the user's on a subway with patchy signal, and why your chat messages don't arrive in random order.

It's basically: **"Prove you can think like an architect, not just a coder."**

---

### Q: Is there a cheat-code structure I can follow every time?

**A:** Yes! There's a beautiful 5-step recipe that works for literally any problem. Think of it like cooking — same steps, different ingredients:

| Step | What You Do | Time | Vibe |
|------|------------|------|------|
| 🔍 **1. Scope it** | Ask questions. Nail down what you're building (and what you're NOT). | ~5 min | Detective mode |
| 📡 **2. Design the API** | Pick protocols, define endpoints, sketch data models. | ~5 min | Backend brain |
| 🏗️ **3. Draw the architecture** | Boxes, arrows, layers. The "big picture" diagram. | ~10 min | Whiteboard artist |
| 🔬 **4. Deep dive** | Pick 1–2 juicy topics and go deep. This is where you shine. | ~20 min | Show-off time |
| 🎁 **5. Wrap up** | Summarize. Mention what else you'd do with more time. | ~5 min | Mic drop |

**The golden rule:** Don't skip Step 1. Jumping straight to architecture is like cooking without knowing if your guest is vegetarian.

---

### Q: Step 1 — What kind of questions should I ask? I don't want to look clueless.

**A:** You won't look clueless — you'll look *smart*. Here's why: the interviewer intentionally leaves the problem vague to see if you'll blindly charge ahead or think first.

Great questions sound like:

- 🎯 *"Are we designing for iOS, Android, or both?"*
- 👥 *"What's our expected scale — are we talking 10K users or 10M?"*
- 📶 *"Should the app work offline?"*
- 🔔 *"Do we need push notifications?"*
- 🔐 *"Any special security needs? Like, is this a banking app?"*
- 🚫 *"What can we explicitly leave out of scope?"*

Then organize what you learn into two buckets:

**Functional** = What the app *does* (features a user can see)
**Non-functional** = How the app *behaves* (fast? secure? works offline? battery-friendly?)

---

### Q: Step 2 — How do I pick a network protocol without overthinking it?

**A:** Here's the cheat sheet. You really only need to know 4:

| Protocol | One-liner | When to pick it |
|----------|-----------|----------------|
| **REST** | The Honda Civic of APIs — reliable, everywhere, gets the job done | Almost always. CRUD operations, searches, uploads. |
| **WebSocket** | An always-open phone call between client and server | Chat apps, live stock prices, multiplayer games |
| **SSE** | A one-way radio broadcast from server to client | Feed updates, notification streams, live scores |
| **gRPC** | REST's faster, nerdier cousin that speaks binary | Ultra-high-scale apps (Google/Netflix territory) |

**Default move in interviews:** Start with REST. Then say *"For the real-time features, I'll add WebSocket/SSE on top."* Boom — you sound thoughtful, not dogmatic.

---

### Q: In React Native, how do I actually make network requests — Fetch, Axios, or React Query?

**A:** They solve different problems and you'll often use all three layers together:

| Layer | Tool | Does what |
|-------|------|-----------|
| **HTTP client** | `fetch` (built-in) or `axios` | Makes the raw HTTP request. Axios adds interceptors and timeout support. |
| **Server state manager** | **React Query (TanStack Query)** | Caches responses, deduplicates requests, handles background refresh, loading/error states |
| **WebSocket** | Built-in `WebSocket` API | RN ships with the same WebSocket API as browsers — no library needed |

In an interview, the answer that stands out:

*"I'd use React Query as my server state layer. It sits on top of fetch/axios and gives me automatic caching, background refetching when the app comes to foreground, and deduplication — so if two components both need the same user profile, only one request goes out. For real-time, I'd pair it with WebSocket and invalidate or update the React Query cache when events arrive."*

This is the pattern modern RN teams use in production. Mentioning React Query by name shows you're current.

**What about `Axios` vs `fetch`?**
- `fetch` is built-in, no bundle cost. Fine for most apps.
- `axios` shines when you need request/response interceptors (e.g., attaching auth tokens to every request, refreshing expired JWTs transparently).
- Either one can be used as React Query's `queryFn`.

---

### Q: What does a good API design actually look like in an interview?

**A:** Keep it simple and RESTful. You're not writing production docs — you're showing you understand the pattern:

```
GET    /v1/posts              → Get a list of posts (paginated)
GET    /v1/posts/{id}         → Get one specific post
POST   /v1/posts              → Create a new post
PUT    /v1/posts/{id}         → Update a post
DELETE /v1/posts/{id}         → Delete a post
```

Then sketch a data model:
```
Post {
  id: String
  authorId: String
  content: String
  imageUrl: String?
  createdAt: Timestamp
  likesCount: Int
}
```

**Bonus points:** Mention pagination in your list endpoint. Say something like:
*"I'll use cursor-based pagination here because the feed is constantly changing — offset-based would cause items to shift between pages."*

Interviewer: 😍

---

### Q: I'm a React Native developer — how do I write data models in an interview?

**A:** Use **TypeScript interfaces**. They're clean, expressive, and every mobile interviewer recognizes them:

```typescript
interface Post {
  id: string;
  authorId: string;
  content: string;
  imageUrl?: string;        // ? = optional field
  createdAt: string;        // ISO 8601 timestamp from the API
  likesCount: number;
  isLikedByMe: boolean;     // server-computed field, saves a client-side lookup
}

interface PaginatedResponse<T> {
  items: T[];
  nextCursor: string | null; // null = you've reached the end
}
```

One thing worth mentioning: TypeScript interfaces only enforce types at compile time — there's no runtime guarantee that the API actually returns what you expect. In a production RN app, you'd validate responses with **Zod**:

```typescript
const PostSchema = z.object({
  id: z.string(),
  authorId: z.string(),
  content: z.string(),
  imageUrl: z.string().optional(),
  createdAt: z.string(),
  likesCount: z.number(),
  isLikedByMe: z.boolean(),
});

type Post = z.infer<typeof PostSchema>; // Type is derived from the schema — one source of truth
```

This is particularly valuable when your backend team changes a field name without telling anyone. Zod catches it at the boundary instead of letting a `undefined` crash bubble up to the UI.

---

### Q: How does React Native's architecture affect my interview answers?

**A:** React Native runs in **two threads** — you need to understand this to give senior-level answers:

| Thread | Runs | Can be blocked by |
|--------|------|-------------------|
| **JS Thread** | Your React components, state, business logic | Heavy JSON parsing, complex renders, synchronous computation |
| **UI Thread (Main)** | Native rendering, gestures, animations | Synchronous JS calls crossing the bridge |

This split has real design implications you can drop into answers:

- **Animations**: Say *"I'd use React Native Reanimated with worklets so animations run on the UI thread — they stay smooth even when the JS thread is busy fetching data."*
- **Heavy data**: *"Parsing a 500-item feed response on the JS thread can cause frame drops. I'd use a streaming JSON parser or process the response in a background worker."*
- **State updates**: React reconciliation happens on the JS thread — keep renders cheap. Memoize expensive components with `React.memo`.

The **New Architecture** (Fabric + JSI + TurboModules) largely eliminates the serialization overhead of the bridge, making native calls direct function pointers. Mention it casually: *"With the New Architecture enabled, synchronous native module calls become possible — useful for things like reading from the local database without async overhead."*

---

### Q: Step 3 — The architecture diagram feels scary. What do I actually draw?

**A:** Relax, it's just boxes and arrows. Think of it as a sandwich:

```
🍞 Top bread    = UI Layer (screens + ViewModels)
🥩 The filling  = Data Layer (repositories + data sources)
🍞 Bottom bread = External stuff (backend, CDN, push provider)
```

Here's what goes in each layer:

**UI Layer:**
- Your screens (Home, Detail, Settings, etc.)
- Each screen gets a "state holder" (ViewModel) that manages its data and logic
- The screen is dumb — it just renders whatever the ViewModel tells it to

**Data Layer:**
- **Repositories** — the traffic cops. They decide: *"Should I grab this from the local database or call the API?"*
- **Remote Data Source** — talks to the backend through a Network Dispatcher
- **Local Data Source** — talks to your on-device database (Room, Core Data)

**External Services:**
- Backend server (your APIs live here)
- CDN (makes images/videos load fast by serving them from nearby servers)
- API Gateway (the bouncer — handles auth, rate limiting, security)
- Push Notification Provider (FCM for Android, APNs for iOS)

Draw it, label it, and walk through a single user flow: *"When a user opens the feed, the ViewModel asks the Repository, which checks the local DB first, then fetches fresh data from the API..."*

---

### Q: Step 4 — What should I deep-dive into?

**A:** This is your **"I'm not just a framework monkey"** moment. Pick topics that are relevant to the problem and genuinely interesting to discuss.

Here's a menu — pick 1 or 2 per interview:

| 🎯 Topic | Great for... | Why it impresses |
|----------|-------------|-----------------|
| Offline support | Chat, cloud storage, feeds | Shows you think about real-world mobile pain |
| Caching strategy | Any data-heavy app | "In-memory + disk dual cache with LRU eviction" = chef's kiss |
| Real-time updates | Chat, trading, collab tools | WebSocket buffering, reconnection, message ordering |
| Image/video optimization | Feeds, YouTube, Instagram | Lazy loading, downsampling, adaptive bitrate |
| Pagination | Any list-based app | Cursor vs. offset, prefetching, infinite scroll |
| Security | Fintech, medical, enterprise | Encryption, certificate pinning, token rotation |
| Background uploads | Cloud storage, social media | Resumable uploads, chunk management, retry with backoff |

**Pro tip:** Don't wait for the interviewer to ask. Say: *"I think the most interesting challenge here is X — let me walk you through how I'd solve it."* That's leadership energy.

---

### Q: Step 5 — What's the wrap-up? Isn't the interview just... over?

**A:** The wrap-up is your **encore**. It's where you show breadth by mentioning things you'd explore with more time:

- *"If we had more time, I'd design the offline sync strategy using a CRDT-based approach."*
- *"We'd also want to add analytics and A/B testing to measure feature engagement."*
- *"For international users, I'd implement localization with RTL support."*

It signals: *"I could go deeper. I just ran out of time."*

---

### Q: What are the biggest mistakes people make in these interviews?

**A:** Oh, I've got a top-5 list:

1. **🚀 Jumping to solutions** — Start coding architecture before asking a single question. *"So I'll use MVVM with Clean Architecture and..."* Slow down, cowboy.

2. **📚 Reciting textbook definitions** — *"MVVM stands for Model-View-ViewModel, a pattern introduced by..."* Nobody cares. Show how you *use* it, not that you memorized it.

3. **🏝️ Ignoring the unhappy path** — Everything works perfectly in your design? No network errors? No low battery? No offline? Suspicious.

4. **🔇 Going silent** — The interviewer can't read your mind. Think out loud. *"I'm considering WebSockets here because..."*

5. **🎨 Over-designing** — Adding microservices, event sourcing, and a blockchain to a todo app. Match the complexity to the problem.

---

### Q: Any last-minute tips before I walk into the interview room?

**A:** 

- **Always present trade-offs.** Don't say "I'll use WebSocket." Say "I'll use WebSocket *because* we need bidirectional real-time comms, but the tradeoff is more complex connection management compared to SSE."

- **Use real-world examples.** *"This is similar to how WhatsApp handles message delivery receipts"* makes you sound experienced, not theoretical.

- **Sketch as you talk.** A diagram is worth a thousand words. Even ugly boxes and arrows help.

- **Don't fear saying "I don't know."** But follow it with: *"Here's how I'd figure it out..."* or *"My instinct would be to..."*

- **Have fun.** Seriously. The best interviews feel like two engineers nerding out together. If you're enjoying it, the interviewer probably is too.

---

*Next up: [02 — News Feed & Chat App →](./02_News_Feed_and_Chat_App.md)*
