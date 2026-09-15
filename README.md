# ✈️ Airline Reservation System

**A production-style distributed booking platform — built to solve the same consistency, coupling, and reliability problems real airline and e-commerce backends face at scale.**

[![Node.js](https://img.shields.io/badge/Node.js-Express.js%205-339933)](https://expressjs.com/)
[![Database](https://img.shields.io/badge/Database-MySQL%208-4479A1)](https://www.mysql.com/)
[![Messaging](https://img.shields.io/badge/Messaging-RabbitMQ-FF6600)](https://www.rabbitmq.com/)
[![ORM](https://img.shields.io/badge/ORM-Sequelize-52B0E7)](https://sequelize.org/)
[![License](https://img.shields.io/badge/License-MIT-blue)](#license)

---

### Highlights

- **5 independently deployable services** communicating over REST and an async message broker, each owning its own MySQL schema (database-per-service)
- **Fail-closed seat reservation** — closes a real overbooking race condition present in naive check-then-decrement designs
- **Transactional outbox pattern** — a broker outage can never silently drop a booking confirmation
- **Circuit breaker + local JWT verification** at the API Gateway — no cascading failures, no auth service in the hot path
- **Correlation-ID tracing** across every service boundary, without a full distributed tracing stack
- **Ownership-scoped authorization** enforced on every booking read/cancel via JWT claims

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Microservices](#microservices)
- [Diagrams](#diagrams)
- [Design Patterns & Principles](#design-patterns--principles)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Tech Stack](#tech-stack)
- [API Overview](#api-overview)
- [Running Locally](#running-locally)
- [Testing](#testing)
- [License](#license)

---

## Overview

This repository is the **umbrella/reference repo** for the system — it does not contain application code itself. Each service lives in its own repository and is developed, versioned, and deployed independently, following a database-per-service pattern with event-driven communication for cross-service consistency.

The system is deliberately scoped around the hardest problems a real booking platform has to solve: preventing overbooking under concurrent writes, guaranteeing a confirmed booking is never silently lost even if the message broker goes down, and keeping a single request traceable across five independently-deployed services without a shared database or a monolithic call stack.

## Architecture

```mermaid
graph TD
    Client[Client]
    GW["API Gateway :3004<br/>JWT verify · rate limit · circuit breaker"]
    Auth["Auth Service :3001"]
    Booking["Booking Service :3002"]
    Flights["Flights & Search Service :3000"]
    Reminder["Reminder Service :3003"]
    AuthDB[(MySQL — Auth DB)]
    BookingDB[(MySQL — Booking DB)]
    ReminderDB[(MySQL — Reminder DB)]
    MQ{{RabbitMQ<br/>BOOKING_SERVICE exchange}}

    Client --> GW
    GW --> Auth
    GW --> Booking
    GW --> Flights
    Auth --> AuthDB
    Booking --> BookingDB
    Booking -->|outbox poller| MQ
    Booking -.->|seat availability| Flights
    MQ --> Reminder
    Reminder --> ReminderDB
```

## Microservices

| Service | Description | Port | Repository |
|---|---|---|---|
| 🛡️ **API Gateway** | Single entry point for all client traffic. Local JWT verification, rate limiting, request proxying, circuit breaking, correlation-ID tracing. | `3004` | [API_GATEWAY](https://github.com/Rohit-hub-coder/API_GATEWAY) |
| 🔐 **Auth Service** | User signup/signin, OTP email verification, JWT issuance, role-based authorization. | `3001` | [auth-service](https://github.com/Rohit-hub-coder/auth-service) |
| 🎫 **Booking Service** | Booking creation, cancellation, and status tracking with concurrency-safe seat reservation. | `3002` | [booking-service](https://github.com/Rohit-hub-coder/booking-service) |
| 🛫 **Flights & Search Service** | Flight, airport, airplane, and city management with relational search. | `3000` | [FLIGHTSANDSEARCHSERVICE](https://github.com/Rohit-hub-coder/FLIGHTSANDSEARCHSERVICE) |
| 📣 **Reminder Service** | Asynchronous booking-confirmation notifications, driven by RabbitMQ. | `3003` | [ReminderService](https://github.com/Rohit-hub-coder/ReminderService) |

---

## Diagrams

<details open>
<summary><strong>Sequence diagram — End-to-end booking flow (fail-closed reservation + outbox pattern)</strong></summary>

This is the system's core transaction, and the one most worth walking an interviewer through: it shows a synchronous, consistency-critical path (seat reservation) composed with an asynchronous, at-least-once delivery path (notification), bridged by the outbox pattern so a broker outage can never silently drop a confirmed booking.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant GW as API Gateway
    participant B as Booking Service
    participant F as Flights Service
    participant DB as Booking DB (MySQL)
    participant MQ as RabbitMQ
    participant R as Reminder Service

    C->>GW: POST /api/v1/bookings (JWT, flightId)
    GW->>GW: Verify JWT locally (no network hop to Auth)
    GW->>B: Proxy request + authenticated userId

    B->>F: GET seat availability (flightId)
    alt Seat count unavailable or ambiguous
        F-->>B: No explicit count returned
        B-->>GW: 503 Booking refused (fail-closed)
        GW-->>C: 503 Service Unavailable
    else Seat count confirmed
        F-->>B: Explicit available seat count
        B->>DB: BEGIN TRANSACTION
        B->>DB: Insert booking row (status = confirmed)
        B->>DB: Insert outbox event (booking.created) — same transaction
        B->>DB: COMMIT
        B-->>GW: 201 Booking confirmed
        GW-->>C: 201 Booking confirmed

        loop Outbox poller (background, decoupled from request path)
            B->>DB: Poll unpublished outbox events
            B->>MQ: Publish booking.created
            B->>DB: Mark event as published
        end

        MQ->>R: Consume booking.created (at-least-once)
        R->>R: Send confirmation email (Nodemailer)
    end
```

</details>

<details open>
<summary><strong>Class diagram — Consolidated domain model across services</strong></summary>

Each class below is owned by exactly one service's database (database-per-service), but is shown together here to illustrate the relationships that span service boundaries via foreign-key-style references (`userId`, `flightId`) rather than actual foreign keys.

```mermaid
classDiagram
    namespace AuthService {
        class User {
            +int id
            +string email
            +string passwordHash
            +boolean isVerified
            +Date createdAt
        }
        class Role {
            +int id
            +string name
        }
        class UserRole {
            +int userId
            +int roleId
        }
        class OtpToken {
            +int id
            +int userId
            +string code
            +Date expiresAt
        }
    }

    namespace FlightsService {
        class City {
            +int id
            +string name
        }
        class Airport {
            +int id
            +string code
            +int cityId
        }
        class Airplane {
            +int id
            +string model
            +int totalSeats
        }
        class Flight {
            +int id
            +int airplaneId
            +int originAirportId
            +int destinationAirportId
            +Date departureTime
            +int availableSeats
        }
    }

    namespace BookingService {
        class Booking {
            +int id
            +int userId
            +int flightId
            +string status
            +Date bookedAt
        }
        class OutboxEvent {
            +int id
            +string eventType
            +json payload
            +boolean published
        }
    }

    namespace ReminderService {
        class ReminderRecord {
            +int id
            +int bookingId
            +string recipientEmail
            +string status
            +int retryCount
        }
    }

    User "1" --> "*" UserRole
    Role "1" --> "*" UserRole
    User "1" --> "*" OtpToken
    City "1" --> "*" Airport
    Airplane "1" --> "*" Flight
    Airport "1" --> "*" Flight : origin/destination
    Booking "1" --> "1" OutboxEvent : emits
    Booking ..> User : userId (cross-service ref)
    Booking ..> Flight : flightId (cross-service ref)
    ReminderRecord ..> Booking : bookingId (cross-service ref)
```

</details>

---

## Design Patterns & Principles

| Pattern / Principle | Where it's used |
|---|---|
| **API Gateway** | Single entry point, offloading auth, rate limiting, and routing from every downstream service |
| **Database-per-Service** | Each service owns an isolated MySQL schema; no cross-service table access |
| **Transactional Outbox** | Booking Service — atomic write of state + event, reliably relayed to RabbitMQ |
| **Circuit Breaker** | Gateway → Flights proxy, via `opossum`, to fail fast instead of cascading latency |
| **Fail-Closed Validation** | Booking Service refuses to proceed on ambiguous downstream responses rather than assuming success |
| **Event-Driven Notification** | Reminder Service reacts to `booking.created` asynchronously, decoupled from the request path |
| **Role-Based Access Control** | Auth Service issues JWTs carrying role claims, enforced downstream |
| **Correlation ID Propagation** | `X-Request-Id` generated/forwarded at the Gateway, logged at every hop |

## Key Engineering Decisions

| Decision | Problem it solves | Trade-off accepted |
|---|---|---|
| **Fail-closed seat reservation** | Prevents overbooking from a non-atomic check-then-decrement race under concurrent requests | Slightly more conservative — a transient Flights-service hiccup can delay a valid booking rather than risk overselling |
| **Transactional outbox** | Guarantees a booking and its `booking.created` event are never inconsistent, even through a broker outage | Adds a polling component and at-least-once (not exactly-once) delivery, so the Reminder Service must tolerate duplicate events |
| **Local JWT verification at the gateway** | Removes a network hop and a single point of failure from every authenticated request | Token revocation isn't instant — a revoked token stays valid until expiry |
| **Circuit breaker on the Flights proxy** | Stops a slow/down Flights service from cascading latency back to the client | Adds a failure mode (open circuit) that must be surfaced clearly to the client as retryable |
| **Correlation IDs, no full tracing stack** | Makes a single request traceable across five services' logs without extra infra | Less powerful than a real distributed tracer (e.g. no automatic span timing) — a deliberate scope trade-off |
| **Ownership-scoped `403` on cross-user access** | Enforces that users can only read/cancel their own bookings | Confirms a resource's existence to an unauthorized caller — `404` would hide that, at the cost of a less informative error |
| **Database-per-service** | Keeps services independently deployable; no schema coupling | Cross-service reads require an API call or event instead of a JOIN, adding latency and eventual consistency |

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js, Express.js 5 |
| Databases | MySQL 8, per-service schema isolation |
| ORM | Sequelize + Sequelize CLI (versioned migrations) |
| Messaging | RabbitMQ (`amqplib`), outbox pattern for reliable publishing |
| Auth | JWT (`jsonwebtoken`), bcrypt password hashing, OTP email verification |
| API Gateway | `http-proxy-middleware`, `express-rate-limit`, `opossum` (circuit breaker), `helmet`, `cors`, `morgan` |
| Email (dev) | Nodemailer + Ethereal (disposable test SMTP sandbox) |
| Testing | Jest, Supertest |
| Dev Tooling | Nodemon, Docker (RabbitMQ container) |

## API Overview

All client traffic goes through the gateway at `http://localhost:3004`.

| Method | Path | Auth Required | Description |
|---|---|---|---|
| `POST` | `/api/v1/auth/signup` | No | Register a new user |
| `POST` | `/api/v1/auth/signin` | No | Authenticate and receive a JWT |
| `POST` | `/api/v1/auth/verify-otp` | No | Verify the OTP sent on signup |
| `GET` | `/api/v1/flights/flight` | No | List all flights |
| `GET` | `/api/v1/flights/flight/:id` | No | Get a single flight |
| `POST` | `/api/v1/bookings` | Yes | Create a booking |
| `GET` | `/api/v1/bookings` | Yes | List the authenticated user's bookings |
| `GET` | `/api/v1/bookings/:id` | Yes | Get a single booking (owner only) |
| `PATCH` | `/api/v1/bookings/:id/cancel` | Yes | Cancel a booking (owner only) |
| `GET` | `/api/v1/health` | No | Gateway liveness check |
| `GET` | `/api/v1/health/aggregate` | No | Combined health of all downstream services |

## Running Locally

Each service is independently runnable. Clone all five repos as siblings, then for each:

```bash
git clone <service-repo-url>
cd <service-directory>
npm install
cp .env.example .env   # fill in real values
npm start
```

**Startup order matters** for a full working system:

1. MySQL running locally, with a separate database per service (`AUTH_DB_DEV`, `booking_db_dev`, etc.)
2. RabbitMQ running: `docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management`
3. Auth Service → Flights & Search Service → Booking Service → Reminder Service → API Gateway

Once all five are up, verify with:

```bash
curl http://localhost:3004/api/v1/health/aggregate
```

## Testing

The Booking Service includes a Jest test suite covering the fail-closed seat-reservation logic, input validation, and payment-failure rollback:

```bash
cd booking-service
npm test
```

## License

MIT — see [LICENSE](LICENSE) for details.
