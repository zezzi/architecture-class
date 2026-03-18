# Diagram 1: High-Level System Architecture (C4 Context)

This C4-style context diagram shows how external actors (clients, third-party services) interact with the platform's core application boundary. The internal box represents either a single deployable monolith (Approach A) or a set of independently deployed microservices (Approach B). All external dependencies are shown at the edges.

```mermaid
graph TB
    subgraph Clients["Client Applications"]
        WebApp["Web App<br/>(React SPA)"]
        MobileApp["Mobile App<br/>(React Native)"]
        CoachDashboard["Coach Dashboard<br/>(Web)"]
    end

    subgraph CDN_Layer["Content Delivery"]
        CDN["CloudFlare CDN<br/>(Static Assets, Video Cache)"]
    end

    subgraph Gateway_Layer["API Layer"]
        APIGW["API Gateway<br/>(Kong / AWS API GW)<br/>Rate Limiting, Auth, Routing"]
        LB["Load Balancer<br/>(NGINX / ALB)"]
    end

    subgraph Core["Core Platform Boundary"]
        direction TB
        subgraph AppA["Approach A: Modular Monolith"]
            Monolith["Single Deployable<br/>12 Bounded Context Modules<br/>(Identity, Coaching, Scheduling,<br/>Payments, Programs, Communication,<br/>Content, Video, Notifications,<br/>Analytics, Search, Support)"]
            Workers_A["Background Workers<br/>(Async Job Processing)"]
        end
        subgraph AppB["Approach B: Microservices"]
            IdentitySvc["Identity & Access Service"]
            CoachingSvc["Coaching Service"]
            SchedulingSvc["Scheduling Service"]
            PaymentsSvc["Payments Service"]
            ProgramsSvc["Programs Service"]
            CommSvc["Communication Service"]
            ContentSvc["Content Service"]
            VideoSvc["Video Sessions Service"]
            NotifSvc["Notifications Service"]
            AnalyticsSvc["Analytics Service"]
            SearchSvc["Search Service"]
            SupportSvc["Support Service"]
        end
    end

    subgraph Messaging["Message Broker"]
        RabbitMQ["RabbitMQ<br/>(Approach A: Async Queues)"]
        Kafka["Apache Kafka<br/>(Approach B: Event Streaming)"]
    end

    subgraph DataStores["Data Stores"]
        PG["PostgreSQL<br/>(Primary Database)"]
        Redis["Redis<br/>(Cache, Sessions, Rate Limits)"]
        S3["S3 / Object Storage<br/>(Videos, Images, Documents)"]
    end

    subgraph External["External Services"]
        Auth0["Auth0<br/>(Authentication / OAuth2)"]
        Stripe["Stripe<br/>(Payment Processing)"]
        Zoom["Zoom API<br/>(Video Conferencing)"]
        SendGrid["SendGrid<br/>(Transactional Email)"]
        Twilio["Twilio<br/>(SMS / Push Notifications)"]
    end

    subgraph Observability["Observability Stack"]
        Grafana["Grafana<br/>(Dashboards)"]
        Prometheus["Prometheus<br/>(Metrics)"]
        Loki["Loki<br/>(Log Aggregation)"]
        Tempo["Tempo<br/>(Distributed Tracing)"]
    end

    subgraph AnalyticsPipeline["Analytics Pipeline"]
        EventCollector["Event Collector"]
        DataWarehouse["Data Warehouse<br/>(ClickHouse / BigQuery)"]
        BIDashboard["BI Dashboard"]
    end

    WebApp --> CDN
    MobileApp --> CDN
    CoachDashboard --> CDN
    CDN --> APIGW
    WebApp --> APIGW
    MobileApp --> APIGW
    CoachDashboard --> APIGW
    APIGW --> LB
    LB --> Core

    Core --> RabbitMQ
    Core --> Kafka
    Core --> PG
    Core --> Redis
    Core --> S3
    Core --> Auth0
    Core --> Stripe
    Core --> Zoom
    Core --> SendGrid
    Core --> Twilio

    RabbitMQ --> Workers_A
    Kafka --> AppB

    Core --> Prometheus
    Core --> Loki
    Core --> Tempo
    Prometheus --> Grafana
    Loki --> Grafana
    Tempo --> Grafana

    Core --> EventCollector
    EventCollector --> DataWarehouse
    DataWarehouse --> BIDashboard

    style Clients fill:#e1f5fe,stroke:#0288d1
    style CDN_Layer fill:#fff3e0,stroke:#f57c00
    style Gateway_Layer fill:#fce4ec,stroke:#c62828
    style Core fill:#e8f5e9,stroke:#2e7d32
    style DataStores fill:#f3e5f5,stroke:#7b1fa2
    style External fill:#fff8e1,stroke:#f9a825
    style Messaging fill:#e0f2f1,stroke:#00695c
    style Observability fill:#efebe9,stroke:#4e342e
    style AnalyticsPipeline fill:#e8eaf6,stroke:#283593
```
