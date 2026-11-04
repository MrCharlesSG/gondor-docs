# ADR-002: Hexagonal Architecture for the Go Monolith

**Status:** Accepted

---

## Context

The Gondor backend must simultaneously integrate multiple infrastructure technologies: MySQL (financial data), Redis (price cache), external stock price APIs (Finnhub, Alpaca), and a Polygon blockchain node (RPC). In addition, the roadmap explicitly planned to add blockchain complexity in later phases.

Coupling business logic directly to any of these technologies would create unsustainable technical debt and would make unit testing impossible without running all infrastructure dependencies.

---

## Decision

Adopt **Hexagonal Architecture (Ports and Adapters)** for the entire Go monolith, with a strict boundary between:

- **Domain layer**: Pure business logic — trade processing, ROI calculation, holding management. Expresses its dependencies as Go interfaces (*ports*). Has zero knowledge of MySQL, Redis, or blockchain.
- **Infrastructure layer**: Concrete implementations (*adapters*) — `TransactionRepository` (MySQL), `StockCacheRepository` (Redis), `PolygonClient` (go-ethereum RPC). Each adapter implements a port interface.
- **Application layer**: Orchestration of domain services, HTTP controllers (Gin-gonic).

The `IdentityBridge` interface is a concrete example: `AuthService` (Minas Tirith) declares it as a port dependency; `RegisterService` (Pelargir) implements it. Neither module needs to know the other's internal implementation.

---

## Consequences

**Positive:**
- Domain logic can be tested in isolation without Docker containers — tests run in pure memory.
- Adding the blockchain layer (Phases 2–3) did not require modifying the financial engine code.
- Achieved >85% test coverage on service layers and 100% on testable infrastructure, using gomock-generated mocks for external ports.
- Future evolution to microservices is straightforward: each port can be replaced by a gRPC contract or message queue.

**Negative / Trade-offs:**
- More boilerplate than a layered architecture — every external interaction requires defining a port interface and at least one adapter.
- Developers unfamiliar with the pattern need onboarding time to understand the indirection layers.
