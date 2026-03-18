# Cache (Redis)

### What It Does

Redis is an in-memory data store that provides sub-millisecond read and write latency. It is far more than a simple key-value cache -- it is a versatile infrastructure component used for multiple purposes in the platform.

Core use cases:

- **Session storage** -- Stores user sessions (JWT metadata, user preferences, active role) so that any application instance can validate and retrieve session data without hitting the database.
- **Availability cache** -- Caches coach availability calendars. When a client opens the booking page, the system reads cached availability instead of computing it from the database on every request. A typical TTL is 30-60 seconds.
- **Rate limiting** -- Implements sliding window or token bucket counters. The API Gateway or application middleware increments a Redis counter per client/IP and rejects requests that exceed the limit.
- **Real-time features** -- Redis Pub/Sub or Streams power real-time notifications (new booking alert for a coach), live session updates, and chat features.
- **Distributed locks** -- Prevents double-booking by acquiring a lock on a time slot before confirming a reservation (using `SET key value NX PX 5000`).
- **Leaderboard / ranking** -- Sorted sets can rank clients by workout completion, enabling gamification features.

### Cache-Aside Pattern

The most common caching pattern in this platform:

1. Application receives a request (e.g., "get coach availability for Coach #42").
2. Application checks Redis: `GET coach:42:availability`.
3. **Cache hit** -- Return the cached data immediately.
4. **Cache miss** -- Query PostgreSQL, store the result in Redis with a TTL (`SET coach:42:availability <data> EX 60`), and return it.

This pattern keeps the application in control of what gets cached and when.

### TTL Strategies

| Data Type                | Suggested TTL   | Reasoning                                          |
|--------------------------|-----------------|-----------------------------------------------------|
| Coach availability       | 30-60 seconds   | Changes frequently with bookings                     |
| Workout plan details     | 5-10 minutes    | Changes infrequently after creation                  |
| User profile             | 2-5 minutes     | Moderate change frequency                            |
| Rate limit counters      | 1 minute window | Sliding window for API rate limiting                 |
| Session data             | 24 hours        | Matches session expiration policy                    |
| Static config (pricing)  | 1 hour          | Changes rarely, safe to cache longer                 |

### Cache Invalidation

Invalidation is the hardest problem in caching. Strategies used in the platform:

- **TTL-based expiration** -- The simplest approach. Data expires automatically. Acceptable for data where slight staleness is tolerable.
- **Event-driven invalidation** -- When a coach updates their availability, the Scheduling module publishes an event. The cache subscriber deletes the relevant key (`DEL coach:42:availability`). The next request triggers a cache miss and refills from the database.
- **Write-through** -- On every write to the database, also update the cache. Ensures consistency but adds write latency.

### Why It Is Used

PostgreSQL can handle significant read traffic, but repeated queries for the same data waste CPU and I/O. When 200 clients check Coach #42's availability within 30 seconds, Redis serves 199 of those from memory in under 1ms. Without Redis, those 200 queries each take 5-20ms against PostgreSQL and compete for database connections.

### Example Technologies

| Technology         | Notes                                               |
|--------------------|-----------------------------------------------------|
| **Redis**          | The standard. Single-threaded, predictable latency.  |
| **Redis Sentinel** | High availability with automatic failover.           |
| **Redis Cluster**  | Horizontal scaling via sharding across nodes.        |
| **KeyDB**          | Multi-threaded Redis fork, drop-in compatible.       |
| **DragonflyDB**    | Modern Redis alternative, multi-threaded.            |
| **Valkey**         | Open-source Redis fork by the Linux Foundation.      |

### Trade-offs

| Advantage                              | Disadvantage                                           |
|----------------------------------------|--------------------------------------------------------|
| Sub-millisecond latency                 | Data loss on crash (unless persistence is configured)   |
| Reduces database load dramatically      | Cache invalidation bugs cause stale data                |
| Versatile (cache, queue, pub/sub, lock) | Adds operational complexity (another system to monitor) |
| Well-understood, massive ecosystem      | Memory is more expensive than disk                      |
| Built-in data structures (sorted sets, streams) | Single-threaded model limits throughput per node  |

### How It Fits

**Approach A -- Modular Monolith:**
A single Redis instance (or Sentinel pair) serves the entire monolith. All modules share the same Redis connection pool but use key prefixes to avoid collisions (`scheduling:cache:*`, `payments:rate-limit:*`, `auth:session:*`). This is simple to operate and sufficient for most workloads.

**Approach B -- Microservices:**
Each service may have its own Redis instance or share a Redis Cluster. The Scheduling Service owns `scheduling:*` keys, the Auth Service owns `auth:*` keys. Shared state (like sessions) lives in a dedicated Redis instance that multiple services can read from. The main risk is services reading each other's cache keys -- clear ownership boundaries are essential.
