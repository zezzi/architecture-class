# ADR-002: Synchronous vs Asynchronous Communication

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-03-18 |
| **Deciders** | Platform architecture team |

### Context

The platform handles operations with fundamentally different latency and reliability requirements. A client booking a session needs an immediate confirmation. A notification that a coach posted new content can arrive seconds later with no user impact. Analytics events should never block a user-facing request. The communication strategy must work inside the modular monolith (Approach A) today and translate cleanly to network-based communication in Approach B.

### Option A — All Synchronous (REST / gRPC)

Every inter-module (or inter-service) call is a synchronous request-response. The caller blocks until the callee responds.

| Pros | Cons |
|------|------|
| Simple mental model — call a function, get a result | Temporal coupling: if the downstream module is slow or down, the caller is blocked |
| Easy to debug — a single call stack or request trace | Cascading failures: a slow analytics write can degrade booking response times |
| No need for a message broker or event infrastructure | Poor fit for fire-and-forget operations (notifications, audit logs) |
| Strong consistency — the caller knows the outcome immediately | Scaling bottlenecks: high-throughput events (analytics) would overload synchronous endpoints |
| Familiar to most developers | Tight coupling between caller and callee contracts |

### Option B — Hybrid (Sync for commands, Async for events)

User-facing commands (book a session, process a payment) use synchronous calls. Side-effects and downstream reactions (send notification, update analytics, sync progress) are emitted as asynchronous events consumed by interested modules.

| Pros | Cons |
|------|------|
| User-facing requests stay fast and predictable | Two communication paradigms to understand and maintain |
| Fire-and-forget events decouple producers from consumers | Eventual consistency: a notification may arrive a few seconds after the booking |
| Adding a new consumer (e.g., a loyalty module) requires zero changes to the producer | Debugging async flows requires correlation IDs and tracing |
| Natural backpressure — consumers process events at their own rate | Message ordering and idempotency must be handled explicitly |
| Mirrors proven architecture patterns used in production systems | Slightly more infrastructure (event bus or message broker) |

### Decision

**Adopt the Hybrid approach (Option B).**

### Concrete Examples

| Operation | Style | Reason |
|-----------|-------|--------|
| `POST /bookings` — create a booking | **Sync** | Client needs immediate confirmation or rejection |
| `POST /payments/charge` — process payment | **Sync** | Payment outcome must be known before confirming the booking |
| `BookingConfirmed` event → send email/push notification | **Async** | Notification delay of a few seconds is acceptable; should not slow down the booking response |
| `BookingConfirmed` event → update analytics counters | **Async** | Analytics is a read-model concern; eventual consistency is fine |
| `WorkoutLogged` event → recalculate progress metrics | **Async** | Background computation; the user sees updated stats on next page load |
| `ContentPublished` event → notify subscribed clients | **Async** | Fan-out to many clients; must not block the coach's publish action |

In Approach A, the async event bus is an in-process `EventEmitter` (NestJS) or `ApplicationEventPublisher` (Spring). In Approach B, the same event contracts are published to a real message broker (see ADR-004).

### Final Justification

The hybrid model matches the natural semantics of each operation. Commands that change state and need confirmation stay synchronous. Events that propagate information to other modules are asynchronous, which improves responsiveness, fault tolerance, and extensibility.
