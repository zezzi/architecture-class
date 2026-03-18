# Payment Provider (Stripe)

### What It Does

The payment provider handles all financial transactions: charging clients, paying out coaches, managing subscriptions, and maintaining PCI compliance. Stripe is the most common choice for platforms because of its marketplace capabilities.

Core capabilities used by the platform:

- **Checkout Sessions** -- Pre-built, hosted payment pages. The client clicks "Book Session," the application creates a Checkout Session, and the client is redirected to Stripe's hosted page to enter card details. The application never touches raw card numbers, maintaining PCI compliance.
- **Payment Intents** -- A lower-level API for custom payment flows. The application creates a Payment Intent (e.g., $50 for a coaching session), and the client completes payment using Stripe Elements (embedded card form). Supports 3D Secure, Apple Pay, and Google Pay.
- **Webhooks** -- Stripe sends HTTP POST requests to your application when events occur: `payment_intent.succeeded`, `invoice.paid`, `customer.subscription.updated`, `charge.disputed`. Webhooks are the primary mechanism for updating your database after a payment -- never rely solely on the client-side redirect.
- **Subscriptions** -- Monthly coaching packages ($99/month for 4 sessions). Stripe manages recurring billing, proration, trial periods, and dunning (retrying failed payments).
- **Stripe Connect** -- The platform collects payment from clients and splits it: 85% to the coach, 15% platform fee. Stripe Connect handles the routing, tax reporting (1099s in the US), and compliance for multi-party payments.
- **Idempotency keys** -- Every Stripe API call includes an `Idempotency-Key` header. If a network failure causes a retry, Stripe returns the same response instead of creating a duplicate charge. This is critical for financial correctness.

### Webhook Flow (Booking Payment)

```
1. Client clicks "Book Session" --> App creates Stripe Checkout Session
2. Client enters card on Stripe's hosted page
3. Stripe processes payment
4. Stripe sends webhook: payment_intent.succeeded
5. App receives webhook, verifies Stripe signature
6. App marks booking as "confirmed" in database
7. App enqueues notification job (email confirmation to client and coach)
8. App enqueues calendar sync job
```

The webhook handler must be idempotent -- if Stripe sends the same event twice (which it does for reliability), processing it twice must not create duplicate bookings or charge the client twice.

### Why It Is Used

Handling payments yourself means PCI DSS compliance (an expensive, ongoing audit process), fraud detection, dispute handling, international payment methods, currency conversion, and regulatory compliance. Stripe absorbs all of this for a ~2.9% + $0.30 per transaction fee. For a platform at any scale, this is a fraction of the cost of building and maintaining payment infrastructure.

### Trade-offs

| Advantage                                  | Disadvantage                                       |
|--------------------------------------------|----------------------------------------------------|
| PCI compliance handled by Stripe            | Transaction fees reduce margins                     |
| Stripe Connect simplifies marketplace payments | Vendor lock-in (payment data lives in Stripe)     |
| Extensive fraud detection (Radar)           | Complex webhook handling requires careful engineering |
| Supports 135+ currencies                    | Stripe's fee structure is non-negotiable at small scale |
| Idempotency keys prevent double charges     | Payouts to coaches have a 2-7 day delay             |
| Excellent documentation and developer experience | Disputes (chargebacks) require manual handling   |

### How It Fits

**Approach A -- Modular Monolith:**
The Payments module encapsulates all Stripe interactions. It exposes internal functions like `createCheckoutSession(bookingId)`, `handleWebhook(event)`, and `createSubscription(clientId, planId)`. The webhook endpoint lives in the monolith's HTTP router. The Payments module updates its own schema and publishes domain events (`payment.succeeded`) for other modules to react to.

Stripe Connect is configured at the platform level. When a coach onboards, the monolith's Coach module triggers Stripe Connect account creation via the Payments module.

**Approach B -- Microservices:**
The Payment Service is an independent service that owns all Stripe interactions. Other services interact with it via API calls (`POST /payments/checkout-sessions`) or by consuming its events (`payment.succeeded` on Kafka). The Payment Service is the only service with Stripe API keys, reducing the blast radius of a key leak.

Webhook handling is critical: the Payment Service receives webhooks from Stripe and translates them into internal domain events. The Scheduling Service listens for `payment.succeeded` to confirm bookings. The Notification Service listens for the same event to send receipts.
