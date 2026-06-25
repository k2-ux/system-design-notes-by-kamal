# NestJS vs Express — By System
> The same backend logic for all 8 systems from HLD_AND_LLD_ANALYSIS.md, written in both frameworks side by side.
> Read this alongside that file — HLD diagrams and DB schemas stay there, backend code comparisons live here.

---

## The One Rule

```
The business logic (what your code DOES) is identical in both.
The difference is only HOW you organize and wire it.

Express:  you wire everything manually — total freedom, total responsibility
NestJS:   framework wires it for you — follow its structure, get guardrails free

Small project / solo / MVP       → Express
Large codebase / team / long-term → NestJS
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

**Core piece:** Register + Login + Protected route

---

### NestJS
```typescript
// auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async register(email: string, password: string) {
    const hashed = await bcrypt.hash(password, 12);
    const user = await this.usersService.create({ email, password: hashed });
    return { token: this.jwtService.sign({ sub: user.id, email }) };
  }

  async login(email: string, password: string) {
    const user = await this.usersService.findByEmail(email);
    if (!user) throw new UnauthorizedException('Invalid credentials');
    const valid = await bcrypt.compare(password, user.password);
    if (!valid) throw new UnauthorizedException('Invalid credentials');
    return { token: this.jwtService.sign({ sub: user.id, email }) };
  }
}

// auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  register(@Body() dto: RegisterDto) {
    return this.authService.register(dto.email, dto.password);
  }

  @Post('login')
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto.email, dto.password);
  }
}

// auth.guard.ts — protects routes
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest();
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) throw new UnauthorizedException();
    try {
      req.user = this.jwtService.verify(token);
      return true;
    } catch { throw new UnauthorizedException(); }
  }
}

// Protected route — one decorator does it
@Get('profile')
@UseGuards(AuthGuard)
getProfile(@Request() req) {
  return req.user;
}
```

---

### Express
```typescript
// auth.service.ts — identical logic, no framework decorators
export class AuthService {
  async register(email: string, password: string) {
    const hashed = await bcrypt.hash(password, 12);
    const user = await usersService.create({ email, password: hashed });
    const token = jwt.sign({ sub: user.id, email }, process.env.JWT_SECRET!, { expiresIn: '7d' });
    return { token };
  }

  async login(email: string, password: string) {
    const user = await usersService.findByEmail(email);
    if (!user) throw new Error('Invalid credentials');
    const valid = await bcrypt.compare(password, user.password);
    if (!valid) throw new Error('Invalid credentials');
    const token = jwt.sign({ sub: user.id, email }, process.env.JWT_SECRET!, { expiresIn: '7d' });
    return { token };
  }
}

// auth.routes.ts — you wire routes manually
const router = Router();
const authService = new AuthService();

router.post('/register', async (req, res, next) => {
  try {
    const result = await authService.register(req.body.email, req.body.password);
    res.status(201).json(result);
  } catch (err) { next(err); }
});

router.post('/login', async (req, res, next) => {
  try {
    const result = await authService.login(req.body.email, req.body.password);
    res.json(result);
  } catch (err) { next(err); }
});

// auth.middleware.ts — protect routes as middleware function
export function requireAuth(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ message: 'Unauthorized' });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET!) as any;
    next();
  } catch { res.status(401).json({ message: 'Invalid token' }); }
}

// Protected route — pass middleware manually
router.get('/profile', requireAuth, (req, res) => {
  res.json(req.user);
});
```

**Difference:** NestJS injects `JwtService` automatically. Express — you call `jwt.sign()` directly. NestJS uses `@UseGuards()` decorator. Express passes `requireAuth` as middleware argument per route.

---

## 2. Real-Time Chat System

**Core piece:** WebSocket server + cross-server message routing via Redis

---

### NestJS
```typescript
// chat.gateway.ts
@WebSocketGateway({ cors: true })
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;
  private users = new Map<string, Socket>();

  handleConnection(socket: Socket) {
    const userId = socket.handshake.auth.userId;
    this.users.set(userId, socket);
    // Subscribe this server to Redis channel for this user
    redisSubscriber.subscribe(`user:${userId}`, (msg) => {
      socket.emit('new_message', JSON.parse(msg));
    });
  }

  handleDisconnect(socket: Socket) {
    this.users.delete(socket.handshake.auth.userId);
  }

  @SubscribeMessage('send_message')
  async handleMessage(socket: Socket, payload: { to: string; content: string }) {
    const from = socket.handshake.auth.userId;
    const message = await this.chatService.save({ from, ...payload });
    // Publish to Redis — recipient's server (whichever it is) will forward it
    await redisPublisher.publish(`user:${payload.to}`, JSON.stringify(message));
  }
}
```

---

### Express
```typescript
// chat.server.ts
import { createServer } from 'http';
import { Server } from 'socket.io';
import express from 'express';

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, { cors: { origin: '*' } });

const users = new Map<string, Socket>();

io.on('connection', (socket) => {
  const userId = socket.handshake.auth.userId;
  users.set(userId, socket);

  // Subscribe this server to Redis channel for this user
  redisSubscriber.subscribe(`user:${userId}`, (msg) => {
    socket.emit('new_message', JSON.parse(msg));
  });

  socket.on('send_message', async (payload: { to: string; content: string }) => {
    const message = await chatService.save({ from: userId, ...payload });
    // Publish to Redis — recipient's server forwards it
    await redisPublisher.publish(`user:${payload.to}`, JSON.stringify(message));
  });

  socket.on('disconnect', () => {
    users.delete(userId);
  });
});

httpServer.listen(3000);
```

**Difference:** NestJS uses `@SubscribeMessage('send_message')` decorator instead of `socket.on('send_message', ...)`. NestJS handles connection/disconnect via `implements OnGatewayConnection`. Express is all inside `io.on('connection', ...)`. The Redis pub/sub logic is identical in both.

---

## 3. URL Shortener

**Core piece:** Shorten a URL + resolve a short code

---

### NestJS
```typescript
// urls.service.ts
@Injectable()
export class UrlsService {
  constructor(
    @InjectRepository(Url) private urlRepo: Repository<Url>,
    private redis: RedisService,
  ) {}

  async shorten(longUrl: string): Promise<string> {
    const shortCode = nanoid(6);
    await this.urlRepo.save({ shortCode, longUrl });
    await this.redis.setex(`url:${shortCode}`, 86400, longUrl);
    return `https://short.ly/${shortCode}`;
  }

  async resolve(shortCode: string): Promise<string> {
    const cached = await this.redis.get(`url:${shortCode}`);
    if (cached) return cached;

    const url = await this.urlRepo.findOne({ where: { shortCode } });
    if (!url) throw new NotFoundException('Short URL not found');
    if (url.expiresAt && url.expiresAt < new Date()) throw new GoneException('URL expired');

    await this.redis.setex(`url:${shortCode}`, 86400, url.longUrl);
    this.urlRepo.increment({ shortCode }, 'clicks', 1); // fire and forget
    return url.longUrl;
  }
}

// urls.controller.ts
@Controller('urls')
export class UrlsController {
  constructor(private urlsService: UrlsService) {}

  @Post('shorten')
  shorten(@Body('url') url: string) {
    return this.urlsService.shorten(url);
  }

  @Get(':code')
  async redirect(@Param('code') code: string, @Res() res: Response) {
    const longUrl = await this.urlsService.resolve(code);
    res.redirect(302, longUrl);
  }
}
```

---

### Express
```typescript
// urls.service.ts — identical logic
export class UrlsService {
  async shorten(longUrl: string): Promise<string> {
    const shortCode = nanoid(6);
    await db.query(
      'INSERT INTO urls (short_code, long_url) VALUES ($1, $2)',
      [shortCode, longUrl]
    );
    await redis.setex(`url:${shortCode}`, 86400, longUrl);
    return `https://short.ly/${shortCode}`;
  }

  async resolve(shortCode: string): Promise<string> {
    const cached = await redis.get(`url:${shortCode}`);
    if (cached) return cached;

    const result = await db.query('SELECT * FROM urls WHERE short_code = $1', [shortCode]);
    const url = result.rows[0];
    if (!url) throw Object.assign(new Error('Not found'), { statusCode: 404 });
    if (url.expires_at && new Date(url.expires_at) < new Date()) {
      throw Object.assign(new Error('URL expired'), { statusCode: 410 });
    }

    await redis.setex(`url:${shortCode}`, 86400, url.long_url);
    db.query('UPDATE urls SET clicks = clicks + 1 WHERE short_code = $1', [shortCode]); // fire and forget
    return url.long_url;
  }
}

// urls.routes.ts — manual routing
const urlsService = new UrlsService();

router.post('/shorten', async (req, res, next) => {
  try {
    const shortUrl = await urlsService.shorten(req.body.url);
    res.json({ shortUrl });
  } catch (err) { next(err); }
});

router.get('/:code', async (req, res, next) => {
  try {
    const longUrl = await urlsService.resolve(req.params.code);
    res.redirect(302, longUrl);
  } catch (err) { next(err); }
});
```

**Difference:** NestJS uses `@InjectRepository(Url)` for DB access (TypeORM). Express uses raw `db.query()` (pg). NestJS `@Param('code')` vs Express `req.params.code`. Error throwing: NestJS has `NotFoundException`, Express uses plain Error with statusCode attached.

---

## 4. File Upload Service

**Core piece:** Generate presigned S3 URL + confirm upload

---

### NestJS
```typescript
// files.service.ts
@Injectable()
export class FilesService {
  constructor(private configService: ConfigService) {
    this.s3 = new S3Client({ region: 'us-east-1' });
  }

  async getPresignedUrl(userId: string, filename: string, mimeType: string) {
    const key = `uploads/${userId}/${randomUUID()}-${filename}`;
    const command = new PutObjectCommand({
      Bucket: this.configService.get('S3_BUCKET'),
      Key: key,
      ContentType: mimeType,
    });
    const uploadUrl = await getSignedUrl(this.s3, command, { expiresIn: 900 });
    const file = await this.filesRepo.save({ userId, storageKey: key, status: 'uploading' });
    return { uploadUrl, fileId: file.id };
  }

  async confirmUpload(fileId: string) {
    const file = await this.filesRepo.findOne({ where: { id: fileId } });
    if (!file) throw new NotFoundException('File not found');
    await this.filesRepo.update(fileId, { status: 'ready' });
    await this.kafka.emit('file_uploaded', { fileId, storageKey: file.storageKey });
    return { url: `https://cdn.app.com/${file.storageKey}` };
  }
}

// files.controller.ts
@Controller('files')
@UseGuards(AuthGuard)
export class FilesController {
  @Post('presigned-url')
  getPresignedUrl(@Body() dto: PresignedUrlDto, @Request() req) {
    return this.filesService.getPresignedUrl(req.user.sub, dto.filename, dto.mimeType);
  }

  @Post(':id/confirm')
  confirmUpload(@Param('id') id: string) {
    return this.filesService.confirmUpload(id);
  }
}
```

---

### Express
```typescript
// files.service.ts — same logic, no decorators
const s3 = new S3Client({ region: 'us-east-1' });

export const filesService = {
  async getPresignedUrl(userId: string, filename: string, mimeType: string) {
    const key = `uploads/${userId}/${randomUUID()}-${filename}`;
    const command = new PutObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: key,
      ContentType: mimeType,
    });
    const uploadUrl = await getSignedUrl(s3, command, { expiresIn: 900 });
    const result = await db.query(
      'INSERT INTO files (user_id, storage_key, status) VALUES ($1, $2, $3) RETURNING id',
      [userId, key, 'uploading']
    );
    return { uploadUrl, fileId: result.rows[0].id };
  },

  async confirmUpload(fileId: string) {
    const result = await db.query('SELECT * FROM files WHERE id = $1', [fileId]);
    const file = result.rows[0];
    if (!file) throw Object.assign(new Error('Not found'), { statusCode: 404 });
    await db.query('UPDATE files SET status = $1 WHERE id = $2', ['ready', fileId]);
    await kafka.emit('file_uploaded', { fileId, storageKey: file.storage_key });
    return { url: `https://cdn.app.com/${file.storage_key}` };
  }
};

// files.routes.ts
router.post('/presigned-url', requireAuth, async (req, res, next) => {
  try {
    const result = await filesService.getPresignedUrl(
      req.user.sub, req.body.filename, req.body.mimeType
    );
    res.json(result);
  } catch (err) { next(err); }
});

router.post('/:id/confirm', requireAuth, async (req, res, next) => {
  try {
    const result = await filesService.confirmUpload(req.params.id);
    res.json(result);
  } catch (err) { next(err); }
});
```

**Difference:** NestJS `this.configService.get('S3_BUCKET')` vs Express `process.env.S3_BUCKET`. NestJS `this.filesRepo.save()` (TypeORM) vs Express `db.query()`. Auth is `@UseGuards(AuthGuard)` on the class vs `requireAuth` middleware on each route.

---

## 5. Social Feed System

**Core piece:** Create post → fanout event → build feed

---

### NestJS
```typescript
// post.service.ts
@Injectable()
export class PostService {
  constructor(
    private kafka: KafkaService,
    private cassandra: CassandraService,
  ) {}

  async createPost(userId: string, content: string, imageUrl?: string) {
    const post = await this.cassandra.execute(
      `INSERT INTO posts (user_id, created_at, post_id, content, image_url)
       VALUES (?, ?, uuid(), ?, ?) RETURNING *`,
      [userId, new Date(), content, imageUrl]
    );
    await this.kafka.emit('post_created', { postId: post.id, userId });
    return post;
  }
}

// feed.service.ts
@Injectable()
export class FeedService {
  constructor(private redis: RedisService, private postService: PostService) {}

  async getFeed(userId: string, page: number) {
    const postIds = await this.redis.lrange(`feed:${userId}`, page * 20, page * 20 + 19);
    return this.postService.getPostsByIds(postIds);
  }
}

// feed-fanout.worker.ts — separate microservice
@EventPattern('post_created')
async handlePostCreated(event: { postId: string; userId: string }) {
  const followerCount = await this.followService.getFollowerCount(event.userId);
  if (followerCount < 1_000_000) {
    const followers = await this.followService.getFollowers(event.userId);
    await Promise.all(
      followers.map(id => this.redis.lpush(`feed:${id}`, event.postId))
    );
  }
  // celebrities → fanout on read instead
}

// post.controller.ts
@Controller('posts')
export class PostController {
  @Post()
  @UseGuards(AuthGuard)
  create(@Body() dto: CreatePostDto, @Request() req) {
    return this.postService.createPost(req.user.sub, dto.content, dto.imageUrl);
  }

  @Get('feed')
  @UseGuards(AuthGuard)
  getFeed(@Request() req, @Query('page') page = 0) {
    return this.feedService.getFeed(req.user.sub, +page);
  }
}
```

---

### Express
```typescript
// post.service.ts
export const postService = {
  async createPost(userId: string, content: string, imageUrl?: string) {
    const post = await cassandra.execute(
      `INSERT INTO posts (user_id, created_at, post_id, content, image_url)
       VALUES (?, ?, uuid(), ?, ?)`,
      [userId, new Date(), content, imageUrl]
    );
    await kafka.emit('post_created', { postId: post.id, userId });
    return post;
  },

  async getPostsByIds(ids: string[]) {
    return cassandra.execute(`SELECT * FROM posts WHERE post_id IN ?`, [ids]);
  }
};

// feed.service.ts
export const feedService = {
  async getFeed(userId: string, page: number) {
    const postIds = await redis.lrange(`feed:${userId}`, page * 20, page * 20 + 19);
    return postService.getPostsByIds(postIds);
  }
};

// fanout.worker.ts — plain Node process consuming Kafka
kafka.consumer.on('post_created', async (event) => {
  const followerCount = await followService.getFollowerCount(event.userId);
  if (followerCount < 1_000_000) {
    const followers = await followService.getFollowers(event.userId);
    await Promise.all(followers.map(id => redis.lpush(`feed:${id}`, event.postId)));
  }
});

// posts.routes.ts
router.post('/', requireAuth, async (req, res, next) => {
  try {
    const post = await postService.createPost(
      req.user.sub, req.body.content, req.body.imageUrl
    );
    res.status(201).json(post);
  } catch (err) { next(err); }
});

router.get('/feed', requireAuth, async (req, res, next) => {
  try {
    const feed = await feedService.getFeed(req.user.sub, +(req.query.page ?? 0));
    res.json(feed);
  } catch (err) { next(err); }
});
```

**Difference:** NestJS `@EventPattern('post_created')` decorator turns a method into a Kafka consumer. Express uses `kafka.consumer.on(...)` manually. The fanout logic — Redis `lpush` per follower — is identical in both.

---

## 6. Video Streaming Service

**Core piece:** Initiate upload + transcoding worker

---

### NestJS
```typescript
// videos.service.ts
@Injectable()
export class VideosService {
  async initiateUpload(userId: string, title: string, mimeType: string) {
    const video = await this.videosRepo.save({ userId, title, status: 'uploading' });
    const rawKey = `raw/${userId}/${video.id}.mp4`;
    const uploadUrl = await getSignedUrl(this.s3, new PutObjectCommand({
      Bucket: this.config.get('S3_BUCKET'), Key: rawKey, ContentType: mimeType,
    }), { expiresIn: 900 });
    return { videoId: video.id, uploadUrl };
  }

  async confirmUpload(videoId: string) {
    await this.videosRepo.update(videoId, { status: 'processing' });
    await this.kafka.emit('video_uploaded', { videoId });
    return { message: 'Transcoding started' };
  }
}

// transcoding.worker.ts — separate NestJS microservice
@EventPattern('video_uploaded')
async handleVideoUploaded({ videoId }: { videoId: string }) {
  const video = await this.videosRepo.findOne({ where: { id: videoId } });
  const rawPath = await this.s3.downloadToTemp(video.rawS3Key);

  for (const quality of ['360p', '720p', '1080p']) {
    await this.ffmpeg.transcodeToHLS(rawPath, videoId, quality);
    await this.s3.uploadDirectory(`/tmp/${videoId}/${quality}`, `hls/${videoId}/${quality}/`);
    await this.videoQualitiesRepo.save({ videoId, quality,
      playlistUrl: `https://cdn.app.com/hls/${videoId}/${quality}/playlist.m3u8` });
  }

  await this.videosRepo.update(videoId, { status: 'ready' });
}

// videos.controller.ts
@Controller('videos')
@UseGuards(AuthGuard)
export class VideosController {
  @Post('initiate')
  initiateUpload(@Body() dto: InitiateUploadDto, @Request() req) {
    return this.videosService.initiateUpload(req.user.sub, dto.title, dto.mimeType);
  }

  @Post(':id/confirm')
  confirmUpload(@Param('id') id: string) {
    return this.videosService.confirmUpload(id);
  }
}
```

---

### Express
```typescript
// videos.service.ts
export const videosService = {
  async initiateUpload(userId: string, title: string, mimeType: string) {
    const result = await db.query(
      `INSERT INTO videos (user_id, title, status) VALUES ($1, $2, 'uploading') RETURNING id`,
      [userId, title]
    );
    const videoId = result.rows[0].id;
    const rawKey = `raw/${userId}/${videoId}.mp4`;
    const uploadUrl = await getSignedUrl(s3, new PutObjectCommand({
      Bucket: process.env.S3_BUCKET!, Key: rawKey, ContentType: mimeType,
    }), { expiresIn: 900 });
    return { videoId, uploadUrl };
  },

  async confirmUpload(videoId: string) {
    await db.query(`UPDATE videos SET status = 'processing' WHERE id = $1`, [videoId]);
    await kafka.emit('video_uploaded', { videoId });
    return { message: 'Transcoding started' };
  }
};

// transcoding.worker.ts — plain Node process
kafka.consumer.on('video_uploaded', async ({ videoId }) => {
  const result = await db.query('SELECT * FROM videos WHERE id = $1', [videoId]);
  const video = result.rows[0];
  const rawPath = await s3downloadToTemp(video.raw_s3_key);

  for (const quality of ['360p', '720p', '1080p']) {
    await ffmpegTranscodeToHLS(rawPath, videoId, quality);
    await s3uploadDirectory(`/tmp/${videoId}/${quality}`, `hls/${videoId}/${quality}/`);
    await db.query(
      `INSERT INTO video_qualities (video_id, quality, playlist_url) VALUES ($1, $2, $3)`,
      [videoId, quality, `https://cdn.app.com/hls/${videoId}/${quality}/playlist.m3u8`]
    );
  }

  await db.query(`UPDATE videos SET status = 'ready' WHERE id = $1`, [videoId]);
});

// videos.routes.ts
router.post('/initiate', requireAuth, async (req, res, next) => {
  try {
    const result = await videosService.initiateUpload(
      req.user.sub, req.body.title, req.body.mimeType
    );
    res.json(result);
  } catch (err) { next(err); }
});

router.post('/:id/confirm', requireAuth, async (req, res, next) => {
  try {
    const result = await videosService.confirmUpload(req.params.id);
    res.json(result);
  } catch (err) { next(err); }
});
```

**Difference:** Transcoding worker in NestJS is a decorated method `@EventPattern`. In Express it's `kafka.consumer.on(...)` in a plain Node file. Both run as a separate process from the API. The ffmpeg + S3 upload logic inside is identical.

---

## 7. Dating App

**Core piece:** Swipe + mutual match detection

---

### NestJS
```typescript
// swipe.service.ts
@Injectable()
export class SwipeService {
  constructor(
    @InjectRepository(Swipe) private swipeRepo: Repository<Swipe>,
    @InjectRepository(Match) private matchRepo: Repository<Match>,
    private redis: RedisService,
    private events: EventEmitter2,
  ) {}

  async swipe(swiperId: string, swipedId: string, direction: 'left' | 'right') {
    await this.swipeRepo.upsert({ swiperId, swipedId, direction }, ['swiperId', 'swipedId']);
    if (direction === 'left') return { match: false };

    // Atomic check: did other person already swipe right on me?
    const mutualKey = `mutual:${[swiperId, swipedId].sort().join(':')}`;
    const alreadyLiked = await this.redis.getset(mutualKey, swiperId);

    if (alreadyLiked && alreadyLiked !== swiperId) {
      const [a, b] = [swiperId, swipedId].sort();
      const match = await this.matchRepo.save({ userAId: a, userBId: b });
      await this.redis.del(mutualKey);
      // Emit event — gateway picks this up and notifies both users via WebSocket
      this.events.emit('match.created', { matchId: match.id, swiperId, swipedId });
      return { match: true, matchId: match.id };
    }

    return { match: false };
  }
}

// swipe.controller.ts
@Controller('swipes')
@UseGuards(AuthGuard)
export class SwipeController {
  @Post()
  swipe(@Body() dto: SwipeDto, @Request() req) {
    return this.swipeService.swipe(req.user.sub, dto.swipedId, dto.direction);
  }
}
```

---

### Express
```typescript
// swipe.service.ts
export const swipeService = {
  async swipe(swiperId: string, swipedId: string, direction: 'left' | 'right') {
    await db.query(
      `INSERT INTO swipes (swiper_id, swiped_id, direction)
       VALUES ($1, $2, $3)
       ON CONFLICT (swiper_id, swiped_id) DO UPDATE SET direction = $3`,
      [swiperId, swipedId, direction]
    );

    if (direction === 'left') return { match: false };

    // Atomic mutual check using Redis
    const mutualKey = `mutual:${[swiperId, swipedId].sort().join(':')}`;
    const alreadyLiked = await redis.getset(mutualKey, swiperId);

    if (alreadyLiked && alreadyLiked !== swiperId) {
      const [a, b] = [swiperId, swipedId].sort();
      const result = await db.query(
        `INSERT INTO matches (user_a_id, user_b_id) VALUES ($1, $2) RETURNING id`,
        [a, b]
      );
      const matchId = result.rows[0].id;
      await redis.del(mutualKey);

      // Notify both users via WebSocket (push to their Redis channels)
      await redis.publish(`user:${swiperId}`, JSON.stringify({ type: 'MATCH', matchId }));
      await redis.publish(`user:${swipedId}`, JSON.stringify({ type: 'MATCH', matchId }));

      return { match: true, matchId };
    }

    return { match: false };
  }
};

// swipes.routes.ts
router.post('/', requireAuth, async (req, res, next) => {
  try {
    const result = await swipeService.swipe(
      req.user.sub, req.body.swipedId, req.body.direction
    );
    res.json(result);
  } catch (err) { next(err); }
});
```

**Difference:** NestJS uses `EventEmitter2` to emit a `match.created` event (then a gateway listener notifies via WebSocket). Express publishes directly to Redis pub/sub channel (which socket.io server picks up). Both achieve the same result. The atomic `redis.getset()` race condition guard is identical in both.

---

## 8. AI App with LLM Provider

**Core piece:** Stream LLM response + semantic cache check

---

### NestJS
```typescript
// chat.service.ts
@Injectable()
export class ChatService {
  constructor(
    private openai: OpenAIService,
    private semanticCache: SemanticCacheService,
    @InjectRepository(Message) private messagesRepo: Repository<Message>,
  ) {}

  async streamResponse(conversationId: string, userMessage: string, res: Response) {
    // 1. Check semantic cache first
    const cached = await this.semanticCache.lookup(userMessage);
    if (cached) {
      res.write(`data: ${JSON.stringify({ content: cached, cached: true })}\n\n`);
      return res.end();
    }

    // 2. Save user message
    await this.messagesRepo.save({ conversationId, role: 'user', content: userMessage });

    // 3. Build context window
    const history = await this.buildContext(conversationId, 3000);

    // 4. Stream from OpenAI
    const stream = await this.openai.client.chat.completions.create({
      model: 'gpt-4o',
      messages: [...history, { role: 'user', content: userMessage }],
      stream: true,
    });

    res.setHeader('Content-Type', 'text/event-stream');
    let fullResponse = '';

    for await (const chunk of stream) {
      const token = chunk.choices[0]?.delta?.content ?? '';
      fullResponse += token;
      res.write(`data: ${JSON.stringify({ content: token })}\n\n`);
    }

    res.write('data: [DONE]\n\n');
    res.end();

    // 5. Persist + cache after stream completes
    await this.messagesRepo.save({ conversationId, role: 'assistant', content: fullResponse });
    await this.semanticCache.store(userMessage, fullResponse);
  }
}

// chat.controller.ts
@Controller('chat')
@UseGuards(AuthGuard)
export class ChatController {
  @Post(':conversationId/stream')
  stream(
    @Param('conversationId') conversationId: string,
    @Body('message') message: string,
    @Res() res: Response,
  ) {
    return this.chatService.streamResponse(conversationId, message, res);
  }
}
```

---

### Express
```typescript
// chat.service.ts
export const chatService = {
  async streamResponse(conversationId: string, userMessage: string, res: Response) {
    // 1. Check semantic cache
    const cached = await semanticCache.lookup(userMessage);
    if (cached) {
      res.write(`data: ${JSON.stringify({ content: cached, cached: true })}\n\n`);
      return res.end();
    }

    // 2. Save user message
    await db.query(
      `INSERT INTO messages (conversation_id, role, content) VALUES ($1, 'user', $2)`,
      [conversationId, userMessage]
    );

    // 3. Build context window
    const history = await buildContext(conversationId, 3000);

    // 4. Stream from OpenAI
    const stream = await openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [...history, { role: 'user', content: userMessage }],
      stream: true,
    });

    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    let fullResponse = '';

    for await (const chunk of stream) {
      const token = chunk.choices[0]?.delta?.content ?? '';
      fullResponse += token;
      res.write(`data: ${JSON.stringify({ content: token })}\n\n`);
    }

    res.write('data: [DONE]\n\n');
    res.end();

    // 5. Persist + cache
    await db.query(
      `INSERT INTO messages (conversation_id, role, content) VALUES ($1, 'assistant', $2)`,
      [conversationId, fullResponse]
    );
    await semanticCache.store(userMessage, fullResponse);
  }
};

// chat.routes.ts
router.post('/:conversationId/stream', requireAuth, async (req, res, next) => {
  try {
    await chatService.streamResponse(
      req.params.conversationId, req.body.message, res
    );
  } catch (err) { next(err); }
});
```

**Difference:** NestJS injects `OpenAIService` via constructor (configured once in a module). Express imports `openai` directly as a module-level instance. Everything else — SSE streaming, `for await` loop over chunks, semantic cache check — is line-for-line identical. This shows that for AI apps specifically, the framework choice barely matters.

---

## Summary — What Actually Differs Between the Two

| Pattern | NestJS | Express |
|---------|--------|---------|
| Route definition | `@Get()`, `@Post()` decorators | `router.get()`, `router.post()` |
| Auth protection | `@UseGuards(AuthGuard)` on class/method | `requireAuth` middleware per route |
| Dependency injection | Constructor params injected automatically | You `new Service()` or import directly |
| DB access | `@InjectRepository()` (TypeORM) | `db.query()` raw SQL or Prisma |
| Kafka consumer | `@EventPattern('topic')` decorator | `kafka.consumer.on('topic', ...)` |
| Config/env vars | `configService.get('KEY')` | `process.env.KEY` |
| Error throwing | `throw new NotFoundException()` | `throw Object.assign(new Error(), { statusCode: 404 })` |
| Validation | `class-validator` DTOs, auto-applied | Manual `zod.parse()` or `joi.validate()` per route |
| Business logic | **Identical** | **Identical** |

> The business logic — the Redis calls, the S3 uploads, the OpenAI streaming, the Kafka publishes — is the same in both. NestJS just organizes it differently. If you can write it in one, you can write it in the other.
