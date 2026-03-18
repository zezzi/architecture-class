# 04 - Infrastructure Components for a Gym Coaching Platform

This document breaks down the 13 core infrastructure components that support a production-grade Gym Coaching Platform. Each section explains what the component does, why it exists, the technologies involved, the trade-offs you face when adopting it, and how the component behaves differently under two architectural approaches:

- **Approach A** -- Modular Monolith (single deployable unit, logically separated modules)
- **Approach B** -- Microservices (independently deployed services communicating over the network)

---

## Table of Contents

1. [API Gateway](#1-api-gateway)
2. [CDN](#2-cdn)
3. [Load Balancer](#3-load-balancer)
4. [Cache (Redis)](#4-cache-redis)
5. [Message Queue / Event Bus](#5-message-queue--event-bus)
6. [Background Workers](#6-background-workers)
7. [Database (PostgreSQL)](#7-database-postgresql)
8. [Object Storage (S3)](#8-object-storage-s3)
9. [Authentication Provider](#9-authentication-provider)
10. [Payment Provider (Stripe)](#10-payment-provider-stripe)
11. [Notification System](#11-notification-system)
12. [Observability Stack](#12-observability-stack)
13. [Analytics Pipeline](#13-analytics-pipeline)

---

---

## Components

1. [API Gateway](./01-api-gateway.md)
2. [CDN](./02-cdn.md)
3. [Load Balancer](./03-load-balancer.md)
4. [Cache (Redis)](./04-cache-redis.md)
5. [Message Queue / Event Bus](./05-message-queue-event-bus.md)
6. [Background Workers](./06-background-workers.md)
7. [Database (PostgreSQL)](./07-database-postgresql.md)
8. [Object Storage (S3)](./08-object-storage-s3.md)
9. [Authentication Provider](./09-authentication-provider.md)
10. [Payment Provider (Stripe)](./10-payment-provider-stripe.md)
11. [Notification System](./11-notification-system.md)
12. [Observability Stack](./12-observability-stack.md)
13. [Analytics Pipeline](./13-analytics-pipeline.md)

---

## Summary Comparison: Approach A vs Approach B

| Component              | Approach A (Modular Monolith)                  | Approach B (Microservices)                         |
|------------------------|------------------------------------------------|----------------------------------------------------|
| API Gateway            | Single upstream, still valuable for cross-cutting concerns | Routes to many services, essential for discovery    |
| CDN                    | One origin, simple config                       | Multiple origins or path-based rules                |
| Load Balancer          | Balances N identical instances                  | Per-service load balancers or shared with routing   |
| Cache (Redis)          | Shared instance, key prefixes per module        | Per-service or shared cluster with ownership rules  |
| Message Queue          | RabbitMQ for async + in-memory for internal     | Kafka for durable event log across services         |
| Background Workers     | Same codebase in worker mode                    | Per-service workers, independently scaled           |
| Database               | One DB, schema-per-module                       | Database-per-service, no cross-service queries      |
| Object Storage         | Shared bucket, prefix conventions               | Per-service buckets or strict prefix isolation      |
| Auth Provider          | One module integrates with external provider    | Shared external provider, each service validates JWTs |
| Payment Provider       | Payments module owns Stripe integration          | Payment Service owns Stripe, others consume events  |
| Notification System    | Notifications module, shared workers            | Independent Notification Service                    |
| Observability          | Simpler (one process), still important           | Essential (distributed tracing across services)     |
| Analytics Pipeline     | PostgreSQL + materialized views for moderate scale | Kafka + ClickHouse for high-volume event processing |

### When to Choose Each Approach

**Start with Approach A** to move fast at launch while maintaining clean boundaries. The modular monolith delivers the same business capabilities without the operational overhead of distributed systems. Every component listed above works in a monolith.

**Migrate to Approach B** when specific modules need independent scaling (e.g., video processing grows 10x while booking stays flat), independent deployment cadence (payments team ships daily, scheduling team ships weekly), or independent technology choices (analytics pipeline in Python, booking system in Go).

The infrastructure components themselves remain the same -- what changes is how they are wired together and who owns them.
