# Zero to Deployed SaaS — Sample (Phases 1-2 of 7)

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

**This is a sample.** The full workflow continues with 5 more phases:

- **Phase 3: Core Features** -- Dashboard, CRUD interfaces, and the "aha moment" feature
- **Phase 4: Billing** -- Stripe integration, usage limits, and upgrade prompts
- **Phase 5: Landing Page** -- High-converting marketing page with pricing
- **Phase 6: Production Readiness** -- Error handling, SEO, performance, and deployment
- **Phase 7: Launch Prep** -- Analytics, monitoring, and final review checklist

Plus a Post-Launch Checklist and a "What You Do NOT Need at Launch" guide that will save you from over-engineering.

Get the complete workflow at: **https://m3phist0s.github.io/promptkit/**
