# Node.js API (Express/Fastify) — Cursor Rules

You are an expert Node.js backend developer building production REST APIs.

## Project Structure

```
src/
  routes/             # Route definitions grouped by resource
    users.ts          # GET /users, POST /users, etc.
    orders.ts
  handlers/           # Request handlers (business logic entry points)
    HandleCreateUser.ts
    HandleListOrders.ts
  services/           # Business logic, orchestration
    UserService.ts
    OrderService.ts
  repositories/       # Data access layer (DB queries)
    UserRepository.ts
  middleware/          # Auth, validation, error handling, rate limiting
  lib/                # Shared utilities, constants, helpers
  types/              # Shared TypeScript types
  db/                 # Database client, migrations, seeds
    migrations/
    schema.ts
  config/             # Environment config with validation
    env.ts
  server.ts           # App bootstrap (listen, middleware registration)
```

## Naming Conventions

- Handlers use the `Handle` prefix: `HandleCreateUser`, `HandleGetOrderById`.
- Services use the `Service` suffix: `UserService`, `PaymentService`.
- Repositories use the `Repository` suffix: `UserRepository`.
- Middleware functions are camelCase verbs: `requireAuth`, `validateBody`, `rateLimit`.
- Route files are lowercase plural nouns: `users.ts`, `orders.ts`.
- Environment variables: `SCREAMING_SNAKE_CASE`. Access only through validated config, never raw `process.env`.

## Request Handling Pattern

Every route follows this pipeline:
```
Route -> Middleware (auth, validation) -> Handler -> Service -> Repository -> Response
```

Handlers are thin. They parse the validated request, call a service, and format the response:
```ts
export async function HandleCreateUser(req: Request, res: Response) {
  const body = req.validated.body; // Already validated by middleware
  const user = await userService.create(body);
  res.status(201).json({ data: user });
}
```

Services contain business logic and are framework-agnostic (no req/res). This makes them testable without HTTP.

## Input Validation

- Validate ALL input at the boundary using `zod` schemas in middleware.
- Define request schemas per-route: `CreateUserSchema`, `UpdateOrderSchema`.
- Validate body, params, and query separately. Reject unknown keys with `.strict()`.
- Never trust client input downstream. By the time it reaches a handler, it must be parsed and typed.
- Coerce types explicitly: `z.coerce.number()` for query params that arrive as strings.

```ts
const CreateUserSchema = z.object({
  body: z.object({
    email: z.string().email().max(255),
    name: z.string().min(1).max(100).trim(),
  }).strict(),
});
```

## Error Handling

- Define a custom `AppError` class with `statusCode`, `code`, and `message` fields.
- Throw `AppError` in services. Never throw raw `Error` for expected failures.
- Use a centralized error-handling middleware as the LAST middleware. It catches `AppError` and formats a consistent JSON response.
- Unexpected errors (not `AppError`) return 500 with a generic message. Log the full error server-side.
- Never expose stack traces, internal paths, or database error details to the client.
- Always use `next(error)` in async handlers. Use an async wrapper or Express 5+ which handles rejected promises.

```ts
// Async wrapper for Express 4
const asyncHandler = (fn: RequestHandler) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);
```

Error response shape (always consistent):
```json
{ "error": { "code": "USER_NOT_FOUND", "message": "User with this ID does not exist" } }
```

## Database

- Use parameterized queries or an ORM/query builder (Drizzle, Prisma, Knex). Never concatenate SQL strings.
- Transactions: wrap multi-step mutations in a transaction. If step 3 fails, steps 1-2 must roll back.
- Connection pooling: configure pool size based on environment (5 for dev, 20-50 for production).
- Migrations are numbered and forward-only in production. Never edit a migration that has been deployed.
- Soft delete by default: add `deleted_at` timestamp column. Hard delete only for GDPR/compliance.
- Indexes: every column used in WHERE, JOIN, or ORDER BY must have an index. No exceptions.

## Authentication and Authorization

- Use JWT in httpOnly cookies for session auth, NOT in Authorization headers from browsers.
- JWT must contain minimal claims: `sub` (user ID), `exp`, `iat`. Never store sensitive data in JWT.
- Refresh token rotation: issue a new refresh token on each use, invalidate the old one.
- Authorization: check permissions in middleware, not in handlers. `requireRole('admin')` pattern.
- Rate limit auth endpoints aggressively: 5 attempts per minute per IP for login.

## Anti-Patterns to Avoid

BAD: Fat handlers with business logic
```ts
app.post('/users', async (req, res) => {
  const user = await db.query('INSERT INTO users...');
  await sendEmail(user.email);
  await createStripeCustomer(user);
  res.json(user);
});
```
GOOD: Thin handler delegating to a service

BAD: Catching errors and returning 200
```ts
try { ... } catch (e) { res.json({ success: false }); }
```
GOOD: Let errors propagate to the centralized handler with proper status codes

BAD: Using `any` for request bodies
GOOD: `req.validated.body` typed by the zod schema inference

BAD: Logging with `console.log`
GOOD: Use a structured logger (pino) with request ID correlation

## Logging

- Use `pino` for structured JSON logging.
- Every request gets a unique `requestId` (UUID) injected by middleware. Include it in all log entries.
- Log levels: `error` for failures, `warn` for degraded states, `info` for business events (user created, payment processed), `debug` for development.
- Never log sensitive data: passwords, tokens, full credit card numbers, PII beyond user IDs.
- Log at service boundaries: when calling external APIs, log the request (method, URL) and response (status, duration).

## Testing

- Framework: Vitest for unit tests, Supertest for integration/HTTP tests.
- Unit test services in isolation — mock the repository layer.
- Integration tests spin up the real app and hit endpoints via Supertest. Use a test database.
- Test file naming: `UserService.test.ts`, `users.integration.test.ts`.
- Test the error paths: invalid input returns 400, missing resource returns 404, unauthorized returns 401.
- Database tests: use transactions that roll back after each test, or truncate tables in `beforeEach`.
- Never mock the framework (Express/Fastify internals). Test through HTTP.

## Import Order

```ts
// 1. Node built-ins
import { randomUUID } from 'node:crypto';
import path from 'node:path';

// 2. Third-party packages
import { z } from 'zod';
import pino from 'pino';

// 3. Internal modules (absolute aliases)
import { UserService } from '@/services/UserService';
import { db } from '@/db/client';

// 4. Relative imports
import { CreateUserSchema } from './schemas';

// 5. Type-only imports
import type { User } from '@/types';
```

Always use the `node:` prefix for built-in modules.

## Security

- Helmet middleware: always enabled for security headers.
- CORS: configure explicitly per environment. Never use `origin: '*'` in production.
- Rate limiting: apply globally (100 req/min) and per-endpoint for sensitive routes.
- Input size limits: `express.json({ limit: '100kb' })`. Reject oversized payloads.
- SQL injection: parameterized queries only. If you write raw SQL, every variable MUST be a parameter.
- Dependency security: run `npm audit` in CI. Block deploys on critical vulnerabilities.
- Secrets: load from environment variables via validated config. Never hardcode.

## Performance

- Use streaming responses for large datasets: `res.write()` chunks instead of building a full array.
- Pagination: cursor-based for large tables, offset for small ones. Always enforce a `limit` max (100).
- Caching: use `Cache-Control` headers for GET endpoints. Redis for application-level caching.
- Avoid N+1 queries: use JOINs or batch loading (DataLoader pattern) for related data.
- Timeouts: set request timeouts (30s), database query timeouts (5s), and external API call timeouts (10s).
- Health check endpoint: `GET /health` returns 200 with uptime, db connectivity, and version.

## Configuration

Validate ALL environment variables at startup with zod. Crash immediately if config is invalid:
```ts
const EnvSchema = z.object({
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  NODE_ENV: z.enum(['development', 'production', 'test']),
});

export const env = EnvSchema.parse(process.env);
```
