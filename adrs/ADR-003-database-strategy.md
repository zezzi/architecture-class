# ADR-003: Database Strategy

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-03-18 |
| **Deciders** | Platform architecture team |

### Context

The platform stores several categories of data:

- **Strongly relational:** Users, bookings, payments, coach-client relationships, schedules — these have strict integrity constraints, foreign keys, and transactional requirements.
- **Semi-structured / document-style:** Coach profiles (variable certifications, specialties), workout templates (nested exercise lists with sets/reps), content metadata (articles, videos with flexible schemas).

The question is whether to use a single database engine or adopt polyglot persistence.

### Option A — PostgreSQL Only (using JSONB for flexible data)

A single PostgreSQL instance handles all data. Relational data uses standard normalized tables. Semi-structured data uses `JSONB` columns, which support indexing (GIN indexes), partial queries, and schema-on-read.

| Pros | Cons |
|------|------|
| Single database to provision, back up, monitor, and secure | JSONB queries can be less intuitive than native document query languages |
| ACID transactions across all data — no distributed transaction headaches | Very large document collections with deep nesting may be less ergonomic than MongoDB |
| JSONB supports GIN indexing, containment queries (`@>`), and path expressions | Team members unfamiliar with JSONB may default to anti-patterns (e.g., storing everything as JSON) |
| Mature ecosystem: pgAdmin, pg_dump, logical replication, extensions (PostGIS, pg_cron) | Horizontal sharding is harder than with natively distributed databases |
| One connection pool, one ORM configuration, one migration pipeline | Write-heavy analytics workloads may eventually need a separate store (but not at moderate scale) |
| Lower operational cost — one managed instance (e.g., Supabase, RDS, Cloud SQL) | |

### Option B — PostgreSQL + MongoDB (Polyglot Persistence)

Relational data stays in PostgreSQL. Document-style data (coach profiles, content, workout templates) moves to MongoDB.

| Pros | Cons |
|------|------|
| MongoDB's document model is a natural fit for nested, variable-schema data | Two databases to provision, monitor, back up, patch, and secure |
| Flexible schema evolution without migrations | Cross-database joins are impossible; data must be denormalized or joined in application code |
| Horizontal scaling via MongoDB sharding (relevant at very large scale) | Transactions spanning both databases require application-level saga coordination |
| Developers with MongoDB experience can be productive immediately | Doubled operational burden without clear performance justification |
| Clear physical separation of concerns | Increased CI/CD complexity — two database containers in local dev and test environments |
| | Consistency risk: a booking references a coach profile that lives in a different database |

### Decision

**Use PostgreSQL with JSONB (Option A).**

### Schema Design Sketch

```sql
-- Relational: strict schema
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       TEXT UNIQUE NOT NULL,
    role        TEXT NOT NULL CHECK (role IN ('client', 'coach', 'admin')),
    created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE bookings (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id   UUID REFERENCES users(id),
    coach_id    UUID REFERENCES users(id),
    starts_at   TIMESTAMPTZ NOT NULL,
    status      TEXT NOT NULL DEFAULT 'pending',
    payment_id  UUID REFERENCES payments(id)
);

-- Semi-structured: JSONB for flexibility
CREATE TABLE coach_profiles (
    user_id         UUID PRIMARY KEY REFERENCES users(id),
    display_name    TEXT NOT NULL,
    bio             TEXT,
    certifications  JSONB DEFAULT '[]',   -- variable list of certs
    specialties     JSONB DEFAULT '[]',   -- tags: "strength", "yoga", etc.
    availability    JSONB DEFAULT '{}'    -- weekly schedule object
);

CREATE TABLE workout_templates (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coach_id    UUID REFERENCES users(id),
    title       TEXT NOT NULL,
    exercises   JSONB NOT NULL           -- nested array of exercises with sets/reps
);

-- GIN index for fast containment queries
CREATE INDEX idx_coach_specialties ON coach_profiles USING GIN (specialties);
```

### Final Justification

PostgreSQL handles both relational and document-style workloads comfortably up to significant scale. The `JSONB` type provides indexing, validation (via CHECK constraints or application-level schemas), and query flexibility that meets the platform's needs without the operational cost of a second database. This follows an important architectural principle: **choose boring technology** — add complexity only when a concrete, measurable need arises, not speculatively. If a future module (e.g., a content recommendation engine) genuinely needs a document store, it can be introduced as a contained decision for that module alone.
