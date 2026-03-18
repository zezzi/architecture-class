# ADR-004: Queue / Event Bus Technology

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-03-18 |
| **Deciders** | Platform architecture team |

### Context

ADR-002 established a hybrid communication model where side-effects are published as asynchronous events. In Approach A (modular monolith), these events flow through an in-process event bus. When modules are extracted into services (Approach B), a real message broker is needed. The choice of broker affects routing flexibility, operational complexity, throughput, and the programming model.

### Option A — RabbitMQ (Traditional Message Broker)

AMQP-based broker with rich routing (direct, topic, fanout, headers exchanges), acknowledgments, dead-letter queues, and mature client libraries.

| Pros | Cons |
|------|------|
| Flexible routing: topic exchanges handle event fan-out naturally | Messages are deleted after consumption — no replay capability |
| Built-in dead-letter queues for failed message handling | Requires broker management (clustering, disk space, upgrades) |
| Mature management UI and CLI tools | Less suited for high-throughput event streaming (millions/sec) |
| Low latency for task-queue patterns (e.g., send email, generate PDF) | Not a good fit for event sourcing (no persistent log) |
| Lightweight — runs comfortably on a single small VM or container | Broker is a single point of failure unless clustered |
| Excellent fit for the platform's initial event volume with room to grow (hundreds of events/hour) | |

### Option B — Apache Kafka (Event Streaming Platform)

Distributed, append-only log where events are retained for a configurable period. Consumers track their own offset and can replay from any point.

| Pros | Cons |
|------|------|
| Persistent event log — consumers can replay events for rebuilding state | Significant operational complexity: ZooKeeper/KRaft, partitions, replication |
| Excellent throughput for high-volume streaming (analytics pipelines) | Higher operational complexity — justified only when event volume and replay requirements demand it |
| Consumer groups enable horizontal scaling of consumers | Topic management, partition rebalancing, and offset management add cognitive load |
| Foundation for event sourcing and CQRS architectures | Minimum resource footprint is larger than RabbitMQ |
| Strong ecosystem: Kafka Connect, Kafka Streams, ksqlDB | Steeper learning curve for the team |
| Industry-standard for large-scale microservices | Higher infrastructure cost (memory, disk, network) |

### Option C — Redis Streams (Lightweight)

Redis Streams provides a log-like data structure with consumer groups, built into Redis (which the platform may already use for caching).

| Pros | Cons |
|------|------|
| Zero additional infrastructure if Redis is already deployed | Redis persistence (RDB/AOF) is less durable than Kafka or RabbitMQ |
| Very low latency | Limited routing capabilities — no exchange/topic matching |
| Simple API — `XADD`, `XREADGROUP`, `XACK` | Consumer group management is more manual than RabbitMQ |
| Good fit for small-scale event processing | Not designed for high-throughput streaming at Kafka scale |
| Minimal operational overhead | Community and tooling are smaller than RabbitMQ or Kafka |
| Can double as a caching layer | Risk of data loss if Redis runs out of memory with `maxmemory-policy` eviction |

### Decision

| Approach | Technology | Rationale |
|----------|-----------|-----------|
| **Approach A** (Modular Monolith) | **RabbitMQ** | Simple to operate, flexible routing, dead-letter support, perfect for the platform's event volume. Can run as a single Docker container in development. |
| **Approach B** (Microservices at scale) | **Apache Kafka** | When the platform grows to handle thousands of concurrent users and event replay or event sourcing becomes valuable, Kafka's persistent log model is the right fit. |

### Event Contract Example

Regardless of the broker, events follow a shared contract:

```json
{
  "eventId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "eventType": "BookingConfirmed",
  "timestamp": "2026-03-18T14:30:00Z",
  "version": 1,
  "payload": {
    "bookingId": "...",
    "clientId": "...",
    "coachId": "...",
    "startsAt": "2026-03-20T10:00:00Z"
  },
  "metadata": {
    "correlationId": "req-xyz-123",
    "source": "booking-module"
  }
}
```

### Final Justification

RabbitMQ is the right starting point because it matches the platform's scale, the team's capacity, and the operational simplicity required by Approach A. Its exchange-based routing maps directly onto the event fan-out patterns identified in ADR-002 (e.g., `BookingConfirmed` fans out to notification, analytics, and progress modules via a topic exchange). RabbitMQ is also easier to reason about and debug, which accelerates onboarding for new team members. The event contract is broker-agnostic, so migrating to Kafka in Approach B requires changing the transport layer, not the application logic.
