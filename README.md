# CineSpot

**A Go-powered cinema booking system with real-time seat availability, temporary reservations, and Redis-based concurrency control.**

CineSpot is a lightweight cinema booking project that demonstrates how to manage seat availability when multiple users may try to reserve the same seat at the same time. Users can browse movies, view seat availability, temporarily hold a seat, and then confirm or release their reservation.

The backend is written in Go and uses Redis to coordinate seat holds. Temporary reservations automatically expire after two minutes, while confirmed bookings are persisted in Redis without an expiration time.

> **Project status:** This is an educational/demo implementation focused on reservation flows and concurrency. It is not a complete production ticketing platform.

## Features

- **Movie listing** — retrieve the available movies and their seat layouts.
- **Seat availability** — view the current booking state of seats for a movie.
- **Temporary seat holds** — reserve a seat for two minutes while the user completes the booking flow.
- **Concurrency control** — use Redis `SET ... NX` so only one concurrent request can acquire a particular seat.
- **Automatic expiration** — Redis TTL releases an unconfirmed seat hold automatically when its time runs out.
- **Booking confirmation** — convert a temporary hold into a confirmed booking by removing its Redis expiration.
- **Reservation release** — let a user release a held seat before confirming it.
- **Interactive web interface** — select a movie, inspect its seats, see reservation states, and confirm or release a hold.
- **Live seat updates** — the browser refreshes seat availability periodically.

## Tech Stack

| Area | Technology |
|---|---|
| Language | Go 1.25.3 |
| HTTP server and routing | Go standard library (`net/http`, `http.ServeMux`) |
| In-memory reservation store | Redis 7 |
| Redis client | `go-redis/v9` |
| Session identifiers | `google/uuid` |
| Frontend | HTML, CSS, and Vanilla JavaScript |
| Local infrastructure | Docker Compose |
| Redis management UI | Redis Commander |

## How It Works

CineSpot uses a short-lived hold before confirming a reservation. This prevents two users from successfully holding the same seat at the same time.

1. The client loads the movie list using `GET /movies`.
2. The client requests the selected movie's seat states.
3. When a user selects an available seat, the API attempts to create a Redis key for that movie and seat.
4. Redis creates the key only when it does not already exist (`NX`) and assigns it a **two-minute TTL**. If the key already exists, the seat cannot be held again.
5. The API returns a session ID and expiration time. The user can confirm the hold or release it.
6. Confirmation removes the expiration from the seat key, making the booking persistent. Releasing a hold deletes its seat and session keys.
7. If the user does nothing, Redis expires the temporary keys automatically.

### Redis key design

The reservation store uses two related keys:

| Key | Purpose |
|---|---|
| `seat:{movieID}:{seatID}` | Stores the booking data for a seat. Temporary holds have a TTL; confirmed bookings do not. |
| `session:{sessionID}` | Maps a booking session back to its seat key. |

The seat key is acquired with Redis' atomic `SET` operation using `NX` and a TTL. This is the central concurrency-control mechanism in the project.

## Project Structure

```text
.
├── cmd/
│   └── main.go                  # HTTP server and route registration
├── internal/
│   ├── adapters/
│   │   └── redis/
│   │       └── redis.go         # Redis client initialization
│   ├── booking/
│   │   ├── handler.go           # HTTP handlers
│   │   ├── service.go           # Booking service
│   │   └── redis_store.go       # Redis-backed booking store
│   └── utils/                   # Shared HTTP/JSON helpers
├── static/
│   └── index.html               # Browser-based booking interface
├── docker-compose.yaml          # Redis and Redis Commander
├── go.mod
└── go.sum
```

## Getting Started

### Prerequisites

- Go **1.25.3** or later
- Docker and Docker Compose

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/cinespot.git
cd cinespot
```

Replace the URL with your repository URL.

### 2. Start Redis

The Compose file starts Redis 7 and Redis Commander:

```bash
docker compose up -d
```

By default, the application connects to Redis at `localhost:6379`.

### 3. Run the application

In the project directory, run:

```bash
go run ./cmd
```

The HTTP server listens on port `8080`.

### 4. Open the application

- **CineSpot UI:** http://localhost:8080/
- **Redis Commander:** http://localhost:8081/

To stop the supporting services:

```bash
docker compose down
```

## API Reference

The backend exposes the following HTTP endpoints:

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/movies` | List available movies and their seat layouts. |
| `GET` | `/movies/{movieID}/seats` | Get the current booking state of a movie's seats. |
| `POST` | `/movies/{movieID}/seats/{seatID}/hold` | Temporarily hold a seat and create a booking session. |
| `PUT` | `/sessions/{sessionID}/confirm` | Confirm a temporary reservation. |
| `DELETE` | `/sessions/{sessionID}` | Release a reservation session. |

### Hold a seat

```http
POST /movies/inception/seats/A1/hold
Content-Type: application/json
```

```json
{
  "user_id": "user-123"
}
```

A successful request returns a session ID and the hold expiration time. The current handler serializes the movie identifier as `movieID` in this response.

```json
{
  "session_id": "<session-uuid>",
  "movieID": "inception",
  "seat_id": "A1",
  "expires_at": "2026-11-05T09:30:00Z"
}
```

### Confirm a reservation

```http
PUT /sessions/<session-id>/confirm
Content-Type: application/json
```

```json
{
  "user_id": "user-123"
}
```

### Release a reservation

```http
DELETE /sessions/<session-id>
Content-Type: application/json
```

```json
{
  "user_id": "user-123"
}
```

A successful release returns `204 No Content`.

### Get seat availability

```http
GET /movies/inception/seats
```

The response contains the seat identifier, associated user ID, and whether the seat is currently held or confirmed.

## Current Limitations

This project intentionally keeps its scope small so the reservation and concurrency flow remains easy to explore. The current implementation has a few demo-level limitations:

- The movie catalog and seat layouts are defined in the application code rather than loaded from a database.
- The browser generates a temporary user ID; there is no real user registration or authentication system.
- There is no payment gateway or external ticketing integration.
- Redis is the reservation store; there is no separate relational database for movie, user, or booking history data.
- The current implementation should be reviewed and strengthened for authorization, validation, error handling, observability, and recovery before any production use.

## Potential Improvements

- Add persistent movie, screening, and seat data.
- Introduce real user authentication and authorization.
- Enforce session ownership when confirming or releasing a reservation.
- Add request validation and consistent API error responses.
- Add automated unit and integration tests, including concurrent booking tests.
- Add structured logging, configuration management, and graceful shutdown.
- Integrate a payment workflow and booking notifications.

## Learning Goals

CineSpot is useful for exploring several backend engineering topics:

- Concurrent access to shared resources
- Atomic Redis operations and key expiration
- Temporary reservations and session lifecycles
- Separation of HTTP handlers, services, and storage adapters
- Designing simple APIs for stateful workflows