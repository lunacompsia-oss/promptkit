# [Project Name] — API Service

## Project Overview

<!-- Replace this block with your project description -->
[One paragraph describing what this API does, what clients consume it, and the domain it serves.]

## Tech Stack

- **Runtime:** Node.js (Fastify/Express) or Python (FastAPI/Django REST)
- **Database:** PostgreSQL (primary), Redis (cache/rate limiting)
- **ORM:** Prisma / SQLAlchemy / Drizzle
- **Auth:** API keys + JWT bearer tokens
- **Docs:** OpenAPI 3.1 (auto-generated from route definitions)
- **Hosting:** [Railway / Fly.io / AWS ECS / Cloudflare Workers]
- **Monitoring:** [Sentry / Datadog / Axiom]

## File Structure

```
src/
  routes/
    v1/
      users.ts            # Route definitions for /v1/users
      users.schema.ts     # Zod/Pydantic schemas for request/response
      users.service.ts    # Business logic, no HTTP concepts
      users.test.ts       # Integration tests for this resource
    v2/                   # Breaking changes go in new version
  middleware/
    auth.ts               # API key + JWT verification
    rate-limit.ts         # Per-key rate limiting
    request-id.ts         # Attach unique ID to every request
    error-handler.ts      # Global error catcher, formats response
    validate.ts           # Generic schema validation middleware
  lib/
    db.ts                 # Database client singleton
    cache.ts              # Redis wrapper with typed get/set
    logger.ts             # Structured JSON logger
    errors.ts             # Error class hierarchy
    pagination.ts         # Cursor and offset pagination helpers
  types/
    api.ts                # Shared API types
  scripts/
    migrate.ts            # Run migrations
    seed.ts               # Seed development data
    generate-openapi.ts   # Generate OpenAPI spec from schemas
```

**Rules:**
- Group by resource, not by layer. `users.ts`, `users.schema.ts`, `users.service.ts` live together.
- One resource per file. Do not combine `/users` and `/teams` in one router.
- Service functions never import `Request`/`Response`. They take typed args, return typed results.
- Every route has a corresponding `.schema.ts` with request and response schemas.

## API Design Conventions

### URL Structure
```
GET    /v1/resources            # List (paginated)
POST   /v1/resources            # Create
GET    /v1/resources/:id        # Read
PATCH  /v1/resources/:id        # Partial update
DELETE /v1/resources/:id        # Delete (soft delete by default)
POST   /v1/resources/:id/action # RPC-style for non-CRUD operations
```

- Use plural nouns for resources: `/users`, not `/user`.
- Nest only one level deep: `/users/:id/invoices`, never `/users/:id/invoices/:invoiceId/items`.
  For deeper access, promote to top-level: `/invoice-items/:id`.
- Query parameters for filtering: `GET /v1/users?status=active&plan=pro`.
- No verbs in URLs. Use HTTP methods. Exception: RPC actions like `/users/:id/deactivate`.

### Response Envelope
```json
// Success (single resource)
{ "data": { "id": "usr_abc", "email": "a@b.com" } }

// Success (list)
{ "data": [...], "pagination": { "next_cursor": "abc", "has_more": true } }

// Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [{ "field": "email", "issue": "required" }]
  }
}
```

- Always wrap in `data` or `error`. Never return a bare array or object.
- Use `snake_case` for all JSON keys. Consistent with PostgreSQL and most API conventions.
- Include `request_id` in error responses for support debugging.

### Status Codes
| Code | When |
|------|------|
| 200 | Successful read or update |
| 201 | Successful create (include `Location` header) |
| 204 | Successful delete (no body) |
| 400 | Validation error, malformed request |
| 401 | Missing or invalid authentication |
| 403 | Authenticated but not authorized |
| 404 | Resource does not exist |
| 409 | Conflict (duplicate key, state conflict) |
| 422 | Business logic rejection (valid syntax, invalid semantics) |
| 429 | Rate limit exceeded (include `Retry-After` header) |
| 500 | Unexpected server error |

## Authentication

### API Keys
- Prefix all keys: `pk_live_`, `pk_test_`, `sk_live_`, `sk_test_`.
- Store only the SHA-256 hash in the database. The raw key is shown once at creation.
- Pass via `Authorization: Bearer sk_live_...` header. Never in query params.
- Keys have scopes: `read`, `write`, `admin`. Check scopes in middleware.
- Support key rotation: allow two active keys per account simultaneously.

### JWT (for user-facing clients)
- Short-lived access tokens (15 minutes). Long-lived refresh tokens (30 days) in httpOnly cookie.
- Include `sub` (user ID), `tid` (tenant ID), `scopes` in JWT payload. Nothing else.
- Validate `iss` and `aud` claims. Reject tokens from other services.

## Rate Limiting

- Default: 100 requests/minute per API key.
- Higher tiers get higher limits. Store limits on the API key record.
- Use sliding window algorithm (Redis `ZRANGEBYSCORE`).
- Return headers on every response:
  ```
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 87
  X-RateLimit-Reset: 1710000000
  ```
- When exceeded, return 429 with `Retry-After` header in seconds.
- Separate rate limits for auth endpoints (stricter: 10/min per IP).

## Versioning

- Version in URL path: `/v1/`, `/v2/`. Not in headers.
- Non-breaking changes go into the current version: adding fields, adding endpoints.
- Breaking changes require a new version: removing fields, changing types, renaming endpoints.
- Support previous version for 12 months after new version launches.
- Version routers are separate modules. Do not use conditionals inside a single handler.

## Error Handling

```typescript
// Error hierarchy
class AppError extends Error {
  constructor(
    public code: string,
    message: string,
    public statusCode: number,
    public details?: unknown
  ) { super(message); }
}

class ValidationError extends AppError {
  constructor(details: { field: string; issue: string }[]) {
    super('VALIDATION_ERROR', 'Request validation failed', 400, details);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super('NOT_FOUND', `${resource} ${id} not found`, 404);
  }
}

class RateLimitError extends AppError {
  constructor(retryAfter: number) {
    super('RATE_LIMITED', 'Too many requests', 429, { retry_after: retryAfter });
  }
}
```

- Throw typed errors in services. The global error handler catches and formats them.
- Log 5xx errors with full context (request ID, user, stack). Log 4xx at `warn` level.
- Never leak internal details: database errors become generic 500s in production.
- Include `request_id` in every error response so users can reference it in support requests.

## Testing

- **Integration tests are primary.** Start the server, hit real endpoints, use a test database.
- **Unit tests for complex business logic only.** Parsers, calculators, validators.
- **No mocks for the database.** Use a real PostgreSQL instance in CI.

```typescript
// Integration test example
describe('POST /v1/users', () => {
  it('creates a user and returns 201', async () => {
    const res = await app.inject({
      method: 'POST',
      url: '/v1/users',
      headers: { authorization: `Bearer ${testApiKey}` },
      payload: { email: 'new@example.com', name: 'Test User' },
    });
    expect(res.statusCode).toBe(201);
    expect(res.json().data).toMatchObject({ email: 'new@example.com' });
    expect(res.headers.location).toMatch(/\/v1\/users\/usr_/);
  });

  it('returns 400 for missing email', async () => {
    const res = await app.inject({
      method: 'POST',
      url: '/v1/users',
      headers: { authorization: `Bearer ${testApiKey}` },
      payload: { name: 'No Email' },
    });
    expect(res.statusCode).toBe(400);
    expect(res.json().error.code).toBe('VALIDATION_ERROR');
  });

  it('returns 401 without auth header', async () => {
    const res = await app.inject({ method: 'POST', url: '/v1/users', payload: {} });
    expect(res.statusCode).toBe(401);
  });
});
```

- Test every status code your endpoint can return.
- Use factory functions for test data: `createTestUser()`, `createTestApiKey({ scopes: ['read'] })`.
- Reset database between test suites, not between individual tests (too slow).

## Security Requirements

- Input validation at the boundary with Zod/Pydantic. Reject unknown fields by default.
- SQL injection: always use parameterized queries via ORM. Audit any raw SQL.
- No mass assignment: explicitly pick allowed fields from the request body.
- CORS: whitelist specific origins. Never `Access-Control-Allow-Origin: *` in production.
- All responses include: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`.
- Webhook endpoints verify signatures before processing payloads.
- Log authentication failures for anomaly detection. Alert on 10+ failures per minute per IP.
- Soft delete by default. Hard delete only for GDPR/compliance, behind admin scope.

## Performance Guidelines

- Target p95 latency: 100ms for reads, 300ms for writes.
- Paginate everything. Default page size 20, max 100. Cursor-based for large datasets.
- Database indexes on every column used in WHERE, JOIN, or ORDER BY.
- Connection pooling: use PgBouncer or built-in pool. Max 20 connections per instance.
- Cache frequently-read, rarely-changed data in Redis with TTL (plan details, feature flags).
- Bulk endpoints for batch operations: `POST /v1/users/bulk` accepts up to 100 items.
- Use `SELECT ... FOR UPDATE` for operations requiring consistency (balance updates, inventory).
- Background jobs for anything over 5 seconds: PDF generation, email sending, data exports.

## Common Pitfalls

1. **Do not use UUIDs as the public-facing ID format.** Use prefixed IDs like `usr_`, `inv_`, `key_` for debuggability.
2. **Do not forget idempotency.** Accept an `Idempotency-Key` header on POST/PATCH. Store results for 24 hours.
3. **Do not return different shapes from the same endpoint.** The response schema is a contract.
4. **Do not log request bodies by default.** They may contain PII or secrets. Log selectively.
5. **Do not use `DELETE` for non-reversible operations without confirmation.** Use `POST /resources/:id/purge`.
6. **Do not skip pagination on internal endpoints.** That table with 50 rows today will have 5M next year.
7. **Do not version at the field level with conditionals.** It creates an untestable mess. Use separate versioned modules.
8. **Do not use 200 for everything.** Status codes exist for a reason. Clients depend on them.

## Database Conventions

- Table names: plural `snake_case` — `user_accounts`, `invoice_items`.
- Primary key: `id` as BIGINT auto-increment internally. Expose prefixed string externally.
- Timestamps: `created_at`, `updated_at` on every table. `deleted_at` for soft deletes.
- Foreign keys: `<singular_table>_id` — `user_id`, `invoice_id`.
- Enums: stored as `VARCHAR` with check constraint or app-level validation, not DB enums (hard to migrate).
- Migrations are append-only. Never modify a deployed migration.

## Git and PR Workflow

### Commit Messages
```
feat(users): add bulk import endpoint
fix(auth): reject expired API keys in rate limit check
perf(invoices): add composite index on tenant_id + created_at
docs(openapi): add webhook event schemas
```
Format: `type(scope): lowercase imperative description`

### PR Checklist
- [ ] Request and response schemas defined and validated
- [ ] Auth middleware applied (no open endpoints by accident)
- [ ] Rate limiting configured for new endpoints
- [ ] OpenAPI spec updated (auto-generated or manually)
- [ ] Integration tests cover success, validation error, auth error, and not-found cases
- [ ] No N+1 queries (check with query logging in tests)
- [ ] Pagination implemented on list endpoints
- [ ] Migration is additive and reversible
- [ ] No secrets, API keys, or PII in logs or error responses
- [ ] Idempotency supported on create/update endpoints
