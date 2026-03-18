# DDD Bounded Context Map

The 12 bounded contexts are split into three domain types. Each diagram below focuses on one set of relationships so the connections are readable.

---

## Domain Classification

```mermaid
graph TB
    subgraph Core["Core Domain — Competitive Advantage"]
        Coaching["Coaching"]
        Programs["Programs"]
    end

    subgraph Supporting["Supporting Domain"]
        Scheduling["Scheduling"]
        Communication["Communication"]
        Content["Content"]
        Video["Video Sessions"]
        Support["Support"]
    end

    subgraph Generic["Generic Domain"]
        Identity["Identity & Access"]
        Payments["Payments"]
        Notifications["Notifications"]
        Analytics["Analytics"]
        Search["Search"]
    end

    style Core fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Supporting fill:#bbdefb,stroke:#1565c0,stroke-width:2px
    style Generic fill:#ffe0b2,stroke:#e65100,stroke-width:2px
```

---

## Identity — Upstream to Everything

Identity & Access is the authority for user data. All other contexts depend on it.

```mermaid
graph LR
    Identity["Identity & Access<br/>Users, Roles, Permissions"]

    Identity -->|"OHS/PL<br/>JWT claims"| Coaching["Coaching"]
    Identity -->|"OHS/PL"| Scheduling["Scheduling"]
    Identity -->|"OHS/PL"| Payments["Payments"]
    Identity -->|"OHS/PL"| Communication["Communication"]
    Identity -->|"OHS/PL"| Programs["Programs"]
    Identity -->|"OHS/PL"| Support["Support"]

    style Identity fill:#ffe0b2,stroke:#e65100,stroke-width:2px
```

---

## Core Domain Relationships

Coaching and Programs are the platform's core value. They feed data to Scheduling, Communication, Content, Search, and Video.

```mermaid
graph LR
    Coaching["Coaching<br/>Profiles, Services"]
    Programs["Programs<br/>Workout Plans, Progress"]

    Coaching -->|"SK<br/>CoachId, ClientId"| Programs
    Coaching -->|"OHS/PL<br/>Availability"| Scheduling["Scheduling"]
    Coaching -->|"OHS/PL<br/>Coach/Client pairs"| Communication["Communication"]
    Coaching -->|"OHS/PL"| Video["Video Sessions"]
    Coaching -->|"OHS/PL<br/>Profiles indexed"| Search["Search"]

    Programs -->|"OHS/PL<br/>Exercise refs"| Content["Content"]
    Programs -->|"OHS/PL<br/>Programs indexed"| Search

    style Coaching fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Programs fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

---

## Scheduling & Payments Flow

Scheduling creates bookings. Payments processes charges. Events flow to Notifications and Analytics.

```mermaid
graph LR
    Scheduling["Scheduling<br/>Bookings, Availability"]
    Payments["Payments<br/>Charges, Subscriptions"]

    Scheduling -->|"ACL<br/>Booking → Payment"| Payments
    Scheduling -->|"OHS/PL<br/>Booking events"| Notifications["Notifications"]
    Scheduling -->|"OHS/PL"| Video["Video Sessions"]
    Scheduling -->|"CF"| Analytics["Analytics"]

    Payments -->|"ACL<br/>Wraps Stripe"| Notifications
    Payments -->|"CF"| Analytics

    style Scheduling fill:#bbdefb,stroke:#1565c0,stroke-width:2px
    style Payments fill:#ffe0b2,stroke:#e65100,stroke-width:2px
```

---

## Event Consumers — Notifications, Analytics, Search

These three contexts consume events from nearly every other context.

```mermaid
graph RL
    Notifications["Notifications<br/>Email, SMS, Push"]
    Analytics["Analytics<br/>Metrics, Reports"]
    Search["Search<br/>Full-text Index"]

    Scheduling["Scheduling"] --> Notifications
    Payments["Payments"] --> Notifications
    Communication["Communication"] --> Notifications
    Video["Video Sessions"] --> Notifications
    Support["Support"] --> Notifications

    Scheduling --> Analytics
    Payments --> Analytics
    Programs["Programs"] --> Analytics
    Communication --> Analytics
    Video --> Analytics
    Support --> Analytics
    Notifications --> Analytics

    Coaching["Coaching"] --> Search
    Content["Content"] --> Search
    Programs --> Search

    style Notifications fill:#ffe0b2,stroke:#e65100,stroke-width:2px
    style Analytics fill:#ffe0b2,stroke:#e65100,stroke-width:2px
    style Search fill:#ffe0b2,stroke:#e65100,stroke-width:2px
```

---

## Relationship Legend

| Pattern | Abbreviation | Description |
|---|---|---|
| Open Host Service / Published Language | OHS/PL | Upstream exposes a well-defined API and published events |
| Anti-Corruption Layer | ACL | Downstream translates upstream models to protect its own domain |
| Conformist | CF | Downstream adopts the upstream model as-is |
| Shared Kernel | SK | Two contexts share a small, co-owned model (e.g., CoachId, ClientId) |
