# [Project Name] — SaaS Application

## Project Overview

<!-- Replace this block with your project description -->
[One paragraph describing what this SaaS does, who it serves, and why it exists.]

## Tech Stack

- **Frontend:** React 18+ with TypeScript, Vite, TanStack Query, Zustand
- **Backend:** Node.js (Express/Fastify) or Python (FastAPI)
- **Database:** PostgreSQL via Prisma ORM (or SQLAlchemy)
- **Auth:** [Clerk / Auth.js / Supabase Auth / custom JWT]
- **Billing:** Stripe (Checkout + Customer Portal + Webhooks)
- **Hosting:** [Vercel / Railway / Fly.io]
- **Email:** [Resend / Postmark / SES]

## File Structure

```
src/
  app/                  # Route pages (if Next.js) or top-level views
  components/
    ui/                 # Primitives: Button, Input, Modal, Badge
    features/           # Domain components: InvoiceTable, PlanSelector
    layouts/            # Shell, Sidebar, DashboardLayout
  hooks/                # Custom hooks — one file per hook
  lib/                  # Pure utilities, no side effects
    api-client.ts       # Typed API caller, wraps fetch
    constants.ts        # App-wide constants, no magic strings
    format.ts           # Date, currency, number formatters
  server/
    routes/             # Route handlers grouped by domain
    middleware/          # Auth, rate limit, error handler, tenant
    services/           # Business logic — no HTTP concepts here
    db/
      schema/           # Database schema definitions
      migrations/       # Ordered migration files
      seed.ts           # Dev seed data
  types/                # Shared TypeScript types and Zod schemas
```

**Rules:**
- One component per file. File name matches the default export.
- Co-locate tests next to source: `UserCard.tsx` + `UserCard.test.tsx`.
- No barrel files (`index.ts` re-exports). Import directly.
- Server code never imports from `components/`. Client code never imports from `server/`.

## Code Style and Conventions

### General
- Use early returns to avoid nesting. No `else` after `return`.
- Prefer `const` arrow functions for React components, named functions for utilities.
- No `any`. If you genuinely cannot type it, use `unknown` and narrow.
- Prefer composition over inheritance. No class components.
- Colocate related logic. A 200-line file with clear sections beats 8 tiny files.
- No comments that restate code. Comments explain WHY, not WHAT.

### Naming
- Components: `PascalCase` — `InvoiceRow.tsx`
- Hooks: `camelCase` with `use` prefix — `useSubscription.ts`
- Utilities: `camelCase` — `formatCurrency.ts`
- Types/interfaces: `PascalCase`, no `I` prefix — `User`, not `IUser`
- Database columns: `snake_case`. TypeScript properties: `camelCase`. Map at the ORM layer.
- Boolean variables: `is`/`has`/`should` prefix — `isLoading`, `hasAccess`

### React Patterns
- Hooks at the top of the component, no conditional hooks.
- Derive state from existing state. If you can compute it, do not store it.
- Use `useMemo`/`useCallback` only when you measure a performance problem, not by default.
- Controlled forms for simple cases, react-hook-form for anything with validation.
- Lift state up only when two siblings need it. Otherwise, keep it local.

### API Layer
- All API responses follow: `{ data: T }` for success, `{ error: { code: string, message: string } }` for failure.
- Use Zod schemas to validate request bodies at the boundary. Never trust client input.
- Service functions are pure business logic. They receive typed inputs and return typed outputs. No `req`/`res` objects.
- Database queries live in service functions, not in route handlers.

## Authentication and Authorization

- Auth middleware runs on every protected route. No "forgetting" to add auth.
- Store user ID and tenant ID on `req.user` after auth middleware.
- Authorization is separate from authentication. Check permissions explicitly:
  ```typescript
  // Good: explicit permission check
  if (invoice.tenantId !== req.user.tenantId) throw new ForbiddenError();

  // Bad: assuming auth means authorized
  const invoice = await getInvoice(req.params.id); // anyone can access?
  ```
- Never expose internal IDs in URLs without ownership checks.
- Rate limit auth endpoints: 5 attempts per minute per IP.

## Billing and Subscriptions

- Stripe is the source of truth for subscription state, not your database.
- Sync subscription status via webhooks (`customer.subscription.updated`, `customer.subscription.deleted`).
- Always verify webhook signatures. No exceptions.
- Gate features by checking `subscription.status === 'active'` and plan tier.
- Handle these states: `active`, `past_due`, `canceled`, `trialing`, `unpaid`.
- Free tier users still get a Stripe customer record for upgrade path.
- Never store card details. Use Stripe Checkout and Customer Portal.

## Error Handling

- Use typed error classes that extend a base `AppError`:
  ```typescript
  class AppError extends Error {
    constructor(public code: string, message: string, public status: number) {
      super(message);
    }
  }
  class NotFoundError extends AppError {
    constructor(resource: string) { super('NOT_FOUND', `${resource} not found`, 404); }
  }
  ```
- Catch errors in a single global error handler middleware. Do not try/catch in every route.
- Log errors with context: user ID, tenant ID, request ID, stack trace.
- Return user-friendly messages. Never leak stack traces or SQL errors to the client.
- On the frontend, show toast notifications for recoverable errors. Show error boundaries for crashes.

## Testing

- **Unit tests** for service functions and utilities. No mocking unless hitting external APIs.
- **Integration tests** for API routes. Use a real test database, not mocks.
- **Component tests** with Testing Library. Test behavior, not implementation.

```typescript
// Good: tests what the user sees
test('shows upgrade prompt when on free plan', async () => {
  render(<BillingPage plan="free" />);
  expect(screen.getByText('Upgrade to Pro')).toBeInTheDocument();
});

// Bad: tests implementation details
test('calls setPlan with correct value', () => {
  // This tests the wiring, not the behavior
});
```

- Seed data goes in `db/seed.ts`. Tests use factory functions, not raw SQL.
- Test the unhappy paths: expired tokens, missing permissions, invalid input, network failures.
- Minimum coverage: 80% on services, 60% on components. 0% coverage on types and constants is fine.

## Security Requirements

- Sanitize all user-generated content before rendering. Use DOMPurify if rendering HTML.
- CSRF protection on all state-changing endpoints.
- Set security headers: `Content-Security-Policy`, `X-Frame-Options`, `Strict-Transport-Security`.
- Environment variables for all secrets. Never hardcode API keys.
- SQL injection is handled by the ORM. If writing raw SQL, use parameterized queries only.
- File uploads: validate MIME type server-side, limit size to 10MB, store in S3/R2 not local disk.
- Tenant isolation: every database query for tenant data must include `WHERE tenant_id = ?`.

## Performance Guidelines

- Paginate all list endpoints. Default 20, max 100 items per page.
- Add database indexes on columns used in WHERE and ORDER BY. Check with `EXPLAIN ANALYZE`.
- Use `select` in ORM queries to fetch only needed columns on list views.
- Lazy load heavy components (charts, editors, file viewers) with `React.lazy`.
- Images: use `next/image` or serve through a CDN with width parameters.
- API responses under 200ms for reads, under 500ms for writes. Measure with middleware.
- N+1 queries: use `include`/`joinRelation` in ORM. Never query in a loop.
- Cache expensive computations (dashboard aggregations) in Redis or in-memory with TTL.

## Common Pitfalls

1. **Do not split into microservices.** One deployable unit until you have 50+ engineers or clear scaling bottlenecks.
2. **Do not build your own auth.** Use an auth provider or a well-tested library (lucia, arctic).
3. **Do not store subscription status only in your DB.** Webhooks fail. Always allow re-sync from Stripe.
4. **Do not use `useEffect` for data fetching.** Use TanStack Query or SWR.
5. **Do not put business logic in API route handlers.** Extract to service functions for testability.
6. **Do not create God components.** If a component file exceeds 300 lines, split by responsibility.
7. **Do not skip database migrations.** Never modify a deployed migration. Create a new one.
8. **Do not use floating-point for money.** Use integers (cents) or a Decimal library.

## Git and PR Workflow

### Commit Messages
```
feat(billing): add annual plan discount toggle
fix(auth): handle expired refresh token redirect loop
chore(deps): upgrade prisma to 5.x
refactor(invoices): extract PDF generation to service
```
Format: `type(scope): lowercase imperative description`
Types: `feat`, `fix`, `chore`, `refactor`, `test`, `docs`, `perf`

### PR Checklist
- [ ] Zod validation on all new API inputs
- [ ] Auth and permission checks on new endpoints
- [ ] Tenant isolation preserved (no cross-tenant data leaks)
- [ ] Database migration is reversible
- [ ] Error states handled in UI (loading, empty, error, forbidden)
- [ ] No `console.log` left in production code (use structured logger)
- [ ] Tests cover the happy path and at least one error path
- [ ] No secrets or credentials in the diff
- [ ] API response shapes match the documented contract
- [ ] Stripe webhook handler tested with Stripe CLI
