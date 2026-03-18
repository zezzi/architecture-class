# Bounded Contexts (DDD)

The platform is decomposed into 12 bounded contexts, each representing a cohesive area of the business domain. This section defines each context's responsibilities, core entities, domain events, and relationships.

---

### Context Map Overview

**Domain Classification:**

```mermaid
graph TB
    subgraph Core["Core Domain"]
        COACHING[Coaching]
        PROGRAMS[Programs]
        SCHEDULING[Scheduling]
    end

    subgraph Supporting["Supporting Domain"]
        COMMUNICATION[Communication]
        VIDEO[Video Sessions]
        CONTENT[Content]
    end

    subgraph Generic["Generic Domain"]
        IDENTITY[Identity & Access]
        PAYMENTS[Payments]
        NOTIFICATIONS[Notifications]
        ANALYTICS[Analytics]
        SEARCH[Search]
        SUPPORT[Support]
    end

    style Core fill:#c8e6c9,stroke:#2e7d32
    style Supporting fill:#bbdefb,stroke:#1565c0
    style Generic fill:#ffe0b2,stroke:#e65100
```

**Core Event Flow — Booking & Payment:**

```mermaid
graph LR
    IDENTITY[Identity] -->|UserRegistered| COACHING[Coaching]
    COACHING -->|ServiceCreated| SCHEDULING[Scheduling]
    SCHEDULING -->|BookingCreated| PAYMENTS[Payments]
    PAYMENTS -->|PaymentConfirmed| SCHEDULING
    SCHEDULING -->|BookingConfirmed| VIDEO[Video Sessions]
```

**Event Consumers — Notifications, Analytics, Search:**

```mermaid
graph RL
    NOTIFICATIONS[Notifications]
    ANALYTICS[Analytics]
    SEARCH[Search]

    SCHEDULING[Scheduling] --> NOTIFICATIONS
    SCHEDULING --> ANALYTICS
    PAYMENTS[Payments] --> NOTIFICATIONS
    PAYMENTS --> ANALYTICS
    PROGRAMS[Programs] --> NOTIFICATIONS
    PROGRAMS --> ANALYTICS
    PROGRAMS --> SEARCH
    COACHING[Coaching] --> SEARCH
    CONTENT[Content] --> SEARCH
```

---

#
