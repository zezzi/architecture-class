# Bounded Contexts (DDD)

The platform is decomposed into 12 bounded contexts, each representing a cohesive area of the business domain. This section defines each context's responsibilities, core entities, domain events, and relationships.

---

### Context Map Overview

```mermaid
graph TB
    subgraph "Core Domain"
        COACHING[Coaching]
        PROGRAMS[Programs]
        SCHEDULING[Scheduling]
    end

    subgraph "Supporting Domain"
        PAYMENTS[Payments]
        COMMUNICATION[Communication]
        VIDEO[Video Sessions]
        CONTENT[Content]
    end

    subgraph "Generic Domain"
        IDENTITY[Identity & Access]
        NOTIFICATIONS[Notifications]
        ANALYTICS[Analytics]
        SEARCH[Search]
        SUPPORT[Support]
    end

    %% Upstream / Downstream relationships
    IDENTITY -->|UserRegistered, RoleAssigned| COACHING
    IDENTITY -->|UserRegistered| NOTIFICATIONS
    IDENTITY -->|UserRegistered| ANALYTICS

    COACHING -->|CoachProfileUpdated, ServiceCreated| SCHEDULING
    COACHING -->|CoachProfileUpdated| SEARCH
    COACHING -->|CoachProfileUpdated| ANALYTICS

    SCHEDULING -->|BookingCreated, BookingCancelled| PAYMENTS
    SCHEDULING -->|BookingCreated| VIDEO
    SCHEDULING -->|BookingCreated, BookingCancelled| NOTIFICATIONS
    SCHEDULING -->|BookingCreated, BookingCancelled| ANALYTICS

    PAYMENTS -->|PaymentConfirmed, PaymentFailed, RefundIssued| SCHEDULING
    PAYMENTS -->|PaymentConfirmed, PaymentFailed| NOTIFICATIONS
    PAYMENTS -->|PaymentConfirmed, RefundIssued| ANALYTICS

    PROGRAMS -->|ProgramCreated, ClientEnrolled, MilestoneCompleted| NOTIFICATIONS
    PROGRAMS -->|ProgramCreated, ClientEnrolled| SEARCH
    PROGRAMS -->|ProgramCreated, ClientEnrolled, MilestoneCompleted| ANALYTICS

    COMMUNICATION -->|MessageSent, FeedbackReceived| NOTIFICATIONS
    COMMUNICATION -->|FeedbackReceived| ANALYTICS

    CONTENT -->|ContentPublished| SEARCH
    CONTENT -->|ContentPublished| NOTIFICATIONS
    CONTENT -->|ContentPublished| ANALYTICS

    VIDEO -->|SessionStarted, SessionEnded| NOTIFICATIONS
    VIDEO -->|SessionStarted, SessionEnded| ANALYTICS

    SUPPORT -->|TicketCreated| NOTIFICATIONS
    SUPPORT -->|TicketCreated| ANALYTICS
```

---

#
