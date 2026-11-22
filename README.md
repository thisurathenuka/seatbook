# SeatBook

A backend service for booking seats at events. Many users can try to book the same seat at the same time, and the system guarantees that **only one of them gets it**.

> Status: work in progress. Tick the boxes in the [Roadmap](#roadmap) as each part is built and tested. Only things that are ticked are claims I can explain.

This is a learning project. I wrote the backend myself and used AI to explain concepts I did not know. There is no frontend; the API is exercised through Swagger UI and tests.

---

## What it does

1. An **admin** creates an event and its seats.
2. A **user** registers and logs in (JWT).
3. A user **holds** a seat. The seat is reserved for 10 minutes.
4. The user **pays** (a fake payment provider is called).
5. On success the booking is **confirmed**. On failure or timeout the seat is released.
6. If the user does nothing, a scheduled job **expires** the hold and frees the seat.

### Booking life cycle

```
            hold                pay ok
  AVAILABLE ------> HELD ---------------> BOOKED
      ^              |
      |              | payment failed / 10 min expired / user cancels
      +--------------+
```

| Booking status | Meaning |
|---|---|
| `PENDING` | Seat is held, waiting for payment |
| `CONFIRMED` | Payment succeeded, seat is booked |
| `CANCELLED` | User cancelled before paying |
| `EXPIRED` | Hold timed out |
| `FAILED` | Payment failed |

### Business rules

- A seat can have **at most one** active booking (`PENDING` or `CONFIRMED`).
- A hold lasts **10 minutes** (configurable: `seatbook.hold.minutes`).
- A user can hold **at most 4 seats** per event.
- Only the user who made a booking can pay for or cancel it.
- A confirmed booking cannot be cancelled in v1.
- Sending the same request twice with the same `Idempotency-Key` must not create two bookings or two charges. The second call returns the first result.
- The payment provider is slow and unreliable on purpose. Calls have a timeout and a limited number of retries. If the outcome is unknown after retries, the booking stays `PENDING` and is resolved by the expiry job, never double-charged.

---

## Tech stack

| Area | Choice |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3 |
| Security | Spring Security, JWT (access token) |
| Persistence | Spring Data JPA (Hibernate), PostgreSQL |
| Migrations | Liquibase |
| Concurrency | Optimistic locking (`@Version`) |
| Scheduling | `@Scheduled` expiry job |
| Resilience | Timeout and retry on the payment call (Resilience4j or Spring Retry) |
| API docs | springdoc-openapi (Swagger UI) |
| Testing | JUnit 5, Mockito, Testcontainers (PostgreSQL) |
| Build and run | Maven, Docker, Docker Compose |
| CI | GitHub Actions |
| Optional | RabbitMQ and a separate notification service |

---

## Data model

```
users(id, email unique, password_hash, role, created_at)

events(id, name, venue, starts_at, created_at)

seats(id, event_id -> events, label, price, status, version)
  unique(event_id, label)

bookings(id, user_id -> users, seat_id -> seats, status,
         idempotency_key, expires_at, created_at, updated_at)
  unique(user_id, idempotency_key)

payments(id, booking_id -> bookings, amount, status,
         provider_ref, created_at)
```

`seats.version` is the optimistic-lock column. If two transactions update the same seat, one commits and the other gets an `OptimisticLockException`, which the API turns into `409 Conflict`.

---

## API

Base path: `/api/v1`. Authenticated endpoints need `Authorization: Bearer <token>`.

### Auth
| Method | Path | Description |
|---|---|---|
| POST | `/auth/register` | Create a user |
| POST | `/auth/login` | Returns a JWT |

### Events and seats
| Method | Path | Role | Description |
|---|---|---|---|
| POST | `/events` | ADMIN | Create an event with its seats |
| GET | `/events` | any | List events (paged) |
| GET | `/events/{id}/seats` | any | List seats and their status |

### Bookings
| Method | Path | Description |
|---|---|---|
| POST | `/bookings` | Hold a seat. Body: `{ "seatId": 1 }`. Header: `Idempotency-Key` required |
| POST | `/bookings/{id}/pay` | Pay for a held seat. Header: `Idempotency-Key` required |
| DELETE | `/bookings/{id}` | Cancel a `PENDING` booking |
| GET | `/bookings/me` | My bookings |

### Error responses

All errors use one JSON shape: `{ "timestamp", "status", "error", "message", "path" }`.

| Status | When |
|---|---|
| 400 | Validation failed |
| 401 | Missing or invalid token |
| 403 | Not allowed (wrong role or not your booking) |
| 404 | Event, seat or booking not found |
| 409 | Seat already taken, concurrent update, or hold limit reached |
| 502 / 504 | Payment provider failed or timed out |

---

## Behaviour I test for

These are the tests that matter most. Each one should be written before I claim the feature.

**Unit (JUnit and Mockito)**
- Holding an available seat creates a `PENDING` booking with `expires_at` 10 minutes ahead.
- Holding a taken seat is rejected.
- Hold limit of 4 per user per event is enforced.
- Payment success moves the booking to `CONFIRMED`; failure moves it to `FAILED` and frees the seat.
- Expiry job only touches `PENDING` bookings whose `expires_at` has passed.

**Integration (Testcontainers + PostgreSQL)**
- Liquibase migrations apply to a clean database.
- Repository queries return what the service expects.
- Full flow: register, login, hold, pay, see confirmed booking.

**Concurrency**
- 50 threads try to hold the same seat at once. Exactly **one** succeeds, the other 49 get `409`. The database has exactly one active booking for that seat.

**Idempotency**
- The same `POST /bookings` sent twice with the same key creates one booking and returns the same response.

**Resilience**
- A payment stub that is slow causes a timeout, then a retry, then a clean error. No double charge.

---

## Run locally

Requirements: Java 17, Maven, Docker.

```bash
# start PostgreSQL (and the app, once the Dockerfile exists)
docker compose up -d

# run the app
./mvnw spring-boot:run

# run all tests (needs Docker for Testcontainers)
./mvnw verify
```

Swagger UI: `http://localhost:8080/swagger-ui.html`

Configuration is in `application.yml`. Secrets (JWT key, DB password) come from environment variables, never from the repo.

| Variable | Purpose |
|---|---|
| `DB_URL`, `DB_USER`, `DB_PASSWORD` | PostgreSQL connection |
| `JWT_SECRET` | Signing key for tokens |
| `SEATBOOK_HOLD_MINUTES` | Hold duration (default 10) |

---

## Project layout

```
seatbook/
  src/main/java/.../
    auth/        controllers, JWT filter, security config
    event/       event and seat entities, service, controller
    booking/     booking entity, service, scheduler, controller
    payment/     payment client (stub), retry and timeout config
    common/      error handling, base classes
  src/main/resources/
    db/changelog/   Liquibase changelogs
    application.yml
  src/test/java/.../   unit, integration and concurrency tests
  Dockerfile
  docker-compose.yml
  .github/workflows/ci.yml
```

---

## Roadmap

Tick each box only when the feature works **and** has a test.

### Foundation
- [ ] Project skeleton, Maven, Java 17
- [ ] Docker Compose with PostgreSQL
- [ ] Liquibase changelogs for all tables
- [ ] Global error handler with the standard error shape
- [ ] Swagger UI

### Auth
- [ ] Register and login
- [ ] JWT filter and role-based access (USER, ADMIN)

### Events and seats
- [ ] Create event with seats (admin)
- [ ] List events and seats

### Booking
- [ ] Hold a seat (`PENDING`)
- [ ] Optimistic locking on seats
- [ ] Hold limit per user
- [ ] Idempotency key on hold and pay
- [ ] Cancel a pending booking
- [ ] Expiry scheduler

### Payment
- [ ] Payment stub (random delay and failures)
- [ ] Timeout and retry
- [ ] Confirm or fail the booking from the result

### Quality
- [ ] Unit tests for the service layer
- [ ] Integration tests with Testcontainers
- [ ] Concurrency test (50 threads, one winner)
- [ ] GitHub Actions: build and test on every push
- [ ] Dockerfile for the app

### Optional
- [ ] RabbitMQ event on booking confirmed
- [ ] Separate notification service that consumes it

---

## What I learned

I will fill this in as I build. For each hard part I will write three lines in my own words:

1. What the problem was.
2. What I tried first and why it failed.
3. What the final solution is and why it works.

Candidates: optimistic vs pessimistic locking, why idempotency keys are needed, why the payment call must not hold a database transaction open, how Testcontainers differs from an in-memory database.

## Known limitations

- No refunds or cancelling confirmed bookings.
- The payment provider is a stub, not a real gateway.
- No frontend.
- Single instance only; the expiry job is not yet safe to run on several instances at once.