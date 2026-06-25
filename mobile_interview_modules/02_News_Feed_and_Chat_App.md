# 📱 News Feed & Chat App — Design Deep Dive

> Two of the most asked MSD interview questions. Master these and you've covered half the game.

---

## 🗞️ PART 1: NEWS FEED APP (Think Instagram / Facebook / X)

---

### Q: "Design a news feed app." Where do I even start?

**A:** First, don't panic. Every social feed app boils down to the same core idea: **a scrollable list of posts that refreshes in real-time.**

Start by scoping with the interviewer:

- ✅ Users can scroll an infinite feed of posts
- ✅ Users can like, comment, share
- ✅ Users can create posts with text + images
- ✅ Feed supports infinite scrolling (pagination)
- ✅ Optimistic updates (likes appear instantly)
- ❌ Out of scope: Stories, reels, DMs, ads (unless they ask)

Non-functional stuff to mention:
- Should handle millions of users
- Feed should load fast even on slow connections
- Offline browsing of cached posts would be nice

---

### Q: What API endpoints does a feed app need?

**A:** Pretty straightforward REST:

```
GET  /v1/feed?cursor={token}&limit=20    → Fetch paginated feed
GET  /v1/posts/{postId}                  → Get a specific post's detail
POST /v1/posts                           → Create a new post
POST /v1/posts/{postId}/like             → Like a post
POST /v1/posts/{postId}/comments         → Add a comment
```

**Why cursor-based pagination for the feed?**
Because feeds are alive — new posts get added constantly. If you use offset-based pagination (`page=2`), posts shift around and users see duplicates or miss content. Cursors say *"give me everything after THIS post"* which stays stable.

Think of it like a bookmark in an ever-growing book vs. saying "page 42" in a book where pages keep getting inserted.

---

### Q: What does the feed's data model look like?

**A:**

```kotlin
data class FeedPost(
    val id: String,
    val author: UserPreview,     // name, avatar, handle
    val content: String,
    val imageUrls: List<String>,
    val likesCount: Int,
    val commentsCount: Int,
    val isLikedByMe: Boolean,
    val createdAt: Timestamp
)
```

Notice `isLikedByMe` — that's a server-computed field so the client can show a filled vs. unfilled heart immediately. No extra API call needed!

---

### Q: How does the architecture look?

**A:** Classic layered setup:

**External services:** Backend (REST API), CDN (serves images blazingly fast), Push Notifications (for new likes/comments)

**Data layer:**
- `FeedRepository` — the brain. Coordinates between local cache and remote API
- `PostsRepository` — handles creating posts, likes, comments
- Local database for offline reading (Room on Android, Core Data on iOS)

**UI layer:**
- **Feed Screen** → shows the scrollable list, managed by its ViewModel
- **Post Detail Screen** → full post with comments
- **Create Post Screen** → compose new content

Data flows like a one-way street:
```
API → Repository → ViewModel → Screen → (user taps something) → ViewModel → Repository → API
```

---

### Q: The juicy stuff — what should I deep-dive into for a feed app?

**A:** Here are the **3 killer deep dives** for feed design:

#### 1. 🖼️ Efficient Image Loading

Images are the #1 performance killer in feed apps. Here's how to handle them like a pro:

- **Progressive loading**: Show a tiny blurry placeholder → load medium quality → sharp final image
- **Image CDN with resizing**: Request `image.jpg?w=400&q=80` instead of the full 4K original. The CDN resizes on-the-fly.
- **Recycled views**: As a post scrolls off screen, reuse its image view for the next post (RecyclerView / UICollectionView)
- **Memory-aware caching**: Two-level cache — fast in-memory (LRU, ~50MB) + slower disk cache (~200MB). Check memory first, then disk, then network.

**In React Native specifically — use `react-native-fast-image`:**

```tsx
import FastImage from 'react-native-fast-image';

<FastImage
  source={{
    uri: post.imageUrl,
    priority: FastImage.priority.normal,
    cache: FastImage.cacheControl.immutable, // CDN images never change — cache forever
  }}
  style={styles.postImage}
  resizeMode={FastImage.resizeMode.cover}
/>
```

`react-native-fast-image` wraps SDWebImage (iOS) and Glide (Android) — the same battle-tested image libraries that native apps use. You get automatic disk caching, memory LRU eviction, request deduplication (two cells requesting the same image only fire one network request), and progressive loading.

For a blurry placeholder while the real image loads, pair it with a small low-res thumbnail from your CDN:

```tsx
// Show a 20px blurred thumbnail until the full image loads
<FastImage source={{ uri: post.thumbnailUrl }} style={[styles.postImage, styles.blur]} blurRadius={10} />
<FastImage source={{ uri: post.imageUrl }} style={[styles.postImage, StyleSheet.absoluteFill]} />
```

The server generates the thumbnail once and includes its URL in the feed response. Zero extra client-side work.

#### 2. 🔄 Optimistic Updates

When a user taps "like":
1. **Immediately** flip the heart to red and increment the count in the UI
2. Fire the API call in the background
3. If it fails? Quietly revert the UI and show a subtle error

This makes the app feel *instant* even on 3G. Instagram does exactly this.

**In React Native with React Query — this is almost trivial:**

```typescript
const { mutate: likePost } = useMutation({
  mutationFn: (postId: string) => api.likePost(postId),

  onMutate: async (postId) => {
    // Cancel in-flight fetches to prevent overwriting our optimistic update
    await queryClient.cancelQueries({ queryKey: ['feed'] });
    const previousFeed = queryClient.getQueryData<FeedPost[]>(['feed']);

    // Optimistically update the cache right now
    queryClient.setQueryData<FeedPost[]>(['feed'], (old = []) =>
      old.map(post =>
        post.id === postId
          ? { ...post, isLikedByMe: true, likesCount: post.likesCount + 1 }
          : post
      )
    );
    return { previousFeed }; // snapshot for rollback
  },

  onError: (_, __, context) => {
    // Something went wrong — silently roll back
    queryClient.setQueryData(['feed'], context?.previousFeed);
  },

  onSettled: () => {
    // Always re-sync with server after mutation completes (success or failure)
    queryClient.invalidateQueries({ queryKey: ['feed'] });
  },
});
```

The key point to emphasize in an interview: React Query handles the snapshot, rollback, and re-sync — the developer writes the *what* (update this post's like state), not the *how* (manually manage three loading states and undo logic).

#### 3. 📡 Feed Refresh Strategy

How does the feed know there's new content?

| Approach | How it works | Tradeoff |
|----------|-------------|----------|
| **Pull-to-refresh** | User manually pulls down to refresh | Simple, but user must actively check |
| **Silent background sync** | App periodically checks for new posts | Drains battery if too frequent |
| **Push notification trigger** | Server pushes "new content available" → app fetches | Efficient! But requires push infra |
| **Hybrid** | Push says "hey, there's new stuff" + pull-to-refresh | Best of both worlds ✅ |

The hybrid approach is what most real apps use. Say that in your interview.

---

### Q: What about offline support for the feed?

**A:** The pattern is simple and elegant:

1. **On first load**: Fetch feed from API → save to local DB → display from DB
2. **On subsequent opens**: Show cached feed from DB immediately (feels instant) → fetch fresh data in background → update DB → UI auto-updates
3. **No internet?** Show whatever's in the DB with a subtle "You're offline" banner

The local database is your **single source of truth**. The UI never talks to the network directly — it only watches the database. The network writes TO the database. This is the magic of UDF (Unidirectional Data Flow).

---

### Q: What are the offline storage options in React Native — which one do I pick?

**A:**

| Need | Library | Why |
|------|---------|-----|
| Simple key-value (tokens, flags, prefs) | **MMKV** | 10× faster than AsyncStorage, **synchronous** reads — critical for tokens you need at app boot |
| Large structured data (posts, messages) | **WatermelonDB** | SQLite-backed, reactive, built for 10k+ record datasets |
| Medium structured data (simpler setup) | **react-native-sqlite-storage + Drizzle ORM** | Direct SQLite with type-safe queries, good for most apps |
| Persisted server cache | **React Query + MMKV persister** | Persist the query cache to MMKV — next app launch gets instant data before network |
| Files, videos, documents | **react-native-fs** | Read/write files in the app sandbox; pairs with `react-native-background-upload` |
| Secrets / auth tokens | **react-native-keychain** | iOS Keychain / Android Keystore — hardware-backed, biometric unlock support |

**The MMKV pitch in an interview:**

*"For simple key-value storage I'd use MMKV instead of AsyncStorage. AsyncStorage is async and JS-based — every read is a promise. MMKV is a C++ key-value store with synchronous reads, which matters when you need the auth token immediately on app boot to decide whether to show the login screen."*

```typescript
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

// Synchronous — no await
storage.set('authToken', 'eyJhbGc...');
const token = storage.getString('authToken'); // immediate read
```

**For feed offline support** — persist the React Query cache:

```typescript
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';
import { persistQueryClient } from '@tanstack/react-query-persist-client';

const persister = createSyncStoragePersister({
  storage: {
    getItem: (key) => storage.getString(key) ?? null,
    setItem: (key, value) => storage.set(key, value),
    removeItem: (key) => storage.delete(key),
  },
});

persistQueryClient({ queryClient, persister, maxAge: 1000 * 60 * 60 * 24 }); // 24h
```

Now when the user opens the app offline, they see yesterday's feed immediately — no loading spinner.

---

### Q: What state management should I propose for a React Native news feed?

**A:** The modern answer splits concerns cleanly:

| State type | Tool | Why |
|-----------|------|-----|
| **Server state** (feed posts, user data) | **React Query** | Caching, deduplication, background refresh, pagination — all free |
| **Global client state** (auth, theme, cart) | **Zustand** or **Jotai** | Tiny, zero-boilerplate, no provider hell |
| **Local UI state** (modal open, input value) | React `useState` | No need to reach for a library |
| **Forms** | **React Hook Form** | Controlled inputs without a re-render per keystroke |

In an interview: *"I'd use React Query for server state because it eliminates the manual loading/error/data state dance, and Zustand for client state because it avoids Redux's ceremony while still keeping state outside components for sharing across screens."*

**Why not Redux for a feed app?** It's not wrong — but it's overkill. You'd manually write actions, reducers, and selectors to manage data that React Query handles automatically. Redux shines when you have complex state with many concurrent writers and need time-travel debugging.

---

---

### Q: How do you make a FlatList perform well with thousands of posts?

**A:** `FlatList` is React Native's virtualized list — it only renders what's on screen. But there are several levers that separate a smooth feed from a janky one:

```tsx
<FlatList
  data={posts}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <PostCard post={item} />}

  // Pagination — fires when user is 50% from the bottom
  onEndReached={() => {
    if (hasNextPage && !isFetchingNextPage) fetchNextPage();
  }}
  onEndReachedThreshold={0.5}

  // Performance knobs
  initialNumToRender={10}        // Only render visible items on first paint
  maxToRenderPerBatch={5}        // How many items to add per batch as user scrolls
  windowSize={7}                 // Render window: 3 screens above + 3 below current
  removeClippedSubviews          // Detach off-screen views from native hierarchy
  ListFooterComponent={isFetchingNextPage ? <ActivityIndicator /> : null}
/>
```

**The `getItemLayout` trick (biggest win for fixed-height items):**

```tsx
const POST_HEIGHT = 120;

<FlatList
  {...props}
  getItemLayout={(_, index) => ({
    length: POST_HEIGHT,
    offset: POST_HEIGHT * index,
    index,
  })}
/>
```

Without `getItemLayout`, FlatList has to measure every item before it can scroll to a specific index — slow. With it, scroll-to-index is instant because the math is trivial. Mention this unprompted and it signals you've hit real performance walls.

**Memoize your `renderItem` component:**

```tsx
// Without memo: re-renders every PostCard when ANY part of the feed state changes
// With memo: only re-renders when that specific post's data changes
const PostCard = React.memo(({ post }: { post: Post }) => {
  return <View>...</View>;
}, (prev, next) => prev.post.id === next.post.id && prev.post.likesCount === next.post.likesCount);
```

---

## 💬 PART 2: CHAT APP (Think WhatsApp / Telegram / Slack)

---

### Q: "Design a chat app." What makes this different from the feed?

**A:** The big difference is **real-time**. In a feed, it's okay if a post appears 5 seconds late. In chat? If my message takes 5 seconds, I'm switching to WhatsApp.

Scope it:
- ✅ 1:1 messaging in real-time
- ✅ Group chats
- ✅ Text messages (+ optional: images, videos)
- ✅ Online/offline status, typing indicators
- ✅ Read receipts (delivered, read)
- ✅ Message history & search
- ❌ Out of scope: Voice/video calls, stories, encryption (unless asked)

---

### Q: This seems like a REST problem, right? Just POST messages?

**A:** REST handles *some* of it, but here's the issue:

If Alice sends Bob a message, how does Bob's phone *know* about it instantly? With REST, Bob's app would have to keep asking the server *"Any new messages? No? How about now? Now?"* That's **polling** — and it's wasteful, laggy, and kills battery.

The answer: a **hybrid approach**.

| What | Protocol | Why |
|------|----------|-----|
| Sending messages, fetching history | **REST (HTTP)** | Simple, reliable, works great for request-response |
| Receiving messages in real-time, typing indicators, read receipts | **WebSocket** | Always-on connection, server pushes updates instantly |

Think of it like this:
- REST = Sending letters through the post office 📮
- WebSocket = Having an open phone line 📞

Most real chat apps (WhatsApp, Slack, Discord) use exactly this hybrid.

---

### Q: How do I implement WebSocket in React Native?

**A:** React Native ships with the browser-standard `WebSocket` API — no library needed. Wrap it in a service class with automatic reconnection:

```typescript
class ChatSocketService {
  private socket: WebSocket | null = null;
  private reconnectAttempts = 0;
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;

  connect(token: string) {
    this.socket = new WebSocket(`wss://api.chat.com/v1/socket?token=${token}`);

    this.socket.onopen = () => {
      this.reconnectAttempts = 0; // Reset on successful connect
    };

    this.socket.onmessage = (event) => {
      const data = JSON.parse(event.data as string);
      this.handleEvent(data);
    };

    this.socket.onclose = () => {
      this.scheduleReconnect(token);
    };
  }

  private scheduleReconnect(token: string) {
    // Exponential backoff: 1s, 2s, 4s, 8s... capped at 30s
    const delay = Math.min(1000 * 2 ** this.reconnectAttempts, 30_000);
    this.reconnectTimer = setTimeout(() => {
      this.reconnectAttempts++;
      this.connect(token);
    }, delay);
  }

  send(event: object) {
    if (this.socket?.readyState === WebSocket.OPEN) {
      this.socket.send(JSON.stringify(event));
    }
    // If socket isn't open, queue the message (see offline handling below)
  }

  disconnect() {
    if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
    this.socket?.close();
  }
}
```

**Key point to mention in your interview**: Mobile apps lose connectivity constantly — tunnels, elevators, switching networks. Without reconnection logic, a user who loses signal for 10 seconds has a broken chat. Exponential backoff prevents hammering the server when it's overwhelmed.

---

### Q: How do WebSocket events work in a chat app?

**A:** Once the app connects to `wss://api.chat.com/v1/socket/updates`, the server can push events like:

```json
{
  "type": "new-message",
  "payload": {
    "messageId": "msg-789",
    "conversationId": "conv-123",
    "senderId": "user-alice",
    "content": "Hey, are you coming tonight?",
    "timestamp": "2025-01-15T18:30:00Z"
  }
}
```

Other event types: `typing-started`, `typing-stopped`, `message-read`, `user-online`, `user-offline`

The client subscribes to these and updates the UI reactively. No polling needed!

---

### Q: What's the data model for messages?

**A:**

```kotlin
data class Message(
    val id: String,           // Server-assigned unique ID
    val localId: String,      // Client-generated (for optimistic display)
    val conversationId: String,
    val senderId: String,
    val content: String,
    val status: MessageStatus, // SENDING → SENT → DELIVERED → READ
    val createdAt: Timestamp
)

enum class MessageStatus {
    SENDING,    // Still on your phone
    SENT,       // Server received it ✓
    DELIVERED,  // Recipient's phone got it ✓✓
    READ        // They actually opened it ✓✓ (blue)
}
```

Those grey checkmarks → blue checkmarks on WhatsApp? That's literally the `status` field moving through those 4 states.

---

### Q: How do messages get ordered correctly on screen?

**A:** This is trickier than it sounds! Consider:

- Alice sends "Yes!" at 6:00:01 PM
- Bob sends "Are you coming?" at 6:00:00 PM
- But Alice's message arrives at the server FIRST because Bob's network was slow

If you sort by server timestamp, the conversation looks correct: Bob's question → Alice's answer.
If you sort by client timestamp, it's backwards because Alice's device clock might be off.

**The answer:** Sort by **server-assigned message ID** (or server timestamp). The server is the source of truth for ordering.

But there's a twist — what about messages you JUST sent that don't have a server ID yet? Use a **local ID** and show them optimistically at the bottom. Once the server confirms, replace with the real ID and re-sort if needed.

---

### Q: How do you handle sending messages when offline?

**A:** This is where the **message lifecycle** gets interesting:

1. User types message and hits send
2. App generates a `localId`, saves to local DB with status = `SENDING`, and shows it in the chat immediately
3. App tries to send via REST API
4. **If online**: Server responds with `messageId` → update local DB to status = `SENT`
5. **If offline**: Message sits in a queue. When connectivity returns, the app replays the queue in order
6. Later, server pushes `DELIVERED` and `READ` updates via WebSocket

The critical design choice: **local-first with background sync**. The user never waits for the network. The UI is always responsive.

---

### Q: What about the architecture diagram for a chat app?

**A:**

```
┌─────────────── UI LAYER ──────────────────┐
│  Chat List Screen    Conversation Screen   │
│  (StateHolder)       (StateHolder)         │
│                      └→ Message List       │
│                      └→ Compose Bar        │
├─────────────── DATA LAYER ────────────────┤
│  Messages Repository                       │
│  ├─ Remote Data Source (REST + WebSocket)  │
│  ├─ Local Data Source (SQLite / Room)      │
│  └─ Message Queue (for offline sends)     │
│                                            │
│  Push Notifications Client                 │
├────────────── EXTERNAL ───────────────────┤
│  Backend │ CDN │ Push Provider (FCM/APNs) │
└────────────────────────────────────────────┘
```

The Messages Repository is the heart. It:
- Writes incoming WebSocket messages to the local DB
- Exposes a reactive stream of messages that the UI subscribes to
- Queues outgoing messages and retries on failure

---

### Q: What are the best deep-dive topics for a chat app interview?

**A:**

#### 1. 🔄 Message Sync Strategy

When a user opens the app after being offline for 3 days, how do you load the 500 messages they missed?

- On app launch, call `GET /v1/messages?lastSyncedMessage={id}&limit=100`
- The `lastSyncedMessage` parameter tells the server: *"I've seen everything up to this ID. What's new?"*
- Paginate through the results until you're caught up
- Store everything in the local DB

This is called **delta sync** — you only fetch what's changed, not the entire history.

#### 2. 🟢 Typing Indicators & Presence

When Alice starts typing, her app sends a `typing-started` event over WebSocket. Bob's app shows "Alice is typing..." for a few seconds. If no `typing-stopped` event comes and no new message arrives, the indicator auto-hides after ~5 seconds (debounce).

For online presence: the server tracks WebSocket connections. If Alice's socket drops, she's "offline." Simple but effective.

#### 3. 💾 Message Storage & Eviction

Messages pile up. A chat app can easily eat gigabytes. Strategies:

- **Per-conversation limits**: Keep the last 1000 messages per chat locally, fetch older ones on demand
- **User-driven cleanup**: "This chat is using 2.3 GB. Clear old media?" (WhatsApp does this)
- **Time-based eviction**: Auto-remove local messages older than 6 months (but they're still on the server)

#### 4. 🔒 End-to-End Encryption (Bonus)

If asked: *"Messages are encrypted on the sender's device and decrypted only on the recipient's. The server can't read them. We'd use the Signal Protocol — it's what WhatsApp and Signal use."*

You don't need to explain the crypto math. Just show you know the concept: **client-side keys, server is just a mailman**.

---

### Q: What's the hardest part of building a chat app?

**A:** Honestly? It's making everything feel **instant** when nothing actually is.

The message has to travel: Your phone → cell tower → internet → server → internet → cell tower → their phone. That's a LOT of hops. But users expect WhatsApp-speed delivery.

The tricks:
- **Optimistic UI** — show the message before the server confirms
- **Local-first architecture** — UI reads from local DB, network writes to local DB
- **Smart reconnection** — when WebSocket drops, reconnect with exponential backoff (wait 1s, 2s, 4s, 8s...) so you don't hammer the server
- **Compact data** — keep WebSocket payloads tiny (just the essentials, not the entire message object)

---

*Next up: [03 — Stock Trading & Hotel Reservation →](./03_Stock_Trading_and_Hotel_Reservation.md)*
