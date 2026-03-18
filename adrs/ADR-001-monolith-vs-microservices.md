# ADR-001: Monolith vs Microservices

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-03-18 |
| **Deciders** | Platform architecture team |

### Context

The platform must support seven distinct functional areas: client booking, coach management, payments, messaging, progress tracking, content delivery, and analytics. The platform must be designed to scale from an initial launch to thousands of concurrent users across multiple markets. The architecture must support evolving from a streamlined deployment model toward a fully distributed one without a full rewrite.

### Option A — Modular Monolith (NestJS or Spring Boot with well-defined modules)

A single deployable unit where each functional area lives inside its own module with explicit public interfaces. Modules communicate through in-process method calls or an internal event bus. The codebase enforces boundaries via package/module visibility rules.

| Pros | Cons |
|------|------|
| Single deployment artifact — simple CI/CD pipeline | All modules share a single process; a memory leak in analytics can crash bookings |
| One database connection pool, one runtime to monitor | Scaling is all-or-nothing; you cannot scale the payment module independently |
| Refactoring across module boundaries is a standard code change, not a distributed migration | Discipline is required to keep module boundaries clean; without enforcement, coupling creeps in |
| Local function calls have zero network latency | A single long deployment pipeline as the codebase grows |
| Easier onboarding — one repo, one build, one run command | Technology choices are locked to the host framework (e.g., all modules must use the same language/runtime) |
| Lower operational overhead, allowing the team to focus on features over infrastructure | Risk of "big ball of mud" if architectural fitness functions are not enforced |

### Option B — Full Microservices (independent deployable services)

Each functional area is a standalone service with its own repository (or mono-repo path), its own database, and its own deployment pipeline. Services communicate over the network via REST/gRPC and async events.

| Pros | Cons |
|------|------|
| Independent deployment — ship a fix to payments without redeploying the whole system | Significant operational overhead: service discovery, load balancing, distributed tracing, circuit breakers |
| Each service can use the technology best suited to its domain | Network calls introduce latency, partial failures, and retry logic |
| Horizontal scaling per service — scale the booking service during peak hours without scaling analytics | Data consistency across services requires sagas or eventual consistency patterns |
| Strong physical enforcement of module boundaries | A growing team may initially spend more time on infrastructure than on features |
| Aligns with industry hiring expectations and team growth goals | Local development requires running many containers or using service stubs |
| Fault isolation — a crash in content delivery does not affect payments | Distributed debugging is harder; requires investment in observability from day one |

### Decision

**Start with Approach A (Modular Monolith), design every module boundary so it can be extracted into Approach B (Microservices) later.**

Each module will:

1. Expose a **public API surface** (a TypeScript interface or Java interface) that other modules depend on — never internal classes.
2. Communicate side-effects through an **internal event bus** that can later be swapped for a real message broker.
3. Own its **database schema** within a shared PostgreSQL instance, using separate schemas (e.g., `booking.*`, `payment.*`) so data ownership is clear.
4. Have its own **integration test suite** that exercises only the public API, proving the module can function behind a network boundary.

### Migration Path (Monolith to Microservices)

```
Phase 1 — Modular Monolith
┌─────────────────────────────────────────────────┐
│  Single Process (NestJS / Spring Boot)          │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐│
│  │ Booking  │ │ Payment  │ │ Coach Management ││
│  └──────────┘ └──────────┘ └──────────────────┘│
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐│
│  │ Messaging│ │ Progress │ │ Content Delivery ││
│  └──────────┘ └──────────┘ └──────────────────┘│
│  ┌──────────┐                                   │
│  │Analytics │   Internal Event Bus              │
│  └──────────┘                                   │
│            Shared PostgreSQL (separate schemas)  │
└─────────────────────────────────────────────────┘

Phase 2 — Extract high-value services first
┌────────────┐   ┌────────────┐    ┌──────────────────────┐
│  Payment   │   │  Booking   │    │  Remaining Monolith  │
│  Service   │◄──┤  Service   │◄──►│  (other modules)     │
└─────┬──────┘   └─────┬──────┘    └──────────┬───────────┘
      │                │                       │
      └────────── Message Broker ──────────────┘

Phase 3 — Full Microservices (if/when scale demands it)
```

### Final Justification

At launch, the modular monolith delivers features faster, costs less to operate, and is far simpler to debug. The deliberate enforcement of module boundaries — separate schemas, public interfaces, event-driven side-effects — means that extracting any module into a standalone service is a mechanical refactoring step rather than a redesign. This approach teaches both the monolith discipline and the microservice extraction pattern, which is more realistic than starting with microservices on day one (a pattern the industry increasingly warns against under the label "premature decomposition").
