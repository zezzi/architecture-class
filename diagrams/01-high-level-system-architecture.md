# High-Level System Architecture (C4 Context)

This section shows how external actors and internal systems interact with the platform. The diagrams are split by concern so each one is readable on its own.

---

## 1A. Request Flow — How Traffic Reaches the Application

```mermaid
graph TB
    Web["Web App"]
    Mobile["Mobile App"]
    Coach["Coach Dashboard"]

    CDN["CloudFlare CDN"]
    GW["API Gateway<br/>Kong / AWS API GW"]
    LB["Load Balancer<br/>NGINX / ALB"]

    App["Core Platform<br/>(Monolith or Microservices)"]

    Web --> CDN
    Mobile --> CDN
    Coach --> CDN
    CDN --> GW
    Web --> GW
    Mobile --> GW
    Coach --> GW
    GW --> LB
    LB --> App

    style CDN fill:#fff3e0,stroke:#f57c00
    style GW fill:#fce4ec,stroke:#c62828
    style LB fill:#fce4ec,stroke:#c62828
    style App fill:#e8f5e9,stroke:#2e7d32
```

---

## 1B. Approach A — Modular Monolith Overview

```mermaid
graph TB
    App["NestJS Monolith<br/>12 Modules"]
    Workers["Background Workers"]
    RMQ["RabbitMQ"]

    PG["PostgreSQL<br/>Schema per Module"]
    Redis["Redis<br/>Cache + Sessions"]
    S3["S3<br/>Media Storage"]
    ES["Elasticsearch<br/>Search Index"]

    App --> PG
    App --> Redis
    App --> S3
    App --> RMQ
    RMQ --> Workers
    Workers --> ES

    style App fill:#e8f5e9,stroke:#2e7d32
    style Workers fill:#e0f7fa,stroke:#00838f
    style RMQ fill:#e0f2f1,stroke:#00695c
    style PG fill:#f3e5f5,stroke:#7b1fa2
    style Redis fill:#f3e5f5,stroke:#7b1fa2
    style S3 fill:#f3e5f5,stroke:#7b1fa2
    style ES fill:#f3e5f5,stroke:#7b1fa2
```

---

## 1C. Approach B — Microservices Overview

```mermaid
graph TB
    GW["API Gateway"]

    subgraph Services["Kubernetes Cluster"]
        S1["Identity"]
        S2["Coaching"]
        S3s["Scheduling"]
        S4["Payments"]
        S5["Programs"]
        S6["Communication"]
    end

    subgraph Services2["  "]
        S7["Content"]
        S8["Video"]
        S9["Notifications"]
        S10["Analytics"]
        S11["Search"]
        S12["Support"]
    end

    Kafka["Apache Kafka"]

    GW --> S1
    GW --> S2
    GW --> S3s
    GW --> S4
    GW --> S5
    GW --> S6
    GW --> S7
    GW --> S8

    S3s --> Kafka
    S4 --> Kafka
    S5 --> Kafka
    S6 --> Kafka

    Kafka --> S9
    Kafka --> S10
    Kafka --> S11

    style GW fill:#fce4ec,stroke:#c62828
    style Services fill:#e8f5e9,stroke:#2e7d32
    style Services2 fill:#e8f5e9,stroke:#2e7d32
    style Kafka fill:#e0f2f1,stroke:#00695c
```

---

## 1D. External Services

```mermaid
graph LR
    App["Core Platform"]

    Auth0["Auth0<br/>Authentication"]
    Stripe["Stripe<br/>Payments"]
    Zoom["Zoom API<br/>Video"]
    SendGrid["SendGrid<br/>Email"]
    Twilio["Twilio<br/>SMS / Push"]

    App --> Auth0
    App --> Stripe
    App --> Zoom
    App --> SendGrid
    App --> Twilio

    style App fill:#e8f5e9,stroke:#2e7d32
    style Auth0 fill:#fff8e1,stroke:#f9a825
    style Stripe fill:#fff8e1,stroke:#f9a825
    style Zoom fill:#fff8e1,stroke:#f9a825
    style SendGrid fill:#fff8e1,stroke:#f9a825
    style Twilio fill:#fff8e1,stroke:#f9a825
```

---

## 1E. Observability & Analytics

```mermaid
graph LR
    App["Core Platform"]

    Prom["Prometheus<br/>Metrics"]
    Loki["Loki<br/>Logs"]
    Tempo["Tempo<br/>Traces"]
    Graf["Grafana<br/>Dashboards"]

    Collector["Event Collector"]
    DW["Data Warehouse<br/>ClickHouse"]
    BI["BI Dashboard"]

    App --> Prom
    App --> Loki
    App --> Tempo
    Prom --> Graf
    Loki --> Graf
    Tempo --> Graf

    App --> Collector
    Collector --> DW
    DW --> BI

    style App fill:#e8f5e9,stroke:#2e7d32
    style Prom fill:#efebe9,stroke:#4e342e
    style Loki fill:#efebe9,stroke:#4e342e
    style Tempo fill:#efebe9,stroke:#4e342e
    style Graf fill:#efebe9,stroke:#4e342e
    style Collector fill:#e8eaf6,stroke:#283593
    style DW fill:#e8eaf6,stroke:#283593
    style BI fill:#e8eaf6,stroke:#283593
```
