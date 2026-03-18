# Diagram 2: DDD Bounded Context Map

This diagram shows all 12 bounded contexts and the strategic DDD relationships between them. Arrows indicate upstream-to-downstream dependency direction. Labels on the edges describe the integration pattern used: ACL (Anti-Corruption Layer), CF (Conformist), SK (Shared Kernel), OHS (Open Host Service), or PL (Published Language).

```mermaid
graph LR
    subgraph CoreDomain["Core Domain"]
        Coaching["Coaching<br/>(Core)<br/>---<br/>Coach profiles,<br/>client relationships,<br/>goals, assessments"]
        Programs["Programs<br/>(Core)<br/>---<br/>Workout plans,<br/>nutrition plans,<br/>progress tracking"]
    end

    subgraph SupportingDomain["Supporting Domain"]
        Scheduling["Scheduling<br/>(Supporting)<br/>---<br/>Bookings, availability,<br/>calendar management"]
        Communication["Communication<br/>(Supporting)<br/>---<br/>Chat, messaging,<br/>file sharing"]
        Content["Content<br/>(Supporting)<br/>---<br/>Exercise library,<br/>articles, media"]
        VideoSessions["Video Sessions<br/>(Supporting)<br/>---<br/>Live sessions,<br/>recordings"]
        Support["Support<br/>(Supporting)<br/>---<br/>Tickets, FAQ,<br/>dispute resolution"]
    end

    subgraph GenericDomain["Generic Domain"]
        Identity["Identity & Access<br/>(Generic)<br/>---<br/>Users, roles,<br/>permissions, auth"]
        Payments["Payments<br/>(Generic)<br/>---<br/>Subscriptions, invoices,<br/>refunds, payouts"]
        Notifications["Notifications<br/>(Generic)<br/>---<br/>Email, SMS, push,<br/>in-app alerts"]
        Analytics["Analytics<br/>(Generic)<br/>---<br/>Events, metrics,<br/>reports, dashboards"]
        Search["Search<br/>(Generic)<br/>---<br/>Full-text search,<br/>filters, ranking"]
    end

    Identity -->|"OHS/PL<br/>User identity shared<br/>via JWT claims"| Coaching
    Identity -->|"OHS/PL"| Scheduling
    Identity -->|"OHS/PL"| Payments
    Identity -->|"OHS/PL"| Communication
    Identity -->|"OHS/PL"| Programs
    Identity -->|"OHS/PL"| Support

    Coaching -->|"SK<br/>Shared Kernel:<br/>CoachId, ClientId"| Programs
    Coaching -->|"OHS/PL<br/>Coach availability<br/>published"| Scheduling
    Coaching -->|"OHS/PL<br/>Coach/Client pairs<br/>published"| Communication
    Coaching -->|"OHS/PL"| VideoSessions

    Programs -->|"OHS/PL<br/>Exercise references"| Content
    Programs -->|"CF<br/>Conformist:<br/>adopts program model"| Analytics

    Scheduling -->|"ACL<br/>Anti-Corruption Layer:<br/>translates booking<br/>to payment request"| Payments
    Scheduling -->|"OHS/PL<br/>Booking events<br/>published"| Notifications
    Scheduling -->|"OHS/PL"| VideoSessions
    Scheduling -->|"CF"| Analytics

    Payments -->|"ACL<br/>Wraps Stripe API"| Notifications
    Payments -->|"CF"| Analytics

    Communication -->|"OHS/PL<br/>Message events"| Notifications
    Communication -->|"CF"| Analytics

    VideoSessions -->|"ACL<br/>Wraps Zoom API"| Notifications
    VideoSessions -->|"CF"| Analytics

    Content -->|"OHS/PL<br/>Content indexed"| Search
    Coaching -->|"OHS/PL<br/>Coach profiles indexed"| Search
    Programs -->|"OHS/PL<br/>Programs indexed"| Search

    Support -->|"ACL"| Payments
    Support -->|"CF"| Analytics
    Support -->|"OHS/PL"| Notifications

    Notifications -->|"CF"| Analytics

    style CoreDomain fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style SupportingDomain fill:#bbdefb,stroke:#1565c0,stroke-width:2px
    style GenericDomain fill:#ffe0b2,stroke:#e65100,stroke-width:2px
```

**Relationship Legend:**

| Pattern | Abbreviation | Description |
|---|---|---|
| Open Host Service / Published Language | OHS/PL | Upstream exposes a well-defined API and published events |
| Anti-Corruption Layer | ACL | Downstream translates upstream models to protect its own domain |
| Conformist | CF | Downstream adopts the upstream model as-is |
| Shared Kernel | SK | Two contexts share a small, co-owned model (e.g., CoachId, ClientId) |
