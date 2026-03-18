# Object Storage (S3)

### What It Does

Object storage is a flat-namespace, highly durable storage system for unstructured data -- files, images, videos, documents. Unlike a filesystem, objects are stored with a key (path), metadata, and the binary content. There are no directories, no file locks, no append operations. You put an object and get it by key.

What the platform stores:

- **Workout videos** -- Coaches upload demonstration videos (50MB - 2GB each). These are the platform's most storage-intensive assets.
- **Progress photos** -- Clients upload before/after photos for their coach to review.
- **PDF reports** -- Monthly progress reports, workout plan exports, invoices.
- **Exercise images** -- Thumbnails and diagrams for the exercise library.
- **Coach documents** -- Certifications, insurance documents, contracts.
- **Backups** -- Database dumps and configuration backups.

### Pre-signed URLs

The application never proxies large files through its own servers. Instead, it generates pre-signed URLs -- temporary, authenticated URLs that allow direct upload to or download from S3.

**Upload flow:**
1. Client requests an upload URL from the API: `POST /api/uploads/request`.
2. The API generates a pre-signed PUT URL valid for 15 minutes: `https://bucket.s3.amazonaws.com/videos/abc123?X-Amz-Signature=...`.
3. The client uploads directly to S3 using this URL.
4. S3 notifies the application (via event or polling) that the upload is complete.

**Download flow:**
1. Client requests a video: `GET /api/videos/abc123`.
2. The API checks permissions, then returns a pre-signed GET URL valid for 1 hour.
3. The client's browser/player fetches the video directly from S3 (or CDN).

This pattern keeps large file traffic off your application servers entirely.

### Lifecycle Policies

Not all data needs to live in expensive, immediately-accessible storage forever:

| Data Type               | Initial Storage   | After 90 days      | After 1 year     |
|-------------------------|-------------------|--------------------|-------------------|
| Workout videos          | S3 Standard       | S3 Standard        | S3 Standard       |
| Progress photos         | S3 Standard       | S3 Infrequent Access | S3 Glacier       |
| Generated reports       | S3 Standard       | S3 Infrequent Access | Delete           |
| Database backups        | S3 Standard       | S3 Glacier          | S3 Deep Archive  |
| Temporary upload chunks | S3 Standard       | Delete              | --               |

Lifecycle rules automate these transitions, reducing storage costs by 50-80% for aging data.

### Why It Is Used

Storing large files in PostgreSQL (as `bytea` columns) is technically possible but operationally disastrous: it bloats the database, slows backups, and wastes expensive database I/O on file serving. Object storage is purpose-built for this: 99.999999999% (eleven nines) durability, unlimited capacity, and pay-per-GB pricing.

### Example Technologies

| Technology        | Type       | Notes                                          |
|-------------------|------------|------------------------------------------------|
| **Amazon S3**     | Managed    | The industry standard, richest feature set      |
| **Google Cloud Storage** | Managed | Equivalent to S3, GCP-native                |
| **MinIO**         | Self-hosted | S3-compatible, runs on your own infrastructure |
| **Cloudflare R2** | Managed    | S3-compatible, zero egress fees                 |
| **DigitalOcean Spaces** | Managed | S3-compatible, simpler pricing model       |

### Trade-offs

| Advantage                                  | Disadvantage                                       |
|--------------------------------------------|----------------------------------------------------|
| Virtually unlimited storage capacity        | Eventual consistency for some operations (rare now)  |
| Eleven nines of durability                  | Egress fees can be significant for video-heavy apps  |
| Pre-signed URLs offload traffic from servers | No append or partial update (must rewrite the whole object) |
| Lifecycle policies reduce costs automatically | Not suitable for frequently changing data           |
| CDN integration is straightforward          | Access control mistakes can expose private data      |

### How It Fits

**Approach A -- Modular Monolith:**
A single S3 bucket (or a few buckets organized by data type: `gym-videos`, `gym-documents`, `gym-backups`) serves the entire monolith. The Content module manages video uploads, the Reporting module writes PDF exports, and the Auth module stores profile photos. All modules use the same S3 client library and configuration.

**Approach B -- Microservices:**
Each service may own its own bucket or prefix within a shared bucket. The Content Service owns `s3://gym-content/videos/*`, the Reporting Service owns `s3://gym-reports/*`. Bucket policies and IAM roles ensure each service can only access its own prefix. The CDN is configured with multiple origins pointing to different bucket paths.
