# Notification System

### What It Does

The notification system delivers messages to users through multiple channels: email, SMS, and push notifications. It is the platform's outbound communication layer, keeping coaches and clients informed about bookings, payments, workouts, and platform updates.

Core responsibilities:

- **Multi-channel delivery** -- The same logical notification ("Your session is confirmed") may be delivered via email (detailed), push (brief alert), and optionally SMS (for users without the app).
- **Template management** -- Notifications use templates with variables: `"Hi {{coach_name}}, {{client_name}} booked a session on {{date}} at {{time}}"`. Templates are maintained per channel and per locale.
- **User preferences** -- Clients and coaches configure which notifications they want and on which channels. A coach might want push notifications for new bookings but only email for payment summaries.
- **Delivery tracking** -- Track whether emails were delivered, opened, bounced, or marked as spam. Track push notification delivery and engagement.
- **Throttling** -- Prevent notification fatigue. If a coach receives 20 bookings in one minute, batch them into a single summary notification instead of 20 individual pings.
- **Scheduling** -- Some notifications are time-sensitive ("Your session starts in 1 hour") and must be sent at a specific time, not immediately.

### Platform Notification Types

| Notification                  | Email | Push | SMS  | Trigger                          |
|-------------------------------|-------|------|------|----------------------------------|
| Booking confirmation          | Yes   | Yes  | No   | `booking.confirmed`              |
| Booking reminder (1 hour)     | No    | Yes  | Yes  | Scheduled job                    |
| Booking cancellation          | Yes   | Yes  | No   | `booking.cancelled`              |
| Payment receipt               | Yes   | No   | No   | `payment.succeeded`              |
| Payment failed                | Yes   | Yes  | Yes  | `payment.failed`                 |
| New client registered (coach) | Yes   | Yes  | No   | `client.registered`              |
| Workout plan assigned         | Yes   | Yes  | No   | `workout_plan.assigned`          |
| Weekly progress summary       | Yes   | No   | No   | Scheduled (weekly cron)          |
| Coach payout processed        | Yes   | No   | No   | `payout.completed`               |
| Platform announcements        | Yes   | Yes  | No   | Admin trigger                    |

### Why It Is Used

Notifications drive engagement and retention. A client who receives a timely session reminder is less likely to no-show. A coach who gets instant new-booking alerts can respond quickly. Payment failure notifications prompt clients to update their card before their subscription lapses.

Without a structured notification system, notification logic scatters across the codebase, templates become inconsistent, and users get spammed with redundant messages.

### Example Technologies

| Technology            | Channel   | Notes                                          |
|-----------------------|-----------|------------------------------------------------|
| **SendGrid**          | Email     | High deliverability, template engine, analytics |
| **Amazon SES**        | Email     | Low cost, AWS-native, requires more setup       |
| **Postmark**          | Email     | Transactional email focus, excellent deliverability |
| **Twilio**            | SMS/Voice | Global coverage, programmable messaging          |
| **Firebase Cloud Messaging (FCM)** | Push | Android and iOS push, free, Google-operated |
| **Apple Push Notification Service (APNs)** | Push | iOS-specific, required for iOS apps     |
| **OneSignal**         | Push/Email/SMS | Multi-channel platform, free tier         |
| **Novu**              | All       | Open-source notification infrastructure          |

### Trade-offs

| Advantage                                 | Disadvantage                                          |
|-------------------------------------------|-------------------------------------------------------|
| Improves user engagement and retention     | External API costs (SendGrid, Twilio) scale with volume |
| Reduces no-shows with timely reminders     | Email deliverability requires ongoing monitoring        |
| Centralized template management            | Multi-channel coordination is complex                  |
| User preferences reduce complaint rates    | Push notification tokens must be managed per device     |
| Delivery tracking enables optimization     | SMS regulations vary by country (opt-in requirements)  |

### How It Fits

**Approach A -- Modular Monolith:**
The Notifications module subscribes to domain events from other modules (via RabbitMQ or in-memory event bus). When `booking.confirmed` is published, the Notifications module checks the user's preferences, selects the appropriate templates, and enqueues delivery jobs (one per channel) to background workers. The workers call SendGrid, FCM, or Twilio APIs.

```
Scheduling Module --event--> Notifications Module --job--> Email Worker --> SendGrid
                                                       --> Push Worker  --> FCM
```

**Approach B -- Microservices:**
The Notification Service is an independent service that consumes events from Kafka topics. It maintains its own database of user preferences, templates, and delivery logs. It runs its own workers for each delivery channel. Other services never call SendGrid or Twilio directly -- they publish events, and the Notification Service decides what to send, when, and how.

This separation means the Notification Service can be maintained by a dedicated team, template changes do not require deploying the Scheduling Service, and notification logic is completely centralized.
