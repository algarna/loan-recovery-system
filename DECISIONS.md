# Design Decisions

This document explains the key architectural and technical decisions made in this project, and the reasoning behind each one. It is intended as a high-level map for anyone reading the codebase — covering *why* things are the way they are, not just *what* they are.

Formal decision records for individual choices are in [`docs/adr/`](./docs/adr/).

---

## 1. Why microservices, and why three services

The decomposition into three services — `loan-service`, `payment-service` and `alert-service` — reflects how these domains would be separated in a real financial platform, based on direct experience with debt recovery products (Recovery, Loan Manager, KX Collection Suite).

The boundaries are drawn around **rate of change and ownership**, not just functionality:

- **loan-service** owns the source of truth for loans. It is the only service that writes to the loan state. Other services react to its events or query it via HTTP — they never write to its database directly. This enforces a hard boundary and avoids distributed data ownership problems.
- **payment-service** owns payment registration and processing logic. It needs to read loan data (to validate and update balances), but it does so via an explicit HTTP call — not by sharing a database. The dependency is visible in the code, not hidden in a shared schema.
- **alert-service** is intentionally stateless. Its only job is to react to events and produce output (structured logs, simulated notifications). Having no database of its own means it can be scaled, restarted or replaced without any data migration concern.

This is not microservices for its own sake. A monolith would have been simpler to build. The tradeoff is accepted here because the goal is to demonstrate real-world architectural thinking, and because the domain genuinely has independent deployment and scaling concerns between these three areas.

---

## 2. Synchronous vs asynchronous communication

The system uses both communication styles, and the choice between them is intentional in each case.

**HTTP (synchronous) — `payment-service` → `loan-service`:**

When a payment is registered, `payment-service` needs to validate that the loan exists and retrieve its current balance before processing the payment. This is a query with a direct dependency on the result — the payment cannot proceed without it. Synchronous HTTP is the right tool: the caller needs an answer before continuing.

This call is wrapped with Polly (retry + circuit breaker) to make it resilient. If `loan-service` is temporarily unavailable, the circuit breaker prevents `payment-service` from cascading failures across the system.

**RabbitMQ events (asynchronous) — all domain events:**

Everything that happens *after* the main operation is complete is communicated via events. `alert-service` does not need to be available for a payment to be processed or a loan to be created. Decoupling these concerns means each service can evolve, be deployed and fail independently.

The rule of thumb applied here: **if you need the result to continue, use HTTP; if you are notifying that something happened, use events.**

---

## 3. MassTransit over the raw RabbitMQ client

Using MassTransit as the messaging abstraction layer rather than the raw `RabbitMQ.Client` was an early decision. The full rationale is in [ADR-001](./docs/adr/ADR-001-masstransit-over-raw-rabbitmq.md).

The short version: the raw client requires hundreds of lines of infrastructure boilerplate (exchange declaration, queue binding, serialization, retry logic, dead-letter routing) before any business logic can be written. MassTransit handles all of this declaratively, is the de facto standard in the .NET ecosystem, and makes the broker swappable — the same consumer code works against RabbitMQ locally and Azure Service Bus in a cloud environment.

---

## 4. Layered architecture: Controllers → Services → Repositories

Every service follows the same internal structure:

```
Controllers   — HTTP boundary. Validates input, calls the service, returns the response.
               Contains no business logic. Contains no try/catch.
Services      — Business logic. Orchestrates domain operations, publishes events.
               Has no knowledge of HTTP or database specifics.
Repositories  — Data access. The only layer that knows about EF Core or SQL.
               Testable in isolation via interface substitution.
```

This layering exists for one reason: **each layer can be tested without the others.** The service layer can be unit tested with mock repositories, without a database. The repository layer can be integration tested with Testcontainers, without the HTTP layer. The controller layer can be tested end-to-end with `WebApplicationFactory`, exercising the full stack.

Alternatives considered: vertical slice architecture (grouping by feature rather than layer). Valid for larger teams where features are owned independently. For a three-service system of this scope, the added complexity of vertical slices is not justified.

---

## 5. Global exception handling — no try/catch in controllers

Controllers do not contain try/catch blocks. All exceptions propagate to a centralized `IExceptionHandler` implementation that maps domain exceptions to HTTP responses.

The reason is consistency and separation of concerns. If each controller handles its own exceptions, the mapping between domain errors and HTTP status codes is scattered across the codebase and will inevitably diverge over time. With a global handler, the mapping is defined once, in one place, and every service behaves the same way.

Domain exceptions (e.g. `LoanNotFoundException`) carry semantic meaning. The global handler translates that meaning into HTTP. Controllers are unaware of HTTP status codes for error cases — they only handle the happy path.

---

## 6. C# records for DTOs

All request and response objects are defined as C# `record` types rather than classes.

Records are immutable by default: once constructed, a DTO cannot be modified. This eliminates a category of bugs where a method receives a DTO, modifies it for its own purposes, and inadvertently affects the caller. It also makes the intent explicit — a DTO is data in transit, not a mutable entity.

Records also give value-based equality for free, which simplifies assertions in tests.

---

## 7. Async/await and CancellationToken throughout

All I/O operations — database queries, HTTP calls, message publishing — are asynchronous and accept a `CancellationToken`. This is not optional in a web API: blocking threads on I/O reduces throughput under load, and ignoring cancellation tokens means the server continues processing requests that the client has already abandoned.

The `CancellationToken` flows from the HTTP request (provided by ASP.NET Core on the controller action) down through the service and repository layers. If the client disconnects mid-request, the entire operation is cancelled cleanly.

---

## 8. Secrets management

No credentials appear in source code or in files committed to the repository. The approach differs by environment:

**Local development:** A `docker/.env` file holds real values. This file is listed in `.gitignore` and never committed. `docker/.env.example` — which contains variable names but no values — is committed and serves as the setup contract for anyone cloning the project.

**CI (GitHub Actions):** Secrets are injected as repository secrets and referenced in workflow files as `${{ secrets.VAR_NAME }}`. The workflow files themselves contain no sensitive values.

**Production:** Outside the scope of this project, but the pattern would be a secrets manager (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault) with the application reading secrets at startup via the .NET configuration system.

The discipline here is not about the sensitivity of a local development password. It is about the habit: normalising credentials in code leads, eventually, to credentials in production code. The `.env` + `.gitignore` + `.env.example` pattern is the correct baseline regardless of environment.

---

## 9. Azure SQL Edge for local development on Apple Silicon (ARM64)

Microsoft's official SQL Server Docker image (`mcr.microsoft.com/mssql/server`) does not have an ARM64 build. Running it on Apple Silicon requires x86 emulation via Rosetta, which works but is slower and occasionally unstable.

Azure SQL Edge (`mcr.microsoft.com/azure-sql-edge`) is Microsoft's ARM64-native SQL Server image. It is functionally compatible with SQL Server 2019 at the API level — the same T-SQL, the same EF Core driver, the same connection strings. The difference is that some enterprise SQL Server features (e.g. SQL Server Agent, certain Always On configurations) are not available, none of which are relevant to this project.

In CI (GitHub Actions, which runs on x86 Linux), the full SQL Server 2022 image is used via Testcontainers. This means the development and CI environments use slightly different images, which is a known and accepted tradeoff documented explicitly so it does not surprise anyone contributing to the project.

---

## 10. Testing strategy

Three levels of tests, each with a distinct purpose:

**Unit tests (xUnit + Moq + FluentAssertions):** Test the service layer in isolation. All dependencies (repositories, HTTP clients, event publishers) are replaced with mocks. These tests are fast, have no external dependencies, and cover business logic and edge cases. They run on every commit.

**Integration tests (xUnit + Testcontainers):** Test the repository layer against a real SQL Server instance spun up in a Docker container by the test runner. These verify that EF Core mappings, migrations and queries work correctly against the real database engine — something in-memory EF providers cannot guarantee. They also test MassTransit consumers against a real RabbitMQ instance.

**End-to-end tests (WebApplicationFactory):** Test the full HTTP stack — routing, model binding, validation, the global exception handler — without mocking the HTTP layer. Use the in-memory EF provider or Testcontainers depending on what is being tested.

The guiding principle: mock at the boundary of what you own. The service layer mocks the repository interface (which it owns). It does not mock the database engine (which it does not own) — that is what integration tests are for.
