# NestJS vs Express — Side by Side
> Same backend pattern, two frameworks. Use this to understand what NestJS actually does under the hood — it's just Express with structure enforced on top.

---

## The Core Difference

```
Express:   You decide everything — folder structure, how middleware works,
           how to organize code. Total freedom. Total responsibility.

NestJS:    Opinionated framework built ON TOP of Express.
           Gives you: decorators, dependency injection, modules, guards, pipes.
           You follow its structure. In return: scales to huge codebases without chaos.

Rule of thumb:
  Side project / startup MVP / small API  → Express (less boilerplate, faster to ship)
  Team of 3+ / growing codebase / enterprise → NestJS (enforced structure saves you later)
```

---

## Table of Contents
1. [Project Setup & Folder Structure](#1-project-setup--folder-structure)
2. [Basic Routing](#2-basic-routing)
3. [Middleware](#3-middleware)
4. [Request Validation](#4-request-validation)
5. [Authentication with JWT](#5-authentication-with-jwt)
6. [Route Guards / Protected Routes](#6-route-guards--protected-routes)
7. [PostgreSQL Database Connection](#7-postgresql-database-connection)
8. [Error Handling](#8-error-handling)
9. [WebSockets / Real-Time](#9-websockets--real-time)
10. [File Upload](#10-file-upload)
11. [Background Jobs / Cron](#11-background-jobs--cron)
12. [Environment Config](#12-environment-config)
13. [When to Choose Which](#13-when-to-choose-which)

---

## 1. Project Setup & Folder Structure

### Express
```bash
npm init -y
npm install express cors helmet dotenv
npm install -D typescript @types/express ts-node nodemon
```

```
my-app/
  src/
    routes/
      auth.routes.ts
      users.routes.ts
    middleware/
      auth.middleware.ts
      error.middleware.ts
    controllers/
      auth.controller.ts
    services/
      auth.service.ts
    db/
      index.ts
    index.ts          ← entry point, you wire everything manually
  .env
  package.json
```

```typescript
// src/index.ts — you set up everything yourself
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { authRouter } from './routes/auth.routes';
import { errorHandler } from './middleware/error.middleware';

const app = express();

app.use(cors());
app.use(helmet());
app.use(express.json());

// Register routes manually
app.use('/api/auth', authRouter);
app.use('/api/users', usersRouter);

// Error handler last
app.use(errorHandler);

app.listen(3000, () => console.log('Running on port 3000'));
```

---

### NestJS
```bash
npm install -g @nestjs/cli
nest new my-app
```

```
my-app/
  src/
    auth/
      auth.module.ts      ← NestJS enforces this module pattern
      auth.controller.ts
      auth.service.ts
      auth.guard.ts
      dto/
        login.dto.ts
    users/
      users.module.ts
      users.controller.ts
      users.service.ts
    app.module.ts         ← root module, imports all others
    main.ts               ← entry point
```

```typescript
// src/main.ts — NestJS bootstraps everything for you
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  app.useGlobalPipes(new ValidationPipe()); // auto-validates all DTOs
  await app.listen(3000);
}
bootstrap();

// src/app.module.ts — declare what exists
@Module({
  imports: [AuthModule, UsersModule, DatabaseModule],
})
export class AppModule {}
```

**Key difference:** In Express you manually `app.use()` everything. In NestJS you declare modules and the framework wires them together.

---

## 2. Basic Routing

### Express
```typescript
// src/routes/users.routes.ts
import { Router } from 'express';
import { UsersController } from '../controllers/users.controller';

const router = Router();
const controller = new UsersController(); // you instantiate manually

router.get('/', controller.getAll);
router.get('/:id', controller.getOne);
router.post('/', controller.create);
router.put('/:id', controller.update);
router.delete('/:id', controller.delete);

export const usersRouter = router;

// src/controllers/users.controller.ts
import { Request, Response, NextFunction } from 'express';

export class UsersController {
  async getAll(req: Request, res: Response, next: NextFunction) {
    try {
      const users = await usersService.findAll();
      res.json(users);
    } catch (err) {
      next(err); // pass to error handler
    }
  }

  async getOne(req: Request, res: Response, next: NextFunction) {
    try {
      const user = await usersService.findOne(req.params.id);
      if (!user) return res.status(404).json({ message: 'Not found' });
      res.json(user);
    } catch (err) {
      next(err);
    }
  }
}
```

---

### NestJS
```typescript
// src/users/users.controller.ts
import { Controller, Get, Post, Put, Delete, Param, Body } from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')  // ← sets base path /users
export class UsersController {
  constructor(private readonly usersService: UsersService) {}
  // ↑ NestJS injects UsersService automatically (dependency injection)
  // You never call "new UsersService()" yourself

  @Get()              // GET /users
  getAll() {
    return this.usersService.findAll();
    // No try/catch needed — NestJS catches exceptions automatically
    // No res.json() needed — return value is auto-serialized to JSON
  }

  @Get(':id')         // GET /users/:id
  getOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }

  @Post()             // POST /users
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Delete(':id')      // DELETE /users/:id
  remove(@Param('id') id: string) {
    return this.usersService.remove(id);
  }
}
```

**Key difference:** NestJS decorators (`@Get`, `@Post`, `@Param`, `@Body`) replace manual `router.get()` + `req.params` + `req.body`. Much less boilerplate per route.

---

## 3. Middleware

### Express
```typescript
// src/middleware/logger.middleware.ts
import { Request, Response, NextFunction } from 'express';

export function loggerMiddleware(req: Request, res: Response, next: NextFunction) {
  console.log(`${req.method} ${req.url} — ${new Date().toISOString()}`);
  next(); // don't forget this — without it the request hangs forever
}

// Register in index.ts
app.use(loggerMiddleware);         // applies to ALL routes
app.use('/api/users', authMiddleware, usersRouter);  // specific routes
```

---

### NestJS
```typescript
// src/common/logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`${req.method} ${req.url} — ${new Date().toISOString()}`);
    next();
  }
}

// Register in app.module.ts
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('*');              // all routes
      // OR: .forRoutes('users')   // specific path
      // OR: .forRoutes({ path: 'users', method: RequestMethod.GET })
  }
}
```

**Key difference:** Express middleware is just a function. NestJS middleware is a class — more structure, injectable dependencies.

---

## 4. Request Validation

### Express
```typescript
// Manual validation — you write it yourself
// or use libraries like: joi, zod, express-validator

import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  age: z.number().min(18),
});

// Validation middleware
export function validate(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({ errors: result.error.flatten() });
    }
    req.body = result.data; // replace with parsed/transformed data
    next();
  };
}

// Use on route
router.post('/', validate(CreateUserSchema), controller.create);
```

---

### NestJS
```typescript
// src/users/dto/create-user.dto.ts
import { IsEmail, IsString, MinLength, Min, IsInt } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsInt()
  @Min(18)
  age: number;
}

// In main.ts — enable once, works for ALL routes automatically
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,    // strip unknown fields (security)
  transform: true,    // auto-convert "18" string → 18 number
}));

// Controller just declares the type — validation is automatic
@Post()
create(@Body() dto: CreateUserDto) {
  // If validation fails, NestJS returns 400 automatically
  // You never write validation logic in the controller
  return this.usersService.create(dto);
}
```

**Key difference:** Express = you wire validation middleware per route. NestJS = declare DTO class with decorators once, validation happens everywhere automatically.

---

## 5. Authentication with JWT

### Express
```typescript
// src/services/auth.service.ts
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import { db } from '../db';

export class AuthService {
  async register(email: string, password: string) {
    const hashed = await bcrypt.hash(password, 12);
    const user = await db.query(
      'INSERT INTO users (email, password) VALUES ($1, $2) RETURNING id, email',
      [email, hashed]
    );
    const token = jwt.sign(
      { sub: user.rows[0].id, email },
      process.env.JWT_SECRET!,
      { expiresIn: '7d' }
    );
    return { token };
  }

  async login(email: string, password: string) {
    const result = await db.query('SELECT * FROM users WHERE email = $1', [email]);
    const user = result.rows[0];
    if (!user) throw new Error('Invalid credentials');

    const valid = await bcrypt.compare(password, user.password);
    if (!valid) throw new Error('Invalid credentials');

    const token = jwt.sign(
      { sub: user.id, email },
      process.env.JWT_SECRET!,
      { expiresIn: '7d' }
    );
    return { token };
  }
}

// src/routes/auth.routes.ts
router.post('/register', async (req, res, next) => {
  try {
    const { email, password } = req.body;
    const result = await authService.register(email, password);
    res.status(201).json(result);
  } catch (err) {
    next(err);
  }
});
```

---

### NestJS
```typescript
// src/auth/auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,  // injected automatically
  ) {}

  async register(dto: RegisterDto) {
    const hashed = await bcrypt.hash(dto.password, 12);
    const user = await this.usersService.create({ ...dto, password: hashed });
    return { token: this.jwtService.sign({ sub: user.id, email: user.email }) };
  }

  async login(dto: LoginDto) {
    const user = await this.usersService.findByEmail(dto.email);
    if (!user) throw new UnauthorizedException('Invalid credentials');
    const valid = await bcrypt.compare(dto.password, user.password);
    if (!valid) throw new UnauthorizedException('Invalid credentials');
    return { token: this.jwtService.sign({ sub: user.id, email: user.email }) };
  }
}

// src/auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  register(@Body() dto: RegisterDto) {
    return this.authService.register(dto);
    // NestJS auto-sends 201 for POST, catches errors, serializes JSON
  }

  @Post('login')
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto);
  }
}

// src/auth/auth.module.ts
@Module({
  imports: [
    JwtModule.register({
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: '7d' },
    }),
    UsersModule,
  ],
  controllers: [AuthController],
  providers: [AuthService],
})
export class AuthModule {}
```

---

## 6. Route Guards / Protected Routes

### Express
```typescript
// src/middleware/auth.middleware.ts
import jwt from 'jsonwebtoken';

export function requireAuth(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.split(' ')[1]; // "Bearer <token>"
  if (!token) return res.status(401).json({ message: 'No token provided' });

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as any;
    req.user = payload; // attach user to request
    next();
  } catch {
    res.status(401).json({ message: 'Invalid token' });
  }
}

// Apply per-route or per-router
router.get('/profile', requireAuth, controller.getProfile);
router.use('/admin', requireAuth, adminRouter); // protect entire sub-router
```

---

### NestJS
```typescript
// src/auth/auth.guard.ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(' ')[1];
    if (!token) throw new UnauthorizedException();

    try {
      request.user = this.jwtService.verify(token);
      return true;
    } catch {
      throw new UnauthorizedException();
    }
  }
}

// Apply with decorator — much cleaner
@Controller('profile')
@UseGuards(AuthGuard)  // ← protects ENTIRE controller
export class ProfileController {
  @Get()
  getProfile(@Request() req) {
    return req.user;
  }

  @Get('settings')
  // also protected — guard applies to all methods
  getSettings(@Request() req) { ... }
}

// Or per-method:
@Controller('posts')
export class PostsController {
  @Get()
  // public — no guard
  getPosts() { ... }

  @Post()
  @UseGuards(AuthGuard)  // ← only this method is protected
  createPost(@Request() req, @Body() dto: CreatePostDto) { ... }
}
```

**Key difference:** Express = pass middleware function to each route. NestJS = `@UseGuards()` decorator, readable at a glance, injectable.

---

## 7. PostgreSQL Database Connection

### Express
```typescript
// src/db/index.ts
import { Pool } from 'pg';

export const db = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,         // max connections in pool
  idleTimeoutMillis: 30000,
});

// Raw SQL queries everywhere
// src/services/users.service.ts
export class UsersService {
  async findAll() {
    const result = await db.query('SELECT id, email, created_at FROM users');
    return result.rows;
  }

  async findOne(id: string) {
    const result = await db.query('SELECT * FROM users WHERE id = $1', [id]);
    return result.rows[0] ?? null;
  }

  async create(email: string, hashedPassword: string) {
    const result = await db.query(
      'INSERT INTO users (email, password) VALUES ($1, $2) RETURNING *',
      [email, hashedPassword]
    );
    return result.rows[0];
  }
}
```

**With an ORM (Prisma — most popular with Express):**
```typescript
// Using Prisma ORM (works with both Express and NestJS)
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// Type-safe queries, no raw SQL needed
const users = await prisma.user.findMany();
const user = await prisma.user.findUnique({ where: { id } });
const newUser = await prisma.user.create({ data: { email, password } });
```

---

### NestJS
```typescript
// NestJS most commonly uses TypeORM or Prisma

// Option A: TypeORM (NestJS-native)
// src/database/database.module.ts
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      url: process.env.DATABASE_URL,
      entities: [User, Post, Message],
      synchronize: false,  // never true in production
    }),
  ],
})
export class DatabaseModule {}

// src/users/user.entity.ts
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;

  @CreateDateColumn()
  createdAt: Date;
}

// src/users/users.service.ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,  // injected automatically
  ) {}

  findAll() {
    return this.userRepository.find();
  }

  findOne(id: string) {
    return this.userRepository.findOne({ where: { id } });
  }

  create(data: Partial<User>) {
    const user = this.userRepository.create(data);
    return this.userRepository.save(user);
  }
}

// Option B: Prisma with NestJS (cleaner, recommended)
// Same Prisma queries as Express — just inject PrismaService instead
@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  findAll() { return this.prisma.user.findMany(); }
  findOne(id: string) { return this.prisma.user.findUnique({ where: { id } }); }
}
```

---

## 8. Error Handling

### Express
```typescript
// src/middleware/error.middleware.ts
import { Request, Response, NextFunction } from 'express';

// Custom error class
export class AppError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

// Global error handler — MUST have 4 params (err, req, res, next)
export function errorHandler(err: any, req: Request, res: Response, next: NextFunction) {
  const statusCode = err.statusCode ?? 500;
  const message = err.message ?? 'Internal server error';

  console.error(`[ERROR] ${req.method} ${req.url}:`, err);

  res.status(statusCode).json({
    status: 'error',
    statusCode,
    message,
    // Don't expose stack trace in production
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
  });
}

// Register LAST in index.ts
app.use(errorHandler);

// Throw anywhere in your code
throw new AppError(404, 'User not found');
throw new AppError(401, 'Invalid credentials');
```

---

### NestJS
```typescript
// NestJS has built-in HTTP exceptions — no custom error class needed

// In any service or controller:
throw new NotFoundException('User not found');        // 404
throw new UnauthorizedException('Invalid token');     // 401
throw new BadRequestException('Email already exists'); // 400
throw new ForbiddenException('Access denied');        // 403
throw new ConflictException('Resource already exists'); // 409
throw new InternalServerErrorException('DB failed');  // 500

// All produce standardized JSON automatically:
// { statusCode: 404, message: "User not found", error: "Not Found" }

// Custom global exception filter (optional — for custom format)
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status = exception instanceof HttpException
      ? exception.getStatus()
      : 500;

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: exception instanceof HttpException
        ? exception.message
        : 'Internal server error',
    });
  }
}

// Register globally in main.ts
app.useGlobalFilters(new AllExceptionsFilter());
```

**Key difference:** Express = you build your error class and handler. NestJS = built-in exception classes for every HTTP status, auto-formatted responses.

---

## 9. WebSockets / Real-Time

### Express (with socket.io)
```typescript
// index.ts
import { createServer } from 'http';
import { Server } from 'socket.io';
import express from 'express';

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, { cors: { origin: '*' } });

// Track connected users manually
const connectedUsers = new Map<string, string>(); // userId → socketId

io.on('connection', (socket) => {
  const userId = socket.handshake.auth.userId;
  connectedUsers.set(userId, socket.id);
  console.log(`User ${userId} connected`);

  socket.on('send_message', async (data) => {
    const { conversationId, content } = data;

    // Save to DB
    const message = await messagesService.create({ conversationId, content, senderId: userId });

    // Find recipient and emit directly
    const recipientSocketId = connectedUsers.get(data.recipientId);
    if (recipientSocketId) {
      io.to(recipientSocketId).emit('new_message', message); // direct push
    }
  });

  socket.on('disconnect', () => {
    connectedUsers.delete(userId);
  });
});

httpServer.listen(3000);
```

---

### NestJS (with @nestjs/websockets)
```typescript
// src/chat/chat.gateway.ts
import {
  WebSocketGateway, WebSocketServer,
  SubscribeMessage, OnGatewayConnection, OnGatewayDisconnect
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({ cors: true })
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private connectedUsers = new Map<string, Socket>();

  handleConnection(socket: Socket) {
    const userId = socket.handshake.auth.userId;
    this.connectedUsers.set(userId, socket);
  }

  handleDisconnect(socket: Socket) {
    const userId = socket.handshake.auth.userId;
    this.connectedUsers.delete(userId);
  }

  @SubscribeMessage('send_message')
  async handleMessage(socket: Socket, payload: { recipientId: string; content: string }) {
    const senderId = socket.handshake.auth.userId;

    const message = await this.chatService.saveMessage({
      senderId, ...payload
    });

    // Push to recipient if online
    const recipientSocket = this.connectedUsers.get(payload.recipientId);
    recipientSocket?.emit('new_message', message);
  }
}

// Register in module
@Module({
  providers: [ChatGateway, ChatService],
})
export class ChatModule {}
```

**Key difference:** Very similar logic — NestJS just organizes it into a class with decorators (`@SubscribeMessage` vs `socket.on`), making large gateway files easier to navigate.

---

## 10. File Upload

### Express (with multer)
```typescript
import multer from 'multer';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({ region: 'us-east-1' });

// Memory storage — file goes to buffer, then we push to S3
const upload = multer({ storage: multer.memoryStorage() });

router.post('/upload', upload.single('file'), async (req, res, next) => {
  try {
    const file = req.file;
    if (!file) return res.status(400).json({ message: 'No file provided' });

    const key = `uploads/${Date.now()}-${file.originalname}`;

    await s3.send(new PutObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype,
    }));

    res.json({ url: `https://${process.env.S3_BUCKET}.s3.amazonaws.com/${key}` });
  } catch (err) {
    next(err);
  }
});
```

---

### NestJS (with @nestjs/platform-express multer)
```typescript
// src/files/files.controller.ts
import { Controller, Post, UploadedFile, UseInterceptors } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { memoryStorage } from 'multer';

@Controller('files')
export class FilesController {
  constructor(private filesService: FilesService) {}

  @Post('upload')
  @UseGuards(AuthGuard)
  @UseInterceptors(FileInterceptor('file', {
    storage: memoryStorage(),
    limits: { fileSize: 50 * 1024 * 1024 }, // 50 MB
    fileFilter: (req, file, cb) => {
      // Allow only images
      if (!file.mimetype.startsWith('image/')) {
        return cb(new BadRequestException('Only images allowed'), false);
      }
      cb(null, true);
    }
  }))
  async uploadFile(@UploadedFile() file: Express.Multer.File) {
    return this.filesService.uploadToS3(file);
  }
}
```

---

## 11. Background Jobs / Cron

### Express
```typescript
// Use node-cron directly
import cron from 'node-cron';

// Schedule in index.ts or a dedicated scheduler file
cron.schedule('*/5 * * * *', async () => {
  // Runs every 5 minutes
  console.log('Flushing like counts to DB...');
  await likeService.flushCountsToDB();
});

cron.schedule('0 0 * * *', async () => {
  // Runs at midnight every day
  await cleanupService.deleteExpiredSessions();
});

// For heavy jobs (email sending, transcoding), use Bull queue
import Bull from 'bull';

const emailQueue = new Bull('email', { redis: process.env.REDIS_URL });

// Add job to queue
await emailQueue.add({ to: 'kamal@gmail.com', subject: 'Welcome!' });

// Process jobs (can run in separate worker process)
emailQueue.process(async (job) => {
  await emailService.send(job.data);
});
```

---

### NestJS
```typescript
// Built-in @nestjs/schedule for cron jobs
// npm install @nestjs/schedule

// src/app.module.ts
@Module({
  imports: [ScheduleModule.forRoot()], // register once
})
export class AppModule {}

// src/tasks/tasks.service.ts
@Injectable()
export class TasksService {
  @Cron('*/5 * * * *')  // every 5 minutes
  async flushLikeCounts() {
    await this.likesService.flushCountsToDB();
  }

  @Cron('0 0 * * *')  // midnight every day
  async cleanupSessions() {
    await this.sessionsService.deleteExpired();
  }

  @Interval(60000)  // every 60 seconds
  async checkHealthStatus() {
    await this.healthService.check();
  }
}

// Bull queue with NestJS (@nestjs/bull)
// src/email/email.processor.ts
@Processor('email')
export class EmailProcessor {
  @Process()
  async handleEmail(job: Job<{ to: string; subject: string; body: string }>) {
    await this.emailService.send(job.data);
  }
}

// Enqueue from anywhere
@Injectable()
export class NotificationService {
  constructor(@InjectQueue('email') private emailQueue: Queue) {}

  async sendWelcomeEmail(user: User) {
    await this.emailQueue.add({
      to: user.email,
      subject: 'Welcome!',
      body: 'Thanks for signing up'
    });
  }
}
```

---

## 12. Environment Config

### Express
```typescript
// .env
DATABASE_URL=postgresql://localhost:5432/myapp
JWT_SECRET=supersecret
REDIS_URL=redis://localhost:6379
PORT=3000

// src/config.ts — manual, no validation
import dotenv from 'dotenv';
dotenv.config();

export const config = {
  databaseUrl: process.env.DATABASE_URL!,
  jwtSecret: process.env.JWT_SECRET!,
  redisUrl: process.env.REDIS_URL!,
  port: parseInt(process.env.PORT ?? '3000'),
};
// No validation — if DATABASE_URL is missing, you find out at runtime when DB connection fails
```

---

### NestJS
```typescript
// npm install @nestjs/config joi

// src/app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,  // available everywhere without importing
      validationSchema: Joi.object({
        DATABASE_URL: Joi.string().required(),
        JWT_SECRET: Joi.string().min(32).required(),
        REDIS_URL: Joi.string().required(),
        PORT: Joi.number().default(3000),
      }),
      // ↑ If DATABASE_URL is missing, app REFUSES TO START with clear error
      // "Missing required environment variable: DATABASE_URL"
    }),
  ],
})
export class AppModule {}

// Use anywhere via injection
@Injectable()
export class AuthService {
  constructor(private configService: ConfigService) {}

  getJwtSecret() {
    return this.configService.get<string>('JWT_SECRET'); // type-safe
  }
}
```

**Key difference:** Express = you read env vars manually, no validation. NestJS = schema validation at startup, app refuses to start with missing config.

---

## 13. When to Choose Which

| Situation | Choose | Why |
|-----------|--------|-----|
| Solo developer, side project | **Express** | Less boilerplate, ship faster |
| MVP that needs to launch in 2 weeks | **Express** | No time to learn NestJS patterns |
| Team of 3+ developers | **NestJS** | Enforced structure prevents chaos as team grows |
| Microservice that does 1 thing | **Express** | NestJS overhead not worth it for tiny service |
| Large monolith or modular monolith | **NestJS** | Modules keep 50+ files manageable |
| You already know Angular | **NestJS** | Same decorator/module/injection patterns |
| You want to move fast and hate boilerplate | **Express + Prisma** | Minimal setup, powerful ORM |
| You want built-in validation, guards, pipes | **NestJS** | These are free with NestJS |
| WebSocket-heavy app | **Either** | Both support socket.io equally well |
| Interview / take-home project | **Express** | Shows raw fundamentals, less magic |

## The Bottom Line

```
Express is a library. NestJS is a framework.

Library: you call it
Framework: it calls you (you fill in the blanks it defines)

Express code:
  app.get('/users', async (req, res) => { ... })  ← you decide everything

NestJS code:
  @Get()
  getUsers() { ... }                               ← framework decides the structure,
                                                      you fill in the logic

For a 500-line app:  Express feels lighter and faster to write
For a 50,000-line app: NestJS saves you from spaghetti code

Both compile to the same thing: an Express HTTP server running on Node.js.
NestJS is Express with guardrails.
```
