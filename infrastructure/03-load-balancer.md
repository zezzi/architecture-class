# Load Balancer

### What It Does

A load balancer distributes incoming network traffic across multiple instances of your application to ensure no single instance is overwhelmed. It operates at Layer 4 (TCP) or Layer 7 (HTTP) of the OSI model.

Core responsibilities:

- **Traffic distribution** -- Spreads requests across healthy backend instances using algorithms like round-robin, least connections, or weighted distribution.
- **Health checks** -- Periodically pings backend instances (e.g., `GET /health`) and removes unhealthy ones from the pool.
- **SSL/TLS termination** -- Decrypts HTTPS traffic so backends receive plain HTTP, offloading CPU-intensive cryptographic work.
- **Session affinity (sticky sessions)** -- Optionally routes all requests from the same client to the same backend (useful when session state is stored in memory, though Redis-based sessions make this unnecessary).
- **Connection draining** -- When a backend instance is being removed (during a deploy, for example), the load balancer stops sending new requests but allows in-flight requests to complete.

### Why It Is Used

A single application instance cannot handle unlimited traffic. When the platform has thousands of coaches and tens of thousands of clients booking sessions, streaming videos, and processing payments simultaneously, you need multiple instances. The load balancer ensures even distribution and automatic failover if an instance crashes.

### Example Technologies

| Technology           | Type          | Best For                                        |
|----------------------|---------------|-------------------------------------------------|
| **Nginx**            | Self-hosted   | Lightweight, well-understood, also serves static files |
| **HAProxy**          | Self-hosted   | High-performance TCP/HTTP, detailed stats page   |
| **AWS ALB**          | Managed       | HTTP/2, WebSocket, path-based routing, AWS native |
| **AWS NLB**          | Managed       | Ultra-low latency Layer 4, static IPs            |
| **Kubernetes Ingress** | Orchestrated | Auto-scales with cluster, native service discovery |

### Trade-offs

| Advantage                                 | Disadvantage                                         |
|-------------------------------------------|------------------------------------------------------|
| Enables horizontal scaling                 | Adds infrastructure complexity                        |
| Automatic failover on instance failure     | Misconfigured health checks can cascade failures      |
| Enables zero-downtime deployments          | SSL termination centralizes a security-sensitive point |
| Offloads TLS from application servers      | Sticky sessions can create uneven load                |
| Provides connection metrics and access logs | Additional cost for managed load balancers            |

### How It Fits

**Approach A -- Modular Monolith:**
The load balancer sits in front of N identical instances of the monolith. All instances run the same code and connect to the same database. Scaling is simple: add more instances behind the load balancer. A typical setup uses 2-4 instances for redundancy and scales to 8-12 during peak hours (e.g., Monday morning when everyone books their weekly sessions).

```
API Gateway --> Load Balancer --> Monolith Instance 1
                              --> Monolith Instance 2
                              --> Monolith Instance 3
```

**Approach B -- Microservices:**
Each service has its own load balancer (or a shared one with path-based routing rules). The Workout Service might run 4 instances, the Payment Service runs 2 (lower traffic but higher criticality), and the Notification Service runs 6 during campaign blasts. Kubernetes typically handles this via internal Services and Ingress controllers, abstracting the load balancer per service.

```
API Gateway --> LB (Workout) --> Workout Instance 1, 2, 3, 4
            --> LB (Payment) --> Payment Instance 1, 2
            --> LB (Notify)  --> Notification Instance 1, 2, 3, 4, 5, 6
```
