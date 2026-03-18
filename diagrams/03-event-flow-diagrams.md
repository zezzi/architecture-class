# Event Flow Diagrams

Domain events are the backbone of decoupled communication. Each flow below is split into focused diagrams so the interactions are easy to follow.

---

## Flow 1: Booking and Payment

### 1A. Client Creates Booking (Synchronous)

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client App
    participant GW as API Gateway
    participant Sched as Scheduling
    participant MQ as Message Broker

    Client->>GW: POST /bookings
    GW->>Sched: Route request
    Sched->>Sched: Validate availability
    Sched->>Sched: Reserve time slot
    Sched-->>MQ: Publish BookingCreated
    Sched-->>Client: 201 Created
```

### 1B. Payment Processing (Async)

```mermaid
sequenceDiagram
    autonumber
    participant MQ as Message Broker
    participant Pay as Payments
    participant Stripe as Stripe API
    participant Sched as Scheduling

    MQ-->>Pay: Consume BookingCreated
    Pay->>Stripe: Create PaymentIntent
    Stripe-->>Pay: PaymentIntent confirmed
    Pay-->>MQ: Publish PaymentConfirmed

    MQ-->>Sched: Consume PaymentConfirmed
    Sched->>Sched: Confirm booking status
```

### 1C. Notifications & Analytics React (Async)

```mermaid
sequenceDiagram
    autonumber
    participant MQ as Message Broker
    participant Notif as Notifications
    participant Email as SendGrid
    participant Push as Twilio
    participant Analytics as Analytics

    MQ-->>Notif: Consume PaymentConfirmed
    Notif->>Email: Confirmation to client
    Notif->>Email: Alert to coach
    Notif->>Push: Push to both

    MQ-->>Analytics: Consume BookingCreated
    MQ-->>Analytics: Consume PaymentConfirmed
    Analytics->>Analytics: Store event data
```

### 1D. Payment Failure Path

```mermaid
sequenceDiagram
    autonumber
    participant MQ as Message Broker
    participant Sched as Scheduling
    participant Notif as Notifications

    Note over MQ: PaymentFailed event
    MQ-->>Sched: Consume PaymentFailed
    Sched->>Sched: Release reserved slot
    Sched-->>MQ: Publish BookingCancelled

    MQ-->>Notif: Consume PaymentFailed
    Notif->>Notif: Send "payment failed" email
```

---

## Flow 2: Program Enrollment and Progress

### 2A. Coach Creates Program

```mermaid
sequenceDiagram
    autonumber
    participant Coach as Coach Dashboard
    participant GW as API Gateway
    participant Prog as Programs
    participant Content as Content
    participant MQ as Message Broker

    Coach->>GW: POST /programs
    GW->>Prog: Create program
    Prog->>Content: Fetch exercise refs
    Content-->>Prog: Exercise metadata
    Prog-->>MQ: Publish ProgramCreated
```

### 2B. Client Enrolls

```mermaid
sequenceDiagram
    autonumber
    participant Coach as Coach Dashboard
    participant GW as API Gateway
    participant Prog as Programs
    participant Coaching as Coaching
    participant MQ as Message Broker

    Coach->>GW: POST /programs/{id}/enrollments
    GW->>Prog: Enroll client
    Prog->>Coaching: Validate relationship
    Coaching-->>Prog: Confirmed
    Prog-->>MQ: Publish EnrollmentStarted
```

### 2C. Progress Tracking (Async Reactions)

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client App
    participant GW as API Gateway
    participant Prog as Programs
    participant MQ as Message Broker
    participant Coaching as Coaching
    participant Notif as Notifications
    participant Analytics as Analytics

    Client->>GW: PUT /programs/{id}/progress
    GW->>Prog: Update progress
    Prog-->>MQ: Publish ProgressUpdated

    MQ-->>Coaching: Update assessment data
    MQ-->>Notif: Notify coach of milestone
    MQ-->>Analytics: Track engagement
```

---

## Flow 3: Communication and Notifications

### 3A. Sending a Message

```mermaid
sequenceDiagram
    autonumber
    participant User as Client / Coach
    participant GW as API Gateway
    participant Comm as Communication
    participant MQ as Message Broker

    User->>GW: POST /messages
    GW->>Comm: Send message
    Comm->>Comm: Store in thread
    Comm-->>MQ: Publish MessageSent
```

### 3B. Notification Delivery

```mermaid
sequenceDiagram
    autonumber
    participant MQ as Message Broker
    participant Notif as Notifications
    participant Push as Push Service
    participant Email as Email Service
    participant SMS as SMS Service

    MQ-->>Notif: Consume MessageSent
    Notif->>Notif: Check preferences

    alt Recipient is online
        Notif->>Push: In-app push
    else Recipient is offline
        Notif->>Push: Mobile push
        Notif->>Email: Email digest
    end

    alt Urgent message
        Notif->>SMS: SMS alert
    end

    Notif-->>MQ: Publish NotificationDelivered
```
