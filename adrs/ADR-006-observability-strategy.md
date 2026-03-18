# ADR-006: Observability Strategy

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-03-18 |
| **Deciders** | Platform architecture team |

### Context

Production systems require three pillars of observability: **logs** (what happened), **metrics** (how the system is performing), and **traces** (how a request flows through the system). The choice affects cost, operational complexity, and the depth of insight available.

### Option A — ELK Stack (Elasticsearch, Logstash, Kibana)

The traditional open-source logging stack extended with APM for metrics and traces.

| Pros | Cons |
|------|------|
| Elasticsearch is extremely powerful for log search and aggregation | Elasticsearch is resource-hungry — requires significant memory (heap sizing) and disk |
| Kibana provides rich visualization and dashboarding | Logstash adds another component to manage; Filebeat/Fluentd are lighter alternatives but add choice complexity |
| Large community with extensive documentation | Elastic's licensing changes (SSPL) have created uncertainty in the open-source community |
| APM module supports distributed tracing | Operational burden is high: index management, shard balancing, cluster health monitoring |
| Full-text search across logs is unmatched | Cost at scale: Elasticsearch clusters are expensive to run and tune |
| | The stack is optimized for logs; metrics and traces feel bolted on |

### Option B — Grafana Stack (Prometheus + Grafana + Loki + Tempo)

A purpose-built stack where each pillar has a dedicated, lightweight tool: Prometheus for metrics, Loki for logs, Tempo for traces, and Grafana as the unified dashboard.

| Pros | Cons |
|------|------|
| Each component is purpose-built and lightweight | Four components to deploy and configure (though all are lightweight) |
| Prometheus is the industry standard for metrics; huge ecosystem of exporters | Loki does not index log content (only labels) — full-text search is less powerful than Elasticsearch |
| Loki uses the same label-based approach as Prometheus — consistent mental model | Prometheus is pull-based; requires service discovery or push gateway configuration |
| Tempo provides distributed tracing with minimal resource usage | Long-term metric storage requires Thanos or Cortex (adds complexity) |
| Grafana unifies logs, metrics, and traces in a single UI with cross-linking | Grafana dashboard creation has a learning curve |
| Fully open source (Apache 2.0) with no licensing concerns | Less mature than ELK for complex log analytics |
| Low resource footprint — runs on a single small VM for gym-scale workloads | |
| Clear separation of pillars unified in one UI | |

### Option C — Managed Observability (Datadog, New Relic)

A fully managed SaaS platform that handles ingestion, storage, querying, alerting, and dashboarding.

| Pros | Cons |
|------|------|
| Zero infrastructure to manage — just install an agent | Cost scales with data volume; can become expensive quickly |
| Best-in-class UI, alerting, and anomaly detection | Vendor lock-in: proprietary query languages, dashboard formats, and agent protocols |
| Distributed tracing, log management, and metrics in one product | Less transparency — the team interacts with a "magic box" with limited customization |
| Enterprise-grade features: SLOs, error tracking, profiling | Free tiers are limited (e.g., Datadog: 5 hosts, 1-day retention) |
| Fastest time to value — works out of the box | Data residency concerns for sensitive user information |
| | Higher cost; many enterprise features may go unused at moderate scale |

### Decision

**Adopt the Grafana Stack (Option B) for self-hosted environments, with an acknowledged migration path to managed (Option C) if operational overhead becomes a burden at scale.**

### Stack Components and Their Roles

| Pillar | Tool | Role | Data Example |
|--------|------|------|-------------|
| **Metrics** | Prometheus | Scrapes numeric time-series data from the application | `http_request_duration_seconds{method="POST", path="/bookings", status="201"}` |
| **Logs** | Loki | Aggregates structured logs from the application | `{"level":"info","module":"booking","msg":"Booking confirmed","bookingId":"..."}` |
| **Traces** | Tempo | Stores distributed traces (spans) from instrumented code | A trace showing: API Gateway → Booking Module → Payment Module → Database |
| **Dashboard** | Grafana | Unified visualization with drill-down from metrics to traces to logs | Dashboard panel showing booking latency P95, with a link to the slowest trace |

### Instrumentation Approach

```
Application (NestJS / Spring Boot)
  │
  ├── OpenTelemetry SDK (vendor-neutral instrumentation)
  │     ├── Exports metrics  → Prometheus (scrape endpoint :9090/metrics)
  │     ├── Exports traces   → Tempo (OTLP gRPC :4317)
  │     └── Exports logs     → Loki (via Promtail or direct push)
  │
  └── Structured JSON logging (correlation ID in every log line)
```

Using **OpenTelemetry** as the instrumentation layer ensures vendor neutrality. If the team later migrates to Datadog or New Relic (Option C), only the exporter configuration changes — application code remains untouched.

### Key Dashboards to Build

1. **Platform Health** — Request rate, error rate, latency percentiles (RED method)
2. **Booking Funnel** — Bookings created, confirmed, cancelled, payment failures
3. **Infrastructure** — CPU, memory, disk, database connections
4. **Business Metrics** — Active clients, revenue per coach, content engagement

### Final Justification

The Grafana Stack provides comprehensive observability at minimal cost, with each component purpose-built for its pillar. At moderate scale, the entire stack runs comfortably on a small Kubernetes cluster and scales horizontally as demand grows. This stack also provides clear separation of concerns: each pillar is configured explicitly, making it straightforward to understand how metrics, logs, and traces work together through correlation IDs in Grafana's unified UI. The use of OpenTelemetry for instrumentation means the application code is not coupled to the observability backend, preserving the option to migrate to a managed provider when operational convenience outweighs cost savings.
