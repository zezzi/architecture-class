# CDN

### What It Does

A Content Delivery Network is a globally distributed network of edge servers that cache and serve static content close to the end user. When a client in Buenos Aires requests a workout video, the CDN serves it from a nearby edge node in South America instead of routing the request all the way to your origin server in Virginia.

Core responsibilities:

- **Static asset delivery** -- JavaScript bundles, CSS, images, fonts, and workout video thumbnails.
- **Video streaming** -- Workout demonstration videos, coach-recorded sessions, and progress review clips. Large files benefit enormously from edge caching.
- **DDoS protection** -- CDN providers absorb volumetric attacks at the edge, preventing malicious traffic from reaching your origin.
- **Edge caching** -- Responses are cached based on URL, headers, and cache-control directives. Subsequent requests for the same resource are served from cache without hitting the origin.
- **TLS termination at the edge** -- Encrypted connections terminate at the nearest edge node, reducing round-trip latency.
- **Compression** -- Automatically compresses responses with Gzip or Brotli.

### Why It Is Used

The platform serves media-heavy content. Coaches upload workout videos (potentially hundreds of megabytes each). Clients stream these videos on mobile networks. Without a CDN, every video request hits your origin server, consuming bandwidth and increasing latency. A CDN reduces origin load by 80-95% for cacheable content and cuts page load times significantly for geographically distributed users.

### Example Technologies

| Technology        | Type       | Strengths                                          |
|-------------------|------------|----------------------------------------------------|
| **CloudFlare**    | Managed    | Free tier, built-in WAF, Workers for edge compute  |
| **CloudFront**    | Managed    | Deep AWS integration, Lambda@Edge                  |
| **Fastly**        | Managed    | Instant purge, VCL configuration, real-time logs   |
| **Bunny CDN**     | Managed    | Low cost, simple API, good for video               |

### Trade-offs

| Advantage                                    | Disadvantage                                            |
|----------------------------------------------|---------------------------------------------------------|
| Dramatically lower latency for static assets  | Cache invalidation is hard (stale content risk)          |
| Reduces origin server bandwidth costs         | Costs scale with bandwidth (video-heavy apps pay more)   |
| Built-in DDoS mitigation                      | Debugging cache behavior can be opaque                   |
| Improves perceived performance globally       | Dynamic content cannot be edge-cached easily             |
| Offloads TLS handshake work from origin       | Adds a dependency on a third-party provider              |

### How It Fits

**Approach A -- Modular Monolith:**
The CDN sits in front of the monolith's static file routes and object storage (S3). The monolith generates pre-signed URLs for private content (client progress photos, paid workout plans) and the CDN caches public content (marketing pages, free workout thumbnails). Configuration is straightforward because there is one origin.

**Approach B -- Microservices:**
The CDN still has one primary origin (typically the API Gateway or a dedicated static file server), but cache rules may vary by path. `/static/*` has long TTLs, `/api/*` is marked as uncacheable. Video content served from S3 has its own CloudFront distribution with specific cache behaviors. Each service that generates static artifacts (e.g., PDF reports from the Reporting Service) pushes them to S3, and the CDN serves them from there.

In both approaches, the CDN configuration is largely the same. The difference is organizational: in Approach B, multiple teams may need to coordinate cache-control headers and invalidation strategies.
