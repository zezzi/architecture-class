# Observability Stack

### What It Does

Observability is the ability to understand what is happening inside your system by examining its external outputs. The three pillars of observability are **metrics**, **logs**, and **traces**. Together, they answer: "Is the system healthy? If not, where is the problem, and why?"

### The Three Pillars

**Metrics** -- Numeric measurements sampled over time.
- Request rate (requests per second)
- Error rate (percentage of 5xx responses)
- Latency (p50, p95, p99 response times)
- Saturation (CPU usage, memory usage, database connection pool utilization)
- Business metrics (bookings per hour, payment success rate, active coaches)

**Logs** -- Timestamped, structured records of discrete events.
- `{"timestamp": "2026-03-18T10:15:03Z", "level": "error", "service": "payments", "message": "Stripe webhook signature verification failed", "webhook_id": "evt_123", "ip": "1.2.3.4"}`
- Structured (JSON) logs are searchable and filterable. Unstructured logs (plain text) are not.

**Traces** -- End-to-end records of a request as it flows through the system.
- A single booking request generates a trace with spans:
  1. API Gateway (2ms)
  2. Auth middleware (1ms)
  3. Scheduling module - check availability (15ms)
  4. Database query - SELECT available slots (8ms)
  5. Payment module - create Stripe checkout (120ms)
  6. Stripe API call (95ms)
  7. Database query - INSERT booking (5ms)
  8. Event publish - booking.confirmed (1ms)
- The trace shows the total request took 152ms, and Stripe was the bottleneck.

### OpenTelemetry

OpenTelemetry (OTel) is the vendor-neutral standard for instrumenting applications. You add the OTel SDK to your application, and it automatically captures metrics, logs, and traces. These are exported to your chosen backend (Prometheus, Loki, Tempo, Jaeger, Datadog, etc.) via the OTel Collector.

```
Application (OTel SDK) --> OTel Collector --> Prometheus (metrics)
                                          --> Loki (logs)
                                          --> Tempo (traces)
```

The key benefit: if you later switch from Grafana/Tempo to Datadog, you change the collector's configuration, not your application code.

### Alerting

Observability without alerting is just data hoarding. Critical alerts for the platform:

| Alert                                    | Condition                        | Severity |
|------------------------------------------|----------------------------------|----------|
| Error rate > 5% for 5 minutes            | `rate(http_errors_total[5m]) > 0.05` | Critical |
| Payment webhook processing latency > 10s | p95 latency exceeds threshold     | Critical |
| Database connection pool > 80% utilized  | `pg_pool_used / pg_pool_max > 0.8` | Warning  |
| Redis memory usage > 80%                 | `redis_used_memory_pct > 80`      | Warning  |
| Background job queue depth > 1000        | Jobs waiting exceed threshold     | Warning  |
| Zero bookings in the last hour (weekday) | Business metric anomaly           | Warning  |
| SSL certificate expires in < 14 days     | Certificate expiration check      | Warning  |

### Why It Is Used

Without observability, debugging a production issue is guesswork. When a client reports "I can't book a session," you need to trace their request through the gateway, load balancer, application, database, Stripe, and notification system to find where it failed. Observability tools make this traceable in minutes instead of hours.

For the Gym Coaching Platform, where payments and bookings are core revenue-generating actions, any downtime or degradation must be detected and resolved quickly.

### Example Technologies

| Technology         | Role            | Notes                                          |
|--------------------|-----------------|------------------------------------------------|
| **Prometheus**     | Metrics store   | Pull-based, time-series database, PromQL       |
| **Grafana**        | Dashboards      | Visualization for metrics, logs, and traces     |
| **Loki**           | Log aggregation | Label-based indexing, pairs with Grafana        |
| **Tempo**          | Trace storage   | Distributed tracing backend, pairs with Grafana |
| **Jaeger**         | Trace storage   | Alternative to Tempo, CNCF project              |
| **Alertmanager**   | Alert routing   | Deduplication, grouping, routing to Slack/PagerDuty |
| **Datadog**        | All-in-one      | Managed, expensive, excellent UX                |
| **New Relic**      | All-in-one      | Managed, generous free tier                     |

### Trade-offs

| Advantage                                   | Disadvantage                                       |
|---------------------------------------------|----------------------------------------------------|
| Dramatically reduces mean time to resolution | Infrastructure cost (storage, compute for metrics)  |
| Enables proactive detection before users notice | Alert fatigue if thresholds are too sensitive     |
| Traces pinpoint bottlenecks across services  | Instrumentation adds slight overhead to requests    |
| Business metrics inform product decisions    | Requires team discipline to maintain dashboards     |
| OpenTelemetry prevents vendor lock-in        | Initial setup and configuration is non-trivial      |

### How It Fits

**Approach A -- Modular Monolith:**
Observability is simpler because all modules run in one process. A single OTel SDK instruments the entire monolith. Traces show module-to-module calls as internal spans (no network hops). Metrics are collected from one application. Logs from all modules share one log stream, differentiated by a `module` label.

Dashboards are organized by module: Scheduling dashboard, Payments dashboard, overall system health dashboard. A single Grafana instance covers everything.

**Approach B -- Microservices:**
Observability is essential because requests cross network boundaries. A single booking request generates spans across the API Gateway, Scheduling Service, Payment Service, and Notification Service. Without distributed tracing, correlating logs across services is nearly impossible.

Each service emits its own metrics, logs, and traces. The OTel Collector aggregates everything into the shared observability backend. Service-level dashboards, cross-service trace views, and service dependency maps become critical operational tools.

The observability infrastructure itself must be highly available -- if Prometheus goes down during an incident, you are flying blind.
