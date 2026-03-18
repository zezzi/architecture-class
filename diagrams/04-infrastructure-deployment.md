# Component / Infrastructure Diagram

Physical deployment topology for both approaches, broken into focused diagrams.

---

## Approach A: Modular Monolith Deployment

### A1. Request Path

```mermaid
graph TB
    Web["Web Browser"]
    Mobile["Mobile App"]

    CF["CloudFlare CDN<br/>SSL, DDoS, Caching"]
    Kong["Kong API Gateway<br/>Auth, Rate Limiting, Routing"]
    NGINX["NGINX / ALB<br/>Round-robin, Health Checks"]

    App1["App Instance 1"]
    App2["App Instance 2"]
    App3["App Instance N"]

    Web --> CF
    Mobile --> CF
    CF --> Kong
    Kong --> NGINX
    NGINX --> App1
    NGINX --> App2
    NGINX --> App3

    style CF fill:#fff3e0,stroke:#f57c00
    style Kong fill:#fce4ec,stroke:#c62828
    style NGINX fill:#fce4ec,stroke:#c62828
    style App1 fill:#e8f5e9,stroke:#2e7d32
    style App2 fill:#e8f5e9,stroke:#2e7d32
    style App3 fill:#e8f5e9,stroke:#2e7d32
```

---

### A2. Internal Modules (Inside Each App Instance)

```mermaid
graph TB
    subgraph Monolith["NestJS Application"]
        Identity["Identity & Access"]
        Coaching["Coaching"]
        Scheduling["Scheduling"]
        Payments["Payments"]
        Programs["Programs"]
        Comm["Communication"]
        Content["Content"]
        Video["Video Sessions"]
        Notif["Notifications"]
        Analytics["Analytics"]
        Search["Search"]
        Support["Support"]
    end

    style Monolith fill:#e8f5e9,stroke:#2e7d32
```

---

### A3. Data Layer

```mermaid
graph TB
    App["App Instances"]

    PG["PostgreSQL Primary<br/>identity.* | coaching.*<br/>scheduling.* | payments.*<br/>programs.* | etc."]
    Replica["PostgreSQL Replica<br/>Read-only"]
    Redis["Redis Cluster<br/>Sessions, Cache,<br/>Rate Limits"]
    S3["S3 / MinIO<br/>Videos, Images,<br/>Documents"]
    ES["Elasticsearch<br/>Search Index"]

    App --> PG
    App --> Replica
    App --> Redis
    App --> S3
    App --> ES

    style App fill:#e8f5e9,stroke:#2e7d32
    style PG fill:#f3e5f5,stroke:#7b1fa2
    style Replica fill:#f3e5f5,stroke:#7b1fa2
    style Redis fill:#f3e5f5,stroke:#7b1fa2
    style S3 fill:#f3e5f5,stroke:#7b1fa2
    style ES fill:#f3e5f5,stroke:#7b1fa2
```

---

### A4. Async Processing

```mermaid
graph LR
    App["App Instances"]

    RMQ["RabbitMQ<br/>booking.created<br/>payment.confirmed<br/>notification.send<br/>analytics.event"]

    W1["Email Worker"]
    W2["Payment Webhook Worker"]
    W3["Analytics Worker"]
    W4["Search Indexing Worker"]
    W5["Video Processing Worker"]

    App --> RMQ
    RMQ --> W1
    RMQ --> W2
    RMQ --> W3
    RMQ --> W4
    RMQ --> W5

    style App fill:#e8f5e9,stroke:#2e7d32
    style RMQ fill:#e0f2f1,stroke:#00695c
    style W1 fill:#e0f7fa,stroke:#00838f
    style W2 fill:#e0f7fa,stroke:#00838f
    style W3 fill:#e0f7fa,stroke:#00838f
    style W4 fill:#e0f7fa,stroke:#00838f
    style W5 fill:#e0f7fa,stroke:#00838f
```

---

### A5. External Services & Observability

```mermaid
graph LR
    App["App Instances"]
    Workers["Workers"]

    Auth0["Auth0"]
    Stripe["Stripe"]
    Zoom["Zoom"]
    SendGrid["SendGrid"]
    Twilio["Twilio"]

    App --> Auth0
    App --> Stripe
    App --> Zoom
    Workers --> SendGrid
    Workers --> Twilio

    style App fill:#e8f5e9,stroke:#2e7d32
    style Workers fill:#e0f7fa,stroke:#00838f
    style Auth0 fill:#fff8e1,stroke:#f9a825
    style Stripe fill:#fff8e1,stroke:#f9a825
    style Zoom fill:#fff8e1,stroke:#f9a825
    style SendGrid fill:#fff8e1,stroke:#f9a825
    style Twilio fill:#fff8e1,stroke:#f9a825
```

```mermaid
graph LR
    App["App Instances"]

    Prom["Prometheus<br/>Metrics"]
    Loki["Loki<br/>Logs"]
    Tempo["Tempo<br/>Traces"]
    Graf["Grafana<br/>Dashboards"]
    Alert["Alertmanager<br/>PagerDuty / Slack"]

    App --> Prom
    App --> Loki
    App --> Tempo
    Prom --> Graf
    Loki --> Graf
    Tempo --> Graf
    Prom --> Alert

    style App fill:#e8f5e9,stroke:#2e7d32
    style Prom fill:#efebe9,stroke:#4e342e
    style Loki fill:#efebe9,stroke:#4e342e
    style Tempo fill:#efebe9,stroke:#4e342e
    style Graf fill:#efebe9,stroke:#4e342e
    style Alert fill:#efebe9,stroke:#4e342e
```

---

## Approach B: Full Microservices Deployment

### B1. Request Path

```mermaid
graph TB
    Web["Web Browser"]
    Mobile["Mobile App"]

    CF["CloudFlare CDN"]
    GW["API Gateway<br/>Kong / AWS API GW"]

    subgraph Mesh["Service Mesh — Istio"]
        Identity["Identity Svc"]
        Coaching["Coaching Svc"]
        Scheduling["Scheduling Svc"]
        Payments["Payments Svc"]
        Programs["Programs Svc"]
        Comm["Communication Svc"]
    end

    Web --> CF
    Mobile --> CF
    CF --> GW

    GW --> Identity
    GW --> Coaching
    GW --> Scheduling
    GW --> Payments
    GW --> Programs
    GW --> Comm

    style CF fill:#fff3e0,stroke:#f57c00
    style GW fill:#fce4ec,stroke:#c62828
    style Mesh fill:#e8f5e9,stroke:#2e7d32
```

```mermaid
graph TB
    GW["API Gateway"]

    subgraph Mesh2["Service Mesh — Istio"]
        Content["Content Svc"]
        Video["Video Svc"]
        Notif["Notification Svc"]
        Analytics["Analytics Svc"]
        Search["Search Svc"]
        Support["Support Svc"]
    end

    GW --> Content
    GW --> Video
    GW --> Notif
    GW --> Search
    GW --> Support

    style GW fill:#fce4ec,stroke:#c62828
    style Mesh2 fill:#e8f5e9,stroke:#2e7d32
```

---

### B2. Database-per-Service

Each service owns its own database. No cross-service queries.

```mermaid
graph LR
    Identity["Identity Svc"] --> ID_DB["PostgreSQL"]
    Coaching["Coaching Svc"] --> Coach_DB["PostgreSQL"]
    Scheduling["Scheduling Svc"] --> Sched_DB["PostgreSQL"]
    Payments["Payments Svc"] --> Pay_DB["PostgreSQL"]
    Programs["Programs Svc"] --> Prog_DB["PostgreSQL"]
    Comm["Communication Svc"] --> Comm_DB["PostgreSQL"]

    style ID_DB fill:#f3e5f5,stroke:#7b1fa2
    style Coach_DB fill:#f3e5f5,stroke:#7b1fa2
    style Sched_DB fill:#f3e5f5,stroke:#7b1fa2
    style Pay_DB fill:#f3e5f5,stroke:#7b1fa2
    style Prog_DB fill:#f3e5f5,stroke:#7b1fa2
    style Comm_DB fill:#f3e5f5,stroke:#7b1fa2
```

```mermaid
graph LR
    Content["Content Svc"] --> Content_DB["PostgreSQL"]
    Video["Video Svc"] --> Video_DB["PostgreSQL"]
    Notif["Notification Svc"] --> Notif_DB["PostgreSQL"]
    Analytics["Analytics Svc"] --> Analytics_DB["ClickHouse"]
    Search["Search Svc"] --> Search_DB["Elasticsearch"]
    Support["Support Svc"] --> Support_DB["PostgreSQL"]

    style Content_DB fill:#f3e5f5,stroke:#7b1fa2
    style Video_DB fill:#f3e5f5,stroke:#7b1fa2
    style Notif_DB fill:#f3e5f5,stroke:#7b1fa2
    style Analytics_DB fill:#f3e5f5,stroke:#7b1fa2
    style Search_DB fill:#f3e5f5,stroke:#7b1fa2
    style Support_DB fill:#f3e5f5,stroke:#7b1fa2
```

---

### B3. Event Streaming (Kafka)

```mermaid
graph LR
    subgraph Producers["Event Producers"]
        Scheduling["Scheduling"]
        Payments["Payments"]
        Programs["Programs"]
        Coaching["Coaching"]
        Communication["Communication"]
        Video["Video"]
    end

    Kafka["Apache Kafka<br/>3 Brokers"]
    Schema["Schema Registry<br/>Avro / Protobuf"]

    subgraph Consumers["Event Consumers"]
        Notifications["Notifications"]
        Analytics["Analytics"]
        Search["Search"]
    end

    Scheduling --> Kafka
    Payments --> Kafka
    Programs --> Kafka
    Coaching --> Kafka
    Communication --> Kafka
    Video --> Kafka

    Kafka --> Notifications
    Kafka --> Analytics
    Kafka --> Search

    Kafka --- Schema

    style Producers fill:#e8f5e9,stroke:#2e7d32
    style Kafka fill:#e0f2f1,stroke:#00695c
    style Schema fill:#e0f2f1,stroke:#00695c
    style Consumers fill:#ffe0b2,stroke:#e65100
```

---

### B4. Shared Infrastructure & CI/CD

```mermaid
graph TB
    subgraph Shared["Shared Infrastructure"]
        Redis["Redis Cluster<br/>Distributed Cache"]
        S3["S3 / Object Storage"]
    end

    subgraph Ext["External Services"]
        Auth0["Auth0"]
        Stripe["Stripe"]
        Zoom["Zoom"]
        SendGrid["SendGrid"]
        Twilio["Twilio"]
    end

    subgraph CICD["CI/CD & Orchestration"]
        K8s["Kubernetes<br/>EKS / GKE"]
        Argo["ArgoCD<br/>GitOps"]
        Registry["Container Registry<br/>ECR / GCR"]
    end

    Registry --> K8s
    Argo --> K8s

    style Shared fill:#f3e5f5,stroke:#7b1fa2
    style Ext fill:#fff8e1,stroke:#f9a825
    style CICD fill:#e8eaf6,stroke:#283593
```

---

## Key Differences Between Approaches

| Aspect | Approach A (Modular Monolith) | Approach B (Microservices) |
|---|---|---|
| **Deployment** | Single artifact, multiple instances | 12+ independent services on Kubernetes |
| **Database** | Shared PostgreSQL with schema-per-module | Database-per-service |
| **Communication** | In-process + RabbitMQ for async | Kafka event streaming + gRPC |
| **Scaling** | Scale entire monolith horizontally | Scale individual services independently |
| **Observability** | Standard logging + Prometheus | Full distributed tracing required |
| **Complexity** | Lower operational overhead | Higher, requires service mesh + orchestration |
| **Team structure** | Single or cross-functional teams | One team per service (Conway's Law) |
| **Recommended for** | Launch to growth (< 15 engineers) | Scale stage (15+ engineers) |
