# Analytics Pipeline

### What It Does

The analytics pipeline collects, processes, stores, and visualizes business events to answer questions like: "Which coaches have the highest client retention?", "What time slots are most popular?", "What is our monthly recurring revenue trend?", and "Where do users drop off in the booking funnel?"

### Pipeline Stages

```
Collection --> Processing --> Storage --> Visualization
```

**1. Collection:**
Events are emitted from the application whenever something business-relevant happens:
- `session_booked`, `session_completed`, `session_cancelled`, `session_no_show`
- `payment_completed`, `subscription_started`, `subscription_churned`
- `workout_plan_created`, `workout_completed`, `exercise_logged`
- `page_viewed`, `feature_used`, `search_performed`
- `coach_onboarded`, `client_registered`, `client_invited`

Events include context: timestamp, user ID, session ID, device type, geo-location, and event-specific properties.

**2. Processing:**
Raw events are transformed, enriched, and aggregated:
- **Enrichment** -- Joining a `session_booked` event with coach and client metadata.
- **Deduplication** -- Removing duplicate events caused by retries.
- **Aggregation** -- Computing daily/weekly/monthly rollups (e.g., "bookings per coach per week").
- **Sessionization** -- Grouping events into user sessions for funnel analysis.

**3. Storage:**
Processed data is stored in an analytics-optimized database. Unlike the operational PostgreSQL database (optimized for transactional reads/writes), the analytics store is optimized for large-scale aggregation queries.

**4. Visualization:**
Dashboards present insights to stakeholders:
- **Platform operators** -- Revenue, active users, churn rate, system health.
- **Coaches** -- Client retention, session completion rate, revenue per client.
- **Product team** -- Feature adoption, booking funnel conversion, A/B test results.

### Why It Is Used

The operational database can answer "How many bookings did Coach #42 have this week?" but it struggles with "What is the correlation between workout plan completion rates and subscription retention across all coaches over the last 6 months?" Running such queries against the production database would be slow and could degrade performance for live users.

The analytics pipeline provides a dedicated system for complex, retrospective queries without impacting the operational workload.

### Example Technologies

| Technology          | Role              | Notes                                        |
|---------------------|-------------------|----------------------------------------------|
| **ClickHouse**      | Analytics database | Column-oriented, extremely fast aggregations  |
| **PostgreSQL** (read replica) | Analytics database | Acceptable for moderate scale, reuses existing tech |
| **Apache Kafka**    | Event transport    | Durable event log between collection and processing |
| **Apache Flink**    | Stream processing  | Real-time aggregation and transformation       |
| **dbt**             | Data transformation | SQL-based, version-controlled data models     |
| **Metabase**        | Visualization      | Open-source BI, SQL-based dashboards           |
| **Apache Superset** | Visualization      | Open-source BI, broader feature set            |
| **Grafana**         | Visualization      | Can also serve analytics dashboards            |
| **Segment**         | Collection          | Managed event collection and routing           |
| **PostHog**         | All-in-one          | Open-source product analytics                  |

### Business Metrics for the Platform

| Metric                          | Query Pattern                               | Audience         |
|---------------------------------|---------------------------------------------|------------------|
| Monthly Recurring Revenue (MRR) | Sum of active subscription values            | Operators        |
| Churn Rate                      | Cancelled subscriptions / total subscriptions | Operators        |
| Booking Conversion Rate         | Bookings / booking page views                | Product          |
| Average Revenue Per Coach       | Total revenue / active coaches               | Operators        |
| Session Completion Rate         | Completed sessions / booked sessions          | Coaches          |
| Client Retention (90-day)       | Clients active after 90 days / registered     | Product          |
| No-Show Rate                    | No-shows / total booked sessions              | Coaches          |
| Popular Time Slots              | Bookings grouped by day-of-week and hour     | Product/Coaches  |
| Workout Plan Adherence          | Completed exercises / assigned exercises      | Coaches          |
| Payment Failure Rate            | Failed payments / attempted payments          | Operators        |

### Trade-offs

| Advantage                                     | Disadvantage                                         |
|-----------------------------------------------|------------------------------------------------------|
| Data-driven product and business decisions     | Infrastructure cost (ClickHouse, processing pipelines) |
| Keeps analytics queries off the production DB  | Event schema evolution requires careful versioning    |
| Historical analysis enables trend detection    | Pipeline bugs can produce incorrect metrics           |
| Coaches get insights that increase retention   | Data privacy regulations (GDPR) apply to analytics data |
| A/B testing infrastructure for product growth  | Initial setup is significant engineering investment   |

### How It Fits

**Approach A -- Modular Monolith:**
The Analytics module collects events from other modules via the in-memory event bus or RabbitMQ. Events are written to a dedicated analytics schema in PostgreSQL (or a separate ClickHouse instance if query performance demands it). A daily dbt job transforms raw events into aggregate tables. Metabase connects to the analytics schema to power dashboards.

For early-stage and moderate scale, PostgreSQL with materialized views can serve as the analytics store, avoiding the need for a separate database:

```
Modules --events--> Analytics Module --writes--> PostgreSQL (analytics schema)
                                                     |
                                                 Materialized Views (daily refresh)
                                                     |
                                                 Metabase (dashboards)
```

**Approach B -- Microservices:**
Each service produces events to Kafka topics. A dedicated Analytics Service (or pipeline) consumes from all topics, processes events (deduplication, enrichment, aggregation), and writes to ClickHouse. This service is independently scalable and does not affect any operational service.

```
All Services --events--> Kafka --> Analytics Service --> ClickHouse --> Metabase
                                                                   --> Grafana
                                                                   --> Coach Dashboard API
```

The Analytics Service also feeds the Coach Dashboard: when a coach logs in and sees "Your client retention is 85%," that number comes from the analytics pipeline, not from a real-time query against the Scheduling database.
