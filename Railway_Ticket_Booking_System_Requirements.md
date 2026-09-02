# Railway Ticket Booking System

## Requirements & Development Guide

Java + Spring Boot | Two-phase backend project | Version 1.0

Purpose: Build a realistic railway ticket booking backend independently. Phase 1 focuses on REST APIs, database design, validation, transactions and exception handling. Phase 2 adds security and microservices.

## 1. Project Overview

The system should support the railway booking lifecycle: manage trains and stations, search trains, check availability, create a booking, view bookings, cancel a booking, and maintain consistent seat inventory.

Keep the first version manageable. This is not a full IRCTC clone. The goal is a clean, production-style Spring Boot backend that demonstrates good engineering decisions.

## 2. Technology Stack

| Area | Choice |
| --- | --- |
| Language | Java 25 |
| Framework | Spring Boot 4.1.1 (Web MVC, Data JPA, Validation, Actuator) |
| Build tool | Maven |
| Database | PostgreSQL |
| Migrations | Flyway |
| API docs | springdoc-openapi (Swagger UI) |
| Testing | JUnit 5, Spring Boot test starters |
| Phase 2 | Spring Security, JWT, OAuth2/OIDC |

Note: `springdoc-openapi` and the Phase 2 security dependencies are not in `pom.xml` yet. Add them when you reach those steps.

## 3. Roles

CUSTOMER: search trains, check availability, book, view own bookings, cancel own bookings.

ADMIN: manage trains, stations, routes, schedules, coaches and inventory.

Phase 2: enforce these permissions with Spring Security; customers must never access another customer's bookings.

## 4. Phase 1 — Functional Requirements

### 4.1 Station Management

Create station with unique code and name.

Get all stations.

Get station by ID/code.

Update station.

Reject duplicate station codes.

Why: Station codes are stable identifiers and are better than free-text station names in routes.

### 4.2 Train Management

Create train with unique train number, name and active/inactive status.

Get train by ID/number.

List trains.

Update train.

Deactivate instead of deleting trains with historical bookings.

Why: Keeping historical data is important; physical DELETE can break history and foreign-key relationships.

### 4.3 Route & Schedule

A train has an ordered list of stops.

Each stop has station, sequence number, arrival time and departure time.

Sequence number is unique within a train.

Departure cannot be earlier than arrival at the same stop.

For a normal route, later stops must have later schedule times.

Provide an API to view a train's complete ordered route.

Design: Do not store a route as a comma-separated string. Model train stops as relational data.

### 4.4 Coaches & Seats

Support a manageable set of classes such as SL, 3A and 2A.

A train has coaches; a coach has seats.

Each seat has coach/seat number and class.

Seat availability is journey-date/route-specific; do not simply mark a physical seat permanently BOOKED.

Initially allocate seats from available inventory for the requested journey/class.

Important: A seat belongs to a train/coach, but availability belongs to a particular journey date and route segment.

### 4.5 Train Search

Request: source station, destination station, journey date, optional class.

Source and destination must differ.

Source stop must occur before destination stop.

Only active trains are returned.

Return train number/name, departure, arrival, duration and available seats.

Return DTOs, not JPA entities.

### 4.6 Booking

Request: user/customer reference, train/journey, source, destination, date, class, passenger details and passenger count.

Validate all input.

Verify the train runs on the requested date.

Verify source occurs before destination.

Verify enough inventory exists.

Create booking and passenger/seat records in one transaction.

Generate a unique booking/PNR.

Reserve/reduce inventory atomically.

Return booking details and status.

Critical: Two users must not both successfully reserve the same last seat. This is the most important concurrency problem in the project.

### 4.7 Booking Retrieval

Get booking by ID/PNR.

List bookings for a customer.

Return passengers, journey, train, class, seats, fare and status.

Do not expose unnecessary internal/sensitive fields.

### 4.8 Cancellation

Cancel an eligible booking.

Change status to CANCELLED.

Release inventory correctly.

Do not physically delete booking history.

Reject repeated/invalid cancellation.

Why: Status changes preserve business history better than DELETE.

## 5. REST API Contract — Phase 1

Best practice: Use /api/v1 so future breaking API changes do not silently break clients.

| Method | Endpoint | Purpose | Success status |
| --- | --- | --- | --- |
| POST | /api/v1/stations | Create station | 201 |
| GET | /api/v1/stations | List stations (paginated) | 200 |
| GET | /api/v1/stations/{id} | Get station | 200 |
| PUT | /api/v1/stations/{id} | Update station | 200 |
| POST | /api/v1/trains | Create train | 201 |
| GET | /api/v1/trains | List trains (paginated) | 200 |
| GET | /api/v1/trains/{id} | Get train | 200 |
| PUT | /api/v1/trains/{id} | Update train | 200 |
| PATCH | /api/v1/trains/{id}/status | Activate/deactivate train | 200 |
| POST | /api/v1/trains/{id}/stops | Add stop to route | 201 |
| GET | /api/v1/trains/{id}/route | Get ordered route | 200 |
| POST | /api/v1/trains/{id}/coaches | Add coach with seats | 201 |
| GET | /api/v1/trains/search | Search trains (source, destination, date, class) | 200 |
| GET | /api/v1/availability | Seat availability for a train/date/class | 200 |
| POST | /api/v1/bookings | Create booking | 201 |
| GET | /api/v1/bookings/{pnr} | Get booking by PNR | 200 |
| GET | /api/v1/bookings?customerId= | List a customer's bookings | 200 |
| POST | /api/v1/bookings/{pnr}/cancel | Cancel booking | 200 |

Status codes: 400 for invalid input, 404 for missing resources, 409 for duplicates and conflicts such as no seats left.

## 6. Database Schema — Phase 1

Database rules: Use PKs, FKs, UNIQUE, NOT NULL and useful indexes. Application validation is not enough; database constraints are a second line of defense.

Core tables:

- `stations` — id, code (UNIQUE), name.
- `trains` — id, train_number (UNIQUE), name, active.
- `train_stops` — id, train_id FK, station_id FK, sequence_no, arrival_time, departure_time. UNIQUE (train_id, sequence_no).
- `coaches` — id, train_id FK, coach_number, travel_class. UNIQUE (train_id, coach_number).
- `seats` — id, coach_id FK, seat_number. UNIQUE (coach_id, seat_number).
- `users` — id, name, email (UNIQUE). Password column is added in Phase 2.
- `seat_inventory` — id, train_id FK, journey_date, travel_class, total_seats, available_seats, version (for optimistic locking). UNIQUE (train_id, journey_date, travel_class).
- `bookings` — id, pnr (UNIQUE), user_id FK, train_id FK, source_station_id FK, destination_station_id FK, journey_date, travel_class, fare, status, created_at.
- `passengers` — id, booking_id FK, name, age, gender, seat_number.

Keep migrations in `src/main/resources/db/migration` as `V1__...sql`, `V2__...sql` and never edit a migration that has already run.

## 7. Key Relationships

Train 1→many TrainStops.

Station 1→many TrainStops.

Train 1→many Coaches.

Coach 1→many Seats.

User 1→many Bookings.

Booking 1→many Passengers.

Train + journey date + class 1→1 SeatInventory row (unique per combination).

## 8. Validation

Best practice: Use request DTOs with @Valid. Do not bind HTTP input directly to JPA entities.

Field-level validation (Bean Validation annotations):

- @NotBlank on station code/name, train number/name, passenger name.
- @Size and @Pattern where a format matters, for example a station code of 2-5 uppercase letters.
- @NotNull on journey date, class, source and destination.
- @FutureOrPresent on journey date.
- @Min / @Max on passenger age and @Positive on passenger count.
- @Valid on nested passenger lists so each element is validated too.

Business validation (in the service layer, not annotations):

- Source and destination must differ and source must come earlier in the route.
- The train must run on the requested date and be active.
- Enough seats must be available for the requested class.
- Only bookings in CONFIRMED status can be cancelled.

## 9. Exception Handling

### 9.1 Custom Exceptions

ResourceNotFoundException

DuplicateResourceException

InvalidJourneyException

InsufficientSeatException

BookingNotAllowedException

### 9.2 Global Exception Handler

Use @RestControllerAdvice and @ExceptionHandler. Controllers should focus on HTTP concerns and the happy path; the handler converts exceptions to consistent API responses.

Example error response:
{
  "timestamp": "2026-08-28T12:00:00Z",
  "status": 404,
  "error": "RESOURCE_NOT_FOUND",
  "message": "Train not found",
  "path": "/api/v1/trains/123",
  "traceId": "..."
}

Security: Never return stack traces, SQL errors, secrets or internal implementation details to clients.

## 10. Layered Architecture

Recommended flow: Controller → Service → Repository → Database

Controller: HTTP, DTOs and validation entry point.

Service: business rules and orchestration.

Repository: database access.

Entity: persistence model.

DTO: API contract.

Mapper: Entity ↔ DTO.

Exception layer: business exceptions and global error handling.

Why: Separation makes code easier to test, maintain and change. Avoid business logic in controllers/repositories.

## 11. DTO vs Entity

Create request DTOs such as CreateTrainRequest and CreateBookingRequest.

Create response DTOs such as TrainResponse and BookingResponse.

Do not return JPA entities directly.

Do not accept JPA entities directly as request bodies.

Why: Database models and public APIs evolve for different reasons. DTOs protect your API contract and prevent accidental field exposure.

## 12. Transactions & Concurrency

Use @Transactional around a booking operation that changes multiple tables.

A booking should either complete all required database changes or none.

Learn optimistic locking with @Version and/or suitable database locking.

Enforce uniqueness at database level.

Do not rely on 'check availability, then save' without considering concurrent requests.

Test two users trying to book the last seat.

Core lesson: Concurrency bugs often do not appear during manual testing. Booking must be designed for atomicity and consistency.

## 13. What ACID Means for Booking

Atomicity: all booking changes succeed or roll back.

Consistency: database rules remain valid.

Isolation: concurrent bookings do not corrupt inventory.

Durability: committed data survives failures.

Learning goal: Understand where @Transactional belongs and why booking is one business transaction.

## 14. API Best Practices

Use nouns: /trains, /bookings, /stations.

Use HTTP methods correctly.

Return meaningful status codes.

Paginate large lists.

Use filtering/sorting where useful.

Keep JSON structure consistent.

Version APIs.

Use PNR as a useful booking lookup identifier.

Document with OpenAPI/Swagger.

## 15. Project Structure

com.example.railway
├── config
├── controller
├── dto
│   ├── request
│   └── response
├── entity
├── repository
├── service
├── mapper
├── exception
├── validation
└── util

Keep configuration in application.yml/properties or environment variables.

Never hard-code passwords, JWT secrets or OAuth client secrets.

Use local/test/prod profiles when needed.

## 16. Logging & Observability

Use SLF4J/Logback.

Log important business events and failures.

Never log passwords, tokens or unnecessary personal data.

Use correlation/trace IDs where practical.

Use structured logs in production.

## 17. Testing

Target: Do not chase coverage blindly. Prioritize important business behavior and failure paths.

Unit tests (service layer, repositories mocked):

- Booking fails when seats are insufficient.
- Booking fails when source is after destination.
- Cancelling an already cancelled booking is rejected.
- Fare/seat allocation logic behaves as expected.

Integration tests (@SpringBootTest or @DataJpaTest with a real database):

- Duplicate station code is rejected by the unique constraint.
- A successful booking writes booking, passengers and reduced inventory together.
- A failed booking rolls back completely and leaves inventory unchanged.
- Two concurrent bookings for the last seat: exactly one succeeds. Use an ExecutorService with two threads.

Web layer tests (@WebMvcTest): validation errors return 400 and the global handler's error shape.

## 18. Phase 1 Definition of Done

Schema and relationships implemented.

Station/train/route/coach/seat/search/availability/booking/cancellation APIs work.

DTOs used at API boundary.

Validation implemented.

Global exception handling implemented.

Booking is transactional.

Concurrency risk addressed and tested.

OpenAPI documentation available.

Core unit/integration tests pass.

README contains setup and API usage.

No secrets committed to Git.

## 19. Phase 2 — Authentication & Authorization

### 19.1 Authentication

Authentication answers: 'Who are you?'

Implement login with Spring Security.

JWT: authenticate → issue access token → validate protected requests.

Keep access tokens short-lived.

Design refresh-token handling separately.

Hash passwords with BCrypt/Argon2; never store plain passwords.

### 19.2 Authorization

Authorization answers: 'What are you allowed to do?'

CUSTOMER manages only their own bookings.

ADMIN manages operational data.

Use endpoint authorization plus service-level ownership checks.

Never rely only on frontend controls.

### 19.3 JWT Concepts

Header, payload, signature.

Claims: subject, roles, expiry.

Access vs refresh tokens.

Signature validation and expiry.

Stateless authentication.

Token transport/storage security.

Important: JWT is a token format, not a complete security architecture. Spring Security handles the security pipeline, but you must configure expiry, validation and authorization correctly.

### 19.4 OAuth2 & OpenID Connect

OAuth2 is an authorization framework.

OIDC adds authentication/identity on top of OAuth2.

Learn Authorization Code flow for browser applications.

Use an Identity Provider such as Microsoft Entra ID when external login is required.

Do not implement OAuth2 cryptography/protocols yourself.

### 19.5 Command Query Responsibility Segregation (CQRS)

Use CQRS in Phase 2 to separate operations that change system state (commands) from operations that only read data (queries).

**Commands**
- Create booking.
- Cancel booking.
- Create/update/deactivate trains, stations and operational data.
- Reserve or release seat inventory.

**Queries**
- Search trains.
- Check seat availability.
- Get booking by ID/PNR.
- List customer bookings.
- View train routes and schedules.

**CQRS principles**
- Commands perform state-changing business operations and should not be used for read-only retrieval.
- Queries retrieve data and should not change system state.
- Keep command and query responsibilities clearly separated at the application/service layer, for example a BookingCommandService and a BookingQueryService.
- Initially, commands and queries may use the same database.
- Do not introduce separate read/write databases unless there is a real requirement for independent scaling, performance, or reporting.
- DTOs should be designed according to the needs of each command or query rather than exposing entities directly.
- Query methods can be marked @Transactional(readOnly = true).

**Learning goal:** Understand why reads and writes can have different models and responsibilities, and how CQRS can make complex booking workflows and read-heavy operations easier to evolve.

## 20. Phase 2 — Microservices

First make Phase 1 a clean modular monolith. Then extract services around real business boundaries. Do not split into microservices only to create more applications.

### 20.1 Service Discovery

Learn service registration/discovery, health checks and load-balanced service communication.

Understand that gateway and service discovery solve different problems.

## 21. Inter-Service Communication

Use REST for suitable synchronous operations.

Use messaging/events for decoupled workflows such as notifications.

Define clear contracts.

Use timeouts.

Retry transient failures carefully.

Never blindly retry operations that can create duplicates.

Use idempotency for retryable business operations.

## 22. Idempotency

An idempotent operation can be safely repeated without creating another business effect.

Example: the client times out after creating a booking and retries POST /bookings. Without protection, two bookings might be created. For critical creation APIs, consider an Idempotency-Key and persist the key/result.

Why: Networks fail. The client cannot always know whether the server already processed a request.

## 23. Notification Service

Booking created/cancelled events can trigger notifications.

Prefer asynchronous processing so email/SMS does not block booking.

Track notification status.

Retry transient failures with limits/backoff.

Do not retry permanent failures forever.

## 24. API Gateway Rules

Route requests to services.

Centralize suitable cross-cutting concerns such as authentication checks, rate limiting and correlation.

Do not put seat allocation/business logic in the gateway.

Keep the gateway thin.

## 25. Microservices Data Rules

Each service owns its data.

Do not let one service directly query another service's database.

Communicate through APIs/events.

Expect distributed transactions to be harder than local transactions.

Learn eventual consistency.

Hardest boundary: Booking and inventory need a clear ownership/reservation contract. Design this carefully before extracting them.

## 26. Distributed-System Topics to Learn

Timeouts

Retries + exponential backoff

Circuit breakers

Idempotency

Eventual consistency

Duplicate events

Dead-letter queues

Distributed tracing

Correlation IDs

Health checks

Graceful failure

Rate limiting

## 27. Production Security Checklist

HTTPS in deployed environments.

Strong password hashing.

Short-lived access tokens.

Secrets outside source control.

Input validation.

Authorization on every protected operation.

Ownership checks.

No stack traces/internal errors in API responses.

Restrictive CORS.

Rate limiting on sensitive endpoints.

Dependency vulnerability scanning and updates.

## 28. Performance & Database Best Practices

Index frequent search/filter columns.

Index foreign keys and common lookups where appropriate.

Avoid JPA N+1 queries.

Use pagination.

Avoid EAGER relationships everywhere.

Fetch only needed data for expensive queries.

Use connection pooling.

Measure before optimizing.

## 29. Git & Development Workflow

Use meaningful commits such as feat, fix, refactor, test, docs.

Use small feature branches.

Keep commits focused.

Never commit .env/secrets.

Review your diff before PR.

Run tests before pushing.

Keep formatting consistent.

## 30. Recommended Build Order

Create Spring Boot project and configure Maven/database.

Create schema migrations; prefer Flyway or Liquibase for repeatable database changes.

Implement Station CRUD.

Implement Train CRUD.

Implement TrainStop/route management.

Implement Coach and Seat model.

Implement train search.

Implement availability.

Implement Booking + Passenger model.

Implement transactional seat reservation.

Implement booking lookup and cancellation.

Add validation.

Add custom exceptions + global exception handler.

Add unit/integration tests.

Add OpenAPI documentation.

Add logging and clean configuration.

After Phase 1 is stable, add Spring Security/JWT.

Then add OAuth2/OIDC.

Then extract services one boundary at a time.

Finally add gateway, discovery, notification and resilience.

## 31. Questions You Should Be Able to Answer

Why relational database?

Why DTOs instead of returning entities?

Where does business logic live?

Why @Transactional for booking?

How do you prevent two users booking the same last seat?

Why global exception handling?

Why 400 vs 404 vs 409?

Why cancel instead of delete?

Why is availability journey-date-specific?

Authentication vs authorization?

JWT vs OAuth2 vs OIDC?

Why should gateway stay thin?

Why should each service own its database?

What if notification service is down after booking succeeds?

What if a booking request is retried?

How do you test concurrent booking?

## 32. Final Learning Checklist

### Phase 1
- JPA entity mapping and relationships
- Flyway migrations and database constraints
- DTOs, mappers and Bean Validation
- Layered architecture
- Global exception handling
- Transactions, ACID and concurrency control
- Pagination and OpenAPI documentation
- Unit and integration testing

### Phase 2
- Spring Security, JWT, OAuth2/OIDC
- Command Query Responsibility Segregation (CQRS)
- Microservices, service discovery and API gateway
- Idempotency, resilience and eventual consistency

## 33. Most Important Development Principle

Build it yourself first. Use this document as the requirement and design reference, not as a copy-paste implementation. When stuck, identify the exact concept—JPA mapping, validation, transaction, security, concurrency, etc.—learn that concept, implement it, and then continue.

End of Requirements Document
