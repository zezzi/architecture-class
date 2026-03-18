# ADR-005: Authentication Strategy

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-03-18 |
| **Deciders** | Platform architecture team |

### Context

The platform has three user roles — client, coach, and admin — each with different permissions. Clients book sessions and track progress. Coaches manage schedules, publish content, and view client data. Admins configure the platform. Authentication must be secure, support role-based access control (RBAC), and ideally offer social login (Google, Apple) for client convenience. Maintaining custom security infrastructure diverts engineering effort from core product development.

### Option A — Self-Managed (JWT + bcrypt, custom auth module)

Build a custom authentication module: password hashing with bcrypt/argon2, JWT issuance and verification, refresh token rotation, password reset flows, and RBAC middleware.

| Pros | Cons |
|------|------|
| Full control over every aspect of the auth flow | Security is hard — custom implementations are a common attack surface |
| No external dependency or vendor lock-in | Must implement and maintain: password hashing, token rotation, rate limiting, brute-force protection, MFA |
| No per-user pricing at any scale | Password reset, email verification, and account recovery flows are non-trivial |
| Can be tailored exactly to the platform's needs | Social login (Google, Apple) requires integrating each OAuth provider manually |
| Full control over customization | Ongoing security patching and vulnerability monitoring is the team's responsibility |
| | GDPR/compliance burden for storing credentials |

### Option B — Managed Auth Provider (Auth0, Supabase Auth, Firebase Auth)

Delegate authentication to a managed service. The platform receives JWT tokens from the provider and validates them locally. User management, social login, MFA, and account recovery are handled by the provider.

| Pros | Cons |
|------|------|
| Battle-tested security: brute-force protection, MFA, anomaly detection out of the box | Vendor dependency — migrating away requires re-implementing auth |
| Social login (Google, Apple, GitHub) is a configuration toggle, not a code change | Free tiers have limits (e.g., Auth0: 7,500 MAU; Supabase: 50,000 MAU) |
| RBAC and custom claims supported natively | Less control over the login UI and flow (some providers offer customization, others less so) |
| Compliance certifications (SOC 2, GDPR) handled by the provider | Latency for token issuance depends on the provider's infrastructure |
| Dramatically reduces development time — no password reset, email verification, or token rotation code | Team has less visibility into auth internals |
| Automatic security updates — the team is not on the hook for CVEs | |

### Decision

**Use a managed auth provider (Option B) — specifically Auth0 or Supabase Auth — with JWT tokens validated locally in the application.**

### Implementation Architecture

```
┌──────────┐     ┌──────────────────┐     ┌──────────────────────┐
│  Client  │────►│  Auth Provider   │────►│  JWT issued with     │
│  (App)   │     │  (Auth0/Supabase)│     │  role claim:         │
└──────────┘     └──────────────────┘     │  { role: "coach" }   │
                                          └──────────┬───────────┘
                                                     │
                                                     ▼
                                          ┌──────────────────────┐
                                          │  Platform API        │
                                          │  - Validate JWT sig  │
                                          │  - Extract role      │
                                          │  - Enforce RBAC      │
                                          └──────────────────────┘
```

### Role-Based Access Control Matrix

| Resource | Client | Coach | Admin |
|----------|--------|-------|-------|
| View own bookings | Yes | Yes | Yes |
| Create booking | Yes | No | Yes |
| Manage schedule | No | Yes (own) | Yes (all) |
| Publish content | No | Yes | Yes |
| View client progress | No | Yes (assigned clients) | Yes (all) |
| Process refunds | No | No | Yes |
| Manage users | No | No | Yes |
| View analytics dashboard | No | Yes (own metrics) | Yes (all) |

### Final Justification

Building custom auth is a high-risk, low-reward investment. A single vulnerability in password storage, token handling, or session management can compromise the entire platform. Managed providers like Auth0 and Supabase Auth eliminate this risk class entirely while providing social login, MFA, and RBAC out of the box. The JWT tokens issued by the provider are standard and validated locally, so the platform is not making network calls to the provider on every request. For Approach A, Supabase Auth is a natural fit if the team is already using Supabase for PostgreSQL hosting. The value lies in integrating with an identity provider, validating tokens, enforcing RBAC at the middleware level, and structuring authorization logic — not in re-implementing bcrypt.
