# Database (PostgreSQL)

### What It Does

PostgreSQL is the primary relational database that stores the platform's persistent state: user accounts, coach profiles, workout plans, bookings, payments, subscriptions, and more. It is the system of record -- the authoritative source of truth for all business data.

Core capabilities leveraged by the platform:

- **ACID transactions** -- Atomicity, Consistency, Isolation, Durability. When a booking is created, the time slot is marked as taken, and the payment hold is recorded, all in a single transaction. If any step fails, everything rolls back. No partial state.
- **JSONB columns** -- Store semi-structured data (workout plan details, exercise configurations, user preferences) as queryable JSON. This avoids schema rigidity for data that varies between coaches.
- **Full-text search** -- PostgreSQL's `tsvector` and `tsquery` enable searching workout plans, exercise libraries, and coach bios without an external search engine (sufficient for moderate scale).
- **Advanced indexing** -- B-tree for equality/range queries, GIN for JSONB and full-text search, GiST for geolocation (finding coaches near a client), partial indexes for scoped queries.
- **Row-level security (RLS)** -- Enforce data isolation at the database level. A coach can only query their own clients' data, enforced by PostgreSQL policies, not just application code.
- **Triggers and functions** -- Maintain derived data (e.g., auto-update a `booking_count` field when a booking is inserted).

### Read Replicas

As the platform grows, read traffic (dashboard queries, reporting, search) can overwhelm a single database. Read replicas are asynchronous copies of the primary database that handle read-only queries.

```
Write queries --> Primary (read-write)
Read queries  --> Replica 1 (read-only)
              --> Replica 2 (read-only)
```

Typical replication lag is 10-100ms. For the Gym Coaching Platform, this means a coach might not see a new booking for a fraction of a second after it is created -- acceptable for dashboards but not for real-time availability checks (which should read from the primary or from Redis cache).

### PgBouncer (Connection Pooling)

PostgreSQL creates a new process for each connection, consuming ~10MB of RAM per connection. With 8 monolith instances (or 20 microservice instances) each maintaining 20 connections, you reach 160-400 connections, which strains the database. PgBouncer sits between the application and PostgreSQL, pooling connections:

```
App Instance 1 (20 connections) --> PgBouncer (maintains 50 connections to PG) --> PostgreSQL
App Instance 2 (20 connections) -->
App Instance 3 (20 connections) -->
```

PgBouncer multiplexes hundreds of application connections onto a smaller pool of real PostgreSQL connections.

### Migrations

Database schema changes (adding a column, creating an index, modifying a constraint) are managed through versioned migration files. Each migration is a numbered script that runs exactly once, in order.

```
migrations/
  001_create_users.sql
  002_create_coaches.sql
  003_create_bookings.sql
  004_add_cancellation_policy_to_coaches.sql
  005_add_index_on_bookings_date.sql
```

Tools like **Flyway**, **node-pg-migrate**, **Prisma Migrate**, or **Alembic** track which migrations have been applied and run pending ones on deployment.

### Why It Is Used

PostgreSQL is the default choice for applications that need relational data integrity, complex queries, and transactional guarantees. The platform has inherently relational data: a booking references a coach, a client, a time slot, and a payment. Enforcing these relationships with foreign keys prevents orphaned data and maintains consistency.

### Trade-offs

| Advantage                                | Disadvantage                                        |
|------------------------------------------|-----------------------------------------------------|
| ACID guarantees prevent data corruption   | Vertical scaling has limits (eventually need sharding) |
| Rich query language (SQL)                 | Schema changes on large tables can lock and block    |
| JSONB gives NoSQL flexibility within SQL  | Connection limits require pooling (PgBouncer)        |
| Mature ecosystem, excellent documentation | Full-text search is good but not Elasticsearch-level |
| Read replicas scale read traffic          | Replication lag makes replicas eventually consistent  |
| Row-level security adds defense in depth  | Requires careful index tuning as data grows          |

### How It Fits

**Approach A -- Modular Monolith (Schema-per-Module):**
A single PostgreSQL instance with multiple schemas. Each module owns a schema:

```
database: gym_platform
  schema: scheduling   --> bookings, time_slots, availability
  schema: payments     --> transactions, subscriptions, invoices
  schema: coaching     --> coaches, clients, workout_plans
  schema: auth         --> users, roles, permissions
  schema: content      --> exercises, videos, programs
```

Modules access their own schema directly. Cross-module data access goes through well-defined internal APIs (function calls within the monolith), not direct cross-schema queries. This enforces module boundaries while allowing a single database transaction to span modules when necessary (e.g., creating a booking and a payment hold atomically).

**Approach B -- Microservices (Database-per-Service):**
Each service owns its own PostgreSQL database (or schema with strict access controls). The Scheduling Service cannot read from the Payments database directly. If the Scheduling Service needs payment status, it calls the Payment Service's API or listens to payment events.

```
Scheduling Service --> scheduling_db (PostgreSQL)
Payment Service    --> payments_db (PostgreSQL)
Coach Service      --> coaches_db (PostgreSQL)
Auth Service       --> auth_db (PostgreSQL)
```

This provides complete independence (each service can evolve its schema freely) but sacrifices cross-service transactions. Distributed transactions (saga pattern) replace atomic database transactions, adding significant complexity.
