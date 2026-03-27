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

---

**This is a sample.** The full CLAUDE.md SaaS template includes Error Handling patterns, Testing strategies, Security Requirements, Performance Guidelines, Common Pitfalls, and Git/PR Workflow conventions.

Get the complete template and 11 more CLAUDE.md files at: **https://m3phist0s.github.io/promptkit/**
