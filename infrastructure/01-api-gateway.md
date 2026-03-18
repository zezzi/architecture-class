# API Gateway

### What It Does

An API Gateway is the single entry point for every client request -- mobile apps, web browsers, third-party integrations. It sits between the outside world and your internal services, handling cross-cutting concerns before a request ever reaches your application code.

Core responsibilities:

- **Request routing** -- Inspects the incoming URL, headers, or method and forwards the request to the correct backend service or module.
- **Rate limiting** -- Prevents abuse by capping the number of requests a client can make within a time window (e.g., 100 requests per minute per API key).
- **Authentication validation** -- Verifies JWT tokens, API keys, or OAuth tokens before the request proceeds. It does not issue tokens (that is the Authentication Provider's job) but it rejects invalid ones early.
- **API versioning** -- Routes `/api/v1/workouts` and `/api/v2/workouts` to different backends or handlers, enabling gradual API evolution without breaking existing clients.
- **Request/response transformation** -- Can modify headers, strip internal fields from responses, or aggregate results from multiple services into a single response.
- **SSL/TLS termination** -- Decrypts HTTPS at the edge so backend services can communicate over plain HTTP internally.
- **Logging and metrics emission** -- Produces access logs and latency metrics for every request, feeding the observability stack.

### Why It Is Used

Without a gateway, every service would need to implement its own rate limiting, authentication checks, CORS handling, and logging. This leads to duplicated logic, inconsistent enforcement, and a larger attack surface. The gateway centralizes these concerns so that application code focuses on business logic.

The gateway protects sensitive endpoints (payment webhooks, coach dashboards, client health data) and ensures fair usage across coaches, clients, and third-party apps.

### Example Technologies

| Technology        | Type                | Best For                                      |
|-------------------|---------------------|-----------------------------------------------|
| **Kong**          | Self-hosted / Cloud | Plugin ecosystem, Lua-based extensibility      |
| **Traefik**       | Self-hosted         | Docker/Kubernetes native, auto-discovery       |
| **AWS API Gateway** | Managed           | Serverless architectures, AWS-native stacks    |
| **Envoy**         | Self-hosted         | High-performance proxy, service mesh sidecar   |
| **APISIX**        | Self-hosted         | High performance, dynamic routing              |

### Trade-offs

| Advantage                                     | Disadvantage                                          |
|-----------------------------------------------|-------------------------------------------------------|
| Single place to enforce security policies      | Single point of failure if not deployed with HA        |
| Clients interact with one stable URL           | Adds latency (typically 1-5ms per hop)                 |
| Simplifies client code (no service discovery)  | Configuration complexity grows with the number of routes |
| Enables canary deployments and A/B testing     | Can become a bottleneck under extreme traffic          |
| Centralizes logging and rate limiting          | Requires dedicated team knowledge to operate at scale  |

### How It Fits

**Approach A -- Modular Monolith:**
The gateway has a single upstream target -- the monolith process. Routing is simple: every request goes to the same host and port. The gateway still earns its keep through rate limiting, JWT validation, CORS, and versioning. Internal module routing (e.g., deciding whether a request hits the Scheduling module or the Payments module) happens inside the monolith's own router.

```
Client --> API Gateway --> Monolith (internal router dispatches to modules)
```

**Approach B -- Microservices:**
The gateway becomes essential for routing. `/api/v1/workouts` goes to the Workout Service, `/api/v1/payments` goes to the Payment Service, `/api/v1/coaches` goes to the Coach Service. The gateway maintains a routing table (often auto-discovered via Kubernetes service names or Consul). It may also perform request aggregation -- a single client call to `/api/v1/dashboard` might fan out to three services internally.

```
Client --> API Gateway --> Workout Service
                      --> Payment Service
                      --> Coach Service
                      --> Notification Service
```
