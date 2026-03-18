# Authentication Provider

### What It Does

The authentication provider manages user identity: who is this person, and can they prove it? It handles the entire authentication lifecycle -- registration, login, password reset, multi-factor authentication, social login, and token issuance.

Core responsibilities:

- **User registration and login** -- Email/password, magic links, or social login (Google, Apple, Facebook).
- **OAuth 2.0 / OpenID Connect** -- Industry-standard protocols for token-based authentication. The provider issues JWTs (JSON Web Tokens) that the application and API Gateway validate.
- **Social login** -- "Sign in with Google" for clients, reducing friction. Coaches might use email/password for a more professional onboarding flow.
- **JWT issuance and validation** -- Access tokens (short-lived, 15-60 minutes) for API calls. Refresh tokens (long-lived, 7-30 days) for obtaining new access tokens without re-login.
- **Role-Based Access Control (RBAC)** -- Assigns roles (client, coach, admin) and permissions (can_view_bookings, can_manage_payments, can_access_admin_panel). The JWT includes role claims that the application checks.
- **Multi-Factor Authentication (MFA)** -- TOTP (authenticator apps), SMS codes, or email codes as a second factor. Critical for coaches who access client health data and for admin accounts.
- **Account linking** -- A user who registers with email and later tries "Sign in with Google" (using the same email) should end up with one account, not two.

### JWT Structure in the Platform

```json
{
  "sub": "user_abc123",
  "email": "coach@example.com",
  "role": "coach",
  "permissions": ["read:bookings", "write:bookings", "read:clients", "write:workout_plans"],
  "org_id": "gym_456",
  "iat": 1679000000,
  "exp": 1679003600
}
```

The API Gateway validates the signature and expiration. The application code checks `role` and `permissions` for authorization decisions.

### Why It Is Used

Building authentication from scratch is one of the most dangerous things a development team can do. Password hashing, token management, brute-force protection, session fixation prevention, CSRF protection, and social login integration each have subtle security pitfalls. A dedicated authentication provider handles all of this with battle-tested implementations, security certifications, and automatic vulnerability patching.

### Example Technologies

| Technology          | Type       | Notes                                           |
|---------------------|------------|-------------------------------------------------|
| **Auth0**           | Managed    | Full-featured, extensive SDKs, expensive at scale |
| **Supabase Auth**   | Managed/OSS | PostgreSQL-native, Row Level Security integration |
| **Firebase Auth**   | Managed    | Google ecosystem, generous free tier              |
| **Keycloak**        | Self-hosted | Enterprise-grade, SAML + OIDC, complex to operate |
| **Clerk**           | Managed    | Developer-friendly, modern UI components          |
| **AWS Cognito**     | Managed    | AWS-native, user pools + identity pools           |

### Trade-offs

| Advantage                                  | Disadvantage                                          |
|--------------------------------------------|-------------------------------------------------------|
| Battle-tested security (no DIY crypto)      | Vendor lock-in (user data lives in their system)       |
| Social login out of the box                 | Cost scales with monthly active users                  |
| MFA, brute-force protection included        | Customization of login flows can be limited            |
| Reduces development time significantly       | Latency for token validation if not cached             |
| Compliance certifications (SOC2, GDPR)      | Migration to a different provider is painful           |

### How It Fits

**Approach A -- Modular Monolith:**
The Auth module integrates with the authentication provider's SDK. On login, the provider issues a JWT. The monolith's middleware validates the JWT on every request (using the provider's public key, cached locally). The Auth module also manages the internal RBAC logic: mapping provider-issued identities to platform-specific roles and permissions stored in the `auth` schema.

```
Client --> Auth Provider (login) --> JWT issued
Client --> API Gateway (JWT in header) --> Monolith (middleware validates JWT, Auth module checks permissions)
```

**Approach B -- Microservices:**
The authentication provider is a shared external service. Every microservice validates JWTs independently using the provider's public key (fetched from a JWKS endpoint and cached). The API Gateway performs initial JWT validation, but individual services may perform additional permission checks specific to their domain. An Auth Service may exist to manage RBAC, token enrichment, and account management, acting as a facade over the external provider.
