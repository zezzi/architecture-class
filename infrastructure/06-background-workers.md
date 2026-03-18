# Background Workers

### What It Does

Background workers are processes that execute tasks asynchronously, outside the request-response cycle. When a user action triggers work that does not need to happen immediately (sending an email, generating a PDF, processing a video), the application enqueues a job and a worker picks it up later.

Core concepts:

- **Job** -- A unit of work with a payload (e.g., `{ type: "send_email", to: "client@example.com", template: "booking_confirmed", data: {...} }`).
- **Queue** -- A named list of pending jobs (e.g., `email_queue`, `video_processing_queue`, `report_queue`).
- **Worker** -- A process that polls a queue, picks up jobs, and executes them.
- **Retry logic** -- When a job fails, it is retried with exponential backoff (e.g., after 1s, 4s, 16s, 64s).
- **Dead letter queue (DLQ)** -- After exhausting retries, failed jobs land in a DLQ for manual inspection.
- **Concurrency** -- Workers can process multiple jobs in parallel (configurable per queue based on resource requirements).

### Platform Use Cases

| Task                       | Queue                  | Priority  | Typical Duration |
|----------------------------|------------------------|-----------|------------------|
| Send booking confirmation email | `email`            | High      | 1-3 seconds      |
| Send payment receipt       | `email`                | High      | 1-3 seconds      |
| Generate monthly progress PDF  | `reports`          | Low       | 10-30 seconds    |
| Transcode workout video    | `video_processing`     | Medium    | 2-10 minutes     |
| Sync booking to Google Calendar | `calendar_sync`   | Medium    | 2-5 seconds      |
| Retry failed payment       | `payment_retry`        | High      | 3-10 seconds     |
| Update search index        | `indexing`             | Low       | 1-5 seconds      |
| Compute coach analytics    | `analytics`            | Low       | 15-60 seconds    |
| Clean up expired sessions  | `maintenance`          | Low       | 5-15 seconds     |

### Why It Is Used

If a user books a session and the server must synchronously send a confirmation email, sync the calendar, update analytics, and notify the coach, the API response takes 5-10 seconds instead of 200ms. Background workers move all non-critical work out of the request path, keeping API responses fast and the user experience smooth.

Additionally, some tasks are inherently long-running (video transcoding) or unreliable (third-party API calls). Workers handle retries and failure isolation so that a SendGrid outage does not prevent users from booking sessions.

### Example Technologies

| Technology     | Language   | Backed By           | Notes                                   |
|----------------|------------|---------------------|-----------------------------------------|
| **BullMQ**     | Node.js    | Redis               | Feature-rich, good UI (Bull Board), TypeScript |
| **Celery**     | Python     | Redis or RabbitMQ   | Mature, widely used, periodic tasks (beat) |
| **Sidekiq**    | Ruby       | Redis               | High performance, battle-tested          |
| **Temporal**   | Polyglot   | Own server           | Workflow orchestration, durable execution |
| **AWS SQS + Lambda** | Any  | Managed             | Serverless, auto-scaling                 |

### Trade-offs

| Advantage                                | Disadvantage                                        |
|------------------------------------------|-----------------------------------------------------|
| Keeps API responses fast                  | Adds infrastructure (worker processes, job queues)   |
| Isolates failures from the user           | Jobs can be lost if the queue crashes without persistence |
| Enables retry with backoff                | Monitoring failed jobs requires dedicated tooling    |
| Scales independently from the web server  | Debugging async flows is harder than sync            |
| Handles long-running tasks gracefully     | Priority inversion (low-priority jobs can starve)    |

### How It Fits

**Approach A -- Modular Monolith:**
Workers are separate processes running the same codebase as the monolith but initialized in "worker mode." They share the same database connection, models, and business logic. BullMQ with Redis is a common choice. The monolith enqueues jobs; workers dequeue and process them.

```
Monolith (web) --enqueue--> Redis Queue --dequeue--> Monolith (worker process)
```

The worker process imports the same modules (Notifications, Payments, Analytics) and calls them directly. No network calls between modules.

**Approach B -- Microservices:**
Each service runs its own workers. The Payment Service has dedicated workers for payment retries. The Notification Service has workers for email, SMS, and push delivery. The Content Service has workers for video transcoding. Each service's workers consume from their own queues and scale independently. The Video Processing workers might run on GPU-equipped instances, while Email workers run on cheap, small instances.
