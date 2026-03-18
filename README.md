# Gym Coaching Platform — Architecture Documentation

> **Version:** 1.0
> **Last Updated:** 2026-03-18

This repository contains the architecture documentation for the Gym Coaching Platform — a system supporting client booking, coach management, payments, messaging, progress tracking, content delivery, and analytics.

Two architectural approaches are contrasted throughout:

- **Approach A** — Modular Monolith (recommended starting point)
- **Approach B** — Full Microservices (migration target as scale demands)

---

## Documentation Structure

### [Proposals](./proposals/)

The architecture design broken into four sections:

1. [High-Level Architecture](./proposals/01-high-level-architecture.md) — Approach A vs B comparison, key characteristics, and recommendation
2. [Bounded Contexts (DDD)](./proposals/02-bounded-contexts.md) — 12 bounded contexts with entities, events, and relationships
3. [Service/Module Decomposition](./proposals/03-service-module-decomposition.md) — Directory structures and code organization for both approaches
4. [Data Flow and Interactions](./proposals/04-data-flow-and-interactions.md) — Sequence diagrams and event catalog

### [Architecture Decision Records](./adrs/)

One file per decision, documenting context, options evaluated, and rationale:

1. [ADR-001: Monolith vs Microservices](./adrs/ADR-001-monolith-vs-microservices.md)
2. [ADR-002: Synchronous vs Asynchronous Communication](./adrs/ADR-002-sync-vs-async-communication.md)
3. [ADR-003: Database Strategy](./adrs/ADR-003-database-strategy.md)
4. [ADR-004: Queue / Event Bus Technology](./adrs/ADR-004-queue-event-bus-technology.md)
5. [ADR-005: Authentication Strategy](./adrs/ADR-005-authentication-strategy.md)
6. [ADR-006: Observability Strategy](./adrs/ADR-006-observability-strategy.md)

### [Diagrams](./diagrams/)

Visual representations of the system architecture:

1. [High-Level System Architecture (C4 Context)](./diagrams/01-high-level-system-architecture.md)
2. [DDD Bounded Context Map](./diagrams/02-ddd-bounded-context-map.md)
3. [Event Flow Diagrams](./diagrams/03-event-flow-diagrams.md) — Booking/payment, program enrollment, and communication flows
4. [Infrastructure / Deployment](./diagrams/04-infrastructure-deployment.md) — Physical topology for both approaches

### [Infrastructure Components](./infrastructure/)

Deep dives into the 13 core infrastructure components, each covering what it does, why it exists, trade-offs, and how it fits under both approaches:

1. [API Gateway](./infrastructure/01-api-gateway.md)
2. [CDN](./infrastructure/02-cdn.md)
3. [Load Balancer](./infrastructure/03-load-balancer.md)
4. [Cache (Redis)](./infrastructure/04-cache-redis.md)
5. [Message Queue / Event Bus](./infrastructure/05-message-queue-event-bus.md)
6. [Background Workers](./infrastructure/06-background-workers.md)
7. [Database (PostgreSQL)](./infrastructure/07-database-postgresql.md)
8. [Object Storage (S3)](./infrastructure/08-object-storage-s3.md)
9. [Authentication Provider](./infrastructure/09-authentication-provider.md)
10. [Payment Provider (Stripe)](./infrastructure/10-payment-provider-stripe.md)
11. [Notification System](./infrastructure/11-notification-system.md)
12. [Observability Stack](./infrastructure/12-observability-stack.md)
13. [Analytics Pipeline](./infrastructure/13-analytics-pipeline.md)

### Presentation

A 55-slide architecture review covering all major topics:

- [presentation.pptx](./presentation.pptx) — PowerPoint format (Google Slides / Keynote compatible)
- [presentation.html](./presentation.html) — Open in a browser to present (arrow keys to navigate)
- [presentation.md](./presentation.md) — Marp source (editable)
- [05-presentation-part1.md](./05-presentation-part1.md) — Slides 1–28 with speaker notes
- [06-presentation-part2.md](./06-presentation-part2.md) — Slides 29–55 with speaker notes

### Infrastructure Components Presentation

A 30-slide deep dive into every infrastructure component — API Gateway, CDN, Redis, Kafka, PostgreSQL, S3, Auth0, Stripe, and more:

- [infrastructure-presentation.pptx](./infrastructure-presentation.pptx) — PowerPoint format
- [infrastructure-presentation.html](./infrastructure-presentation.html) — Browser presentation
- [infrastructure-presentation.md](./infrastructure-presentation.md) — Marp source (editable)
