# EventFlow — NestJS Microservices

A monorepo event management platform built with NestJS microservices, Kafka, PostgreSQL, Redis, and a Next.js frontend.

---

## Architecture Overview

| Service | Port | Role |
|---|---|---|
| API Gateway | 3000 | Single HTTP entry point for all clients |
| Auth Service | 3001 | Registration, login, JWT tokens |
| Events Service | 3003 | Create and manage events |
| Tickets Service | 3004 | Purchase and check-in tickets |
| Notifications Service | 3006 | Email notifications via Kafka |
| Client (Next.js) | 4000 | Web frontend |
| PostgreSQL | 5432 | Primary database |
| Redis | 6379 | Rate-limiting cache |
| Kafka | 9093 | Inter-service messaging |
| Kafka UI | 8080 | Kafka debugging dashboard |
| Mailhog SMTP | 1025 | Local email sink |
| Mailhog Web UI | 8025 | View sent emails in browser |

---

## Prerequisites

Make sure the following are installed before you begin:

- [Node.js](https://nodejs.org/) v20+
- [pnpm](https://pnpm.io/installation) v9+
- [Docker](https://docs.docker.com/get-docker/) + Docker Compose v2

Verify:

```bash
node -v
pnpm -v
docker -v
docker compose version
```

---

## Step 1 — Install Dependencies

From the project root:

```bash
pnpm install
```

---

## Step 2 — Configure Environment Variables

Create a `.env` file in the project root by copying the example below. These values are used by both Docker Compose and the local dev scripts.

```bash
cat > .env << 'EOF'
# Database
DATABASE_URL=postgres://eventflowapp:eventflow_password@localhost:5432/eventflowapp

# JWT
JWT_SECRET=change_me_to_a_long_random_secret

# Kafka
KAFKA_BROKER=localhost:9093

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# SMTP (Mailhog — local only)
SMTP_HOST=localhost
SMTP_PORT=1025

# Service URLs (used by API Gateway in local dev)
AUTH_SERVICE_URL=http://localhost:3001
EVENTS_SERVICE_URL=http://localhost:3003
TICKETS_SERVICE_URL=http://localhost:3004

NODE_ENV=development
EOF
```

> **Note:** When running the full stack via Docker Compose (Step 4), the container hostnames (`postgres`, `kafka`, `redis`, `mailhog`) are used instead of `localhost`. The `docker-compose.yaml` already sets the correct values for each service container — you only need the `.env` above for local/native development.

---

## Step 3 — Start Infrastructure Services

Spin up PostgreSQL, Kafka, Zookeeper, Redis, and Mailhog using Docker Compose:

```bash
docker compose up -d postgres redis zookeeper kafka mailhog
```

Wait about 15–20 seconds for Kafka and PostgreSQL to become healthy. You can confirm:

```bash
docker compose ps
```

All listed containers should show `running` or `healthy`.

---

## Step 4 — Run Database Migrations

Generate the migration files from the schema, then apply them:

```bash
npx drizzle-kit generate
npx drizzle-kit migrate
```

`generate` must run first — it creates the `drizzle/migrations/` folder and `_journal.json` that `migrate` requires. If you skip it you will get a `Can't find meta/_journal.json` error.

---

## Option A — Run Everything in Docker (Recommended)

Build and start all services, including the frontend:

```bash
docker compose up --build
```

This starts all five microservices plus the Next.js client. Skip to **Verify** below.

---

## Option B — Run Services Locally (Development Mode)

Open a separate terminal for each service, or use a process manager like [tmux](https://github.com/tmux/tmux) or [concurrently](https://www.npmjs.com/package/concurrently).

**API Gateway:**
```bash
pnpm start:dev api-gateway
```

**Auth Service:**
```bash
pnpm start:dev auth-service
```

**Events Service:**
```bash
pnpm start:dev events-service
```

**Tickets Service:**
```bash
pnpm start:dev tickets-service
```

**Notifications Service:**
```bash
pnpm start:dev notifications-service
```

**Frontend (Next.js):**
```bash
cd client
pnpm install
pnpm dev
```

> All services use `--watch` mode and will reload on file changes.

---

## Verify Everything Is Running

| What to check | URL |
|---|---|
| API Gateway health | http://localhost:3000 |
| Web frontend | http://localhost:4000 |
| Kafka UI | http://localhost:8080 |
| Mailhog email inbox | http://localhost:8025 |

---

## Common Workflows

### Register a new user

```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Jane Doe","email":"jane@example.com","password":"secret123"}'
```

### Log in and get a JWT

```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"jane@example.com","password":"secret123"}'
```

Use the returned `accessToken` as a Bearer token for protected routes.

---

## Useful Commands

```bash
# Rebuild and restart a single service
docker compose up --build auth-service

# View logs for a service
docker compose logs -f api-gateway

# Stop all containers
docker compose down

# Stop and delete volumes (wipes the database)
docker compose down -v

# Run unit tests
pnpm test

# Run tests with coverage
pnpm test:cov

# Format code
pnpm format

# Lint
pnpm lint
```

---

## Project Structure

```
nestjs-microservices/
├── apps/
│   ├── api-gateway/          # HTTP entry point
│   ├── auth-service/         # Auth & JWT
│   ├── events-service/       # Event CRUD
│   ├── tickets-service/      # Ticket purchases
│   └── notifications-service/ # Email via Kafka
├── libs/
│   ├── common/               # Shared DTOs, filters, interceptors
│   ├── database/             # Drizzle ORM schemas
│   └── kafka/                # Kafka client setup
├── client/                   # Next.js frontend
├── drizzle/                  # SQL migrations
├── docker-compose.yaml
└── nest-cli.json
```

---

## Troubleshooting

**Kafka connection refused on startup**
Kafka takes ~15 seconds to start. If a service fails on boot, restart it after Kafka is ready:
```bash
docker compose restart auth-service
```

**`Can't find meta/_journal.json` on migration**
You skipped `generate`. Run `npx drizzle-kit generate` first, then `npx drizzle-kit migrate`.

**Database migration fails**
Confirm PostgreSQL is running and `DATABASE_URL` points to the correct host (`localhost` for local dev, `postgres` inside Docker).

**Emails not arriving**
Check Mailhog at http://localhost:8025. All outgoing mail in development is captured there — nothing is sent to real inboxes.

**Port already in use**
Find and stop the process using the port:
```bash
lsof -i :3000
kill -9 <PID>
```
