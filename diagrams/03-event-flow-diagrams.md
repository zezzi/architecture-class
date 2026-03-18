# Diagram 3: Event Flow Diagram

This sequence diagram shows the three primary event flows through the system. Domain events are the backbone of decoupled communication, whether delivered via RabbitMQ queues (Approach A) or Kafka topics (Approach B).

### Flow 1: Booking and Payment

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client App
    participant APIGW as API Gateway
    participant Sched as Scheduling Context
    participant MQ as Message Broker<br/>(RabbitMQ / Kafka)
    participant Pay as Payments Context
    participant Stripe as Stripe API
    participant Notif as Notifications Context
    participant Email as SendGrid
    participant Push as Twilio
    participant Analytics as Analytics Context
    participant DW as Data Warehouse

    Client->>APIGW: POST /bookings (coachId, slot, type)
    APIGW->>Sched: Route to Scheduling
    Sched->>Sched: Validate availability & reserve slot
    Sched-->>MQ: Publish: BookingCreated{bookingId, clientId, coachId, amount}

    MQ-->>Pay: Consume: BookingCreated
    Pay->>Stripe: Create PaymentIntent
    Stripe-->>Pay: PaymentIntent confirmed
    Pay-->>MQ: Publish: PaymentConfirmed{bookingId, paymentId, amount}

    MQ-->>Sched: Consume: PaymentConfirmed
    Sched->>Sched: Confirm booking status

    MQ-->>Notif: Consume: PaymentConfirmed
    Notif->>Email: Send confirmation email to client
    Notif->>Email: Send booking alert to coach
    Notif->>Push: Send push notification to both

    MQ-->>Analytics: Consume: BookingCreated
    MQ-->>Analytics: Consume: PaymentConfirmed
    Analytics->>DW: Store event data

    Note over Sched,Analytics: If payment fails, Scheduling receives<br/>PaymentFailed and releases the slot.

    MQ-->>Sched: (alt) Consume: PaymentFailed
    Sched->>Sched: (alt) Release reserved slot
    Sched-->>MQ: (alt) Publish: BookingCancelled
```

### Flow 2: Program Enrollment and Progress

```mermaid
sequenceDiagram
    autonumber
    participant Coach as Coach Dashboard
    participant APIGW as API Gateway
    participant Prog as Programs Context
    participant MQ as Message Broker
    participant Coaching as Coaching Context
    participant Content as Content Context
    participant Notif as Notifications Context
    participant Analytics as Analytics Context
    participant Search as Search Context

    Coach->>APIGW: POST /programs (plan details)
    APIGW->>Prog: Create program
    Prog->>Content: Fetch exercise references
    Content-->>Prog: Exercise metadata
    Prog-->>MQ: Publish: ProgramCreated{programId, coachId, exercises}

    MQ-->>Search: Consume: ProgramCreated
    Search->>Search: Index program for discovery

    MQ-->>Analytics: Consume: ProgramCreated
    Analytics->>Analytics: Record program creation metric

    Note over Coach,Analytics: Client enrolls in the program

    Coach->>APIGW: POST /programs/{id}/enrollments (clientId)
    APIGW->>Prog: Enroll client
    Prog->>Coaching: Validate coach-client relationship
    Coaching-->>Prog: Relationship confirmed
    Prog-->>MQ: Publish: EnrollmentStarted{programId, clientId}

    MQ-->>Notif: Consume: EnrollmentStarted
    Notif->>Notif: Send welcome + program details to client

    Note over Coach,Analytics: Client logs workout progress

    Coach->>APIGW: PUT /programs/{id}/progress (workout log)
    APIGW->>Prog: Update progress
    Prog-->>MQ: Publish: ProgressUpdated{programId, clientId, completion%}

    MQ-->>Coaching: Consume: ProgressUpdated
    Coaching->>Coaching: Update client assessment data

    MQ-->>Notif: Consume: ProgressUpdated
    Notif->>Notif: Notify coach of client milestone

    MQ-->>Analytics: Consume: ProgressUpdated
    Analytics->>Analytics: Track engagement metrics
```

### Flow 3: Communication and Notifications

```mermaid
sequenceDiagram
    autonumber
    participant User as Client / Coach
    participant APIGW as API Gateway
    participant Comm as Communication Context
    participant MQ as Message Broker
    participant Notif as Notifications Context
    participant Email as SendGrid
    participant SMS as Twilio SMS
    participant PushSvc as Twilio Push
    participant Analytics as Analytics Context

    User->>APIGW: POST /messages (recipientId, body, attachments)
    APIGW->>Comm: Send message
    Comm->>Comm: Store message, update conversation thread
    Comm-->>MQ: Publish: MessageSent{messageId, senderId, recipientId, channel}

    MQ-->>Notif: Consume: MessageSent
    Notif->>Notif: Check recipient notification preferences

    alt Recipient is online
        Notif->>PushSvc: Send in-app push
    else Recipient is offline
        Notif->>PushSvc: Send mobile push notification
        Notif->>Email: Send email digest (batched)
    end

    alt Urgent / flagged message
        Notif->>SMS: Send SMS alert
    end

    Notif-->>MQ: Publish: NotificationDelivered{notifId, channel, recipientId}

    MQ-->>Analytics: Consume: MessageSent
    MQ-->>Analytics: Consume: NotificationDelivered
    Analytics->>Analytics: Track communication engagement

    Note over Comm,Analytics: All events feed into the analytics<br/>pipeline regardless of the delivery outcome.
```
