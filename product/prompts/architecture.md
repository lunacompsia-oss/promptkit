# Architecture Prompts

15+ prompts for system design, database modeling, API design, and architectural decision-making with AI. Each prompt guides the AI to think through tradeoffs, not just produce diagrams.

---

### System Architecture Design

**When to use:** Starting a new project or major feature that needs architectural planning.
**Tool:** Claude Code / Cursor / Any

```
Design the system architecture for [PRODUCT/FEATURE NAME].

Context:
- What it does: [1-2 sentence description]
- Expected users: [number range, e.g., "100 initially, scaling to 10,000"]
- Team size: [number of developers who will maintain this]
- Timeline: [when does this need to ship]

Design constraints:
- Budget: [infrastructure budget per month]
- Must integrate with: [existing systems, APIs, databases]
- Compliance: [GDPR, SOC2, HIPAA, none]

Provide:
1. High-level architecture diagram (text-based, using boxes and arrows)
2. Component list: each component's responsibility, technology choice, and why
3. Data flow: how does a typical request move through the system?
4. Storage: what data is stored where, and why that storage was chosen
5. Communication: how do components talk to each other? (HTTP, queue, events, direct DB)

For each technology choice, explain:
- Why this over alternatives (tradeoff analysis, not just "it's popular")
- What we give up by choosing this
- When we'd need to reconsider (scale threshold, team growth, feature requirement)

Bias toward simplicity. A monolith that ships is worth more than microservices that don't.
Every additional component is a liability -- justify each one.
```

---

### Database Schema Design

**When to use:** Designing a database from scratch or adding significant new tables.
**Tool:** Claude Code / Cursor / Any

```
Design the database schema for [DOMAIN/FEATURE].

Business context:
[Describe the domain in business terms. What entities exist? How do they relate?
What operations are performed on them? What questions do we need to answer from this data?]

Requirements:
- Database engine: [PostgreSQL / MySQL / SQLite]
- Multi-tenancy model: [shared database, schema per tenant, database per tenant, or single-tenant]
- Expected data volume: [rows per table after 1 year]
- Read/write ratio: [read-heavy, write-heavy, balanced]
- Most common queries: [list the top 5 queries this schema must serve efficiently]

Provide:
1. Table definitions with columns, types, constraints, and comments
2. Primary keys: use [UUIDs / auto-increment / ULIDs] for [reason]
3. Foreign keys: all relationships with ON DELETE behavior specified
4. Indexes: for every column that appears in WHERE, ORDER BY, or JOIN clauses
5. Unique constraints: for business-level uniqueness rules
6. Enum/status columns: list all valid values with their meanings
7. Audit columns: created_at, updated_at, deleted_at (if soft deletes)

Also address:
- Denormalization decisions: where do we intentionally duplicate data for query performance?
- Null vs default: for each nullable column, why is NULL the right choice over a default value?
- Migration plan: what order should tables be created (respecting foreign key dependencies)?

Common pitfalls to avoid:
- Entity-Attribute-Value pattern (use JSONB columns instead if schema is truly dynamic)
- Polymorphic associations (use separate tables or a discriminator column)
- Over-normalization (if you always join two tables together, maybe they should be one)
```

---

### API Design

**When to use:** Designing a new API or major API extension.
**Tool:** Claude Code / Cursor / Any

```
Design the API for [FEATURE/SERVICE].

Context:
- Consumers: [who calls this API -- frontend, mobile app, third-party, internal service]
- Authentication: [API key, JWT, OAuth, none]
- Protocol: [REST, GraphQL, gRPC -- or recommend one]

Business operations to support:
[List every operation the API must support, e.g.:
- Create a project
- List all projects for the current user
- Update a project
- Delete a project
- Add a member to a project
- etc.]

For each endpoint, provide:
1. Method and path (following RESTful conventions)
2. Request: headers, path params, query params, body schema with types
3. Response: status code, body schema with types for success AND error cases
4. Authorization: who can call this and what checks are performed
5. Pagination (for list endpoints): cursor-based with limit parameter
6. Rate limiting: requests per minute per client

Also design:
- Error response format (consistent across all endpoints):
  { "error": { "code": "MACHINE_READABLE", "message": "Human readable", "details": [...] } }
- Versioning strategy: URL prefix (/v1/) or header (Accept: application/vnd.api+json;version=1)
- Webhook format (if events are emitted): consistent envelope with event type and timestamp

Produce an OpenAPI/Swagger-style specification (YAML format) for all endpoints.
```

---

### Monolith to Services Migration Planning

**When to use:** When the monolith is genuinely struggling and you need to extract services.
**Tool:** Claude Code / Cursor / Any

```
Plan the extraction of [MODULE/FEATURE] from the monolith into a separate service.

IMPORTANT: Before proceeding, answer these questions honestly:
1. Is the monolith actually causing problems? (Deployment conflicts? Scaling bottlenecks? Team contention?)
2. Have we tried solving the problem within the monolith first? (Better modules? Database optimization?)
3. Do we have the operational maturity to run multiple services? (Monitoring, deployment, debugging across services)

If you can solve the problem in the monolith, recommend that instead.

If extraction is truly needed:

Current state:
- What does this module do?
- What data does it own?
- What other modules call it?
- What does it call?
- How much traffic does it handle?

Extraction plan:
1. Define the service boundary: what goes in the new service, what stays
2. Define the API contract between the monolith and new service
3. Data migration strategy:
   - Which tables move to the new service's database?
   - How is data synced during migration? (Dual writes? Change data capture? Event sourcing?)
   - What foreign keys cross the boundary? How do we handle them?
4. Incremental migration steps (Strangler Fig):
   - Step 1: Extract the code into a separate module within the monolith
   - Step 2: Add the API contract as an internal interface
   - Step 3: Deploy the new service, route traffic through the monolith
   - Step 4: Route traffic directly to the new service
   - Step 5: Remove the old code from the monolith
5. Rollback plan for each step
6. What new operational complexity does this introduce? (Service discovery, distributed tracing, etc.)

Estimate the total cost: engineering time, infrastructure cost, ongoing maintenance burden.
Compare against "just make the monolith better."
```

---

### Event-Driven Architecture Design

**When to use:** Designing systems that need loose coupling, async processing, or real-time reactivity.
**Tool:** Claude Code / Cursor / Any

```
Design an event-driven architecture for [FEATURE/SYSTEM].

Use cases that need events:
[List the scenarios, e.g.:
- When a user signs up, send a welcome email AND create a Stripe customer AND log analytics
- When a payment succeeds, activate the subscription AND send a receipt AND update usage limits
- When a report is requested, generate it asynchronously AND notify the user when done]

For each event:
1. Event name (past tense: user.signed_up, payment.succeeded, report.requested)
2. Event payload: the minimum data needed by all consumers (prefer IDs over full objects)
3. Producer: which component emits this event
4. Consumers: which components react to this event and what they do
5. Delivery guarantee needed: at-most-once, at-least-once, exactly-once
6. Ordering requirement: do events need to be processed in order?
7. Error handling: what happens if a consumer fails? (Retry? Dead letter? Alert?)

Architecture decisions:
- Event transport: [database-backed queue, Redis streams, SQS, Kafka -- recommend based on scale]
- Event schema: define a standard envelope (id, type, timestamp, version, data)
- Idempotency: how do consumers handle duplicate events?
- Schema evolution: how do we add fields to events without breaking consumers?
- Monitoring: how do we detect if events are stuck, failing, or processing slowly?

Start with the simplest transport that meets the requirements.
A database-backed queue (poll a jobs table) works fine up to thousands of events per minute.
Don't bring in Kafka for 10 events per second.
```

---

### Caching Strategy Design

**When to use:** When performance needs improvement and caching is being considered.
**Tool:** Claude Code / Cursor / Any

```
Design a caching strategy for [APPLICATION/FEATURE].

Current performance problems:
[Describe the slow operations: which endpoints, what latency, what the bottleneck is]

For each cacheable resource, define:
1. What is cached: the exact data shape
2. Cache key pattern: how the key is constructed (include tenant/user scoping)
3. TTL: how long until the cache expires
4. Invalidation strategy: when and how the cache is cleared
   - Time-based: expires after TTL, stale data is acceptable for the TTL duration
   - Event-based: cleared when the underlying data changes
   - Write-through: cache is updated when data is written (strongest consistency)
5. Cache miss behavior: what happens when the cache is cold (thundering herd prevention?)
6. Cache storage: where the cache lives (in-memory, Redis, CDN, HTTP cache headers)

Also address:
- Cache warming: should the cache be pre-populated on deployment?
- Cache monitoring: how do we measure hit rate, miss rate, eviction rate?
- Memory budget: how much memory does the cache need? What's the eviction policy when full?
- Consistency model: what's the maximum staleness we accept?
- Multi-server: if running multiple instances, is the cache shared or per-instance?

Common caching mistakes to avoid:
- Caching everything (cache only what's slow AND frequently accessed)
- No invalidation strategy (stale data is worse than slow data for some use cases)
- Cache stampede (when TTL expires, 1000 requests all rebuild the cache simultaneously)
- Caching user-specific data in a shared cache without proper key scoping

Recommend whether we even need a cache. Sometimes the answer is "add an index"
or "fix the N+1 query" instead of adding caching complexity.
```

---

### Authentication and Authorization Architecture

**When to use:** Designing the auth system for a new application.
**Tool:** Claude Code / Cursor / Any

```
Design the authentication and authorization architecture for [APPLICATION].

Requirements:
- User types: [e.g., individual users, team members, admins]
- Auth methods needed: [email/password, social login, SSO, API keys]
- Multi-tenancy: [single tenant, multi-tenant with teams/orgs]
- Authorization model: [role-based (RBAC), permission-based (ABAC), both]

Design:
1. Authentication flow:
   - How does a user prove their identity? (Full flow diagram)
   - How is the session maintained? (JWT vs server session vs both)
   - Token storage: where and how (httpOnly cookie, localStorage -- and why)
   - Token lifecycle: creation, refresh, expiration, revocation
   - Multi-device: can a user be logged in on multiple devices?

2. Authorization model:
   - What roles exist? What can each role do?
   - Are permissions resource-scoped? (Admin of project A != admin of project B)
   - How are permissions checked in code? (Middleware, decorator, inline check)
   - Where is the authorization logic centralized? (One file, not scattered)

3. Security considerations:
   - Password storage: bcrypt with cost factor >= 12
   - Brute force prevention: rate limiting on auth endpoints
   - Session fixation: new session ID after login
   - CSRF protection: for cookie-based auth
   - Token rotation: refresh tokens rotated on use

4. Database schema:
   - users table: id, email, password_hash, email_verified_at
   - sessions/tokens table (if server-side sessions)
   - roles table and user_roles join table
   - permissions table and role_permissions join table
   - API keys table: key_hash, scopes, last_used_at, expires_at

5. Implementation approach:
   - Use [library/framework] because [reason]
   - Custom implementation for [specific parts] because [reason]

Produce the database schema, middleware code, and a flow diagram.
```

---

### File/Asset Storage Architecture

**When to use:** When the application needs to handle user uploads, media, or generated files.
**Tool:** Claude Code / Cursor / Any

```
Design the file storage architecture for [APPLICATION].

File types to handle:
[List the types, sizes, and access patterns, e.g.:
- User avatars: images, 100KB-5MB, public, accessed frequently
- Documents: PDF/DOCX, 1MB-50MB, private per user, accessed occasionally
- Reports: generated PDFs, 100KB-10MB, private, accessed once then archived]

Design decisions:
1. Storage backend:
   - Where do files live? (S3, R2, GCS, local filesystem)
   - Why this choice? (Cost, latency, CDN integration, compliance)
   - Bucket/container structure (one bucket vs multiple)

2. Upload flow:
   - Direct upload to storage (presigned URL) vs upload through server
   - Maximum file size and how it's enforced
   - File type validation (MIME type check + magic bytes, not just extension)
   - Virus scanning (if required)
   - Upload progress indication for the user

3. Access control:
   - Public files: CDN with long cache headers
   - Private files: signed URLs with expiration
   - How is ownership verified before granting access?

4. Processing:
   - Image resizing: when does it happen? (On upload vs on demand)
   - Thumbnail generation: sizes and formats
   - Document preview generation (if needed)
   - Processing queue for heavy operations

5. Database model:
   - files table: id, user_id, filename, content_type, size_bytes, storage_key,
     storage_bucket, status (uploading, ready, processing, error), metadata (JSONB)
   - Polymorphic association: how files relate to different entities
     (avatar -> user, attachment -> comment, etc.)

6. Cleanup:
   - How are orphaned files detected and removed?
   - What happens to files when the owning entity is deleted?
   - Storage lifecycle rules (move to cold storage after X days)

7. Cost estimation:
   - Storage cost per month at expected volume
   - Bandwidth cost per month at expected traffic
   - Processing cost (if using image resize services)
```

---

### Background Job Architecture

**When to use:** When the application needs to process work asynchronously.
**Tool:** Claude Code / Cursor / Any

```
Design the background job architecture for [APPLICATION].

Jobs that need to run in the background:
[List each job, e.g.:
- Send transactional emails (after signup, purchase, etc.)
- Generate reports (user-initiated, takes 30 seconds to 5 minutes)
- Process webhooks from external services
- Sync data with third-party APIs (hourly)
- Clean up expired resources (daily)]

For each job type, define:
1. Trigger: what causes this job to be enqueued (user action, schedule, event)
2. Priority: how quickly must it start? (immediate, within minutes, within hours)
3. Duration: how long does it typically run?
4. Retry strategy: how many retries, what backoff schedule, what to do on final failure
5. Concurrency: can multiple instances of this job run in parallel?
6. Idempotency: is it safe to run the same job twice? (It must be)

Architecture:
1. Queue backend: [PostgreSQL-backed queue, Redis/BullMQ, SQS -- recommend based on needs]
   For most applications: a database-backed queue (e.g., Postgres with SKIP LOCKED) is sufficient
   and eliminates an infrastructure dependency.

2. Worker deployment:
   - Same process as the web server? (Simpler but shares resources)
   - Separate worker process? (Better isolation, more operational complexity)
   - How many workers? How is concurrency controlled?

3. Job schema:
   - Job table: id, queue, type, payload (JSONB), status, attempts, max_attempts,
     run_at, started_at, completed_at, failed_at, error, locked_by, locked_at

4. Monitoring:
   - How do we know if jobs are backed up? (Queue depth metric)
   - How do we know if jobs are failing? (Error rate metric)
   - How do we inspect a failed job? (Admin UI or query)
   - Alert thresholds: queue depth > X for Y minutes, error rate > Z%

5. Scheduled jobs:
   - Where is the schedule defined? (Cron expressions in code)
   - How is the scheduler deployed? (Only one instance must run the scheduler)
   - What happens if the scheduler misses a window? (Run immediately or skip)

Keep it simple. For under 1000 jobs/hour, a database queue is the right answer.
```

---

### Search Architecture

**When to use:** When the application needs search functionality beyond simple SQL LIKE queries.
**Tool:** Claude Code / Cursor / Any

```
Design the search architecture for [APPLICATION].

What needs to be searchable:
[List each searchable entity and what fields are searched, e.g.:
- Products: name, description, category, tags -- full text, fuzzy matching
- Users: name, email -- exact and prefix matching
- Orders: order number, customer name -- exact matching]

Requirements:
- Expected index size: [number of documents]
- Search latency target: [e.g., under 100ms at p95]
- Features needed: [full-text, fuzzy, faceted, autocomplete, typo-tolerance, relevance ranking]
- Write delay tolerance: [how quickly must new/updated records appear in search?]

Design options (recommend one):
1. PostgreSQL full-text search (tsvector + GIN index)
   - Good for: under 1M documents, simple full-text search, no additional infrastructure
   - Limited: no fuzzy matching, basic relevance ranking
2. Application-level search (in-memory index with FlexSearch, MiniSearch)
   - Good for: under 100K documents, fast autocomplete, zero infrastructure
   - Limited: memory usage, no persistence across restarts
3. Dedicated search engine (Meilisearch, Typesense, Elasticsearch)
   - Good for: large datasets, complex queries, faceting, typo tolerance
   - Cost: additional infrastructure to deploy and maintain

For the recommended approach, provide:
1. Index schema: which fields are indexed, field types, analyzers
2. Sync strategy: how data gets from the database to the search index
   (trigger-based, polling, event-driven)
3. Query building: how user input is transformed into search queries
4. Relevance tuning: how search results are ranked
5. Autocomplete: if needed, how suggestions are generated as the user types
6. Infrastructure: deployment, memory requirements, backup

Start with PostgreSQL full-text search unless you have a specific reason not to.
Migrating to a dedicated search engine later is straightforward.
```

---

### Multi-Tenancy Architecture

**When to use:** Designing a SaaS application that serves multiple customers/organizations.
**Tool:** Claude Code / Cursor / Any

```
Design the multi-tenancy architecture for [APPLICATION].

Context:
- Tenant count: [expected number of tenants at 1 year]
- Largest tenant: [expected data volume for the biggest customer]
- Isolation requirements: [shared everything, logically isolated, physically isolated]
- Compliance: [any requirements for data isolation -- GDPR, HIPAA, enterprise clients]

Evaluate these approaches:
1. Shared database, shared schema (tenant_id column on every table)
   - Pros: simplest, cheapest, easiest to maintain
   - Cons: risk of data leakage if tenant_id filter is missed, noisy neighbor

2. Shared database, schema per tenant (PostgreSQL schemas)
   - Pros: better isolation, per-tenant backup/restore
   - Cons: migration complexity (must run on all schemas), connection pooling challenges

3. Database per tenant
   - Pros: strongest isolation, per-tenant performance, easy data export
   - Cons: expensive, complex operations, hard to query across tenants

Recommend one approach and design:

1. Tenant resolution: how do we determine which tenant a request belongs to?
   (Subdomain, header, URL path, API key, JWT claim)

2. Query scoping: how do we ensure EVERY query is scoped to the correct tenant?
   (Middleware that sets tenant context, ORM default scope, row-level security)
   THIS IS THE MOST CRITICAL PART -- a missed filter means data leakage.

3. Data isolation testing: how do we verify that tenant A cannot see tenant B's data?
   (Automated tests that create data for two tenants and verify isolation)

4. Tenant lifecycle: create tenant, onboard, migrate, export data, delete tenant

5. Cross-tenant operations: what admin queries need to span all tenants?
   (Usage reporting, billing, analytics)

6. Performance isolation: how do we prevent one tenant from affecting others?
   (Rate limiting per tenant, query timeout per tenant, resource quotas)

Default to the shared schema approach unless there's a specific compliance reason
for stronger isolation. Shared schema + row-level security covers 95% of SaaS apps.
```

---

### Notification System Architecture

**When to use:** Designing a system that sends notifications across multiple channels.
**Tool:** Claude Code / Cursor / Any

```
Design the notification system for [APPLICATION].

Notification types:
[List each notification, e.g.:
- Welcome email after signup
- Payment receipt after purchase
- Task assigned (in-app + email)
- Weekly digest (email)
- Real-time alerts (in-app push)]

Channels: [email, in-app, push notification, SMS, Slack/webhook]

Design:
1. Notification dispatch:
   - A notification is triggered by a business event
   - The system determines: which channels, which template, which recipients
   - Each channel delivery is a separate background job (one failure doesn't block others)

2. Channel implementations:
   - Email: [SendGrid / Postmark / SES] -- template approach (Markdown + renderer vs email builder)
   - In-app: database table (notifications) polled by frontend or pushed via WebSocket
   - Push: [FCM / APNs / Web Push] -- device token management
   - SMS: [Twilio / MessageBird] -- use sparingly (expensive, intrusive)

3. User preferences:
   - Per-notification-type toggle (user can mute specific types)
   - Per-channel toggle (user can disable email but keep in-app)
   - Quiet hours / do-not-disturb
   - Frequency control (immediate vs digest)

4. Database schema:
   - notification_types: type, default_channels, template
   - notification_preferences: user_id, notification_type, channel, enabled
   - notifications: id, user_id, type, title, body, data (JSONB), read_at, created_at
   - notification_deliveries: notification_id, channel, status, sent_at, error

5. Template system:
   - Where are templates stored? (Code, database, or template service)
   - How are they rendered? (Variable substitution, Markdown to HTML, MJML for email)
   - How are they tested? (Preview mode, test send)

6. Delivery guarantees:
   - Deduplication: prevent sending the same notification twice
   - Ordering: do notifications need to arrive in order?
   - Batching: aggregate multiple events into one notification (digest mode)

Keep the first version simple: a notifications table + email via transactional email service.
Add push notifications and real-time only when users ask for them.
```

---

### Rate Limiting and Abuse Prevention

**When to use:** Designing protection against abuse, DoS, and resource exhaustion.
**Tool:** Claude Code / Cursor / Any

```
Design rate limiting and abuse prevention for [APPLICATION].

Endpoints/resources to protect:
[List each endpoint with its acceptable usage pattern, e.g.:
- POST /login: max 5 attempts per minute per IP (brute force prevention)
- POST /signup: max 3 per hour per IP (spam prevention)
- GET /api/*: 100 requests per minute per API key (fair usage)
- POST /upload: 10 per hour per user (resource protection)]

Design:
1. Rate limiting algorithm: [token bucket, sliding window, fixed window -- recommend one]
   - Token bucket: smooth rate limiting, allows bursts. Best for APIs.
   - Sliding window: more precise, prevents boundary abuse. Best for login.
   - Fixed window: simplest. Fine if boundary abuse is acceptable.

2. Rate limit storage:
   - In-memory (single server only, lost on restart)
   - Redis (shared across servers, persistent)
   - Database (works but slower -- acceptable for low-traffic apps)

3. Rate limit keys: what identifies a client?
   - Unauthenticated: IP address (consider shared IPs behind NAT)
   - Authenticated: user ID or API key
   - Combined: IP + user ID for sensitive endpoints

4. Response when limited:
   - HTTP 429 Too Many Requests
   - Retry-After header with seconds until limit resets
   - X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset headers
   - Friendly error message explaining the limit

5. Additional abuse prevention:
   - CAPTCHA on signup/login after N failures
   - Email verification before access to expensive operations
   - IP reputation checking (optional, for high-value targets)
   - Request size limits on all endpoints
   - Timeout on all database queries and external API calls

6. Monitoring:
   - Alert when rate limits are hit frequently (possible attack)
   - Alert when a single IP/user hits limits repeatedly
   - Dashboard showing rate limit events over time

Implementation should be middleware that applies to all routes by default,
with per-route overrides for stricter or looser limits.
```

---

### Data Migration Architecture

**When to use:** Planning a migration from one data store, format, or system to another.
**Tool:** Claude Code / Cursor / Any

```
Design the data migration from [SOURCE] to [DESTINATION].

Migration scope:
- Tables/collections to migrate: [list]
- Total data volume: [rows/GB]
- Downtime tolerance: [zero downtime, maintenance window of X hours, flexible]
- Data transformation needed: [schema changes, format changes, data enrichment]

Design the migration in phases:

Phase 1 -- Preparation:
1. Map source schema to destination schema (column by column)
2. Identify transformations needed for each field
3. Identify data that cannot be migrated (orphaned records, invalid data)
4. Create validation queries that verify data integrity post-migration

Phase 2 -- Dual Write Setup (for zero-downtime):
1. Application writes to BOTH source and destination
2. Reads continue from source
3. Monitor that dual writes are consistent

Phase 3 -- Backfill:
1. Migrate historical data in batches (1000-10000 rows per batch)
2. Track progress (last migrated ID or timestamp)
3. Handle failures gracefully (retry the batch, not the whole migration)
4. Run continuously until caught up with dual writes

Phase 4 -- Verification:
1. Count comparison: same number of records in both
2. Checksum comparison: data content matches
3. Application-level verification: key business queries return same results

Phase 5 -- Cutover:
1. Switch reads from source to destination
2. Continue dual writes for rollback safety
3. Monitor for errors and performance
4. After confidence period, stop dual writes
5. Deprecate source (but keep it read-only for 30 days as safety net)

Rollback plan:
- At any phase, how do we go back to the previous state?
- What data might be lost if we rollback?
- How long does rollback take?

Write the migration scripts, validation queries, and runbook.
```

---

### Architectural Decision Record (ADR)

**When to use:** When making any significant technical decision that future developers will question.
**Tool:** Claude Code / Cursor / Any

```
Write an Architectural Decision Record for the following decision:

**Decision:** [what we decided to do]
**Context:** [what situation prompted this decision]

Use this ADR format:

# ADR-[NUMBER]: [Title]

## Status
[Proposed / Accepted / Deprecated / Superseded by ADR-X]

## Context
What is the issue that we're seeing that motivates this decision?
What are the constraints? (Team size, timeline, budget, existing tech)

## Decision Drivers
- [driver 1: e.g., "Must support 10,000 concurrent users"]
- [driver 2: e.g., "Team has no experience with Go"]
- [driver 3: e.g., "Must integrate with existing PostgreSQL database"]

## Considered Options
1. [Option A]: [brief description]
2. [Option B]: [brief description]
3. [Option C]: [brief description]

## Decision
We chose [Option X].

## Rationale
Why this option over the others. Address each decision driver.
Be honest about tradeoffs -- what do we lose with this choice?

## Consequences

### Positive
- [consequence 1]

### Negative
- [consequence 1]

### Risks
- [risk 1 and mitigation]

## Follow-up Actions
- [ ] [action needed to implement this decision]

---

Be intellectually honest. If we're choosing the easier option over the technically superior one,
say so and explain why that's the right call for our situation.
The goal is that a developer reading this in 2 years understands not just WHAT we chose,
but WHY, and under what conditions we'd reconsider.
```
