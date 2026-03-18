# Architecture Design Project — Step-by-Step Student Guide

This guide walks you through building your architecture project from start to finish. Follow the phases in order — each one builds on the previous.

---

## Phase 0: Pick Your Project (Day 1)

**Goal:** Choose a product with enough complexity to design around.

1. Pick a real-world software product (e.g., food delivery app, healthcare scheduling, e-commerce platform, learning management system, fleet management, property rental marketplace).
2. Validate complexity: Can you identify at least 6 distinct business areas? If you can only think of 2-3, the project is too simple.
3. Write a 2-3 sentence description of your product. What does it do? Who uses it?

**Checkpoint:** You should be able to list 6+ business areas (e.g., for a food delivery app: ordering, restaurants, delivery, payments, notifications, analytics).

---

## Phase 1: Understand the Domain (Days 1-2)

**Goal:** Break your product into bounded contexts before touching any technology.

### Step 1.1 — Identify Your Actors
List every type of user or external system that interacts with your product.

> Example: Customer, Restaurant Owner, Delivery Driver, Admin, Payment Gateway, SMS Provider

### Step 1.2 — List Business Capabilities
Write down everything your system needs to do, grouped by business area. Don't think about code yet — think about what the business does.

> Example:
> - **Ordering:** Browse menus, add to cart, place order, track order status
> - **Payments:** Charge customer, refund, payout to restaurant
> - **Delivery:** Assign driver, track location, confirm delivery

### Step 1.3 — Define Bounded Contexts
Turn your business areas into bounded contexts. For each one, write:

| Field | What to Write |
|-------|--------------|
| **Name** | A clear name (e.g., "Order Management") |
| **Purpose** | One sentence — what is this context responsible for? |
| **Core Entities** | List 3-5 entities with their key fields |
| **Events Published** | What events does this context announce? (use past tense: `OrderPlaced`, `PaymentCompleted`) |
| **Events Consumed** | What events from other contexts does it listen to? |
| **Upstream/Downstream** | Which contexts does it depend on? Which depend on it? |

**Checkpoint:** You have 6+ bounded contexts, each with entities, events, and relationships defined. No context should do "everything" — if one feels too big, split it.

---

## Phase 2: Compare Architectural Approaches (Days 2-3)

**Goal:** Evaluate two architecture styles and pick one with real justification.

### Step 2.1 — Pick Two Approaches to Compare
Common pairs:
- Modular monolith vs. microservices
- Monolith vs. event-driven microservices
- Serverless vs. containerized services

### Step 2.2 — Build a Comparison Table
Compare them across these dimensions:

| Dimension | Approach A | Approach B |
|-----------|-----------|-----------|
| Deployment complexity | | |
| Team size fit | | |
| Scalability | | |
| Fault isolation | | |
| Development speed (initial) | | |
| Operational cost | | |
| Data consistency | | |

### Step 2.3 — Make a Recommendation
Choose one and write 2-3 paragraphs explaining **why it fits your specific project**. Don't just say "microservices scale better" — explain why your project needs (or doesn't need) that scalability.

**Deliverable:** Write `proposals/01-high-level-architecture.md`

---

## Phase 3: Document Bounded Contexts & Data Flows (Days 3-5)

**Goal:** Turn your domain understanding into formal documentation.

### Step 3.1 — Write the Bounded Contexts Document
Take what you did in Phase 1 and formalize it. Include a context map showing relationships between all contexts. Label relationships using DDD patterns:
- **OHS/PL** — Open Host Service / Published Language
- **ACL** — Anti-Corruption Layer
- **CF** — Conformist
- **SK** — Shared Kernel

**Deliverable:** Write `proposals/02-bounded-contexts.md`

### Step 3.2 — Define Code Organization
Map your bounded contexts to a concrete directory structure. Show:
- Folder layout
- Which modules/services correspond to which bounded context
- Shared libraries or common code

**Deliverable:** Write `proposals/03-service-module-decomposition.md`

### Step 3.3 — Describe Key Data Flows
Pick at least 3 important user journeys (e.g., "User places an order," "Payment fails and triggers a refund"). For each flow:

1. List every step sequentially
2. Mark which steps are synchronous (API calls) vs. asynchronous (events/queues)
3. Describe the **happy path**
4. Describe what happens when something **fails** (compensation/rollback)

**Deliverable:** Write `proposals/04-data-flow-and-interactions.md`

**Checkpoint:** All 4 proposal documents are complete. Read through them — do they tell a consistent story?

---

## Phase 4: Write Architecture Decision Records (Days 5-7)

**Goal:** Document the "why" behind every major technical choice.

### Step 4.1 — Pick Your ADR Topics
Choose at least 5 from this list:

1. Monolith vs. Microservices
2. Synchronous vs. Asynchronous Communication
3. Database Strategy
4. Queue / Event Bus Technology
5. Authentication Strategy
6. Observability Strategy
7. API Design (REST vs. GraphQL vs. gRPC)
8. Caching Strategy
9. Deployment Strategy
10. Frontend Architecture

### Step 4.2 — Write Each ADR
For every ADR, follow this process:

1. **State the problem** — What decision do you need to make? What constraints exist?
2. **List at least 2 options** — Research real tools/approaches. For each, write an honest pros/cons table.
3. **Choose one** — Explain why it's the best fit **for your project** (not in general).
4. **Acknowledge trade-offs** — What do you give up? What risks remain?

Use this template for each file:

```markdown
# ADR-XXX: [Decision Title]

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | YYYY-MM-DD |

### Context
[What problem are you solving? What constraints exist?]

### Options Considered

**Option 1: [Name]**

| Pros | Cons |
|------|------|
| ... | ... |

**Option 2: [Name]**

| Pros | Cons |
|------|------|
| ... | ... |

### Decision
[Which option did you choose and why?]

### Consequences
[What are the trade-offs? What does this enable or prevent?]
```

**Deliverable:** Write `adrs/ADR-001-*.md` through `adrs/ADR-005-*.md` (or more)

**Checkpoint:** Read your ADRs together. Do they contradict each other? Does ADR-003 (database) align with what you wrote in the proposals about data flows?

---

## Phase 5: Create Architecture Diagrams (Days 7-9)

**Goal:** Visualize your architecture with clear, readable diagrams.

### Step 5.1 — Choose a Diagramming Tool
Options: Mermaid (works in Markdown), PlantUML, Draw.io, Excalidraw, Lucidchart

### Step 5.2 — Create Each Required Diagram

**Diagram 1: System Context** (`diagrams/01-system-context.md`)
- Show your system as a single box in the center
- Surround it with external actors (users, external APIs, third-party services)
- Draw arrows showing who talks to what
- Keep it high-level — no internal details

**Diagram 2: Bounded Context Map** (`diagrams/02-bounded-context-map.md`)
- Show all your bounded contexts
- Group them by type: Core Domain, Supporting, Generic
- Label the relationships (OHS/PL, ACL, CF, SK)

**Diagram 3: Data Flow / Sequence Diagrams** (`diagrams/03-data-flow.md`)
- Create at least 2 sequence diagrams for key flows
- Max 6 participants per diagram — split if needed
- Show both sync calls (solid arrows) and async events (dashed arrows)

**Diagram 4: Deployment / Infrastructure** (`diagrams/04-deployment.md`)
- Show what runs where (containers, servers, managed services)
- Include data stores, message brokers, CDN, load balancers
- Show network boundaries (public internet, private VPC)

### Rules for Good Diagrams
- **Max ~10 nodes per diagram.** If it's too dense, split it.
- **Label everything.** Arrows should say what data flows through them.
- **Write an explanation** under each diagram (2-3 sentences minimum).

**Checkpoint:** Show your diagrams to someone else. Can they understand your architecture in 2 minutes?

---

## Phase 6: Document Infrastructure Components (Days 9-11)

**Goal:** Justify every piece of infrastructure in your system.

### Step 6.1 — Pick at Least 8 Components
Choose from: API Gateway, CDN, Load Balancer, Cache, Message Queue, Background Workers, Database, Object Storage, Auth Provider, Payment Provider, Notification System, Observability Stack, Analytics Pipeline, Search Engine, CI/CD Pipeline.

### Step 6.2 — Write Each Component Document
For every component, address these 5 sections:

| Section | What to Write |
|---------|--------------|
| **What it does** | 3-5 bullet points of core responsibilities |
| **Why your project needs it** | Specific to YOUR use case — not generic |
| **Technology choice** | Compare at least 2 options in a table, then pick one |
| **Trade-offs** | Advantages vs. disadvantages table |
| **How it fits** | How does it connect to other components? Reference your diagrams |

**Common mistake to avoid:** Don't just describe what Redis is. Explain why YOUR project needs caching and what specific data you'd cache.

**Deliverable:** Write `infrastructure/01-*.md` through `infrastructure/08-*.md` (or more)

---

## Phase 7: Assemble and Polish (Days 11-13)

**Goal:** Make your repository professional and navigable.

### Step 7.1 — Create Folder READMEs
Each folder (`/proposals`, `/adrs`, `/diagrams`, `/infrastructure`) needs a `README.md` that:
- Briefly describes what's in the folder
- Lists and links to every document inside

### Step 7.2 — Write the Root README
Your root `README.md` must include:
- Project name and 2-3 sentence description
- The two architectural approaches you compared and which you chose
- Links to every folder and document with brief descriptions
- A clean folder structure overview

### Step 7.3 — Consistency Check
Go through this checklist before submitting:

- [ ] If an ADR says "use PostgreSQL," does the infrastructure doc also say PostgreSQL?
- [ ] Do the sequence diagrams match the data flows described in the proposals?
- [ ] Are the bounded contexts in the proposal the same ones in the context map diagram?
- [ ] Does the deployment diagram include all the infrastructure components you documented?
- [ ] Do all links in your READMEs work?
- [ ] Are files named consistently?
- [ ] Are there any empty placeholder files?

---

## Phase 8: Final Review (Day 14)

**Goal:** Self-evaluate before submitting.

Rate yourself on each section using the rubric:

| Section | Question to Ask Yourself | Points |
|---------|------------------------|--------|
| **Proposals (4 pts)** | Did I genuinely compare two approaches? Are my bounded contexts well-scoped with real entities and events? Do my data flows cover failure cases? | /4 |
| **ADRs (4 pts)** | Does every ADR have 2+ real options with honest pros/cons? Are decisions justified for MY project specifically? | /4 |
| **Diagrams (3 pts)** | Are diagrams readable (<10 nodes)? Do they have written explanations? Are they consistent with my documents? | /3 |
| **Infrastructure (2 pts)** | Did I compare technology options? Are trade-offs specific to my project? Does "how it fits" reference my architecture? | /2 |
| **Repo Structure (2 pts)** | Is the README complete with working links? Is naming consistent? Is it easy to navigate? | /2 |

---

## Suggested Timeline

| Days | Phase | What You're Doing |
|------|-------|------------------|
| 1-2 | 0-1 | Pick project, understand domain, identify bounded contexts |
| 2-3 | 2 | Compare architectural approaches, write high-level proposal |
| 3-5 | 3 | Document bounded contexts, code organization, data flows |
| 5-7 | 4 | Write 5+ ADRs |
| 7-9 | 5 | Create all 4 diagrams |
| 9-11 | 6 | Document 8+ infrastructure components |
| 11-13 | 7 | Assemble repo, write READMEs, consistency check |
| 14 | 8 | Final self-review and submit |

---

## Tips for Success

1. **Start with the domain, not the technology.** Understand what your system does before deciding how to build it.
2. **Be specific to YOUR project.** "PostgreSQL is popular" is not a justification. "Our system has complex relational data between orders, inventory, and payments that benefits from ACID transactions" is.
3. **Consistency is worth more than perfection.** A simpler architecture where all documents agree is better than an ambitious one full of contradictions.
4. **Failures matter.** Don't just describe happy paths. What happens when a payment fails mid-order? How does your system recover?
5. **Diagrams should be readable.** A clean diagram with 8 nodes beats a comprehensive one with 30 that nobody can follow.
6. **Use the example repo as a format reference** — but don't copy its content. Your architecture must be original and specific to your project.
