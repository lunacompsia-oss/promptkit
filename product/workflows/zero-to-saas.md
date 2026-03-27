# Zero to Deployed SaaS

A step-by-step workflow for building and deploying a complete SaaS application using Claude Code. From an empty directory to a production deployment with authentication, billing, and a landing page.

**Time estimate:** 4-8 hours for a skilled developer following this workflow.
**Prerequisites:** Node.js 18+, a GitHub account, a Stripe account, a deployment target (Vercel, Railway, or Fly.io).

---

## Phase 1: Foundation (30 minutes)

### Step 1.1 -- Scaffold the Project

**When to use:** You have a validated idea and are ready to build.

```
I'm building a SaaS called [PRODUCT NAME] that [ONE SENTENCE DESCRIPTION].

Target users: [WHO]
Core value: [WHAT PROBLEM IT SOLVES]

Set up a new project with:
- Next.js 14+ with App Router
- TypeScript (strict mode)
- Tailwind CSS v4
- PostgreSQL with Drizzle ORM
- Project structure following convention: app/, lib/, components/, db/

Create the initial project structure. Do NOT add authentication or billing yet -- just the skeleton.
Include a .env.example with all required variables documented.
```

### Step 1.2 -- Database Schema Design

**When to use:** Immediately after scaffolding.

```
Design the database schema for [PRODUCT NAME].

Business context: [2-3 sentences about what the product does]

Requirements:
- Multi-tenant: each user belongs to an organization
- Core entities: [LIST YOUR DOMAIN ENTITIES, e.g., "projects, tasks, comments"]
- Soft deletes on all user-facing entities
- Created/updated timestamps on everything
- Use UUIDs for public-facing IDs, auto-increment integers for internal references

Create Drizzle ORM schema files in db/schema/. One file per logical domain.
Include indexes for any column that will be filtered or sorted frequently.
Add a seed file at db/seed.ts with realistic sample data.
```

### Step 1.3 -- Environment and Dev Tooling

```
Set up the development environment:

1. Add a docker-compose.yml with PostgreSQL 16 for local dev
2. Add drizzle-kit config for migrations
3. Add these npm scripts:
   - db:push (apply schema changes)
   - db:seed (seed sample data)
   - db:studio (open Drizzle Studio)
   - dev (start everything needed for development)
4. Add Biome for linting and formatting (NOT ESLint -- keep it simple)
5. Add a .nvmrc pinning the Node version

Do NOT add husky, lint-staged, commitlint, or any other git hook tools.
Keep the dev setup minimal -- I want to run one command and start coding.
```

---

## Phase 2: Authentication (45 minutes)

### Step 2.1 -- Auth Setup

```
Add authentication to the project using NextAuth.js v5 (Auth.js).

Requirements:
- Email/password login with bcrypt hashing
- Google OAuth as a social provider
- Session strategy: JWT stored in httpOnly cookies
- Middleware that protects all routes under /app/*
- Public routes: /, /login, /signup, /pricing, /api/webhooks/*
- After login, redirect to /app/dashboard

Create these files:
- lib/auth.ts (auth configuration)
- app/login/page.tsx (login form)
- app/signup/page.tsx (signup form)
- middleware.ts (route protection)

Use server actions for the login/signup form submissions -- no API routes needed.
Style the forms with Tailwind. Keep them clean and minimal -- centered card layout.
```

### Step 2.2 -- User Onboarding Flow

```
Add a post-signup onboarding flow.

After a new user signs up, redirect them to /app/onboarding with these steps:
1. "What's your name?" (update profile)
2. "Create your first [CORE ENTITY]" (so they immediately see value)

Requirements:
- Multi-step form with progress indicator
- Each step is a server action that saves to the database
- After completing onboarding, set a `onboarded_at` timestamp on the user
- If a user who hasn't onboarded tries to access /app/*, redirect to onboarding
- Keep it to 2 steps maximum -- every extra step loses users
```

---

## Phase 3: Core Features (2-3 hours)

### Step 3.1 -- Dashboard and Layout

```
Build the authenticated app layout and dashboard.

Layout (/app/layout.tsx for the /app/* routes):
- Left sidebar with navigation (collapsible on mobile)
- Top bar with user avatar dropdown (settings, logout)
- Main content area with max-width constraint
- Responsive: sidebar becomes bottom nav on mobile

Dashboard (/app/dashboard/page.tsx):
- Welcome message with user's name
- Quick stats cards showing [RELEVANT METRICS FOR YOUR APP]
- Recent activity feed
- "Quick action" button for the most common task

Use server components by default. Only add "use client" where interactivity requires it.
Fetch data with async server components -- no useEffect/fetch pattern.
```

### Step 3.2 -- CRUD for Core Entity

```
Build the complete CRUD interface for [CORE ENTITY, e.g., "projects"].

Requirements:
- List view at /app/[entities]/page.tsx
  - Table layout on desktop, card layout on mobile
  - Search/filter functionality
  - Pagination (20 items per page)
  - Sort by name, created date, status
- Detail view at /app/[entities]/[id]/page.tsx
  - All fields displayed in a clean layout
  - Edit inline or via modal
- Create form at /app/[entities]/new/page.tsx
  - Server action for form submission
  - Validation with Zod schemas (shared between client and server)
  - Redirect to detail view after creation
- Delete with confirmation dialog

Use server actions for all mutations.
Use Zod schemas in a shared lib/validators/ directory.
Add optimistic UI updates where it improves perceived performance.
Add loading.tsx skeletons for each page.
```

### Step 3.3 -- The "Aha Moment" Feature

```
Build the feature that delivers the core value of [PRODUCT NAME].

[DESCRIBE THE SPECIFIC FEATURE IN DETAIL -- 3-5 sentences about what it does,
what the user inputs, what they get back, and why it's valuable]

Technical requirements:
- [LIST SPECIFIC TECHNICAL NEEDS -- e.g., "Calls OpenAI API", "Processes CSV files", "Generates PDF reports"]
- Show a progress indicator for any operation taking more than 500ms
- Handle errors gracefully -- show what went wrong and how to fix it
- Cache results where appropriate to avoid redundant processing

This is the feature that makes users say "wow, I need this."
Make it work perfectly before making it pretty.
```

---

## Phase 4: Billing (1 hour)

### Step 4.1 -- Stripe Integration

```
Add Stripe billing with the following plan structure:

Plans:
- Free: [LIMITS -- e.g., "3 projects, 100 tasks"]
- Pro ($19/month): [LIMITS -- e.g., "unlimited projects, 10,000 tasks"]
- Team ($49/month): [LIMITS -- e.g., "everything in Pro + team features, 5 seats"]

Implementation:
1. Create a pricing page at /pricing with a toggle for monthly/annual (20% discount for annual)
2. Integrate Stripe Checkout for subscription creation
3. Add a webhook handler at /api/webhooks/stripe that handles:
   - checkout.session.completed
   - customer.subscription.updated
   - customer.subscription.deleted
   - invoice.payment_failed
4. Store subscription status in the database (plan, status, current_period_end)
5. Add a lib/billing.ts with helper functions:
   - getCurrentPlan(userId)
   - canAccess(userId, feature)
   - getRemainingUsage(userId, resource)
6. Add a billing settings page at /app/settings/billing

Use Stripe's test mode. Include the test clock setup for testing renewals.
Gate features using the canAccess helper -- check at the server action level, not just UI.
```

### Step 4.2 -- Usage Limits and Upgrade Prompts

```
Implement usage tracking and upgrade prompts.

Requirements:
1. Track usage of limited resources in the database:
   - [RESOURCE 1, e.g., "number of projects created"]
   - [RESOURCE 2, e.g., "number of API calls this month"]
2. When a user hits 80% of their limit, show a subtle banner: "You've used X of Y [resources]. Upgrade for more."
3. When a user hits 100%, block the action with a friendly message and upgrade button
4. Reset monthly counters on billing cycle renewal (use the Stripe webhook)
5. Add a usage display in the dashboard sidebar showing current consumption

The upgrade flow should be frictionless: one click from the limit message to Stripe Checkout.
Never show an error without a clear path to resolution.
```

---

## Phase 5: Landing Page (45 minutes)

### Step 5.1 -- Marketing Landing Page

```
Build a high-converting landing page at the root route (/).

Sections (in order):
1. Hero: headline, subheadline, CTA button, product screenshot/mockup
2. Social proof: "Trusted by X developers" or logos (use placeholder logos for now)
3. Problem/Solution: 3 pain points with your solution to each
4. Features: 3-4 key features with icons and descriptions
5. How it works: 3 steps with illustrations
6. Pricing: embed the pricing cards from /pricing
7. FAQ: 5-6 common questions
8. Final CTA: repeat the main call-to-action

Design requirements:
- Clean, modern SaaS aesthetic
- Dark/light mode support
- Performance: no client-side JS for the landing page (pure server component)
- Mobile-first responsive design
- Use next/image for all images with proper sizing
- Above-the-fold content should load with zero layout shift

Copy should focus on the outcome the user gets, not the features you built.
Use specific numbers where possible ("Save 4 hours per week" not "Save time").
```

---

## Phase 6: Production Readiness (1 hour)

### Step 6.1 -- Error Handling and Edge Cases

```
Audit the entire application for error handling and edge cases.

Check and fix:
1. All server actions: wrap in try/catch, return typed error responses
2. All database queries: handle connection failures gracefully
3. All external API calls: add timeouts (10s default), retry logic (3 attempts with exponential backoff)
4. Form validation: both client-side (immediate feedback) and server-side (security)
5. Auth edge cases: expired sessions, deleted users, concurrent logins
6. Billing edge cases: failed payments, plan downgrades, cancelled subscriptions
7. Rate limiting: add rate limits to auth endpoints (5 attempts/minute) and API routes

Add a global error boundary at app/error.tsx that:
- Logs the error (console for now, we'll add Sentry later)
- Shows a friendly message to the user
- Offers a "Try again" button

Add not-found.tsx pages for 404 handling.
```

### Step 6.2 -- SEO and Performance

```
Optimize for SEO and performance.

SEO:
1. Add metadata to all public pages using Next.js Metadata API
2. Create app/sitemap.ts that generates a dynamic sitemap
3. Create app/robots.ts with appropriate rules
4. Add Open Graph images for social sharing (use next/og)
5. Add JSON-LD structured data for the landing page (Organization, SoftwareApplication)

Performance:
1. Audit all images -- ensure proper sizes, formats (WebP), and lazy loading
2. Add appropriate cache headers to static assets
3. Use React.lazy() for heavy components not needed on initial load
4. Ensure no layout shift (check all dynamic content has reserved space)
5. Add preconnect hints for external domains (Stripe, analytics)

Run through the checklist and fix everything. Show me a summary of changes made.
```

### Step 6.3 -- Deployment

```
Prepare for production deployment to [Vercel / Railway / Fly.io].

1. Create a production-ready Dockerfile (if using Railway/Fly) or verify vercel.json (if Vercel)
2. Set up the production database:
   - Connection pooling configuration
   - SSL mode enabled
   - Migration strategy (drizzle-kit push for now, migrate for later)
3. Create a deployment checklist document at DEPLOY.md covering:
   - Required environment variables (every single one)
   - Stripe webhook endpoint configuration
   - DNS setup instructions
   - Post-deployment verification steps
4. Add a health check endpoint at /api/health that checks:
   - Database connectivity
   - Stripe API key validity
   - Returns version number from package.json
5. Set up GitHub Actions CI:
   - Type checking (tsc --noEmit)
   - Lint check (biome check)
   - Build verification
   - Auto-deploy to production on push to main

Keep the CI pipeline under 3 minutes. Fast feedback loops matter.
```

---

## Phase 7: Launch Prep (30 minutes)

### Step 7.1 -- Analytics and Monitoring

```
Add essential analytics and monitoring. Nothing fancy -- just what we need to know if things are working.

1. Add Plausible Analytics (privacy-friendly, no cookie banner needed)
   - Track page views automatically
   - Add custom events for: signup, login, subscription_created, [CORE_ACTION]
2. Add a simple error tracking setup:
   - Create lib/logger.ts that logs errors with context (userId, action, metadata)
   - In production, these should go to stdout (the platform will capture them)
   - Add error tracking to all server actions using a wrapper function
3. Add uptime monitoring:
   - The /api/health endpoint from earlier is the target
   - Document how to set up BetterUptime or similar free monitoring

Do NOT add: Google Analytics, Mixpanel, Segment, Amplitude, or any heavy tracking SDK.
We need to know three things: are users signing up, are they using the product, is it broken.
```

### Step 7.2 -- Final Review

```
Do a final review of the entire codebase before launch.

Check:
1. Security: no secrets in code, all env vars documented, CSRF protection active
2. No console.log statements left in production code
3. All TODO comments are either resolved or tracked as GitHub issues
4. README.md has setup instructions that a new developer can follow
5. package.json has correct name, version, description
6. License file exists
7. .gitignore covers: .env*, node_modules, .next, *.local
8. No unused dependencies in package.json
9. TypeScript strict mode passes with zero errors
10. All pages have proper loading and error states

Create GitHub issues for anything that should be improved post-launch but isn't blocking.
List them in a LAUNCH_TODOS.md file.
```

---

## Post-Launch Checklist

After deploying, verify these manually:

- [ ] Landing page loads in under 2 seconds
- [ ] Signup flow works end-to-end
- [ ] Stripe test payment succeeds
- [ ] Webhook receives events from Stripe
- [ ] Protected routes redirect to login when not authenticated
- [ ] Mobile layout looks correct on a real phone
- [ ] Open Graph preview looks correct (use https://opengraph.xyz)
- [ ] Health check endpoint returns 200
- [ ] Error boundary triggers on a forced error (add `?error=test` to a page)

---

## What You Do NOT Need at Launch

Stop yourself from adding these before you have paying users:

- Admin dashboard (use Drizzle Studio or direct database access)
- Email templates (transactional emails can wait -- users will find you)
- i18n / localization (ship in one language, expand when demand appears)
- GraphQL (REST or server actions are fine)
- Redis caching layer (PostgreSQL is fast enough)
- Kubernetes (a single container is fine)
- Feature flags service (use environment variables)
- A/B testing framework (you don't have enough traffic yet)
