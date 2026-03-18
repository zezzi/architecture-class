## 2.2 Coaching

| Aspect | Detail |
|---|---|
| **Purpose** | Manages coach profiles, professional certifications, offered services, and specializations. Represents the supply side of the marketplace. |
| **Core Entities** | `Coach`, `Service`, `Certification`, `Specialization`, `CoachReview` |
| **Events Published** | `CoachProfileCreated`, `CoachProfileUpdated`, `CoachVerified`, `ServiceCreated`, `ServiceUpdated`, `ServiceDeactivated`, `ReviewSubmitted` |
| **Events Consumed** | `UserRegistered` (from Identity -- creates coach profile shell), `PaymentConfirmed` (from Payments -- marks service as paid/active) |
| **Upstream Contexts** | Identity & Access |
| **Downstream Contexts** | Scheduling, Search, Analytics, Programs |

**Entity Details:**

- **Coach** -- `id`, `userId` (FK to Identity), `bio`, `headline`, `specializations[]`, `experienceYears`, `hourlyRate`, `currency`, `verificationStatus` (pending/verified/rejected), `rating`, `totalReviews`, `isActive`
- **Service** -- `id`, `coachId`, `name`, `description`, `type` (one-on-one, group, program), `durationMinutes`, `price`, `currency`, `maxParticipants`, `isActive`
- **Certification** -- `id`, `coachId`, `name`, `issuingOrganization`, `issueDate`, `expiryDate`, `documentUrl`, `verifiedAt`
- **Specialization** -- `id`, `name`, `category` (strength, cardio, nutrition, yoga, rehabilitation, etc.)
- **CoachReview** -- `id`, `coachId`, `clientUserId`, `rating`, `comment`, `createdAt`

**Key Rules:**
- A coach profile is created automatically when a user registers with the coach role; it remains in `pending` verification status until an admin approves.
- Services cannot be booked until the coach is verified.
- Rating is a computed aggregate, updated whenever a new review is submitted.

---

#
