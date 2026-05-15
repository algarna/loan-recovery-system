# ADR-001 — MassTransit over raw RabbitMQ client

**Date:** 2025-05-15
**Status:** Accepted

---

## Context

The system requires asynchronous communication between three microservices via RabbitMQ. The two implementation options evaluated were:

**Option A — RabbitMQ.Client (raw):** The official .NET client published by the RabbitMQ team. Provides full control over the AMQP protocol but requires manual management of channels, exchanges, queue bindings, serialization, retry logic and dead-letter routing.

**Option B — MassTransit:** A mature open-source messaging abstraction layer that runs on top of RabbitMQ (and other brokers). Standard in the .NET ecosystem for production messaging workloads.

---

## Decision

Use **MassTransit** as the messaging abstraction layer over RabbitMQ.

---

## Rationale

### What MassTransit handles automatically

| Concern | Raw RabbitMQ.Client | MassTransit |
|---|---|---|
| Exchange / queue topology | Manual `ExchangeDeclare`, `QueueDeclare`, `QueueBind` per consumer | Declared automatically at startup from consumer registrations |
| Serialization | Manual JSON serialization / deserialization | Built-in System.Text.Json; configurable |
| Retry policies | Manual implementation with try/catch and requeue logic | Declarative: `UseMessageRetry(r => r.Exponential(...))` |
| Dead-letter queues | Manual DLX/DLQ exchange binding | Automatic `*_error` queues per consumer |
| Consumer lifecycle | Manual thread management | Managed host; clean shutdown on cancellation |
| Correlation / tracing | Manual header propagation | Built-in via `CorrelationId` and OpenTelemetry support |

### Why this matters for this project

The raw client would require several hundred lines of infrastructure boilerplate before writing a single line of business logic. For a portfolio project that aims to demonstrate domain design and architectural decisions — not AMQP wire protocol knowledge — that boilerplate would obscure the intent of the code.

More importantly, MassTransit is the de facto standard in the .NET ecosystem for this use case. Using it reflects the kind of decision a senior engineer would make in a real production context: choose the well-supported abstraction that reduces operational risk, not the lower-level primitive.

### Broker portability

MassTransit's transport layer is pluggable. Switching from RabbitMQ to Azure Service Bus (a common move when going cloud-native on Azure) requires changing only the MassTransit configuration in `Program.cs`. Consumer and producer code — the business logic — is unchanged. This is a concrete application of the Dependency Inversion Principle at the infrastructure level.

---

## Consequences

**Positive:**
- Exchange/queue topology is declared automatically at startup.
- Retry policies and dead-letter queues are configured declaratively in one place.
- The broker can be swapped (e.g. to Azure Service Bus) by changing only the MassTransit configuration, not consumer/producer code.
- Correlation IDs and structured logging are handled by the framework.

**Negative:**
- Adds a dependency (`MassTransit.RabbitMQ`). Evaluated as acceptable given the library's maturity (10+ years, actively maintained, widely adopted in enterprise .NET).
- Abstracts some AMQP concepts that may be useful to understand directly. Mitigated by the fact that MassTransit's documentation explicitly maps its concepts to the underlying AMQP primitives.

---

## Alternatives considered

**Rebus:** Similar abstraction layer. Smaller community, less active maintenance. Ruled out.

**NServiceBus:** Enterprise-grade, excellent feature set. Commercial license for production use. Not appropriate for an open-source portfolio project.
