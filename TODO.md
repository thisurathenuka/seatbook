# SeatBook – My To-Do List

Work top to bottom. Don't skip ahead. One small task at a time.

**Rules for every task**
- [ ] Write the code myself. Use AI only to explain, never to paste.
- [ ] After each task: explain it aloud in my own words, then write a 3-line note in Obsidian.
- [ ] Commit after each finished task with a clear message.
- [ ] Tick a README box only when the feature works AND has a test.

---

## Step 0: Setup (about 1 evening)
- [x] Install JDK 17, Maven, Docker Desktop, IntelliJ, Postman. Check: `java -version` shows 17.
- [x] Create the GitHub repo `seatbook`. Add a `.gitignore` for Java and IntelliJ.
- [x] Generate the project at start.spring.io: Maven, Java 17, Spring Boot 3.x, deps: Web, Security, Data JPA, Validation, PostgreSQL Driver, Liquibase, Lombok.
- [ ] Add springdoc-openapi to the pom. Open Swagger UI to confirm it runs.
- [ ] Add `docker-compose.yml` with PostgreSQL. Run `docker compose up -d`.
- [ ] Connect the app to the database through env variables. App starts without errors.
- [ ] Push the first commit and paste in the README.

## Step 1: Database and errors
- [ ] Write Liquibase changelogs: users, events, seats, bookings, payments.
- [ ] Add the unique constraints (event+seat label, user+idempotency key).
- [ ] Create the JPA entities. Add `@Version` to Seat.
- [ ] Write the global exception handler with the standard error shape.
- [ ] Test: migrations run on a clean database.

## Step 2: Auth
- [ ] Register endpoint with password hashing (BCrypt).
- [ ] Login endpoint that returns a JWT.
- [ ] JWT filter that reads the token and sets the user.
- [ ] Roles: USER and ADMIN. Lock the admin endpoints.
- [ ] Tests: register, login, wrong password, no token gives 401, wrong role gives 403.

## Step 3: Events and seats
- [ ] Create event with seats (admin only).
- [ ] List events with paging.
- [ ] List seats of an event with their status.
- [ ] Validation on request bodies.
- [ ] Tests for each endpoint.

## Step 4: Booking (the core)
- [ ] Hold a seat: create a PENDING booking, set `expires_at`.
- [ ] Reject a seat that is already taken (409).
- [ ] Optimistic locking: map `OptimisticLockException` to 409.
- [ ] Hold limit of 4 per user per event.
- [ ] Idempotency key on hold: same key, same result, no duplicate.
- [ ] Cancel a pending booking.
- [ ] `GET /bookings/me`.
- [ ] Concurrency test: 50 threads, one seat, exactly one winner. This is the key test.

## Step 5: Payment
- [ ] Write the payment stub with random delay and random failures.
- [ ] Call it from `POST /bookings/{id}/pay`.
- [ ] Add a timeout and limited retries.
- [ ] Success: booking CONFIRMED. Failure: FAILED, seat freed.
- [ ] Unknown result after retries: stay PENDING, no double charge.
- [ ] Idempotency key on pay.
- [ ] Tests: success, failure, timeout, retry.

## Step 6: Expiry job
- [ ] `@Scheduled` job that expires PENDING bookings past `expires_at` and frees the seats.
- [ ] Test it with a short hold time.

## Step 7: Quality and shipping
- [ ] Unit tests for the service layer with Mockito.
- [ ] Integration tests with Testcontainers (PostgreSQL).
- [ ] Dockerfile for the app. Whole system runs with `docker compose up`.
- [ ] GitHub Actions: build and test on every push. Add the badge to the README.
- [ ] Fill in the "What I learned" section of the README.
- [ ] Clean up: no secrets in the repo, no commented-out code, clear commit history.

## Step 8: Showcase
- [ ] Pin SeatBook on my GitHub profile.
- [ ] Add a short project line to my profile README.
- [ ] Add the GitHub link to my CV header.
- [ ] Write a blog post: "How I stopped two people booking the same seat".
- [ ] Practise the 2-minute explanation aloud: problem, design, the hard part, what I'd improve.

## Optional (only after everything above)
- [ ] RabbitMQ event on booking confirmed.
- [ ] Separate notification service that consumes it.

---

## Daily routine
1. Pick the next unticked task. Only that one.
2. Try it myself for 30 minutes before asking AI.
3. Ask AI to explain, not to write. Then rewrite from memory.
4. Explain it aloud. Write the 3-line note.
5. Commit. Tick the box.