# Loan Recovery System

A microservices-based loan recovery platform built with **.NET 9**, **RabbitMQ** and **SQL Server**. Inspired by real-world experience developing debt recovery and loan management products in a critical financial environment.

This project is part of a portfolio designed to demonstrate production-grade backend engineering in the .NET ecosystem — complementing a [Java/Spring Boot portfolio project](https://github.com/algarna85/medical-appointment-system) that covers the same architectural patterns in a different ecosystem.

---

## Architecture Overview

The system is composed of three independent microservices that communicate via HTTP (synchronous) and RabbitMQ (asynchronous events):

```
┌─────────────────┐     HTTP (Polly)     ┌──────────────────┐
│   loan-service  │◄────────────────────►│ payment-service  │
│   (port 5001)   │                      │   (port 5002)    │
└────────┬────────┘                      └────────┬─────────┘
         │                                        │
         │  LoanOverdueEvent                      │  PaymentProcessedEvent
         │  LoanSettledEvent                      │
         └──────────────────┬─────────────────────┘
                            │
                    ┌───────▼────────┐
                    │  RabbitMQ      │
                    │  (MassTransit) │
                    └───────┬────────┘
                            │
                   ┌────────▼────────┐
                   │  alert-service  │
                   │  (port 5003)    │
                   │  (stateless)    │
                   └─────────────────┘
```

### Services

| Service | Responsibility | Database |
|---|---|---|
| `loan-service` | Manages the loan lifecycle (creation, status transitions, overdue detection) | SQL Server (port 1433) |
| `payment-service` | Records and processes payments; updates outstanding balances | SQL Server (port 1434) |
| `alert-service` | Stateless event consumer; evaluates risk and logs structured alerts | None |

### Messaging (RabbitMQ via MassTransit)

| Event | Producer | Consumer |
|---|---|---|
| `LoanOverdueEvent` | `loan-service` | `alert-service` |
| `LoanSettledEvent` | `loan-service` | `alert-service` |
| `PaymentProcessedEvent` | `payment-service` | `alert-service` |

MassTransit is used as the messaging abstraction layer. See [ADR-001](./docs/adr/ADR-001-masstransit-over-raw-rabbitmq.md) for the rationale.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | C# 13 / .NET 9 |
| API Framework | ASP.NET Core Web API |
| ORM | Entity Framework Core 9 |
| Database | SQL Server (Azure SQL Edge on ARM64) |
| Messaging | RabbitMQ + MassTransit |
| API Docs | Scalar (OpenAPI) |
| Validation | FluentValidation |
| Logging | Serilog (structured JSON) |
| Testing | xUnit + Moq + FluentAssertions + Testcontainers |
| Containers | Docker + Docker Compose |
| CI | GitHub Actions |

---

## Project Structure

```
loan-recovery-system/
├── src/
│   ├── loan-service/               # Loan lifecycle management
│   │   ├── Controllers/
│   │   ├── Domain/
│   │   ├── Dto/
│   │   ├── Events/
│   │   ├── Exceptions/
│   │   ├── Infrastructure/
│   │   ├── Repositories/
│   │   ├── Services/
│   │   └── Program.cs
│   ├── payment-service/            # Payment processing
│   └── alert-service/              # Event-driven alert engine (stateless)
├── tests/
│   ├── LoanService.Tests/          # Unit + integration tests
│   ├── PaymentService.Tests/
│   └── AlertService.Tests/
├── docker/
│   ├── docker-compose.infra.yml    # Infrastructure only (DB + RabbitMQ)
│   ├── docker-compose.yml          # Full stack
│   └── .env.example                # Environment variable template
├── docs/
│   └── adr/                        # Architecture Decision Records
│       └── ADR-001-masstransit-over-raw-rabbitmq.md
├── .github/
│   └── workflows/
│       ├── loan-service.yml
│       ├── payment-service.yml
│       └── alert-service.yml
├── .gitignore
├── LoanRecoverySystem.sln
└── README.md
```

---

## Getting Started

### Prerequisites

| Tool | Version | Notes |
|---|---|---|
| .NET SDK | 9.x | `dotnet --version` |
| Docker Desktop | 4.x+ | Apple Silicon: enable Rosetta in settings |
| Git | any | — |

### 1. Clone the repository

```bash
git clone https://github.com/algarna85/loan-recovery-system.git
cd loan-recovery-system
```

### 2. Configure environment variables

```bash
cp docker/.env.example docker/.env
```

Edit `docker/.env` with your local values.

> **Never commit `docker/.env`** — it is listed in `.gitignore`. Use `docker/.env.example` as the reference template.

### 3. Start infrastructure

```bash
# Starts SQL Server instances + RabbitMQ only
docker compose -f docker/docker-compose.infra.yml up -d
```

RabbitMQ Management UI: [http://localhost:15672](http://localhost:15672) (guest / guest)

### 4. Apply database migrations

```bash
# loan-service database
dotnet ef database update --project src/loan-service

# payment-service database
dotnet ef database update --project src/payment-service
```

### 5. Run the services

**Option A — individually (recommended during development):**

```bash
# Each in a separate terminal
dotnet run --project src/loan-service
dotnet run --project src/payment-service
dotnet run --project src/alert-service
```

**Option B — full stack via Docker Compose:**

```bash
docker compose -f docker/docker-compose.yml up --build
```

### 6. Explore the API

| Service | Scalar UI | Base URL |
|---|---|---|
| loan-service | http://localhost:5001/scalar | http://localhost:5001/api/v1 |
| payment-service | http://localhost:5002/scalar | http://localhost:5002/api/v1 |

---

## API Reference

### loan-service

```
GET    /api/v1/loans                        Paginated list of loans
GET    /api/v1/loans/{id}                   Loan detail
GET    /api/v1/loans/customer/{customerId}  Loans by customer
GET    /api/v1/loans/status/{status}        Loans by status
POST   /api/v1/loans                        Create a loan
PATCH  /api/v1/loans/{id}/status            Update loan status
```

### payment-service

```
GET    /api/v1/payments                     Paginated list of payments
GET    /api/v1/payments/{id}                Payment detail
GET    /api/v1/payments/loan/{loanId}       Payments for a loan
POST   /api/v1/payments                     Register a payment
```

---

## Running Tests

```bash
# All tests
dotnet test

# Specific service
dotnet test tests/LoanService.Tests

# With coverage report
dotnet test --collect:"XPlat Code Coverage"
```

Integration tests use [Testcontainers for .NET](https://dotnet.testcontainers.org/) — Docker must be running.

---

## Architecture Decisions

Design decisions are documented as Architecture Decision Records (ADRs) in [`docs/adr/`](./docs/adr/):

| ADR | Decision |
|---|---|
| [ADR-001](./docs/adr/ADR-001-masstransit-over-raw-rabbitmq.md) | MassTransit over raw RabbitMQ client |

---

## Implementation Phases

| Phase | Scope | Status |
|---|---|---|
| 1 | `loan-service` — CRUD, EF Core, migrations, Swagger, tests, CI | 🔄 In progress |
| 2 | `payment-service` — payments, HTTP client (Polly), events | ⏳ Pending |
| 3 | `alert-service` — stateless consumer, structured logging | ⏳ Pending |
| 4 | Full integration — Docker Compose, end-to-end tests | ⏳ Pending |

---

## Design Principles

- **Layered architecture:** Controllers → Services → Repositories. Each layer has a single responsibility and can be tested in isolation.
- **Dependency inversion:** All dependencies are injected via constructor and resolved through interfaces. No `new ConcreteClass()` inside services.
- **Async by default:** All I/O operations use `async`/`await` with `CancellationToken` propagation.
- **Immutable DTOs:** Request and response objects are C# `record` types.
- **Explicit error handling:** A global exception handler maps domain exceptions to HTTP responses. Controllers do not contain try/catch blocks.
- **No credentials in code:** Secrets are managed via `.env` locally and GitHub Secrets in CI. `.env` is git-ignored; `.env.example` is the contract.
- **Resilience by default:** HTTP calls between services use Polly (retry + circuit breaker). Message consumers use MassTransit retry policies with exponential backoff and dead-letter queues.

---

## Author

**Alberto García Nava** — Senior Developer / Tech Lead  
12+ years in LegalTech, FinTech and Healthcare IT  
Valencia, Spain  

[LinkedIn](https://www.linkedin.com/in/alberto-garcía-nava-820001b1) · [GitHub](https://github.com/algarna85)
