# High-Level Architecture

This section contrasts two architectural approaches for the platform. **Approach A** is recommended for production delivery. **Approach B** is presented to illustrate full microservices patterns and serve as a reference for future migration.

---

### Approach A (Recommended): Modular Monolith with Light Event-Driven Architecture

**Philosophy:** Start with a single deployable unit that enforces strong module boundaries internally. Use asynchronous messaging only where it provides clear value (decoupling, resilience, eventual consistency). This approach optimizes for developer productivity, operational simplicity, and refactoring safety.

**Key characteristics:**

| Property | Detail |
|---|---|
| Runtime | Single NestJS (or Spring Boot) application |
| Module isolation | Each bounded context is a self-contained module with its own directory, DTOs, and internal services |
| Database | Single PostgreSQL instance, one schema per module (e.g., `identity`, `coaching`, `scheduling`) |
| Cross-module communication | In-process method calls for synchronous queries; RabbitMQ for asynchronous domain events |
| Deployment | Single container (Docker), horizontally scalable behind a load balancer |
| Async broker | RabbitMQ with topic exchanges and per-consumer queues |
| API Gateway | Not required -- the monolith serves its own REST/GraphQL endpoints |

**Why this works for a coaching platform:**

- The domain is moderately complex (12 bounded contexts) but the team can start lean and scale as the platform grows.
- Transactional consistency across modules (e.g., booking + payment) is dramatically simpler within a single process.
- RabbitMQ handles the cases where true async decoupling matters: notifications, analytics, search indexing.
- Extracting a module into its own service later requires only lifting the module directory, pointing it at its own schema, and replacing in-process calls with HTTP/gRPC.

**Application Layer:**

```mermaid
graph TB
    WEB[Web App] --> GW[NestJS Application]
    MOBILE[Mobile App] --> GW
    ADMIN[Admin Dashboard] --> GW

    subgraph Modules["12 Internal Modules"]
        IAM[Identity]
        COACH[Coaching]
        SCHED[Scheduling]
        PAY[Payments]
        PROG[Programs]
        COMM[Communication]
        CONT[Content]
        VIDEO[Video]
        NOTIF[Notifications]
        ANALYTICS[Analytics]
        SEARCH[Search]
        SUPPORT[Support]
    end

    GW --> Modules
```

**Data & Async Layers:**

```mermaid
graph LR
    App[NestJS App] --> PG[(PostgreSQL<br/>Schema per Module)]
    App --> REDIS[(Redis<br/>Cache & Sessions)]
    App --> RMQ[RabbitMQ]
    App --> S3[(S3 - Media)]

    RMQ --> NOTIF[Notifications]
    RMQ --> ANALYTICS[Analytics]
    RMQ --> SEARCH[Search / ES]

    App --> STRIPE[Stripe]
    App --> ZOOM[Zoom]
    NOTIF --> EMAIL[SendGrid]
    NOTIF --> PUSH[FCM]
```

---

### Approach B: Full Microservices with Event-Driven Architecture: Full Microservices with Event-Driven Architecture

**Philosophy:** Each bounded context is an independently deployable service with its own database, CI/CD pipeline, and scaling policy. Communication is predominantly asynchronous via an event streaming platform. This approach maximizes autonomy and scalability at the cost of operational complexity.

**Key characteristics:**

| Property | Detail |
|---|---|
| Runtime | Independent services (mix of NestJS, Spring Boot, Go, Python as appropriate) |
| Module isolation | Process-level isolation -- each service is a separate deployable |
| Database | Database-per-service (PostgreSQL, MongoDB, or specialized stores) |
| Synchronous communication | REST or gRPC for queries that require immediate responses |
| Asynchronous communication | Apache Kafka for event streaming; events are the primary integration mechanism |
| Orchestration | Kubernetes with Helm charts per service |
| API Gateway | Kong, AWS API Gateway, or custom NestJS gateway for routing, auth, rate limiting |
| Observability | Distributed tracing (Jaeger), centralized logging (ELK), metrics (Prometheus + Grafana) |

**Why this approach exists (and when it is justified):**

- It implements real-world patterns used by large organizations (Netflix, Uber, Spotify).
- It requires developers to think about service boundaries, eventual consistency, and failure modes.
- It leverages essential infrastructure: service discovery, circuit breakers, distributed tracing, saga orchestration.
- However, the operational overhead is significant and often unjustified for teams smaller than 20-30 engineers.

**Services & Gateway:**

```mermaid
graph TB
    WEB[Web App] --> GW[API Gateway - Kong]
    MOBILE[Mobile App] --> GW
    ADMIN[Admin Dashboard] --> GW

    subgraph K8s["Kubernetes Cluster"]
        SVC_IAM[Identity - NestJS]
        SVC_COACH[Coaching - NestJS]
        SVC_SCHED[Scheduling - NestJS]
        SVC_PAY[Payments - Spring Boot]
        SVC_PROG[Programs - NestJS]
        SVC_COMM[Communication - NestJS]
        SVC_CONT[Content - Go]
        SVC_VIDEO[Video - NestJS]
        SVC_NOTIF[Notification - NestJS]
        SVC_ANALYTICS[Analytics - Python]
        SVC_SEARCH[Search - Go]
        SVC_SUPPORT[Support - NestJS]
    end

    GW --> K8s
```

**Event Streaming & Database-per-Service:**

```mermaid
graph LR
    subgraph Producers["Event Producers"]
        Sched[Scheduling]
        Pay[Payments]
        Prog[Programs]
        Comm[Communication]
    end

    KAFKA[Apache Kafka + Schema Registry]

    subgraph Consumers["Event Consumers"]
        Notif[Notifications]
        Analytics[Analytics]
        Search[Search]
    end

    Producers --> KAFKA
    KAFKA --> Consumers
```

```mermaid
graph LR
    Identity[Identity] --> DB1[(PostgreSQL)]
    Coaching[Coaching] --> DB2[(PostgreSQL)]
    Scheduling[Scheduling] --> DB3[(PostgreSQL)]
    Payments[Payments] --> DB4[(PostgreSQL)]
    Analytics[Analytics] --> DB5[(ClickHouse)]
    Search[Search] --> DB6[(Elasticsearch)]
```

---

### Comparison Table: Approach A vs. Approach B

| Dimension | Approach A -- Modular Monolith | Approach B -- Full Microservices |
|---|---|---|
| **Deployment complexity** | Low -- single container, simple CI/CD | High -- 12+ containers, Helm charts, K8s |
| **Team size fit** | Up to ~15 developers | 15-50+ developers |
| **Cross-module transactions** | Simple -- in-process, single DB transaction | Complex -- Saga pattern, compensating actions |
| **Latency (inter-module)** | Nanoseconds (in-process calls) | Milliseconds (network calls + serialization) |
| **Schema evolution** | Coordinated migrations, single DB | Independent per service, schema registry |
| **Operational overhead** | Minimal -- one app to monitor | Significant -- distributed tracing, log aggregation, service mesh |
| **Scalability** | Vertical + horizontal (whole app) | Granular -- scale individual services |
| **Fault isolation** | Module failure can crash the process | Service failure is contained |
| **Developer onboarding** | Fast -- single codebase, single repo | Slow -- must understand infra, messaging, multiple repos |
| **Refactoring to microservices** | Straightforward -- modules have clean boundaries | Already there |
| **Cost (infrastructure)** | Low -- single VM / small cluster | High -- K8s cluster, Kafka cluster, multiple DBs |

**Recommendation:** Start with Approach A. Establish clean module boundaries from day one. Extract modules into services only when a specific module needs independent scaling, a different technology stack, or team ownership isolation.
