# Message Queue / Event Bus

### What It Does

A message queue decouples producers (services that emit events) from consumers (services that react to those events). Instead of Service A calling Service B directly via HTTP, Service A publishes a message to a queue, and Service B consumes it asynchronously.

Core concepts:

- **Producer** -- Publishes a message (e.g., "BookingConfirmed" event after a session is booked).
- **Consumer** -- Subscribes to a queue or topic and processes messages.
- **Exchange (RabbitMQ)** -- A routing layer that decides which queue(s) receive a message based on routing keys and binding patterns.
- **Queue (RabbitMQ)** -- A buffer that holds messages until a consumer processes them. Messages are removed from the queue after acknowledgment.
- **Topic (Kafka)** -- A named log that messages are appended to. Multiple consumer groups can read from the same topic independently.
- **Partition (Kafka)** -- A topic is split into partitions for parallelism. Each partition is an ordered, immutable sequence of messages.

### Platform Use Cases

| Event                     | Producer            | Consumer(s)                                  |
|---------------------------|---------------------|----------------------------------------------|
| `booking.confirmed`       | Scheduling          | Notifications, Analytics, Calendar Sync       |
| `payment.succeeded`       | Payments            | Notifications, Coach Payout, Analytics        |
| `payment.failed`          | Payments            | Notifications, Retry Worker                   |
| `workout.completed`       | Workouts            | Analytics, Gamification, Coach Dashboard       |
| `coach.profile.updated`   | Coach Management    | Search Index, Cache Invalidation              |
| `video.uploaded`          | Content             | Video Transcoding Worker, CDN Warm-up          |
| `user.registered`         | Auth                | Welcome Email Worker, Analytics, CRM Sync      |

### Why It Is Used

Synchronous communication creates tight coupling. If the Notification Service is down when a booking is confirmed, the booking request fails -- even though booking and notification are independent concerns. A message queue absorbs the message and delivers it when the Notification Service recovers. This provides:

- **Temporal decoupling** -- Producer and consumer do not need to be available at the same time.
- **Load leveling** -- A spike of 1,000 bookings in one minute does not overwhelm the Notification Service. Messages queue up and are processed at the consumer's pace.
- **Fan-out** -- One event triggers multiple downstream actions without the producer knowing about any of them.

### Example Technologies

| Technology       | Model            | Best For                                        |
|------------------|------------------|-------------------------------------------------|
| **RabbitMQ**     | Message Queue    | Complex routing, moderate throughput, simple ops |
| **Apache Kafka** | Event Log        | High throughput, event replay, stream processing |
| **Amazon SQS**   | Managed Queue    | Serverless, minimal ops, AWS-native              |
| **Amazon SNS**   | Managed Pub/Sub  | Fan-out to SQS queues, Lambda, HTTP endpoints    |
| **NATS**         | Message System   | Lightweight, low latency, cloud-native           |
| **Redis Streams** | Event Log       | Simple event log when Redis is already in the stack |

### Trade-offs

| Advantage                                  | Disadvantage                                          |
|--------------------------------------------|-------------------------------------------------------|
| Decouples services temporally               | Adds latency (eventual consistency, not immediate)     |
| Absorbs traffic spikes (load leveling)      | Message ordering guarantees vary by technology         |
| Enables event replay (Kafka)                | Dead letter queues need monitoring and handling        |
| Fan-out without producer changes            | Debugging distributed flows is harder than tracing a function call |
| Improves system resilience                  | Duplicate messages require idempotent consumers        |

### RabbitMQ vs Kafka (Deeper Comparison)

| Aspect             | RabbitMQ                                | Kafka                                     |
|--------------------|-----------------------------------------|-------------------------------------------|
| **Model**          | Messages are consumed and removed        | Messages are appended to a log and retained |
| **Ordering**       | Per-queue FIFO                           | Per-partition FIFO                         |
| **Replay**         | Not possible (message is gone)           | Consumers can rewind to any offset         |
| **Throughput**     | Tens of thousands of messages/sec        | Millions of messages/sec                   |
| **Routing**        | Flexible (direct, topic, fanout, headers)| Topic-based only                           |
| **Ops complexity** | Moderate                                 | High (ZooKeeper/KRaft, partition management) |
| **Best for**       | Task queues, RPC, moderate scale         | Event sourcing, analytics, high scale      |

### How It Fits

**Approach A -- Modular Monolith:**
RabbitMQ is the natural choice. Modules within the monolith publish domain events to RabbitMQ exchanges. A `booking.confirmed` event on a `topic` exchange is routed to three queues: `notifications.booking`, `analytics.booking`, and `calendar.booking`. Background workers (running as separate processes) consume from these queues. This gives the monolith async capabilities without the complexity of Kafka.

Internal module-to-module communication within the same process can use in-memory event dispatch (e.g., an EventEmitter). RabbitMQ is reserved for cross-process communication (background workers, external integrations).

**Approach B -- Microservices:**
Kafka becomes justified at this scale. Each service produces events to its own Kafka topic (`scheduling-events`, `payment-events`, `workout-events`). Other services subscribe as consumer groups. Kafka's log retention means events can be replayed when a new service comes online or when a bug requires reprocessing. The Analytics Service, for example, consumes from all topics to build aggregate dashboards.

The trade-off is operational overhead: Kafka requires partition management, consumer group coordination, and schema evolution (typically via a Schema Registry with Avro or Protobuf).
