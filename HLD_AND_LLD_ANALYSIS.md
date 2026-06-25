# HLD & LLD Analysis — 8 Core Systems
> Bridging the gap between "big picture architecture" and "actual implementation."
> All backend examples use **Node.js / NestJS**. Frontend examples use **React / Next.js**.

---

## How to Read This Document

```
HLD = What components exist and how they connect (boxes and arrows)
LLD = What lives INSIDE each box (schemas, classes, methods, algorithms)

Read HLD first → understand the map
Read LLD next  → understand how each city on the map actually works
```

---

## Table of Contents
1. [REST API with Authentication](#1-rest-api-with-authentication)
2. [Real-Time Chat System](#2-real-time-chat-system)
3. [URL Shortener](#3-url-shortener)
4. [File Upload Service](#4-file-upload-service)
5. [Social Feed System](#5-social-feed-system)
6. [Video Streaming Service](#6-video-streaming-service)
7. [Dating App](#7-dating-app)
8. [AI App with LLM Provider](#8-ai-app-with-llm-provider)

---

## 1. REST API with Authentication

> The system every web developer builds first. Understanding its HLD and LLD gives you the vocabulary for every other system.

---

### HLD

```
[React/Next.js Frontend]
         │
         │ HTTPS requests
         ▼
[Load Balancer]  ← distributes traffic across multiple API instances
         │
[NestJS API Servers]  ← stateless (can scale horizontally)
    │         │
    ▼         ▼
[PostgreSQL]  [Redis]
(users,       (sessions,
 data)         rate limiting)
```

**Key decisions:**
- API servers are **stateless** — no user data stored on the server itself. Stateless = any server can handle any request = easy to scale.
- **Redis** holds sessions so all servers share the same session state.
- **PostgreSQL** is the source of truth for all persistent data.

---

### LLD

#### Database Schema
```sql
-- Users table
CREATE TABLE users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email       VARCHAR(255) UNIQUE NOT NULL,
  password    VARCHAR(255) NOT NULL,  -- bcrypt hash, NEVER plain text
  created_at  TIMESTAMP DEFAULT NOW()
);

-- Sessions table (alternative: store in Redis instead)
CREATE TABLE sessions (
  id          UUID PRIMARY KEY,
  user_id     UUID REFERENCES users(id),
  expires_at  TIMESTAMP NOT NULL
);
```

#### NestJS Module Structure
```
src/
  auth/
    auth.module.ts        ← wires everything together
    auth.controller.ts    ← handles HTTP routes
    auth.service.ts       ← business logic
    auth.guard.ts         ← protects routes
    dto/
      login.dto.ts        ← validates request body shape
      register.dto.ts
  users/
    users.module.ts
    users.service.ts      ← DB queries for users
    users.controller.ts
```

#### Auth Service (the important logic)
```typescript
// auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async register(email: string, password: string) {
    // 1. Hash password — NEVER store plain text
    const hashed = await bcrypt.hash(password, 12);

    // 2. Save to DB
    const user = await this.usersService.create({ email, password: hashed });

    // 3. Return JWT token
    return { token: this.jwtService.sign({ sub: user.id, email }) };
  }

  async login(email: string, password: string) {
    const user = await this.usersService.findByEmail(email);
    if (!user) throw new UnauthorizedException('Invalid credentials');

    // Compare input password against stored hash
    const valid = await bcrypt.compare(password, user.password);
    if (!valid) throw new UnauthorizedException('Invalid credentials');

    return { token: this.jwtService.sign({ sub: user.id, email }) };
  }
}
```

#### JWT Flow
```
Login:
  Client sends: { email, password }
  Server verifies → signs JWT: { sub: userId, email, exp: 7days }
  Client stores token in localStorage or httpOnly cookie

Protected Request:
  Client sends: Authorization: Bearer <token>
  auth.guard.ts intercepts → verifies JWT signature
  If valid → extracts userId → attaches to request object
  Controller receives request with req.user.id already populated

Why JWT?
  Token is self-contained — server doesn't need DB lookup per request
  Any server can verify it (just needs the secret key)
  → Enables truly stateless, horizontally scalable APIs
```

#### Rate Limiting in Redis
```typescript
// Simple rate limiter: max 100 requests per IP per minute
async checkRateLimit(ip: string): Promise<boolean> {
  const key = `rate:${ip}`;
  const count = await redis.incr(key);      // increment counter
  if (count === 1) await redis.expire(key, 60); // set 60s TTL on first request
  return count <= 100;
}
```

---

## 2. Real-Time Chat System

> Introduces persistent connections, event-driven architecture, and the difference between storing data vs routing data.

---

### HLD

```
[React Frontend]          [React Frontend]
  Alice's browser           Bob's browser
       │                         │
       │ WebSocket               │ WebSocket
       ▼                         ▼
[Load Balancer]           [Load Balancer]
       │                         │
[NestJS Server A]         [NestJS Server B]
       │                         │
       └─────────[Redis Pub/Sub]─┘
                      │
               [PostgreSQL]
              (message history)
```

**The core problem:** Alice connects to Server A. Bob connects to Server B. How does Alice's message reach Bob?

**Solution:** Redis Pub/Sub — servers publish messages to Redis, all other servers subscribed to that channel receive them instantly.

---

### LLD

#### Database Schema
```sql
CREATE TABLE conversations (
  id         UUID PRIMARY KEY,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE conversation_members (
  conversation_id UUID REFERENCES conversations(id),
  user_id         UUID REFERENCES users(id),
  PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID REFERENCES conversations(id),
  sender_id       UUID REFERENCES users(id),
  content         TEXT NOT NULL,
  created_at      TIMESTAMP DEFAULT NOW()
);

-- Index for fast message history lookup
CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at DESC);
```

#### NestJS WebSocket Gateway
```typescript
// chat.gateway.ts
@WebSocketGateway({ cors: true })
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;

  // Map: userId → socket (so we can push to specific users)
  private connectedUsers = new Map<string, Socket>();

  handleConnection(socket: Socket) {
    const userId = socket.handshake.auth.userId;
    this.connectedUsers.set(userId, socket);

    // Subscribe to Redis channel for this user
    this.redisSubscriber.subscribe(`user:${userId}`, (message) => {
      socket.emit('new_message', JSON.parse(message));
    });
  }

  handleDisconnect(socket: Socket) {
    const userId = socket.handshake.auth.userId;
    this.connectedUsers.delete(userId);
  }

  @SubscribeMessage('send_message')
  async handleMessage(socket: Socket, payload: { conversationId: string; content: string }) {
    const senderId = socket.handshake.auth.userId;

    // 1. Save to PostgreSQL (durable storage)
    const message = await this.messagesService.create({
      conversationId: payload.conversationId,
      senderId,
      content: payload.content,
    });

    // 2. Find all members of this conversation
    const members = await this.conversationsService.getMembers(payload.conversationId);

    // 3. Publish to Redis for each recipient
    // (their server — whichever it is — will receive and push to their socket)
    for (const memberId of members) {
      if (memberId !== senderId) {
        await this.redisPublisher.publish(`user:${memberId}`, JSON.stringify(message));
      }
    }
  }
}
```

#### Redis Pub/Sub — Why It Solves the Multi-Server Problem
```
Without Redis Pub/Sub:
  Alice (Server A) sends message
  Server A looks for Bob's socket → not here (he's on Server B) → message lost ❌

With Redis Pub/Sub:
  Alice (Server A) sends message
  Server A publishes to Redis channel "user:bob"
  Server B is subscribed to "user:bob" → receives it → pushes to Bob's socket ✅

Redis here is a MESSAGE ROUTER, not a database.
Messages are not stored in Redis — just passed through.
PostgreSQL stores them permanently.
```

#### React Frontend Hook
```typescript
// useChatSocket.ts
export function useChatSocket(conversationId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const socketRef = useRef<Socket>();

  useEffect(() => {
    // Connect WebSocket on mount
    socketRef.current = io('http://api.yourapp.com', {
      auth: { userId: getCurrentUserId() }
    });

    // Listen for incoming messages
    socketRef.current.on('new_message', (message: Message) => {
      setMessages(prev => [...prev, message]);
    });

    return () => socketRef.current?.disconnect(); // cleanup on unmount
  }, []);

  const sendMessage = (content: string) => {
    socketRef.current?.emit('send_message', { conversationId, content });
  };

  return { messages, sendMessage };
}
```

---

## 3. URL Shortener

> Deceptively simple. The LLD reveals interesting tradeoffs around encoding, collisions, and caching that appear in many other systems.

---

### HLD

```
[Next.js Frontend]  ← paste long URL, get short URL
       │
[NestJS API]
    │        │
    ▼        ▼
[PostgreSQL] [Redis]
(url mappings) (cache: short_code → long_url)
       │
[Next.js Redirect Handler]
  (handles /:code → redirects)
```

**Two separate flows:**
1. **Create**: user submits long URL → generate short code → store → return short URL
2. **Redirect**: user visits short URL → look up in Redis (or DB) → 301/302 redirect

---

### LLD

#### Database Schema
```sql
CREATE TABLE urls (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  short_code  VARCHAR(10) UNIQUE NOT NULL,
  long_url    TEXT NOT NULL,
  user_id     UUID REFERENCES users(id),  -- nullable for anonymous
  clicks      INTEGER DEFAULT 0,
  expires_at  TIMESTAMP,                  -- nullable = never expires
  created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_urls_short_code ON urls(short_code);  -- fast lookup
```

#### Base62 Encoding (the core algorithm)
```typescript
// Why Base62? Uses [a-z A-Z 0-9] = 62 chars
// 6 characters of Base62 = 62^6 = 56 billion unique codes
// More than enough, URL-safe (no special characters)

const CHARS = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';

function encodeBase62(num: number): string {
  let result = '';
  while (num > 0) {
    result = CHARS[num % 62] + result;
    num = Math.floor(num / 62);
  }
  return result.padStart(6, 'a'); // always 6 chars
}

// Alternative: just use nanoid(6) — generates random URL-safe 6-char string
// Simpler, no need to track auto-increment IDs
import { nanoid } from 'nanoid';
const shortCode = nanoid(6); // e.g. "V1StGX"
```

#### URL Service
```typescript
// urls.service.ts
@Injectable()
export class UrlsService {
  async shorten(longUrl: string): Promise<string> {
    // 1. Generate short code
    const shortCode = nanoid(6);

    // 2. Check collision (rare but possible)
    const existing = await this.db.urls.findOne({ shortCode });
    if (existing) return this.shorten(longUrl); // retry with new code

    // 3. Save to DB
    await this.db.urls.create({ shortCode, longUrl });

    // 4. Cache in Redis immediately (will be hot on first use)
    await this.redis.setex(`url:${shortCode}`, 86400, longUrl); // 24h TTL

    return `https://short.ly/${shortCode}`;
  }

  async resolve(shortCode: string): Promise<string> {
    // 1. Check Redis cache first (fast path)
    const cached = await this.redis.get(`url:${shortCode}`);
    if (cached) return cached;

    // 2. Cache miss → hit PostgreSQL
    const url = await this.db.urls.findOne({ shortCode });
    if (!url) throw new NotFoundException('Short URL not found');

    // 3. Check expiry
    if (url.expiresAt && url.expiresAt < new Date()) {
      throw new GoneException('This short URL has expired');
    }

    // 4. Re-cache for next request
    await this.redis.setex(`url:${shortCode}`, 86400, url.longUrl);

    // 5. Increment click counter (async, don't await — don't slow down redirect)
    this.db.urls.increment({ shortCode }, 'clicks', 1);

    return url.longUrl;
  }
}
```

#### 301 vs 302 — The Important Interview Distinction
```
301 Permanent Redirect:
  Browser caches it forever
  Future visits to short.ly/abc → browser redirects WITHOUT hitting your server
  ✅ Less load on your servers
  ❌ You can't track clicks or update the destination later

302 Temporary Redirect:
  Browser does NOT cache it
  Every visit hits your server
  ✅ You can track clicks, change destination, expire URLs
  ❌ More server load

For a URL shortener: use 302
  → You want analytics (clicks matter)
  → You want URLs to be deletable/expirable
```

---

## 4. File Upload Service

> Introduces chunked uploads, streaming, and the pattern of separating metadata from binary data — used in Google Drive, WhatsApp media, Instagram photos.

---

### HLD

```
[React Frontend]
  │  1. Request presigned URL from API
  │  2. Upload directly to S3 (bypasses your servers)
  │  3. Notify API that upload is complete
  ▼
[NestJS API]
  │  - generates presigned S3 URLs
  │  - saves file metadata to DB
  │  - triggers post-processing (resize, transcode)
  ▼
[PostgreSQL]      [S3]              [Redis]
(file metadata)   (actual files)    (upload progress)
                       │
                  [Kafka] ← "file_uploaded" event
                       │
              [Worker Service]
              (resize images,
               generate thumbnails,
               scan for viruses)
```

**Key insight:** Your API servers never touch the file bytes. S3 handles the upload directly from the browser. Your servers only handle lightweight metadata.

---

### LLD

#### Database Schema
```sql
CREATE TABLE files (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id),
  original_name VARCHAR(500) NOT NULL,
  storage_key   VARCHAR(500) NOT NULL,  -- S3 key: "uploads/userId/uuid.jpg"
  mime_type     VARCHAR(100),
  size_bytes    BIGINT,
  status        VARCHAR(20) DEFAULT 'uploading',  -- uploading | ready | failed
  created_at    TIMESTAMP DEFAULT NOW()
);
```

#### Presigned URL Flow
```typescript
// files.service.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

@Injectable()
export class FilesService {
  async getPresignedUploadUrl(userId: string, filename: string, mimeType: string) {
    // 1. Generate unique storage key
    const storageKey = `uploads/${userId}/${randomUUID()}-${filename}`;

    // 2. Create S3 presigned URL (expires in 15 minutes)
    const command = new PutObjectCommand({
      Bucket: process.env.S3_BUCKET,
      Key: storageKey,
      ContentType: mimeType,
    });
    const uploadUrl = await getSignedUrl(this.s3Client, command, { expiresIn: 900 });

    // 3. Save pending record to DB
    const file = await this.db.files.create({
      userId, originalName: filename, storageKey, mimeType, status: 'uploading'
    });

    return { uploadUrl, fileId: file.id, storageKey };
    // Frontend uploads directly to uploadUrl (PUT request to S3)
    // S3 never touches your server — saves bandwidth and cost
  }

  async confirmUpload(fileId: string, userId: string) {
    // Called by frontend after S3 upload completes
    const file = await this.db.files.findOne({ id: fileId, userId });

    // Verify file actually exists in S3
    const metadata = await this.s3Client.headObject({
      Bucket: process.env.S3_BUCKET, Key: file.storageKey
    });

    // Update DB record
    await this.db.files.update(fileId, {
      status: 'ready',
      sizeBytes: metadata.ContentLength,
    });

    // Trigger async processing (don't make user wait for this)
    await this.kafka.emit('file_uploaded', { fileId, storageKey: file.storageKey });

    return { fileId, url: `https://cdn.yourapp.com/${file.storageKey}` };
  }
}
```

#### React Frontend Upload
```typescript
// useFileUpload.ts
export function useFileUpload() {
  const [progress, setProgress] = useState(0);

  const upload = async (file: File) => {
    // Step 1: Get presigned URL from your API
    const { uploadUrl, fileId } = await api.post('/files/presigned-url', {
      filename: file.name,
      mimeType: file.type,
    });

    // Step 2: Upload directly to S3 (NOT through your API)
    await axios.put(uploadUrl, file, {
      headers: { 'Content-Type': file.type },
      onUploadProgress: (e) => {
        setProgress(Math.round((e.loaded / e.total) * 100));
      },
    });

    // Step 3: Tell your API the upload completed
    const result = await api.post(`/files/${fileId}/confirm`);
    return result.data; // { fileId, url }
  };

  return { upload, progress };
}
```

#### Chunked Upload for Large Files
```typescript
// For files > 100 MB — split into 10 MB chunks and upload in parallel

async function uploadLargeFile(file: File) {
  const CHUNK_SIZE = 10 * 1024 * 1024; // 10 MB
  const chunks = Math.ceil(file.size / CHUNK_SIZE);

  // Get multipart upload ID from S3
  const { uploadId } = await api.post('/files/multipart/start', { filename: file.name });

  // Upload all chunks in parallel (max 5 at a time)
  const uploadedParts = await Promise.all(
    Array.from({ length: chunks }, async (_, i) => {
      const start = i * CHUNK_SIZE;
      const chunk = file.slice(start, start + CHUNK_SIZE);

      const { presignedUrl, partNumber } = await api.post('/files/multipart/part', {
        uploadId, partNumber: i + 1
      });

      const response = await axios.put(presignedUrl, chunk);
      return { partNumber, etag: response.headers.etag };
    })
  );

  // Tell S3 all parts are done — S3 assembles them
  await api.post('/files/multipart/complete', { uploadId, parts: uploadedParts });
}
```

---

## 5. Social Feed System

> The most architecturally complex of the five. Introduces the fanout problem, pre-computation vs real-time tradeoffs, and the celebrity problem — concepts that appear in every social platform design.

---

### HLD

```
[Next.js Frontend]
       │
[NestJS API Gateway]  ← routes to specific microservices
    │        │        │          │
    ▼        ▼        ▼          ▼
[Post     [Feed    [Follow   [Media
 Service]  Service] Service]  Service]
    │        │                   │
    ▼        ▼                   ▼
[Cassandra] [Redis]           [S3+CDN]
(posts,     (pre-computed      (images,
 comments)   feeds per user)    videos)
    │
  [Kafka]
"post_created" events
    │
[Feed Fanout Workers]
→ write post_id to followers' Redis feeds
```

---

### LLD

#### Database Schema
```sql
-- PostgreSQL: relational data (users, follows)
CREATE TABLE users (
  id       UUID PRIMARY KEY,
  username VARCHAR(50) UNIQUE,
  bio      TEXT,
  avatar   VARCHAR(500)  -- S3 URL
);

CREATE TABLE follows (
  follower_id UUID REFERENCES users(id),
  followee_id UUID REFERENCES users(id),
  created_at  TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_followee ON follows(followee_id);
```

```typescript
// Cassandra schema for posts (high write volume, time-ordered)
// Defined as CQL (Cassandra Query Language)

CREATE TABLE posts (
  user_id    UUID,
  created_at TIMESTAMP,
  post_id    UUID,
  content    TEXT,
  image_url  TEXT,
  like_count INT,
  PRIMARY KEY (user_id, created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
// Partition key: user_id (all posts by a user on same node)
// Sort key: created_at DESC (newest posts first automatically)
```

#### Post Service — Creating a Post
```typescript
// post.service.ts
@Injectable()
export class PostService {
  async createPost(userId: string, content: string, imageUrl?: string) {
    // 1. Save post to Cassandra
    const post = await this.cassandra.execute(`
      INSERT INTO posts (user_id, created_at, post_id, content, image_url)
      VALUES (?, ?, uuid(), ?, ?)
    `, [userId, new Date(), content, imageUrl]);

    // 2. Publish event to Kafka — fanout workers handle the rest
    await this.kafka.emit('post_created', {
      postId: post.id,
      userId,
      createdAt: new Date(),
    });

    return post;
  }
}
```

#### Feed Fanout Worker — The Core Problem
```typescript
// feed-fanout.worker.ts
// This runs as a separate NestJS microservice consuming Kafka events

@EventPattern('post_created')
async handlePostCreated(event: { postId: string; userId: string }) {
  // Get follower count to decide fanout strategy
  const followerCount = await this.followService.getFollowerCount(event.userId);

  if (followerCount < 1_000_000) {
    // Regular user: FANOUT ON WRITE
    // Pre-compute — push post_id to every follower's feed NOW
    const followers = await this.followService.getFollowers(event.userId);

    // Process in batches of 1000 to avoid memory issues
    for (const batch of chunk(followers, 1000)) {
      await Promise.all(batch.map(followerId =>
        // Redis list: push post_id to front of follower's feed
        this.redis.lpush(`feed:${followerId}`, event.postId)
      ));
    }
  }
  // Celebrity (> 1M followers): FANOUT ON READ
  // Don't pre-compute — too expensive
  // Their posts are fetched and merged when followers open their feed
}
```

#### Feed Service — Reading the Feed
```typescript
// feed.service.ts
@Injectable()
export class FeedService {
  async getFeed(userId: string, page: number, limit = 20): Promise<Post[]> {
    const offset = page * limit;

    // 1. Get pre-computed feed from Redis (post IDs only)
    const postIds = await this.redis.lrange(`feed:${userId}`, offset, offset + limit - 1);

    // 2. Fetch actual post data from Cassandra
    const regularPosts = await this.postService.getPostsByIds(postIds);

    // 3. Get celebrities this user follows (< 5 typically)
    const celebrities = await this.followService.getCelebrityFollowees(userId);

    // 4. Pull latest posts from celebrities in real-time
    const celebrityPosts = await Promise.all(
      celebrities.map(c => this.postService.getLatestPosts(c.id, 5))
    );

    // 5. Merge and sort all posts by time
    const allPosts = [...regularPosts, ...celebrityPosts.flat()];
    return allPosts.sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
  }
}
```

#### Like Counter — Redis Atomic Increment
```typescript
// Likes don't go to DB on every click — too many writes
// Instead: Redis counter, batch flush to DB every 5 minutes

async likePost(postId: string, userId: string) {
  // Track who liked (for deduplication)
  const alreadyLiked = await this.redis.sismember(`likes:${postId}`, userId);
  if (alreadyLiked) return; // idempotent — clicking like twice does nothing

  await this.redis.sadd(`likes:${postId}`, userId);
  await this.redis.incr(`likecount:${postId}`);  // atomic increment — thread safe
}

// Cron job runs every 5 minutes
@Cron('*/5 * * * *')
async flushLikeCountsToDB() {
  const keys = await this.redis.keys('likecount:*');
  for (const key of keys) {
    const postId = key.replace('likecount:', '');
    const count = await this.redis.get(key);
    await this.cassandra.execute(
      'UPDATE posts SET like_count = ? WHERE post_id = ?',
      [parseInt(count), postId]
    );
    await this.redis.del(key); // clear after flush
  }
}
```

#### Next.js Infinite Scroll Feed
```typescript
// pages/feed.tsx
export default function FeedPage() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [page, setPage] = useState(0);
  const loaderRef = useRef(null);

  // Intersection Observer — triggers when bottom of list is visible
  useEffect(() => {
    const observer = new IntersectionObserver(async ([entry]) => {
      if (entry.isIntersecting) {
        const newPosts = await api.get(`/feed?page=${page}&limit=20`);
        setPosts(prev => [...prev, ...newPosts.data]);
        setPage(p => p + 1);
      }
    });
    if (loaderRef.current) observer.observe(loaderRef.current);
    return () => observer.disconnect();
  }, [page]);

  return (
    <div>
      {posts.map(post => <PostCard key={post.id} post={post} />)}
      <div ref={loaderRef} /> {/* invisible trigger at bottom */}
    </div>
  );
}
```

---

---

## 6. Video Streaming Service

> Introduces the most complex media pipeline — upload, transcode, chunk, stream. The gap between "user uploads a video" and "someone watches it in HD" is enormous.

---

### HLD

```
[Next.js Frontend - Upload]          [Next.js Frontend - Watch]
        │                                      │
        │ 1. Presigned URL                     │ HLS playlist request
        │ 2. Upload raw video to S3            │
        ▼                                      ▼
[NestJS API]                          [CDN (CloudFront)]
        │                                      │
        │ "video_uploaded" event               │ cache miss
        ▼                                      ▼
     [Kafka]                              [S3 — chunked HLS files]
        │                                 (.m3u8 playlist + .ts chunks)
        ▼
[Transcoding Workers] ← pull jobs from Kafka
        │
        │ ffmpeg converts raw video into:
        │   360p  → chunks → S3
        │   720p  → chunks → S3
        │   1080p → chunks → S3
        │   thumbnail → S3
        ▼
[PostgreSQL] ← update video status: "processing" → "ready"
```

**The key insight:** Users never stream raw uploaded video. Raw video is transcoded into HLS (HTTP Live Streaming) format — split into 2–10 second `.ts` chunks at multiple quality levels. The player downloads chunks one at a time and switches quality based on internet speed (Adaptive Bitrate Streaming).

---

### LLD

#### Database Schema
```sql
CREATE TABLE videos (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id),
  title         VARCHAR(500) NOT NULL,
  description   TEXT,
  status        VARCHAR(20) DEFAULT 'uploading',
  -- uploading → processing → ready → failed
  raw_s3_key    VARCHAR(500),   -- original upload
  duration_sec  INTEGER,
  thumbnail_url VARCHAR(500),
  view_count    BIGINT DEFAULT 0,
  created_at    TIMESTAMP DEFAULT NOW()
);

CREATE TABLE video_qualities (
  video_id      UUID REFERENCES videos(id),
  quality       VARCHAR(10) NOT NULL,  -- '360p', '720p', '1080p'
  playlist_url  VARCHAR(500),          -- S3 URL of .m3u8 playlist file
  size_bytes    BIGINT,
  PRIMARY KEY (video_id, quality)
);
```

#### Upload Flow — NestJS
```typescript
// videos.service.ts
@Injectable()
export class VideosService {
  async initiateUpload(userId: string, title: string, mimeType: string) {
    // 1. Create DB record in 'uploading' status
    const video = await this.db.videos.create({ userId, title, status: 'uploading' });

    // 2. Generate presigned S3 URL for raw video upload
    const rawKey = `raw/${userId}/${video.id}.mp4`;
    const uploadUrl = await this.s3.getPresignedUrl(rawKey, mimeType);

    return { videoId: video.id, uploadUrl };
    // Frontend uploads raw video directly to S3 using this URL
  }

  async confirmUpload(videoId: string) {
    // Called by frontend after S3 upload finishes
    await this.db.videos.update(videoId, {
      status: 'processing',
      rawS3Key: `raw/${videoId}.mp4`
    });

    // Publish to Kafka — transcoding worker picks it up
    await this.kafka.emit('video_uploaded', { videoId });
  }
}
```

#### Transcoding Worker — The Heavy Lifting
```typescript
// transcoding.worker.ts (separate NestJS microservice)
@EventPattern('video_uploaded')
async handleVideoUploaded({ videoId }: { videoId: string }) {
  const video = await this.db.videos.findOne(videoId);

  // 1. Download raw video from S3 to local temp disk
  const rawPath = await this.s3.downloadToTemp(video.rawS3Key);

  // 2. Transcode to multiple qualities using ffmpeg
  const qualities = ['360p', '720p', '1080p'];

  for (const quality of qualities) {
    const outputDir = `/tmp/${videoId}/${quality}`;

    // ffmpeg command: convert to HLS format at given quality
    await this.ffmpeg.run([
      '-i', rawPath,
      '-vf', `scale=${this.getResolution(quality)}`,
      '-c:v', 'libx264', '-crf', '23',
      '-hls_time', '6',           // 6-second chunks
      '-hls_playlist_type', 'vod',
      '-hls_segment_filename', `${outputDir}/chunk_%03d.ts`,
      `${outputDir}/playlist.m3u8`
    ]);

    // 3. Upload all chunks + playlist to S3
    await this.s3.uploadDirectory(outputDir, `hls/${videoId}/${quality}/`);

    // 4. Save quality record to DB
    await this.db.videoQualities.create({
      videoId, quality,
      playlistUrl: `https://cdn.app.com/hls/${videoId}/${quality}/playlist.m3u8`
    });
  }

  // 5. Generate thumbnail from frame at 5 seconds
  await this.ffmpeg.extractThumbnail(rawPath, `thumbnails/${videoId}.jpg`);

  // 6. Mark video as ready
  await this.db.videos.update(videoId, { status: 'ready' });

  // 7. Clean up raw file (optional — save storage cost)
  await this.s3.delete(video.rawS3Key);
}

getResolution(quality: string): string {
  return { '360p': '-2:360', '720p': '-2:720', '1080p': '-2:1080' }[quality];
}
```

#### HLS Streaming — How the Player Works
```
S3 structure after transcoding:
  hls/video_123/360p/playlist.m3u8
  hls/video_123/360p/chunk_000.ts  (6 seconds of video)
  hls/video_123/360p/chunk_001.ts
  hls/video_123/360p/chunk_002.ts  ...
  hls/video_123/720p/playlist.m3u8
  hls/video_123/720p/chunk_000.ts  ...
  hls/video_123/1080p/...
  hls/video_123/master.m3u8        ← lists all quality options

Master playlist (master.m3u8):
  #EXTM3U
  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
  360p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720
  720p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
  1080p/playlist.m3u8

Player behaviour (Adaptive Bitrate Streaming):
  → Measures download speed of each chunk
  → Slow internet (1 Mbps)  → switches to 360p
  → Fast internet (10 Mbps) → switches to 1080p
  → Happens mid-video, seamlessly
```

#### React Video Player
```typescript
// VideoPlayer.tsx
import Hls from 'hls.js'; // npm install hls.js

export function VideoPlayer({ videoId }: { videoId: string }) {
  const videoRef = useRef<HTMLVideoElement>(null);

  useEffect(() => {
    const masterUrl = `https://cdn.app.com/hls/${videoId}/master.m3u8`;

    if (Hls.isSupported()) {
      // Chrome, Firefox — native HLS not supported, use hls.js
      const hls = new Hls();
      hls.loadSource(masterUrl);
      hls.attachMedia(videoRef.current!);
      return () => hls.destroy();
    } else if (videoRef.current?.canPlayType('application/vnd.apple.mpegurl')) {
      // Safari — native HLS support
      videoRef.current.src = masterUrl;
    }
  }, [videoId]);

  return <video ref={videoRef} controls style={{ width: '100%' }} />;
}
```

#### View Count — Avoiding Write Hotspot
```typescript
// Same Redis counter + batch flush pattern as likes
// But view counts fire on every play — much higher volume

async recordView(videoId: string) {
  // Don't write to DB per view — DB would collapse under YouTube-scale traffic
  await this.redis.incr(`views:${videoId}`);
}

// Cron: flush every 60 seconds
@Cron('*/1 * * * *')
async flushViewCounts() {
  const keys = await this.redis.keys('views:*');
  for (const key of keys) {
    const videoId = key.replace('views:', '');
    const count = parseInt(await this.redis.getdel(key));
    if (count > 0) {
      await this.db.videos.increment(videoId, 'viewCount', count);
    }
  }
}
```

---

## 7. Dating App

> Introduces geospatial queries, the matching/swipe problem, and how to handle a system where the entire UX depends on WHO sees WHO.

---

### HLD

```
[React Native / Next.js Frontend]
           │
    [NestJS API Gateway]
    │          │          │
    ▼          ▼          ▼
[Profile   [Discovery  [Match &
 Service]   Service]    Chat Service]
    │          │               │
    ▼          ▼               ▼
[PostgreSQL] [Redis GEO    [PostgreSQL]  [Redis Pub/Sub]
(profiles,   + Candidate    (matches,    (real-time chat)
 photos)      Cache]         messages)
    │
  [S3+CDN]
  (profile photos)
```

**Three hard problems:**
1. **Discovery** — show me people nearby who I haven't seen yet, who match my filters
2. **The Swipe** — if both people swipe right, create a match instantly (race condition)
3. **Privacy** — never reveal exact location, only approximate distance

---

### LLD

#### Database Schema
```sql
CREATE TABLE profiles (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID REFERENCES users(id) UNIQUE,
  name         VARCHAR(100),
  age          INTEGER,
  gender       VARCHAR(20),
  bio          TEXT,
  photos       TEXT[],        -- array of S3 URLs
  latitude     DECIMAL(10,8), -- NEVER exposed to other users directly
  longitude    DECIMAL(11,8),
  last_active  TIMESTAMP,
  min_age_pref INTEGER DEFAULT 18,
  max_age_pref INTEGER DEFAULT 99,
  gender_pref  VARCHAR(20) DEFAULT 'all',
  max_distance INTEGER DEFAULT 50  -- km
);

CREATE TABLE swipes (
  swiper_id   UUID REFERENCES profiles(id),
  swiped_id   UUID REFERENCES profiles(id),
  direction   VARCHAR(5) NOT NULL,  -- 'left' or 'right'
  created_at  TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (swiper_id, swiped_id)
);

CREATE TABLE matches (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_a_id  UUID REFERENCES profiles(id),
  user_b_id  UUID REFERENCES profiles(id),
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE (user_a_id, user_b_id)  -- prevent duplicate matches
);
```

#### Discovery Service — Finding Candidates
```typescript
// discovery.service.ts
@Injectable()
export class DiscoveryService {
  async getCandidates(userId: string, limit = 10): Promise<Profile[]> {
    const me = await this.profileService.getProfile(userId);

    // 1. Find nearby users using Redis GEO (approximate location only)
    const nearbyIds = await this.redis.georadius(
      'user_locations',
      me.longitude, me.latitude,
      me.maxDistance, 'km',
      'ASC',    // closest first
      'COUNT', 200  // get 200 candidates, we'll filter down
    );

    // 2. Filter out:
    //    - already swiped
    //    - doesn't match age/gender preferences
    //    - already matched
    const seenIds = await this.getAlreadySwiped(userId); // from PostgreSQL
    const unseenIds = nearbyIds.filter(id => !seenIds.has(id) && id !== userId);

    // 3. Apply preference filters
    const candidates = await this.profileService.getProfiles(unseenIds);
    const filtered = candidates.filter(p =>
      p.age >= me.minAgePref &&
      p.age <= me.maxAgePref &&
      (me.genderPref === 'all' || p.gender === me.genderPref)
    );

    // 4. Cache this candidate list in Redis (avoid recomputing on every swipe)
    await this.redis.setex(
      `candidates:${userId}`,
      300, // 5 minute TTL
      JSON.stringify(filtered.slice(0, 50).map(p => p.id))
    );

    return filtered.slice(0, limit);
  }

  async updateLocation(userId: string, lat: number, lng: number) {
    // Store approximate location in Redis GEO
    // Never store exact location — snap to ~500m grid for privacy
    const approxLat = Math.round(lat * 200) / 200;
    const approxLng = Math.round(lng * 200) / 200;

    await this.redis.geoadd('user_locations', approxLng, approxLat, userId);

    // Also update DB for persistence (Redis can restart)
    await this.db.profiles.update(userId, { latitude: approxLat, longitude: approxLng });
  }
}
```

#### Swipe Service — The Match Race Condition
```typescript
// swipe.service.ts
@Injectable()
export class SwipeService {
  async swipe(swiperId: string, swipedId: string, direction: 'left' | 'right') {
    // 1. Record the swipe
    await this.db.swipes.upsert({ swiperId, swipedId, direction });

    if (direction === 'left') return { match: false };

    // 2. Right swipe — check if other person already swiped right on us
    // Use Redis atomic check to prevent race condition:
    // Two people swipe right simultaneously → both think they're creating the match
    const mutualKey = `mutual:${[swiperId, swipedId].sort().join(':')}`;
    const alreadyLiked = await this.redis.getset(mutualKey, swiperId);
    // getset: atomically GET old value and SET new value
    // First right swipe: alreadyLiked = null → just store, no match yet
    // Second right swipe: alreadyLiked = other person's ID → MATCH!

    if (alreadyLiked && alreadyLiked !== swiperId) {
      // It's a match!
      const match = await this.createMatch(swiperId, swipedId);
      await this.redis.del(mutualKey); // cleanup

      // Notify both users via WebSocket
      this.notifyMatch(swiperId, swipedId, match.id);
      return { match: true, matchId: match.id };
    }

    return { match: false };
  }

  private async createMatch(userA: string, userB: string) {
    // Sort IDs so (A,B) and (B,A) always create same record
    const [idA, idB] = [userA, userB].sort();
    return this.db.matches.create({ userAId: idA, userBId: idB });
  }

  private notifyMatch(userA: string, userB: string, matchId: string) {
    // Push "it's a match!" event to both users' WebSocket connections
    this.redis.publish(`user:${userA}`, JSON.stringify({ type: 'MATCH', matchId }));
    this.redis.publish(`user:${userB}`, JSON.stringify({ type: 'MATCH', matchId }));
  }
}
```

#### Privacy — Never Expose Exact Location
```typescript
// When returning profile data to other users, compute distance label
// NEVER send raw lat/lng coordinates

function getDistanceLabel(myLat: number, myLng: number, theirLat: number, theirLng: number): string {
  const km = haversineDistance(myLat, myLng, theirLat, theirLng);

  if (km < 1) return 'Less than 1 km away';
  if (km < 10) return `${Math.round(km)} km away`;
  return `${Math.round(km / 10) * 10} km away`; // round to nearest 10 for privacy
}

// Profile response to other users:
// ✅ name, age, bio, photos, distance label
// ❌ exact latitude/longitude (never sent)
```

#### React Swipe Cards
```typescript
// SwipeCard.tsx — the iconic Tinder gesture
export function SwipeCard({ profile, onSwipe }: SwipeCardProps) {
  const [dragX, setDragX] = useState(0);

  const handleDragEnd = () => {
    if (dragX > 100) onSwipe('right');   // dragged right enough → like
    else if (dragX < -100) onSwipe('left'); // dragged left enough → pass
    else setDragX(0); // snap back to center
  };

  const rotation = dragX / 15; // tilt card as it's dragged
  const opacity = Math.abs(dragX) > 50
    ? dragX > 0 ? '💚' : '❌'  // show like/nope indicator
    : null;

  return (
    <div
      style={{ transform: `translateX(${dragX}px) rotate(${rotation}deg)` }}
      onMouseMove={(e) => setDragX(e.movementX + dragX)}
      onMouseUp={handleDragEnd}
    >
      <img src={profile.photos[0]} />
      <h2>{profile.name}, {profile.age}</h2>
      {opacity && <div className="indicator">{opacity}</div>}
    </div>
  );
}
```

---

## 8. AI App with LLM Provider

> Introduces the unique challenges of LLM-backed systems: streaming responses, token costs, prompt management, rate limits, and semantic caching.

---

### HLD

```
[Next.js Frontend]
  │  streams tokens as they arrive (SSE)
  ▼
[NestJS API]
  │         │            │
  ▼         ▼            ▼
[OpenAI   [Vector DB   [PostgreSQL]
 API]      (Pinecone)]   (conversations,
  │        (semantic      users, usage)
  │         cache +
  │         RAG docs)
  ▼
[Redis]
(rate limit per user,
 active session context)
```

**What makes LLM apps different:**
- Responses take 3–30 seconds — you must stream them word by word, not wait for the full response
- Every request costs money (tokens) — caching identical questions saves real $$$
- Context window is limited — you can't send the entire conversation history forever
- Rate limits from OpenAI — you need queuing and backoff

---

### LLD

#### Database Schema
```sql
CREATE TABLE conversations (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID REFERENCES users(id),
  title      VARCHAR(500),   -- auto-generated from first message
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID REFERENCES conversations(id),
  role            VARCHAR(20) NOT NULL,  -- 'user' or 'assistant'
  content         TEXT NOT NULL,
  token_count     INTEGER,               -- track cost per message
  created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE usage_stats (
  user_id        UUID REFERENCES users(id),
  date           DATE DEFAULT CURRENT_DATE,
  tokens_used    INTEGER DEFAULT 0,
  requests_made  INTEGER DEFAULT 0,
  PRIMARY KEY (user_id, date)
);
```

#### Chat Service — Streaming Response
```typescript
// chat.service.ts
@Injectable()
export class ChatService {
  async streamResponse(
    conversationId: string,
    userMessage: string,
    res: Response  // Express response object for SSE streaming
  ) {
    // 1. Check semantic cache — has someone asked the same thing?
    const cached = await this.semanticCache.lookup(userMessage);
    if (cached) {
      // Stream cached response instantly (free, no API call)
      res.write(`data: ${JSON.stringify({ content: cached, cached: true })}\n\n`);
      res.end();
      return;
    }

    // 2. Save user message to DB
    await this.db.messages.create({
      conversationId, role: 'user', content: userMessage
    });

    // 3. Build context window — last N messages (can't send entire history)
    const history = await this.buildContextWindow(conversationId, maxTokens: 3000);

    // 4. Call OpenAI with streaming enabled
    const stream = await this.openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [
        { role: 'system', content: 'You are a helpful assistant.' },
        ...history,
        { role: 'user', content: userMessage }
      ],
      stream: true,  // ← key: stream tokens as they're generated
    });

    // 5. Stream each token to the frontend via SSE
    let fullResponse = '';
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');

    for await (const chunk of stream) {
      const token = chunk.choices[0]?.delta?.content ?? '';
      fullResponse += token;

      // Send each token to frontend immediately
      res.write(`data: ${JSON.stringify({ content: token })}\n\n`);
    }

    res.write('data: [DONE]\n\n');
    res.end();

    // 6. Save complete response to DB (after streaming finishes)
    const tokenCount = this.countTokens(fullResponse);
    await this.db.messages.create({
      conversationId, role: 'assistant',
      content: fullResponse, tokenCount
    });

    // 7. Store in semantic cache for future identical questions
    await this.semanticCache.store(userMessage, fullResponse);

    // 8. Track usage for billing/limits
    await this.trackUsage(conversationId, tokenCount);
  }
}
```

#### Context Window Management
```typescript
// You can't send 100 messages of history — it costs too many tokens
// Strategy: keep last N messages that fit within token budget

async buildContextWindow(conversationId: string, maxTokens: number): Promise<Message[]> {
  // Get messages newest-first
  const allMessages = await this.db.messages.findMany({
    where: { conversationId },
    orderBy: { createdAt: 'desc' },
    take: 50  // never look back more than 50 messages
  });

  // Add messages until we hit token budget (oldest messages dropped first)
  const context = [];
  let tokenCount = 0;

  for (const msg of allMessages) {
    const msgTokens = this.countTokens(msg.content);
    if (tokenCount + msgTokens > maxTokens) break;
    context.unshift(msg);  // add to front (maintain chronological order)
    tokenCount += msgTokens;
  }

  return context;
}

countTokens(text: string): number {
  // Rough estimate: 1 token ≈ 4 characters
  // For accuracy: use tiktoken library
  return Math.ceil(text.length / 4);
}
```

#### Semantic Cache — Save Token Costs
```typescript
// Two users ask: "What is Redis?" and "Can you explain Redis?"
// They mean the same thing — don't call OpenAI twice

@Injectable()
export class SemanticCacheService {
  async lookup(query: string): Promise<string | null> {
    // 1. Convert query to embedding vector (meaning in numbers)
    const embedding = await this.openai.embeddings.create({
      model: 'text-embedding-3-small',
      input: query,
    });
    const queryVector = embedding.data[0].embedding;

    // 2. Search vector DB for semantically similar past questions
    const results = await this.pinecone.query({
      vector: queryVector,
      topK: 1,
      includeMetadata: true,
    });

    const topMatch = results.matches[0];
    // If similarity score > 0.95, it's basically the same question
    if (topMatch && topMatch.score > 0.95) {
      return topMatch.metadata.answer as string; // return cached answer
    }

    return null; // cache miss
  }

  async store(query: string, answer: string) {
    const embedding = await this.openai.embeddings.create({
      model: 'text-embedding-3-small',
      input: query,
    });

    await this.pinecone.upsert([{
      id: randomUUID(),
      values: embedding.data[0].embedding,
      metadata: { query, answer }
    }]);
  }
}
```

#### RAG — Making the AI Know Your Data
```typescript
// RAG = Retrieval-Augmented Generation
// Problem: LLM doesn't know about YOUR company's docs, products, internal data
// Solution: retrieve relevant docs → inject into prompt → LLM answers using them

async answerWithContext(question: string, userId: string): Promise<string> {
  // 1. Embed the question
  const questionEmbedding = await this.openai.embeddings.create({
    model: 'text-embedding-3-small', input: question
  });

  // 2. Find most relevant documents from your knowledge base
  const relevantDocs = await this.pinecone.query({
    vector: questionEmbedding.data[0].embedding,
    topK: 3,  // get top 3 most relevant chunks
    filter: { type: 'knowledge_base' }
  });

  // 3. Build prompt with retrieved context
  const context = relevantDocs.matches
    .map(doc => doc.metadata.text)
    .join('\n\n---\n\n');

  const prompt = `
    You are a helpful assistant. Answer the question using ONLY the context below.
    If the answer isn't in the context, say "I don't have that information."

    Context:
    ${context}

    Question: ${question}
  `;

  // 4. Send to LLM with injected context
  const response = await this.openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: prompt }]
  });

  return response.choices[0].message.content;
}
```

#### Rate Limiting Per User (Token Budget)
```typescript
// AI apps need per-user rate limiting at the TOKEN level, not just request level
// A user sending 1 request with a 10-page document uses 100x more compute

async checkTokenBudget(userId: string, estimatedTokens: number): Promise<void> {
  const key = `token_budget:${userId}:${new Date().toISOString().slice(0, 10)}`; // daily

  const used = parseInt(await this.redis.get(key) ?? '0');
  const DAILY_LIMIT = 100_000; // 100k tokens/day per user

  if (used + estimatedTokens > DAILY_LIMIT) {
    throw new TooManyRequestsException(
      `Daily token limit reached. Resets at midnight. Used: ${used}/${DAILY_LIMIT}`
    );
  }

  await this.redis.incrby(key, estimatedTokens);
  await this.redis.expire(key, 86400); // expires after 24h
}
```

#### Next.js Frontend — Streaming Chat UI
```typescript
// ChatInterface.tsx
export function ChatInterface({ conversationId }: { conversationId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [streamingContent, setStreamingContent] = useState('');

  const sendMessage = async (userInput: string) => {
    // Add user message instantly (optimistic UI)
    setMessages(prev => [...prev, { role: 'user', content: userInput }]);
    setStreamingContent(''); // clear previous streaming content

    // Connect to SSE stream
    const eventSource = new EventSource(
      `/api/chat/${conversationId}/stream?message=${encodeURIComponent(userInput)}`
    );

    eventSource.onmessage = (event) => {
      if (event.data === '[DONE]') {
        // Streaming finished — add complete message to list
        setMessages(prev => [...prev, { role: 'assistant', content: streamingContent }]);
        setStreamingContent('');
        eventSource.close();
        return;
      }

      const { content } = JSON.parse(event.data);
      // Append each token to the streaming display
      setStreamingContent(prev => prev + content);
    };
  };

  return (
    <div className="chat">
      {messages.map((msg, i) => (
        <div key={i} className={`message ${msg.role}`}>{msg.content}</div>
      ))}

      {/* Show streaming response in real-time as tokens arrive */}
      {streamingContent && (
        <div className="message assistant streaming">
          {streamingContent}
          <span className="cursor">▋</span>
        </div>
      )}

      <input onKeyDown={(e) => e.key === 'Enter' && sendMessage(e.target.value)} />
    </div>
  );
}
```

---

## The Patterns That Repeat Everywhere

> After studying these 5 systems, you'll notice the same decisions appearing over and over in system design books:

| Pattern | Appears In | Why |
|---------|-----------|-----|
| **Stateless servers + Redis for shared state** | All 5 systems | Enables horizontal scaling |
| **Metadata in SQL, files in S3** | Chat, File Upload, Feed | DBs aren't for binary blobs |
| **Kafka for async event fanout** | Chat, File Upload, Feed | Decouple producers from consumers |
| **Redis cache in front of DB** | All 5 systems | DB can't handle all read traffic |
| **Presigned URLs** | File Upload, Feed (images) | API servers shouldn't touch file bytes |
| **Atomic Redis counters + batch flush** | URL Shortener (clicks), Feed (likes) | High-frequency writes kill DBs |
| **Optimistic fanout vs lazy fanout** | Feed (fanout on write vs read) | Celebrity problem appears everywhere |
| **JWT for stateless auth** | All 5 systems | Any server can verify without DB |

---

## What System Design Books Assume You Know

> After reading this document, you now have the vocabulary to understand books like *Designing Data-Intensive Applications* (DDIA) and *System Design Interview* by Alex Xu:

```
Book says:          You now know:
"horizontal scaling"  → stateless servers behind a load balancer
"cache-aside pattern" → check Redis first, fallback to DB, re-cache
"event-driven"        → Kafka decoupling services
"fanout"              → pushing to all followers on write
"sharding"            → splitting DB by key (user_id % N)
"presigned URL"       → client uploads directly to S3, bypassing API
"idempotency"         → same operation twice = same result (like counters)
"eventual consistency"→ Redis counter ≠ DB count for a few minutes (that's OK)
```
