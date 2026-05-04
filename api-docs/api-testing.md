# EventFlow API — QA Testing Guide

All requests go through the **API Gateway** at `http://localhost:3000`. Never call microservice ports (3001, 3003, 3004) directly.

---

## Prerequisites

- API Gateway and all services running (see root `README.md`)
- Postman v10+ installed
- Import `eventflow-postman-collection.json` (in this folder) into Postman

---

## Environment Setup in Postman

Create a Postman Environment called **EventFlow Local** with these variables:

| Variable | Initial Value | Notes |
|---|---|---|
| `base_url` | `http://localhost:3000` | API Gateway |
| `access_token` | *(empty)* | Populated automatically after login |
| `event_id` | *(empty)* | Fill in after creating an event |
| `ticket_id` | *(empty)* | Fill in after purchasing a ticket |
| `ticket_code` | *(empty)* | Fill in after purchasing a ticket |

The collection's login request has a **Tests** script that sets `access_token` automatically. All authenticated requests use `{{access_token}}` as a Bearer token.

---

## Step 1 — Health Check

Confirm the gateway is reachable before running any other tests.

**Request:** `GET /`

**Expected response:**
```json
{
  "success": true,
  "data": "Hello World!",
  "meta": { "timestamp": "...", "path": "/", "method": "GET" }
}
```

If this fails, the API Gateway is not running.

---

## Step 2 — Authentication

### 2.1 Register a new user

**Request:** `POST /auth/register`

**Body:**
```json
{
  "name": "Jane QA",
  "email": "jane.qa@example.com",
  "password": "password123"
}
```

**Expected:** `201 Created`
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJ...",
    "user": { "id": "...", "email": "jane.qa@example.com", "name": "Jane QA", "role": "USER" }
  }
}
```

**Edge cases to test:**
- Duplicate email → `409 Conflict`
- Invalid email format → `400 Bad Request`
- Password shorter than 6 characters → `400 Bad Request`
- Missing required field → `400 Bad Request`
- Trigger throttle: send more than 3 requests within 60 seconds → `429 Too Many Requests`

---

### 2.2 Login

**Request:** `POST /auth/login`

**Body:**
```json
{
  "email": "jane.qa@example.com",
  "password": "password123"
}
```

**Expected:** `200 OK`
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJ...",
    "user": { "id": "...", "email": "jane.qa@example.com", "name": "Jane QA", "role": "USER" }
  }
}
```

Copy the `accessToken` value into the `access_token` Postman environment variable (done automatically if using the collection).

**Edge cases to test:**
- Wrong password → `401 Unauthorized`
- Non-existent email → `401 Unauthorized`
- Trigger throttle: more than 5 requests in 60 seconds → `429 Too Many Requests`

---

### 2.3 Get Profile

**Request:** `GET /auth/profile`

**Headers:** `Authorization: Bearer {{access_token}}`

**Expected:** `200 OK`
```json
{
  "success": true,
  "data": { "id": "...", "email": "jane.qa@example.com", "name": "Jane QA", "role": "USER" }
}
```

**Edge cases to test:**
- No token → `401 Unauthorized`
- Malformed or expired token → `401 Unauthorized`

---

## Step 3 — Events

> Register a second user with role `ORGANIZER` for full event lifecycle testing. Currently role is set at the database level — update the `role` column to `ORGANIZER` directly, or register and manually update via DB.

### 3.1 Get All Events (public)

**Request:** `GET /events`

**Auth required:** No

**Expected:** `200 OK` — array of events (may be empty on fresh setup)

---

### 3.2 Create an Event

**Request:** `POST /events`

**Headers:** `Authorization: Bearer {{access_token}}`

**Body:**
```json
{
  "title": "QA Test Event",
  "description": "An event created during QA testing",
  "date": "2026-12-01T18:00:00.000Z",
  "location": "Test Venue, Manila",
  "capacity": 100,
  "price": 500
}
```

**Expected:** `201 Created`
```json
{
  "success": true,
  "data": {
    "id": "...",
    "title": "QA Test Event",
    "status": "DRAFT",
    ...
  }
}
```

Save the returned `id` into the `event_id` Postman variable.

**Edge cases to test:**
- Missing required field (`title`, `date`, `location`, `capacity`) → `400 Bad Request`
- `capacity` less than 1 → `400 Bad Request`
- `price` negative → `400 Bad Request`
- No token → `401 Unauthorized`

---

### 3.3 Get Single Event

**Request:** `GET /events/{{event_id}}`

**Auth required:** No

**Expected:** `200 OK` — the event object

**Edge cases to test:**
- Non-existent UUID → `404 Not Found`
- Invalid UUID format → `400 Bad Request`

---

### 3.4 Update an Event

**Request:** `PUT /events/{{event_id}}`

**Headers:** `Authorization: Bearer {{access_token}}`

**Body (all fields optional):**
```json
{
  "title": "QA Test Event — Updated",
  "capacity": 200
}
```

**Expected:** `200 OK` — updated event object

**Edge cases to test:**
- Updating another user's event → `403 Forbidden`
- No token → `401 Unauthorized`

---

### 3.5 Publish an Event

**Request:** `POST /events/{{event_id}}/publish`

**Headers:** `Authorization: Bearer {{access_token}}`

**Body:** none

**Expected:** `200 OK` — event with `"status": "PUBLISHED"`

> Only a `PUBLISHED` event can be purchased. Run this before the ticket purchase tests.

**Edge cases to test:**
- Publishing another user's event → `403 Forbidden`
- Publishing an already-cancelled event → `400 Bad Request`

---

### 3.6 Get My Events

**Request:** `GET /events/my-events`

**Headers:** `Authorization: Bearer {{access_token}}`

**Expected:** `200 OK` — array of events belonging to the authenticated user

---

### 3.7 Cancel an Event

**Request:** `POST /events/{{event_id}}/cancel`

**Headers:** `Authorization: Bearer {{access_token}}`

> Run this **after** finishing ticket tests — cancelling an event may affect ticket status.

**Expected:** `200 OK` — event with `"status": "CANCELLED"`

---

## Step 4 — Tickets

All ticket endpoints require a valid JWT token.

> The event must be in `PUBLISHED` status before purchasing tickets (run Step 3.5 first).

### 4.1 Purchase a Ticket

**Request:** `POST /tickets/purchase`

**Headers:** `Authorization: Bearer {{access_token}}`

**Body:**
```json
{
  "eventId": "{{event_id}}",
  "quantity": 2
}
```

**Expected:** `201 Created`
```json
{
  "success": true,
  "data": {
    "id": "...",
    "ticketCode": "TKT-XXXXXXXX",
    "status": "CONFIRMED",
    "quantity": 2,
    "totalPrice": 1000,
    ...
  }
}
```

Save `id` → `ticket_id` and `ticketCode` → `ticket_code` in Postman variables.

**Edge cases to test:**
- `quantity` of 0 → `400 Bad Request`
- `quantity` greater than 10 → `400 Bad Request`
- Non-existent `eventId` → `404 Not Found`
- Event in `DRAFT` status → `400 Bad Request`
- Purchasing more than remaining capacity → `400 Bad Request`
- No token → `401 Unauthorized`

---

### 4.2 Get My Tickets

**Request:** `GET /tickets/my-tickets`

**Headers:** `Authorization: Bearer {{access_token}}`

**Expected:** `200 OK` — array of the user's tickets

---

### 4.3 Get Single Ticket

**Request:** `GET /tickets/{{ticket_id}}`

**Headers:** `Authorization: Bearer {{access_token}}`

**Expected:** `200 OK` — ticket object

**Edge cases to test:**
- Another user's ticket ID → `403 Forbidden` or `404 Not Found`
- Non-existent UUID → `404 Not Found`

---

### 4.4 Check In a Ticket

**Request:** `POST /tickets/check-in`

**Headers:** `Authorization: Bearer {{access_token}}`

**Body:**
```json
{
  "ticketCode": "{{ticket_code}}"
}
```

**Expected:** `200 OK` — ticket with `"status": "CHECKED_IN"`

**Edge cases to test:**
- Already checked-in ticket code → `400 Bad Request`
- Invalid or non-existent ticket code → `404 Not Found`

---

### 4.5 Get Tickets for an Event

**Request:** `GET /tickets/event/{{event_id}}`

**Headers:** `Authorization: Bearer {{access_token}}`

**Expected:** `200 OK` — array of tickets for that event

---

### 4.6 Cancel a Ticket

**Request:** `POST /tickets/{{ticket_id}}/cancel`

**Headers:** `Authorization: Bearer {{access_token}}`

**Expected:** `200 OK` — ticket with `"status": "CANCELLED"`

**Edge cases to test:**
- Already-cancelled ticket → `400 Bad Request`
- Already-checked-in ticket → `400 Bad Request`
- Another user's ticket → `403 Forbidden`

---

## Step 5 — Global Error Shape Verification

For every error response, confirm it matches this shape:

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "...",
    "timestamp": "...",
    "path": "..."
  }
}
```

Common `code` values: `BAD_REQUEST`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `TOO_MANY_REQUESTS`, `INTERNAL_SERVER_ERROR`.

---

## Recommended Test Order

Run requests in this sequence to avoid dependency failures:

1. `GET /` — health check
2. `POST /auth/register`
3. `POST /auth/login` — saves token
4. `GET /auth/profile`
5. `POST /events` — saves event_id
6. `GET /events`
7. `GET /events/{{event_id}}`
8. `PUT /events/{{event_id}}`
9. `POST /events/{{event_id}}/publish`
10. `GET /events/my-events`
11. `POST /tickets/purchase` — saves ticket_id, ticket_code
12. `GET /tickets/my-tickets`
13. `GET /tickets/{{ticket_id}}`
14. `GET /tickets/event/{{event_id}}`
15. `POST /tickets/check-in`
16. `POST /tickets/{{ticket_id}}/cancel`
17. `POST /events/{{event_id}}/cancel`
