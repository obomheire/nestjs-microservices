# EventFlow — Platform Overview

## What Is EventFlow?

EventFlow is a cloud-ready event management platform that allows organizers to create and sell tickets to events, and attendees to discover, purchase, and check in to those events — all through a single API.

The platform is built as a set of independent microservices that work together behind a single entry point. Each service owns a specific business domain, can be scaled independently, and communicates with other services through a message broker (Kafka) so that no single failure brings down the whole system.

---

## The Problem It Solves

Traditional event platforms are built as monoliths — one large application handling everything. When traffic spikes (e.g. a popular event goes on sale), the whole system slows down. When one feature breaks, everything goes down.

EventFlow is architected differently:

- The ticketing system can scale independently during high-demand sales without touching the auth or notifications system.
- A failure in email delivery does not affect ticket purchases.
- Each domain (auth, events, tickets, notifications) can be updated, deployed, and maintained by separate teams without risking the rest of the platform.

---

## How the System Is Structured

```
                        ┌─────────────────────────────┐
   All client requests  │       API Gateway :3000      │
   ──────────────────▶  │  Rate limiting · JWT auth    │
                        └──────┬──────────┬────────────┘
                               │ HTTP     │ HTTP
               ┌───────────────┼──────────┼───────────────┐
               ▼               ▼          ▼               ▼
        ┌─────────────┐ ┌──────────────┐ ┌──────────────┐
        │ Auth Service│ │Events Service│ │Ticket Service│
        │   :3001     │ │    :3003     │ │    :3004     │
        └──────┬──────┘ └──────┬───────┘ └──────┬───────┘
               │               │                │
               └───────────────┼────────────────┘
                               │ Kafka events (async)
                               ▼
                    ┌──────────────────────┐
                    │ Notifications :3006  │
                    │  Email via Mailhog   │
                    └──────────────────────┘

   Shared Infrastructure:
   PostgreSQL (data) · Redis (rate limiting) · Kafka (messaging)
```

### The API Gateway

The single door into the platform. No client ever talks directly to individual services. The gateway is responsible for:

- **Authentication** — validates JWT tokens on every protected request so individual services don't have to.
- **Rate limiting** — backed by Redis, limits each IP to 10 requests per 60 seconds globally (tighter limits on auth endpoints: 3 per minute for registration, 5 per minute for login). This protects against abuse and brute-force attacks.
- **Routing** — forwards requests to the correct downstream service over internal HTTP.
- **Consistent response shape** — every response from the platform, success or error, follows the same envelope so clients always know what to expect.

### Auth Service

Owns everything to do with user identity.

- Registers new users, hashing passwords with bcrypt (industry-standard, 10 rounds).
- Issues signed JWT tokens on login. Tokens are validated by the gateway locally — no round-trip to the auth service on every request.
- Publishes a `user.registered` event to Kafka after signup so the notifications service can send a welcome email without the auth service needing to know anything about email.

### Events Service

Owns the event lifecycle.

- Organizers create events that start in **DRAFT** status — invisible to the public.
- Publishing an event moves it to **PUBLISHED** and makes it available for ticket purchases.
- Events can be **CANCELLED** at any point, which triggers a Kafka event that downstream consumers (e.g. notifications) can act on.
- Only the event organizer or an ADMIN can modify, publish, or cancel an event. The service enforces this at the data layer.
- `GET /events` only returns PUBLISHED events to the public.

### Tickets Service

Owns the full ticket lifecycle from purchase to check-in.

- Validates that an event is PUBLISHED before allowing purchase.
- Tracks real-time capacity by summing confirmed ticket quantities against the event's capacity — preventing overselling even under concurrent load.
- Generates a unique, cryptographically random ticket code (12-character hex) per purchase. This code is the physical artifact used at the door.
- Check-in is gated to the event organizer only — attendees cannot check themselves in.
- Status transitions are strictly enforced: a cancelled or already-checked-in ticket cannot be acted on again.

### Notifications Service

Handles all outbound communication. It does not expose an HTTP API — it listens exclusively to Kafka events published by other services and reacts to them:

| Kafka Event | Email Sent |
|---|---|
| `user.registered` | Welcome email to new user |
| `ticket.purchased` | Ticket confirmation with code, event details, price |
| `ticket.cancelled` | Cancellation confirmation |
| `event.cancelled` | Organizer notification |

Because notifications are decoupled via Kafka, a slow email provider or a notifications outage has zero impact on ticket purchases or event management.

---

## The Data Model

Three core entities power the platform:

**Users**
Each user has a role: `USER` (attendee), `ORGANIZER` (event creator), or `ADMIN`. Role controls what actions the system permits.

**Events**
Belong to an organizer. Move through a strict status lifecycle: `DRAFT → PUBLISHED → CANCELLED`. Capacity and price are set at creation. Only PUBLISHED events are purchasable.

**Tickets**
Join a user to an event. Each ticket has a unique `ticketCode` used for physical check-in. Status lifecycle: `CONFIRMED → CHECKED_IN` (normal flow) or `CONFIRMED → CANCELLED` (refund flow). Capacity is tracked in real time against confirmed tickets.

---

## Security Model

| Concern | How It Is Handled |
|---|---|
| Password storage | bcrypt with 10 salt rounds — passwords are never stored in plain text |
| Authentication | JWT tokens signed with a shared secret — stateless, no database lookup per request |
| Authorization | Role and ownership checks enforced at the service layer, not just the gateway |
| Brute force protection | Redis-backed rate limiting per IP on all auth endpoints |
| Internal traffic | Services communicate over Docker's internal network — never exposed to the public internet |
| Input validation | Every incoming request is validated against typed DTOs with strict rules before touching business logic |

---

## Request & Response Contract

Every response from the platform — success or failure — follows the same shape. Client applications always know where to find their data and never need to guess the error format.

**Success:**
```json
{
  "success": true,
  "data": { },
  "meta": {
    "timestamp": "2026-05-04T11:49:03.033Z",
    "path": "/tickets/purchase",
    "method": "POST"
  }
}
```

**Error:**
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Event not found",
    "timestamp": "2026-05-04T11:49:03.033Z",
    "path": "/events/abc"
  }
}
```

Error codes are machine-readable (`BAD_REQUEST`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `TOO_MANY_REQUESTS`, `INTERNAL_SERVER_ERROR`) so client applications can handle them programmatically.

---

## End-to-End User Journeys

### Attendee Journey

```
Register → Login → Browse events → Purchase ticket → Receive confirmation email → Check in at door
```

1. Attendee registers. Welcome email arrives instantly.
2. They browse published events (no login required).
3. They log in and purchase tickets for an event. A unique ticket code is returned immediately and emailed as confirmation.
4. At the event, the organizer scans/enters the ticket code. The system marks it `CHECKED_IN`.

### Organizer Journey

```
Register → Login → Create event (DRAFT) → Set details → Publish → Monitor ticket sales → Check in attendees
```

1. Organizer creates an event — it starts as DRAFT, invisible to attendees.
2. When ready, they publish it. It immediately appears in public listings.
3. Attendees begin purchasing. The organizer can view all tickets sold for their event.
4. At the door, they check in attendees by ticket code.
5. If needed, they can cancel the event. All stakeholders are notified via the notifications pipeline.

---

## Scalability & Reliability

**Independent scaling** — because each service is a separate process, the tickets service can run on 10 instances during a high-demand sale while auth runs on 2. No other service is affected.

**Asynchronous communication** — Kafka decouples services so that slow or failed downstream consumers (like notifications) never block the critical path. A ticket purchase completes and returns to the user in milliseconds. The welcome email is sent whenever the notifications service gets to it.

**Restart resilience** — all services are configured with `restart: unless-stopped`. If a service crashes, Docker restarts it automatically without operator intervention.

**Rate limiting with Redis** — distributed rate limiting means the limits are enforced correctly even when the API gateway is running on multiple instances.

**Data integrity** — ticket codes are enforced as unique at the database level. Capacity is calculated against live confirmed ticket data — not a cached counter — so overselling is structurally impossible.

---

## Technology Stack

| Layer | Technology | Why |
|---|---|---|
| API framework | NestJS (Node.js) | Structured, production-grade framework with built-in DI, guards, and pipes |
| Language | TypeScript | Type safety across the entire codebase reduces runtime errors |
| Database | PostgreSQL 16 | Battle-tested relational database with ACID guarantees |
| ORM | Drizzle ORM | Type-safe query builder — SQL errors are caught at compile time, not runtime |
| Message broker | Apache Kafka | Industry-standard for high-throughput async event streaming |
| Cache / rate limiting | Redis | In-memory store for sub-millisecond rate limit checks |
| Authentication | JWT + Passport | Stateless auth — no session storage, horizontally scalable |
| Email (dev/test) | Mailhog | Captures all outgoing email locally — no real emails sent during testing |
| Containerization | Docker + Docker Compose | One-command startup, consistent environment across dev and production |
| Frontend | Next.js 16 + React 19 | Modern, server-rendered web frontend |

---

## Frequently Asked Questions

**How does the platform prevent someone from buying more tickets than are available?**

The tickets service calculates remaining capacity in real time by querying the sum of all confirmed ticket quantities for the event at the moment of purchase. It does not rely on a cached counter. If two users attempt to buy the last ticket simultaneously, the database transaction ensures only one succeeds.

**What happens if the notifications service goes down?**

Nothing visible to the user. Kafka retains the published events in its log. When the notifications service recovers, it picks up from where it left off and processes any queued events. Ticket purchases, event management, and authentication are completely unaffected.

**Can an attendee check themselves in?**

No. The check-in endpoint validates that the requester is the organizer of the specific event the ticket belongs to. An attendee presenting their own ticket code cannot mark it as checked in.

**How are ticket codes generated?**

Each ticket code is a 12-character uppercase hexadecimal string derived from 6 cryptographically random bytes (`crypto.randomBytes`). The uniqueness is enforced at both the application and database level (unique constraint on the `ticketCode` column).

**What prevents someone from using an expired or cancelled ticket?**

The check-in endpoint checks the ticket's current status before proceeding. A `CANCELLED` or already `CHECKED_IN` ticket is rejected with a clear error. The ticket code alone is not sufficient — the status must be `CONFIRMED`.

**Can the same user buy multiple tickets to the same event?**

Yes, within the per-purchase quantity limit (1–10 tickets per transaction). Multiple purchases by the same user are allowed as long as capacity remains.

**How is the platform secured between services internally?**

All microservices run on Docker's internal network and are not reachable from the public internet. The API Gateway is the only publicly exposed port. Inter-service calls use Docker's internal DNS (`http://events-service:3003`), which is only resolvable inside the container network. The gateway passes authenticated user identity via internal headers (`x-user-id`, `x-user-role`) that clients cannot forge.

**Is this ready for production?**

The architecture, security model, and data integrity guarantees are production-grade. For a production deployment, the following would be added: HTTPS/TLS termination at the gateway, secrets management (e.g. AWS Secrets Manager instead of env vars), a managed Kafka cluster (e.g. Confluent Cloud), a managed PostgreSQL instance (e.g. RDS), and a CI/CD pipeline. The codebase is structured to support all of these with minimal changes.

**Can new services be added without changing existing ones?**

Yes. Any new service simply subscribes to the relevant Kafka topics. For example, a future analytics service could listen to `ticket.purchased` and `event.created` events to build dashboards without touching any existing service. This is the core extensibility guarantee of the event-driven architecture.

**What user roles exist and what can each do?**

| Action | USER | ORGANIZER | ADMIN |
|---|---|---|---|
| Browse published events | ✓ | ✓ | ✓ |
| Purchase tickets | ✓ | ✓ | ✓ |
| Create events | ✓ | ✓ | ✓ |
| Publish / cancel own events | — | ✓ | ✓ |
| Update / cancel any event | — | — | ✓ |
| Check in attendees | — | ✓ (own events) | ✓ |
| View tickets for an event | — | ✓ (own events) | ✓ |
