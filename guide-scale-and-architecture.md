# Guide: How to Think About Scale and Choose an Architecture

> This guide helps you reason about scale and make architecture decisions based on real constraints — not hype.

---

## Part 1: Understanding Scale

Scale is not just "how many users." It has multiple dimensions, and each one puts pressure on different parts of your system.

### The Five Dimensions of Scale

| Dimension | What It Means | Example Pressure |
|-----------|--------------|-----------------|
| **Request volume** | How many requests per second your system handles | 10 req/s is trivial; 10,000 req/s requires careful design |
| **Data volume** | How much data you store and query | 1 GB fits in memory; 1 TB needs indexing strategy; 1 PB needs partitioning |
| **User concurrency** | How many users are active simultaneously | 100 concurrent users vs. 100,000 changes everything about session management and state |
| **Team size** | How many developers work on the system | 3 developers vs. 30 vs. 300 changes how you organize code and deploy |
| **Feature complexity** | How many distinct business domains the system covers | A todo app vs. an e-commerce platform with payments, inventory, shipping, fraud detection |

### How to Estimate Scale for Your Project

Don't guess — estimate with napkin math:

1. **Start with users.** How many total users? How many are active daily?
2. **Estimate peak concurrency.** Typically 5-10% of daily active users are online at the same time.
3. **Estimate requests per user.** A typical web session generates 10-50 API calls.
4. **Multiply.** `peak_concurrent_users × requests_per_session / avg_session_duration_seconds = requests_per_second`

**Example:**
- 50,000 daily active users
- 5,000 concurrent at peak (10%)
- 20 API calls per session, session lasts ~10 minutes (600 seconds)
- 5,000 × 20 / 600 ≈ **~167 requests/second**

This is well within what a single well-optimized server can handle. You probably don't need microservices for throughput.

### The Scale Reality Check

Most projects overestimate their scale needs. Here's what a single modern server can handle:

| Component | Rough Capacity |
|-----------|---------------|
| Node.js / Spring Boot / Django | 1,000 - 10,000 req/s (depending on complexity) |
| PostgreSQL | 10,000+ simple queries/s on modest hardware |
| Redis | 100,000+ operations/s |
| A single VM (4 CPU, 16 GB RAM) | More than most startups will ever need |

**Rule of thumb:** If your napkin math says fewer than 1,000 req/s, a monolith with a single database will almost certainly work. You don't need microservices for performance — you might need them for other reasons.

---

## Part 2: Choosing an Architecture

Architecture is not about picking the "best" pattern. It's about picking the pattern whose trade-offs align with your constraints.

### The Three Most Common Approaches

#### 1. Monolith

```
[ Single Deployable Application ]
        |
  [ Single Database ]
```

| When to Choose | When to Avoid |
|---------------|---------------|
| Small team (1-10 developers) | Large team where groups step on each other |
| Rapid prototyping / MVP phase | Need independent deployment of components |
| Simple operational requirements | Components have wildly different scaling needs |
| Strong data consistency needs | Regulatory requirement to isolate certain data |

**Key insight:** A monolith is not "bad architecture." It is the right starting point for most projects. Instagram served 400 million users on a monolith.

#### 2. Modular Monolith

```
[ Application ]
  ├── Module A (own schema/tables)
  ├── Module B (own schema/tables)
  └── Module C (own schema/tables)
        |
  [ Single Database (separate schemas) ]
```

| When to Choose | When to Avoid |
|---------------|---------------|
| Medium team (5-20 developers) | You need modules to scale independently right now |
| You want clear boundaries but simple deployment | Team is too small to maintain module discipline |
| Planning for a future split into services | Modules need different technology stacks |
| Data consistency across modules is important | |

**Key insight:** This gives you most of the organizational benefits of microservices without the operational cost. Modules communicate through defined interfaces, not random function calls. You can extract a module into its own service later if needed.

#### 3. Microservices

```
[ Service A ] ←→ [ Message Broker ] ←→ [ Service B ]
     |                                       |
 [ DB A ]                                [ DB B ]
```

| When to Choose | When to Avoid |
|---------------|---------------|
| Large team (20+ developers across multiple squads) | Small team — the overhead will slow you down |
| Components need independent scaling (e.g., search vs. payments) | You don't have DevOps/platform capacity |
| Independent deployability is critical | Data consistency across services is your top priority |
| Different services need different tech stacks | You're building an MVP or prototype |
| Regulatory/compliance requires isolation | |

**Key insight:** Microservices solve organizational problems, not technical ones. They let large teams deploy independently. If you have 5 developers, microservices add complexity without benefit.

---

## Part 3: The Decision Framework

Use this flowchart to guide your decision:

### Step 1 — How big is your team?

| Team Size | Default Starting Point |
|-----------|----------------------|
| 1-5 developers | Monolith |
| 5-15 developers | Modular monolith |
| 15+ developers, multiple squads | Consider microservices |

### Step 2 — Do you have a forcing function?

A "forcing function" is a constraint that overrides the team-size default:

| Forcing Function | What It Pushes You Toward |
|-----------------|--------------------------|
| Regulatory isolation (e.g., PCI for payments) | Extract that one component as a service |
| Extreme scale difference between components (e.g., real-time chat vs. billing) | Separate the hot path |
| Multi-language requirement (ML team uses Python, backend uses Go) | Service boundary at the language boundary |
| Independent release cadence needed | Service per release boundary |
| You already have a working monolith that's hitting walls | Incremental extraction, not a rewrite |

If none of these apply, stick with the team-size default.

### Step 3 — What trade-offs are you accepting?

Every architecture choice has a cost. Be explicit about what you're giving up:

| Choice | You Gain | You Pay |
|--------|---------|---------|
| Monolith | Simplicity, fast development, easy debugging, strong consistency | Harder to scale individual components; large codebase can become tangled without discipline |
| Modular monolith | Clear boundaries, single deployment, easier refactoring | Requires discipline to maintain module boundaries; still a single deployable |
| Microservices | Independent scaling, independent deployment, fault isolation | Network complexity, distributed transactions, eventual consistency, operational overhead, harder debugging |

### Step 4 — Write it down

Whatever you choose, document:
1. **What** you chose
2. **Why** it fits your project's specific constraints
3. **What you're giving up** and why that's acceptable
4. **When you would reconsider** — what signals would trigger a different choice?

This becomes your ADR.

---

## Part 4: Scaling Strategies (Within Any Architecture)

Regardless of whether you pick a monolith or microservices, these strategies help you handle growth:

### Vertical Scaling (Scale Up)
Add more CPU/RAM to your existing server.

- **When:** Your bottleneck is raw compute or memory
- **Limit:** There's a ceiling to how big one machine can get
- **Cost:** Simple but expensive at the top end

### Horizontal Scaling (Scale Out)
Run multiple instances of your application behind a load balancer.

- **When:** Your application is stateless (or can be made stateless)
- **Requirement:** Session state must live outside the application (in Redis, database, or JWT)
- **Works for:** Both monoliths and microservices

### Database Scaling Strategies

| Strategy | What It Does | When to Use |
|----------|-------------|-------------|
| **Read replicas** | Copies of your database that serve read queries | Read-heavy workloads (90%+ reads) |
| **Connection pooling** | Reuses database connections instead of opening new ones | Always — this is free performance |
| **Caching** | Store frequent query results in Redis/Memcached | Hot data that doesn't change often (product catalog, user profiles) |
| **Partitioning / Sharding** | Split data across multiple databases | Very large datasets (100M+ rows) or multi-region requirements |
| **CQRS** | Separate read and write models | Complex queries that don't match your write schema |

### Async Processing
Move slow work out of the request path:

- **What:** User clicks "submit" → your API returns immediately → a background worker processes the task
- **When:** Email sending, report generation, image processing, payment settlement, analytics
- **How:** Use a message queue (RabbitMQ, SQS, Kafka) between your API and worker processes

### Caching Layers

```
User → CDN (static assets) → API Gateway → App Cache (Redis) → Database
```

| Layer | What to Cache | TTL |
|-------|--------------|-----|
| **CDN** | Static files, images, CSS/JS | Hours to days |
| **API Gateway** | Common API responses | Seconds to minutes |
| **Application (Redis)** | Session data, computed results, frequently read records | Minutes to hours |
| **Database** | Query result cache (built-in) | Automatic |

---

## Part 5: Common Mistakes to Avoid

### 1. "We might need to scale, so let's use microservices"
You don't architect for imaginary scale. Start simple, measure, then optimize. Premature distribution is the root of all evil in system design.

### 2. Confusing high availability with microservices
A monolith behind a load balancer with health checks, running on 3 instances, is highly available. You don't need microservices for uptime.

### 3. Choosing technology before understanding the problem
"Let's use Kafka" is not an architecture decision. "We have 50,000 events/second from IoT devices that need to be processed in order per device" is the problem — Kafka might be the answer.

### 4. Ignoring operational cost
Every service you add needs monitoring, logging, deployment pipelines, error handling, and an on-call rotation. If you have 20 microservices and 3 developers, you'll spend more time on infrastructure than on features.

### 5. Treating the architecture as permanent
Good architecture evolves. Start with a monolith, extract modules, graduate to services only where needed. The best systems grow incrementally, not by upfront grand design.

---

## Quick Reference: Architecture Decision Cheat Sheet

```
Is your team < 10 people?
  └─ YES → Start with a monolith or modular monolith
       └─ Do you need regulatory isolation for a component?
            └─ YES → Extract only that component as a service
            └─ NO  → Stay with the monolith until you hit real pain
  └─ NO  → Consider microservices
       └─ Can teams deploy independently today?
            └─ YES → Microservices make sense
            └─ NO  → Start with modular monolith, define boundaries first

Is your scale > 10,000 req/s?
  └─ YES → Profile first. Is it CPU? Memory? Database? Network?
       └─ CPU/Memory → Scale horizontally (more instances)
       └─ Database   → Read replicas, caching, then consider CQRS
       └─ Network    → CDN, compression, connection pooling
  └─ NO  → Scale is not your problem. Focus on correctness and velocity.
```

---

## Applying This to Your Project

When writing your proposals and ADRs:

1. **Do the napkin math.** Estimate your scale numbers and include them in your proposal.
2. **Name your forcing functions.** If you chose microservices, explain what forces you toward it beyond "it's modern."
3. **Be honest about trade-offs.** The graders want to see that you understand what you give up, not just what you gain.
4. **Match your infrastructure to your architecture.** If you chose a monolith, you probably don't need Kafka. If you chose microservices, explain how you handle distributed transactions.
5. **Show evolution.** Describe how your system could grow from the starting architecture to the next stage, and what triggers would drive that change.
