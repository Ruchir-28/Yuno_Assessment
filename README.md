# Payment Module - Java Spring Boot Assignment

## Features

- Create Payment API
- Fetch Payment API
- Routing Engine
  - CARD routes to Provider A first, then Provider B as failover
  - UPI routes to Provider B first, then Provider A as failover
- Retry and failover across providers
- Idempotency using `Idempotency-Key` request header
- Payment status tracking
- MySQL persistence with Spring Data JPA
- Unit tests for core orchestration behavior

## Architecture as per assignment

```text
Client
  ↓
Controller Layer
  ↓
Service Layer / Orchestration Engine
  ↓
Routing Engine
  ↓
Provider Connectors A/B
  ↓
Persistence Layer MySQL
  ↓
Idempotency Store DB
```

## Tech Stack

- Java 17
- Spring Web
- Spring Data JPA
- MySQL
- Maven

## Run Locally

Start MySQL and create the database: tables will be created automatically due to JPA in app.properties is auto

```sql
CREATE DATABASE sit;
```

Update `src/main/resources/application.properties` if your MySQL username/password differs. I used XAMPP server for apache and mysql

Run the application:

```bash
mvn spring-boot:run
```

The service starts on:

```text
http://localhost:8080
```

## APIs

### Create Payment

```http
POST /api/v1/payments
Idempotency-Key: unique-request-key-123
Content-Type: application/json
```

Request:

```json
{
  "amount": 100.00,
  "currency": "INR",
  "paymentMethod": "CARD",
  "customerId": "customer-001"
}
```

Response:

```json
{
  "paymentId": "uuid",
  "amount": 100.00,
  "currency": "INR",
  "paymentMethod": "CARD",
  "status": "SUCCESS",
  "providerName": "PROVIDER_A",
  "providerTransactionId": "A-xxxx",
  "failureReason": null,
  "createdAt": "2026-05-28T10:00:00Z",
  "updatedAt": "2026-05-28T10:00:01Z"
}
```

### Fetch Payment

```http
GET /api/v1/payments/{paymentId}
```

## Routing Rules

| Payment Method | Primary Provider | Failover Provider |
|---|---|---|
| CARD | Provider A | Provider B |
| UPI | Provider B | Provider A |

## Status Lifecycle

```text
INITIATED → PROCESSING → SUCCESS
INITIATED → PROCESSING → FAILED
```

## Test Cases

### Positive Scenarios

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Create CARD payment | Routed to Provider A and marked SUCCESS |
| 2 | Create UPI payment | Routed to Provider B and marked SUCCESS |
| 3 | Fetch existing payment | Payment details returned |
| 4 | Repeat request with same Idempotency-Key | Existing payment returned, no duplicate charge |
| 5 | Primary provider fails and secondary succeeds | Payment succeeds through failover provider |

### Negative Scenarios

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Missing amount | HTTP 400 validation error |
| 2 | Amount less than 1 | HTTP 400 validation error |
| 3 | Missing currency | HTTP 400 validation error |
| 4 | Missing payment method | HTTP 400 validation error |
| 5 | Fetch unknown payment ID | HTTP 404 |
| 6 | Both providers fail | Payment status becomes FAILED |
| 7 | Provider throws exception | module catches it and attempts failover |

## Run Tests

```bash
mvn test
```

## Production Improvements

- Replace simulated providers with real PSP REST clients.
- Add Redis-backed idempotency with TTL, although idempotency can be spoofed from frontend, we can do request hashing + salting via bcrypt or similar algos to prevent same payment twice.
- Add optimistic locking for concurrent payment updates, this can be done at db level. Also we can add locking for multiple jvms.
- Add observability: structured logs, metrics, traces, dashboards, alerts.
- Add provider-specific error mapping.
- Add webhooks for asynchronous provider status updates.
- Add feature for notification / alerts post payment is success / failure.
