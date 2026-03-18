## 2.1 Identity & Access

| Aspect | Detail |
|---|---|
| **Purpose** | Manages user registration, authentication, authorization, roles, and permissions. Acts as the upstream authority for user identity across the entire platform. |
| **Core Entities** | `User`, `Role`, `Permission`, `RefreshToken`, `OAuthConnection` |
| **Events Published** | `UserRegistered`, `UserUpdated`, `UserDeactivated`, `RoleAssigned`, `RoleRevoked`, `PasswordReset` |
| **Events Consumed** | None -- this is a pure upstream context |
| **Upstream Contexts** | None |
| **Downstream Contexts** | All other contexts consume user identity data |

**Entity Details:**

- **User** -- `id`, `email`, `passwordHash`, `firstName`, `lastName`, `phone`, `avatarUrl`, `status` (active/suspended/deactivated), `emailVerifiedAt`, `createdAt`, `updatedAt`
- **Role** -- `id`, `name` (client, coach, admin, super_admin), `description`, `permissions[]`
- **Permission** -- `id`, `resource`, `action` (create, read, update, delete), `conditions`
- **RefreshToken** -- `id`, `userId`, `token`, `expiresAt`, `revokedAt`
- **OAuthConnection** -- `id`, `userId`, `provider` (google, apple, facebook), `providerUserId`, `accessToken`, `refreshToken`

**Key Rules:**
- A user can hold multiple roles (e.g., a coach is also a client).
- Permissions are additive -- the union of all role permissions applies.
- `UserRegistered` triggers downstream profile creation in Coaching (if role=coach) and default notification preferences.

---

#
