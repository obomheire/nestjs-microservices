## Project Overview

EventFlow is a production-ready event management platform built as a NestJS CLI monorepo. Users can register, browse and create events, purchase tickets, and receive email notifications. The backend is composed of five independent HTTP microservices behind an API Gateway, communicating asynchronously via Apache Kafka. The frontend is a Next.js app.

## Commands

**Package manager:** `pnpm` (do not use npm or yarn)

### Development

```bash
# Start a specific service in watch mode
pnpm start:dev api-gateway
pnpm start:dev auth-service
pnpm start:dev events-service
pnpm start:dev tickets-service
pnpm start:dev notifications-service

# Build a specific service
pnpm build api-gateway

# Production start (API Gateway only)
pnpm start:prod
```

### Infrastructure (must be running before services)

```bash
docker compose up -d postgres redis zookeeper kafka mailhog
```

### Database Migrations

```bash
npx drizzle-kit generate   # Generate migration from schema changes
npx drizzle-kit migrate    # Apply migrations to the database
```

Schema lives in `libs/database/src/schema/`. Output goes to `drizzle/migrations/`.

### Testing

```bash
pnpm test                  # All unit tests
pnpm test:watch            # Watch mode
pnpm test:cov              # Coverage
pnpm test:e2e              # E2E (uses apps/api-gateway/test/jest-e2e.json)
pnpm test:debug            # With Node inspector
```

Test files use `*.spec.ts`. Jest roots are `apps/` and `libs/`. No separate `jest.config.ts` — config is in `package.json`.

### Code Quality

```bash
pnpm lint      # ESLint with autofix (matches {src,apps,libs,test}/**/*.ts)
pnpm format    # Prettier (matches apps/**/*.ts and libs/**/*.ts)
```

## Architecture

### Monorepo Layout

NestJS CLI monorepo (`nest-cli.json`) with five apps and three shared libraries:

- `apps/api-gateway` — The only HTTP-facing service (port 3000). All client traffic enters here.
- `apps/auth-service` — HTTP service on port 3001. Issues JWTs and publishes auth events to Kafka.
- `apps/events-service` — HTTP service on port 3003.
- `apps/tickets-service` — HTTP service on port 3004.
- `apps/notifications-service` — HTTP service on port 3006. Consumes Kafka events and sends email via Nodemailer.
- `libs/common` — DTOs, interfaces, interceptors, exception filters, and service constants. Aliased as `@app/common`.
- `libs/database` — Drizzle ORM setup and schema. Aliased as `@app/database`.
- `libs/kafka` — `KafkaModule.register(consumerGroup?)` dynamic module. Aliased as `@app/kafka`.
- `client/` — Next.js 16 frontend (port 4000).

### Inter-Service Communication

The API Gateway proxies to the other services over HTTP using `AUTH_SERVICE_URL`, `EVENTS_SERVICE_URL`, and `TICKETS_SERVICE_URL` env vars. Services are **not** wired together as NestJS microservice transports — they all run plain HTTP servers.

Kafka is used for async events only (fire-and-forget after an HTTP response). Services inject `KAFKA_SERVICE` from `KafkaModule` to produce messages. Topic names live in `KAFKA_TOPICS` in `libs/kafka/src/kafka.constants.ts`.

**Kafka broker addresses:**
- Inside Docker: `kafka:29092`
- Local dev: `localhost:9093`

The API Gateway passes authenticated user identity to downstream services via custom HTTP headers: `x-user-id` and `x-user-role`.

### Authentication Flow

JWT strategy is defined inside `apps/auth-service/src/` but imported by `apps/api-gateway/src/` for use as a global guard. The API Gateway validates JWTs locally (shared `JWT_SECRET`) — it does not call the auth service for every request.

### Global Response Shape

`TransformInterceptor` (applied globally in `api-gateway/main.ts`) wraps every successful response:

```json
{
  "success": true,
  "data": "<T>",
  "meta": { "timestamp": "", "path": "", "method": "" }
}
```

`AllExceptionsFilter` and `HttpExceptionFilter` are also global. Errors return:

```json
{
  "success": false,
  "error": { "code": "NOT_FOUND", "message": "", "timestamp": "", "path": "" }
}
```

### Shared Libraries

**`@app/common`** — Import DTOs (`RegisterDto`, `LoginDto`, `CreateEventDto`, `UpdateEventDto`, `PurchaseTicketDto`, `CheckInTicketDto`), interfaces (`IUser`, `IAuthUser`, response types), `SERVICES` / `SERVICES_PORTS` constants, `TransformInterceptor`, `HttpExceptionFilter`, `AllExceptionsFilter`.

**`@app/database`** — Import `DatabaseModule` and `DatabaseService`. The service exposes a `db` property typed as `NodePgDatabase<typeof schema>`. Tables: `users`, `events`, `tickets`. Enums: `UserRole` (`USER`/`ORGANIZER`/`ADMIN`), `EventStatus` (`DRAFT`/`PUBLISHED`/`CANCELLED`), `TicketStatus` (`PENDING`/`CONFIRMED`/`CHECKED_IN`/`CANCELLED`). Drizzle-inferred types (`User`, `NewUser`, `Event`, etc.) are also exported from the schema files.

**`@app/kafka`** — Call `KafkaModule.register('my-service-group')` in a service module to get `ClientsModule` with the `KAFKA_SERVICE` injection token. Use `KAFKA_TOPICS` constants for topic names. The default consumer group is `'eventflowapp-consumer'`.

### Rate Limiting

API Gateway uses a custom `ThrottlerStorageRedis` class backed by Redis (env vars `REDIS_HOST` / `REDIS_PORT`). Default throttle: 10 requests per 60 seconds. Per-route overrides use `@Throttle({ default: { limit: N, ttl: ms } })` and `@SkipThrottle()`.

### Environment Variables

| Variable | Used by |
|---|---|
| `DATABASE_URL` | auth, events, tickets, notifications |
| `JWT_SECRET` | api-gateway, auth-service |
| `KAFKA_BROKER` | auth, events, tickets, notifications |
| `REDIS_HOST` / `REDIS_PORT` | api-gateway |
| `AUTH_SERVICE_URL` | api-gateway |
| `EVENTS_SERVICE_URL` | api-gateway |
| `TICKETS_SERVICE_URL` | api-gateway |
| `SMTP_HOST` / `SMTP_PORT` | notifications-service |

### Known Quirks

- `SERVICES_PORTS.PAYEMENTS_SERVICE` has a typo ("PAYEMENTS") — match it exactly if referencing the constant.
- `users-service` and `payments-service` are referenced in constants but do not exist as apps.
- `notifications-service` module does not declare `KafkaModule` in its `@Module` imports — investigate before adding Kafka consumers there.

---

## Coding Style & Conventions

### TypeScript

- **Prettier:** single quotes, trailing commas everywhere (`'all'`), semicolons on (default).
- **ESLint:** `@typescript-eslint/no-explicit-any` is off — `any` is allowed but avoid it. `no-floating-promises` and `no-unsafe-argument` are warnings, not errors.
- Return types are explicitly declared on all service methods: `async findAll(): Promise<EventResponse[]>`.
- Controller methods are left untyped (they rely on the global interceptor wrapping).
- Use `async/await` throughout. Never use raw `.then()` chains. Wrap RxJS observables with `firstValueFrom()` when a Promise is needed (used in API Gateway HTTP client calls).

### NestJS Patterns

**Services** implement `OnModuleInit` when setup is needed (e.g., Kafka connection):

```typescript
@Injectable()
export class MyService implements OnModuleInit {
  constructor(
    @Inject(KAFKA_SERVICE) private readonly kafkClient: ClientKafka,
    private readonly dbService: DatabaseService,
  ) {}

  async onModuleInit() {
    await this.kafkClient.connect();
  }
}
```

**Controllers** use route prefixes, NestJS built-in exception classes, and `ParseUUIDPipe` for UUID params:

```typescript
@Controller('events')
export class EventsController {
  constructor(private readonly eventsService: EventsService) {}

  @UseGuards(AuthGuard('jwt'))
  @Get(':id')
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.eventsService.findOne(id);
  }
}
```

**Error handling** uses NestJS built-in exception classes only — never throw raw `Error`:

```typescript
throw new NotFoundException('Event not found');
throw new ConflictException('User already exists');
throw new UnauthorizedException('Invalid credentials');
throw new ForbiddenException('Not authorized to perform this action');
throw new BadRequestException('Insufficient ticket availability');
```

**Kafka events** are emitted fire-and-forget after mutations, always with a timestamp:

```typescript
this.kafkClient.emit(KAFKA_TOPICS.EVENT_CREATED, {
  eventId: event.id,
  timestamp: new Date().toISOString(),
});
```

### DTOs

Use `class-validator` decorators on every property. Always provide a custom `message`:

```typescript
export class RegisterDto {
  @IsEmail({}, { message: 'Please provide a valid email' })
  @IsNotEmpty({ message: 'Email is required' })
  email: string;

  @IsString({ message: 'Password must be a string' })
  @MinLength(6, { message: 'Password must be at least 6 characters long' })
  password: string;
}
```

Common decorators in use: `@IsEmail`, `@IsNotEmpty`, `@IsString`, `@IsInt`, `@IsOptional`, `@IsUUID`, `@IsDateString`, `@MinLength`, `@MaxLength`, `@Min`, `@Max`.

### Database (Drizzle ORM)

Always use `eq()`, `and()`, `sql<T>` from `drizzle-orm`. Chain fluent calls. Use `.returning()` for all inserts and updates. Destructure single-row results:

```typescript
const [event] = await this.dbService.db
  .insert(events)
  .values({ ...createEventDto, organizerId })
  .returning();

const [ticket] = await this.dbService.db
  .select()
  .from(tickets)
  .where(and(eq(tickets.id, id), eq(tickets.userId, userId)))
  .limit(1);

const [{ total }] = await this.dbService.db
  .select({ total: sql<number>`COALESCE(SUM(${tickets.quantity}), 0)` })
  .from(tickets)
  .where(eq(tickets.eventId, eventId));
```

### Naming Conventions

| Type | Convention | Example |
|---|---|---|
| Variables / parameters | camelCase | `userId`, `createEventDto`, `ticketCode` |
| Methods | camelCase | `findOne()`, `generateTicketCode()` |
| Classes / DTOs / Interfaces | PascalCase | `EventsService`, `CreateEventDto`, `IAuthUser` |
| Constants / Kafka topics | UPPER_SNAKE_CASE | `KAFKA_SERVICE`, `KAFKA_TOPICS` |
| Enum values | UPPER_SNAKE_CASE string literals | `'USER'`, `'CHECKED_IN'`, `'CANCELLED'` |
| Drizzle inferred types | PascalCase | `User`, `NewUser`, `Event`, `NewTicket` |

### Interfaces vs Types

Prefer `interface` for contracts and API shapes (`IUser`, `ApiResponse<T>`, `AuthResponse`). Use `type` only for Drizzle schema inference (`export type User = typeof users.$inferSelect`).

### Imports

Group in this order, separated by a blank line:
1. NestJS / framework packages
2. `@app/*` shared libraries
3. Third-party libraries (`bcrypt`, `drizzle-orm`, etc.)
4. Local relative imports

Use barrel `index.ts` files for all shared library exports. Import from `@app/common`, not from deep paths like `@app/common/src/dto/register.dto`.
