# ☁️ Google Drive, 🎬 YouTube & 📚 Pagination Library

> Cloud storage, video streaming, and designing a library instead of an app.
> Three very different beasts, one file.

---

## ☁️ PART 1: GOOGLE DRIVE APP

---

### Q: "Design a Google Drive mobile client." What's the big deal?

**A:** Google Drive is deceptively complex. On the surface it's "upload files, download files, done." But under the hood:

- Files can be **gigabytes** — you can't just yolo-upload 5GB over a flaky cell connection
- Users have **multiple devices** — change a file on your laptop, it should appear on your phone
- Files need **version history** — "oops, I overwrote my thesis" should be fixable
- Enterprise users want **encryption** — "can my IT admin read my files?" matters

Scope it:
- ✅ Upload/download files (up to 10GB!)
- ✅ Create/delete folders
- ✅ Resumable uploads (pick up where you left off)
- ✅ File versioning (view + restore old versions)
- ✅ Sync across devices
- ❌ Out of scope: Real-time collaborative editing, sharing/permissions

---

### Q: What's the API design?

**A:** REST for everything, plus SSE or WebSocket for real-time sync notifications:

```
GET    /v1/files?parentId={folderId}     → List files in a folder
GET    /v1/files/{fileId}                → Get file metadata
POST   /v1/files                         → Create file/folder
DELETE /v1/files/{fileId}                → Delete file/folder
PATCH  /v1/files/{fileId}               → Rename/move file

GET    /v1/files/{fileId}/revisions      → List version history
POST   /v1/files/{fileId}/revisions/{id}/restore → Roll back to old version
```

**Clever design note:** Folders are just files with `mimeType: "application/folder"`. One endpoint handles both. Less code, fewer bugs.

---

### Q: Resumable uploads — how do you upload a 5GB file without crying?

**A:** This is the **star deep dive** of the Google Drive interview. Regular uploads are boring. Resumable uploads are where you shine.

The problem: You're uploading a 2GB video. At 73% complete, you walk into an elevator. Connection dies. Without resumable uploads, you start over. With them? You pick up right where you left off.

**The 3-step dance:**

```
Step 1: "Hey server, I want to upload a big file"
POST /v1/upload?uploadType=resumable
Body: { name: "vacation.mp4", size: 2147483648, mimeType: "video/mp4" }
Response: { uploadId: "abc-123", chunkSize: 5242880, totalChunks: 410 }

Step 2: Send the file piece by piece
PUT /v1/upload/abc-123/chunk
Header: Content-Range: bytes 0-5242879/2147483648
Body: [raw binary data for chunk 1]
Response: 308 Resume Incomplete (keep going!)

...repeat for each chunk...

Step 3: Last chunk → server assembles the file
Response: 201 Created 🎉
```

**If the connection drops mid-upload:**
```
GET /v1/upload/abc-123
Response: { chunksProcessed: 298, totalChunks: 410 }
→ Resume from chunk 299!
```

**Real-world examples:**
- Google Drive, Google Photos, Dropbox, Box, OneDrive — they ALL use resumable uploads
- Google offers 3 upload types: simple (≤5MB), multipart (≤5MB + metadata), and resumable (everything else)

---

### Q: What are the mobile-specific challenges with uploads?

**A:** Oh, there are several fun ones:

| Challenge | Solution |
|-----------|----------|
| **App gets killed while uploading** | Save upload state (uploadId, chunksProcessed) to disk. Resume on next app launch. |
| **Running out of storage** | Check available space before upload. Show "Low storage" warning. Let users clear upload cache. |
| **User needs to pick files** | Use system file picker (temporary read access, no broad permission needed) |
| **Background uploads** | Use WorkManager (Android) / BGTasks (iOS) to continue uploading when app isn't visible |
| **Network changes** | Pause on cellular (if user prefers), resume on Wi-Fi |

---

### Q: What about file versioning and conflicts?

**A:** Two scenarios to discuss:

**Uploading a new version:**
- **Full copy upload**: Upload the entire file every time. Simple, reliable, used by Google Drive, Box, OneDrive.
- **Block-level sync**: Only upload the changed parts. Way more efficient for small edits to big files. Dropbox pioneered this.

For an interview, go with full copy upload unless asked. It's simpler and most services use it.

**Handling conflicts** (two devices edit the same file):

Use **optimistic concurrency control** — fancy name for a simple idea:

1. Every file has a version number
2. When uploading, send: `If-Match: version-7`
3. Server checks: "Is version 7 still the latest?"
   - ✅ Yes → accept upload, increment to version 8
   - ❌ No (someone else uploaded version 8 first) → `409 Conflict`
4. Client fetches the latest version and asks the user: "Keep both files or replace?"

Google Drive literally shows a dialog: *"A file with this name already exists. Keep both or replace?"*

---

### Q: What about encryption for enterprise users?

**A:** If the interviewer throws this curveball, here's your answer:

- Encrypt files locally using **AES-256** before writing to disk
- Store encryption keys in **Android Keystore** / **iOS Keychain** (hardware-backed, can't be extracted)
- Derive the key from user credentials using **PBKDF2** (so only authenticated users can decrypt)

Even if someone steals the phone and mounts the storage, the files are gibberish without the key. 🔐

---

## 🎬 PART 2: YOUTUBE APP

---

### Q: "Design the YouTube mobile app." This feels huge.

**A:** It IS huge. YouTube has 2.5 billion active users. But the interviewer doesn't want you to design ALL of YouTube. Focus on:

- ✅ Browse a home feed of recommended videos
- ✅ Watch videos with a feature-rich player (play/pause/seek, quality, subtitles)
- ✅ Prefetch video recommendations for instant loading
- ❌ Out of scope: Uploads, comments, live streaming, subscriptions

Non-functional: Design for 1 billion DAU. Yeah, a BILLION. 😳

---

### Q: How does video streaming actually work on mobile?

**A:** This is the meat of the YouTube interview. Here's the "aha" moment most people miss:

**You don't download the whole video and then play it.** That'd take forever. Instead:

1. Video is pre-split into tiny **segments** (2-10 seconds each)
2. Each segment exists in multiple **quality levels** (360p, 720p, 1080p, 4K)
3. The player downloads segments one at a time
4. Based on your internet speed, it picks the best quality for each segment
5. Slow Wi-Fi? Next segment loads in 480p. Fast LTE? Jumps to 1080p.

This is called **Adaptive Bitrate Streaming (ABR)** and it's powered by protocols like:

| Protocol | Who uses it | Key trait |
|----------|-------------|-----------|
| **HLS** (HTTP Live Streaming) | Apple, most mobile apps | Works everywhere, especially good on iOS |
| **DASH** (Dynamic Adaptive Streaming) | YouTube, Netflix | Open standard, codec-flexible |
| **CMAF** | Modern services | Bridges HLS and DASH — one format for both |

**The player is smart.** It constantly monitors:
- Download speed (can I get the next segment fast enough?)
- Buffer level (how many seconds of video are pre-loaded?)
- Battery level (should I drop quality to save power?)
- Device capabilities (this phone can't even decode 4K, so don't bother)

For Android: use **ExoPlayer** (now Media3). For iOS: use **AVPlayer**. These handle all the ABR magic natively.

---

### Q: How do subtitles and audio tracks work?

**A:** Here's a cool detail: **video, audio, and subtitles are separate streams**.

The video file doesn't contain the subtitles baked in. Instead:
- The streaming manifest (`.m3u8` for HLS, `.mpd` for DASH) lists available subtitle tracks
- Subtitles are in **WebVTT** format (basically a text file with timestamps)
- The player downloads and overlays them on the video

Why separate?
- Adding a new language doesn't require re-encoding the video
- Subtitles use almost zero bandwidth (they're just text!)
- You can update captions without re-processing the entire video

Same for audio — you can have English, Spanish, and Japanese audio tracks. The user picks one, and the player switches streams seamlessly.

---

### Q: What about those thumbnail previews when you scrub the seek bar?

**A:** Two approaches:

1. **Sprite sheets**: Pre-generate a big image with 100+ tiny thumbnails stitched together. When user scrubs to 2:35, show the thumbnail at position (row 4, col 7). Works, but requires storing extra files.

2. **Low-res adaptive streaming** (what YouTube actually does): When you scrub to 2:35, the app requests that timestamp's video segment but at 144p. It's tiny, loads instantly, and shows actual motion — not a static image. Way cooler.

YouTube uses approach #2 because:
- It reuses existing streaming infrastructure (no extra thumbnail server)
- Shows actual motion, giving better context
- No extra storage for sprite sheets

---

### Q: Tell me about prefetching — how does the feed load instantly?

**A:** When you open YouTube, the feed is already there. How? **Intelligent prefetching.**

While you're NOT using the app, it sneaks in and pre-loads your personalized feed in the background:

```
Is user likely to open app soon? (based on usage patterns)
  AND is device on Wi-Fi or charging?
  AND is battery > 20%?
    → YES → Background-fetch recommendations from server
    → Cache them locally
    → When user opens app → 💨 instant feed
```

The backend helps by:
- Assigning each recommendation a `relevanceScore` (0-100)
- Setting a `ttl` (time-to-live) so stale recommendations get refreshed
- Sending a lightweight config that tells the client how many items to prefetch and how often

On Android: **WorkManager** handles the scheduling.
On iOS: **Background Tasks framework** with `earliestBeginDate`.

Both platforms may delay or batch your background work — your code needs to handle that gracefully.

---

### Q: Any bonus deep dives for YouTube?

**A:**

**Background audio playback** — When you minimize the app, video stops but audio keeps playing. The app switches to an audio-only stream (way less data). Think podcast mode.
- Android: Foreground Service + Media3 APIs
- iOS: AVAudioSession with proper category

**Picture-in-Picture (PiP)** — That little floating video window while you check your messages. Both platforms support it natively. Reduce video quality in PiP to save battery.

**Battery optimization** — Video is the #1 battery killer. Lower quality when battery is low. Use hardware decoding (GPU) instead of software (CPU). Pause everything when the app is backgrounded.

---

## 📚 PART 3: PAGINATION LIBRARY

---

### Q: "Design a pagination library." Wait... a LIBRARY? Not an app?

**A:** Yep! This is a curveball question and it tests something different. When you design an app, your "users" are end-users. When you design a library, your "users" are **developers**.

The question is really: *"Can you design a great API that other programmers will love using?"*

Key mindset shift:
- App design → "How does the user interact with screens?"
- Library design → "How does a developer interact with your code?"

---

### Q: What should this pagination library do?

**A:** Scope:
- ✅ Works with any data type (posts, messages, products — anything)
- ✅ Supports offset-based AND cursor-based pagination
- ✅ Works with remote AND local data sources
- ✅ Built-in caching (in-memory + optional disk)
- ✅ Prefetching (auto-load the next page before user needs it)
- ✅ Open-source friendly (clean docs, stable API)
- ❌ Out of scope: UI components, fast-scrolling

---

### Q: What's the core API design?

**A:** The hero of the library is `Paginator<T>` — generic, so it works with ANY type:

```kotlin
interface Paginator<T> {
    fun fetch(key: String, pageSize: Int): PaginatorPage<T>
    fun fetchAround(key: String, pageSize: Int, depth: Int, direction: Direction)
    fun clear(key: String)
}
```

That `<T>` is the magic. Instead of:
- `PostPaginator` for posts
- `MessagePaginator` for messages
- `ProductPaginator` for products

You write ONE paginator that handles ALL of them. Less code, fewer bugs, happy developers.

The `key` is intentionally flexible:
- For offset pagination: key = `"page-2"` or `"page-3"`
- For cursor pagination: key = `"cursor-abc123"`

Same interface, different strategies. Developers don't need to learn a new API for each approach.

---

### Q: How does the caching work?

**A:** Dual-layer, like a well-organized kitchen:

```
Request comes in for page "cursor-xyz"
  ↓
1. Check IN-MEMORY cache (HashMap) → Found? Return instantly! ⚡
  ↓ (miss)
2. Check DISK cache (optional) → Found? Promote to memory + return 🏎️
  ↓ (miss)
3. Fetch from data source (API call) → Validate → Save to both caches → Return 📡
```

Each cached page has a state:
- `Loading` — we're fetching it right now
- `Success(data)` — here's your data!
- `Error(message)` — something went wrong

**Eviction policies** (because you can't cache forever):
- **LRU** (Least Recently Used) — evict the page nobody's looked at lately
- **FIFO** (First In, First Out) — evict the oldest page
- **TTL** (Time To Live) — evict pages older than X minutes

Developers configure this when creating a Paginator instance:

```kotlin
val paginator = CallbackPaginator<Post>(
    dataSource = myApiDataSource,
    validator = myPageValidator,
    store = myDiskStore,            // optional
    config = PaginatorConfig(
        inMemoryEvictionStrategy = LRU(maxPages = 50),
        maxParallelCalls = 3
    )
)
```

---

### Q: Why callbacks instead of coroutines/Combine?

**A:** Because you're building an **open-source library** that every developer should be able to use. Not everyone uses coroutines. Some projects still use RxJava. Some use LiveData. iOS devs might use Combine or async/await.

If you bake in coroutines, you force EVERY consumer to add a coroutines dependency. That's rude.

**The solution: Modular design**

```
paginator-core         → Pure callbacks, zero dependencies ✅
paginator-coroutines   → Kotlin Flow/suspend extensions (optional)
paginator-rx           → RxJava extensions (optional)
paginator-combine      → Swift Combine extensions (optional)
```

Core stays lightweight. Extensions add syntactic sugar for specific frameworks. Developers include only what they need.

This is what Google's Paging 3 library does — it supports LiveData, RxJava, AND Flow through separate artifacts.

---

### Q: What about testing support?

**A:** A library without testing support is like a car without a spare tire. Ship a separate **testing artifact**:

```
paginator-testing → Pre-built test doubles, fake data sources, error simulators
```

This gives developers:
- `FakePaginatorDataSource` — returns predefined pages without network calls
- `FakePageValidator` — always passes or always fails (configurable)
- Helper functions to generate sample `PaginatorPage` objects
- Error simulation (network timeout, corrupted data)

Developers love libraries that make testing easy. It's a huge adoption driver.

---

### Q: How do you version an open-source library?

**A:** **Semantic Versioning (SemVer)**: `MAJOR.MINOR.PATCH`

| Version bump | When | Example |
|-------------|------|---------|
| **MAJOR** (1.0 → 2.0) | Breaking change | Changed the Paginator interface |
| **MINOR** (1.0 → 1.1) | New feature, backwards compatible | Added prefetch priority support |
| **PATCH** (1.0.0 → 1.0.1) | Bug fix | Fixed a race condition in cache |

Why it matters: Both Gradle (Android) and Swift Package Manager (iOS) rely on SemVer for dependency resolution. Use weird versioning and their tools can't auto-resolve conflicts. Nobody will use your library.

---

*Next up: [05 — The Building Blocks Cheat Sheet →](./05_Building_Blocks_Cheat_Sheet.md)*
