# Test Suite from Scratch

A systematic workflow for generating comprehensive tests for an existing codebase that has little to no test coverage. Covers: identifying what to test first, writing integration tests that prove the system works, unit tests that verify business logic, and building a sustainable testing culture.

**Philosophy:** Integration tests first, unit tests second. A green integration test suite means your product works. A green unit test suite means your functions work. You need both, but the first gives you more confidence per test written.

**Time estimate:** 2-5 days for initial critical path coverage, then ongoing expansion.

---

## Phase 1: Test Infrastructure (2 hours)

### Step 1.1 -- Test Framework Setup

**When to use:** The codebase has no test setup, or the existing setup is broken.

```
Set up a modern test infrastructure for this codebase.

Current tech stack: [LANGUAGE/FRAMEWORK]

Requirements:
1. Test runner: choose the simplest option that works with our stack
   - JavaScript/TypeScript: Vitest (fast, ESM-native, compatible with Jest API)
   - Python: pytest
   - Ruby: Minitest (ships with Rails) or RSpec
   - Go: built-in testing package
2. Test database: configure a separate test database that resets between test suites
3. Test utilities file at [test/helpers.ts or equivalent]:
   - Database cleanup function (truncate all tables)
   - Factory functions for creating test data (use a factory pattern, NOT fixtures)
   - Authenticated request helper (create user, login, return auth token)
   - Common assertions (assertDatabaseHas, assertEmailSent, etc.)
4. CI integration: add a GitHub Actions workflow that runs tests on every PR
5. NPM/make scripts:
   - test (run all tests)
   - test:watch (run in watch mode for development)
   - test:coverage (run with coverage report)

Configuration:
- Parallel test execution enabled
- Timeout: 10 seconds per test (fail fast)
- Coverage thresholds: do NOT set them yet (we'll add them once we have baseline coverage)

Keep the setup minimal. We want to be writing tests within 15 minutes.
```

### Step 1.2 -- Factory Functions

```
Create factory functions for generating test data.

Review the database schema and create a factory for each model/table.

Requirements for each factory:
1. Sensible defaults for all required fields (a factory call with zero arguments should work)
2. Override any field via parameters
3. Auto-generate unique values where needed (use incrementing counters or random suffixes)
4. Handle relationships: creating a Post factory should auto-create a User if none is provided
5. Support traits/variants: factory.build("user", { trait: "admin" }) for common variations
6. Both "build" (in-memory only) and "create" (persisted to database) methods

Example pattern:
- userFactory.create() -> creates user with defaults, returns the user
- userFactory.create({ name: "Alice", role: "admin" }) -> creates admin user named Alice
- postFactory.create({ author: existingUser }) -> creates post linked to existing user

Create factories for these models: [LIST ALL YOUR MODELS]

Do NOT use random data libraries like faker.js for core fields.
Use predictable, readable values: "Test User 1", "test-1@example.com", etc.
Random data makes test failures harder to debug.
```

---

## Phase 2: Critical Path Tests (Day 1-2)

### Step 2.1 -- Identify Critical Paths

```
Analyze this codebase and identify the critical paths that need test coverage first.

A "critical path" is any code path where a bug would:
1. Lose revenue (payment processing, subscription management)
2. Lose data (create, update, delete operations on core entities)
3. Compromise security (authentication, authorization, input validation)
4. Block users entirely (signup, login, core product action)

For each critical path found, document:
- Description: what the user is trying to do
- Entry point: the route/function where it starts
- Key decision points: where the code branches based on conditions
- Side effects: database writes, emails sent, external API calls
- Error scenarios: what can go wrong and what should happen

Rank them by severity (revenue > data > security > blocking) and frequency of use.

Output a prioritized list of the top 15 critical paths with enough detail
to write tests for each one. This becomes our test backlog.
```

### Step 2.2 -- Authentication and Authorization Tests

**When to use:** Always write these first. Auth bugs are the most dangerous.

```
Write comprehensive tests for the authentication and authorization system.

Authentication tests:
1. Signup with valid credentials -> account created, session established
2. Signup with existing email -> appropriate error, no duplicate account
3. Signup with weak password -> rejected with clear message
4. Login with valid credentials -> session established, correct user data returned
5. Login with wrong password -> rejected, no information leakage (don't say "wrong password")
6. Login with non-existent email -> rejected, same error as wrong password
7. Logout -> session destroyed, subsequent requests rejected
8. Session expiry -> after [timeout], requests are rejected, redirect to login
9. Password reset flow -> request, email sent, token valid, password changed, old sessions invalidated
10. OAuth flow (if applicable) -> redirect, callback, account created/linked

Authorization tests:
1. Unauthenticated user accessing protected route -> 401/redirect to login
2. User accessing their own resources -> 200 with correct data
3. User accessing another user's resources -> 403 (NOT 404, unless you intentionally hide existence)
4. Admin accessing user resources -> 200 (if admin has permission)
5. User attempting admin actions -> 403
6. API key authentication (if applicable) -> valid key works, invalid key rejected, expired key rejected
7. Role changes -> permissions update immediately, not on next login
8. Deleted/suspended user -> all requests rejected, session invalidated

Each test should be a complete flow: setup state, perform action, verify result AND side effects.
Test both the happy path response AND that the database state is correct after each action.
```

### Step 2.3 -- Payment and Billing Tests

**When to use:** If the application handles money in any form.

```
Write tests for all payment and billing flows.

Subscription lifecycle:
1. New subscription created -> correct plan, correct amount, stored in database
2. Subscription renewed -> period extended, payment recorded
3. Payment failed -> subscription status updated, user notified (check side effects)
4. Payment retry succeeded -> subscription reactivated
5. Subscription cancelled -> access continues until period end, then stops
6. Plan upgrade -> prorated charge, immediate access to new features
7. Plan downgrade -> takes effect at period end, no refund
8. Free trial -> full access during trial, prompted to pay at end

Webhook handling:
1. Valid Stripe webhook -> processed correctly, database updated
2. Invalid webhook signature -> rejected with 400
3. Duplicate webhook (same event ID) -> idempotent, no double-processing
4. Webhook for unknown customer -> logged, not crash
5. Out-of-order webhooks -> handled gracefully (e.g., payment before subscription creation)

Usage and limits:
1. User within limits -> action succeeds
2. User at limit -> action blocked with upgrade prompt
3. User upgrades -> limits immediately increased
4. Monthly reset -> counters reset on billing cycle date

Mock the Stripe API for these tests. Use recorded webhook payloads for webhook tests.
Verify database state after every test, not just the HTTP response.
```

### Step 2.4 -- Core CRUD Integration Tests

```
Write integration tests for the core CRUD operations on [PRIMARY ENTITY, e.g., "projects"].

For each operation (Create, Read, Update, Delete), test:

CREATE:
1. Valid input -> entity created, correct data stored, 201 response
2. Missing required fields -> 422 with field-specific error messages
3. Invalid field values -> 422 with validation errors (test each validation rule)
4. Duplicate unique fields -> 409 with clear message
5. Unauthorized user -> 401
6. User creating entity for another user's account -> 403

READ (List):
1. Returns only current user's entities (multi-tenancy)
2. Pagination works (page 1, page 2, last page, beyond last page)
3. Search/filter returns correct subset
4. Sort order is correct (default and explicit)
5. Empty state returns empty array, not error
6. Deleted entities are not returned (if using soft deletes)

READ (Single):
1. Valid ID -> correct entity returned with all fields
2. Non-existent ID -> 404
3. Another user's entity -> 403
4. Deleted entity -> 404 (or 410 Gone)

UPDATE:
1. Valid partial update -> only specified fields changed, others preserved
2. Invalid field values -> 422, entity unchanged
3. Updating non-existent entity -> 404
4. Updating another user's entity -> 403
5. Concurrent updates -> last write wins or optimistic locking error (document which)

DELETE:
1. Valid delete -> entity marked as deleted (soft) or removed (hard)
2. Deleting non-existent entity -> 404
3. Deleting another user's entity -> 403
4. Cascade effects -> related entities handled correctly
5. Deleted entity cannot be accessed anymore

Use the factory functions to set up test data.
Each test is independent -- no test relies on another test's data.
```

---

## Phase 3: Unit Tests for Business Logic (Day 3-4)

### Step 3.1 -- Extract and Test Pure Business Logic

```
Identify all pure business logic in the codebase -- functions that take inputs and
return outputs without side effects (no database, no API calls, no file system).

Common places to find pure business logic:
- Pricing calculations
- Permission checks
- Data transformations
- Validation rules
- Sorting/filtering algorithms
- Status/state computations
- Date/time calculations
- String formatting functions

For each pure function found:
1. Write tests covering:
   - Typical inputs (the common case)
   - Boundary values (0, 1, max, empty string, empty array)
   - Edge cases (null, undefined, negative numbers, special characters)
   - Type coercion traps (if using JavaScript: "1" vs 1, null vs undefined)
2. Use parameterized/table-driven tests where the logic has many input/output combinations
3. Name tests descriptively: "calculates 20% discount for annual billing"
   NOT "test calculatePrice"

If business logic is tangled with I/O (database calls mixed with calculations),
note it but do NOT refactor yet. Write integration tests for those.
We refactor to extract pure functions AFTER we have test coverage.
```

### Step 3.2 -- Validation Logic Tests

```
Write exhaustive tests for all validation logic in the codebase.

For each validated entity/form:
1. List every validation rule (required, format, length, range, uniqueness, custom)
2. Write a test for each rule with:
   - Input that passes the rule
   - Input that fails the rule
   - Boundary values (exactly at min length, exactly at max length, one over)
3. Test combinations of invalid fields (multiple errors returned at once, not just the first)
4. Test that validation error messages are user-friendly and actionable

Common validation edge cases to test:
- Email: unicode characters, plus addressing, subdomains, very long addresses
- Passwords: minimum length boundary, spaces allowed?, unicode allowed?
- URLs: missing protocol, trailing slashes, query parameters, fragments
- Phone numbers: with/without country code, spaces, dashes, parentheses
- Dates: leap years, timezone boundaries, future/past constraints
- Numbers: zero, negative, decimal precision, very large numbers
- Strings: empty, whitespace only, SQL injection attempts, XSS payloads, null bytes
- Files: zero bytes, max size boundary, wrong mime type, double extensions

Use parameterized tests to keep validation tests compact and readable.
Each test case should be one line: [input, expectedValid, expectedError].
```

### Step 3.3 -- Error Handling Tests

```
Write tests that verify the application handles errors correctly.

For each layer of the application, test error scenarios:

Database errors:
1. Connection refused -> graceful error, not a stack trace
2. Query timeout -> timeout error returned, not hanging forever
3. Unique constraint violation -> meaningful error message
4. Foreign key violation -> meaningful error message
5. Transaction rollback -> partial writes are not persisted

External API errors:
1. API timeout -> retry or graceful fallback
2. API returns 500 -> error logged, user sees friendly message
3. API returns unexpected response shape -> handled without crash
4. API rate limited (429) -> back off and retry
5. Network error (DNS failure, connection reset) -> timeout and graceful error

Application errors:
1. Unhandled exception in request handler -> 500 with generic message (no stack trace in response)
2. Invalid JSON in request body -> 400 with "invalid JSON" message
3. Request body too large -> 413 with size limit in message
4. Unsupported content type -> 415 with supported types listed
5. Method not allowed -> 405 with allowed methods in header

For each test, verify:
- The error response is safe (no internal details, no stack traces, no SQL queries)
- The error is logged with enough context to debug
- The system state is consistent (no partial writes, no locked resources)
- The error response format matches your API conventions
```

---

## Phase 4: Mocking Strategy (Use Throughout)

### Step 4.1 -- External Service Mocks

```
Create reusable mock configurations for all external services this application depends on.

For each external service:
1. Identify all API calls made to the service (endpoints, methods, payloads)
2. Create mock responses for:
   - Success case (200/201 with realistic response body)
   - Common error cases (400 bad request, 401 unauthorized, 404 not found, 429 rate limited, 500 server error)
   - Timeout case
3. Store mock responses in test/mocks/[service-name]/ as JSON files
4. Create a mock setup function that intercepts HTTP calls to the service

External services to mock:
[LIST YOUR EXTERNAL SERVICES -- e.g., Stripe, SendGrid, OpenAI, AWS S3]

Rules for mocking:
- Mock at the HTTP level (intercept requests), not at the SDK level
  (this tests that we're calling the API correctly)
- Use recorded real responses as the basis for mock data
  (call the API once, save the response, use it in tests)
- Mock setup should be one line: mockStripe() or mockSendGrid()
- Each test can override specific mock responses when needed
- All mocks should assert they were called with expected parameters

NEVER mock the database. Use a real test database.
Tests that mock the database are testing your mock, not your code.
```

### Step 4.2 -- Time and Randomness Control

```
Make time-dependent and random behavior deterministic in tests.

Time:
1. All functions that use "current time" should accept an optional time parameter
   - Production: defaults to Date.now() / Time.current
   - Tests: inject a specific timestamp
2. Create a test helper: withFrozenTime(timestamp, callback)
   that freezes time for the duration of the test
3. Test cases that need time:
   - Token expiration: create token, advance time past expiry, verify rejection
   - Subscription renewals: advance time to renewal date, verify behavior
   - Rate limiting: verify limit, advance time past window, verify reset
   - Cron jobs: trigger at expected time boundaries

Randomness:
1. Identify all uses of Math.random(), UUID generation, or similar
2. For functions that need randomness, accept an optional seed/generator parameter
3. In tests, use a seeded random generator for reproducible results
4. Test that "random" selections have the expected distribution (if applicable)

Create these test utilities and document their usage with examples.
```

---

## Phase 5: Coverage Analysis and Expansion (Day 5+)

### Step 5.1 -- Coverage Gap Analysis

```
Run the test suite with coverage enabled and analyze the gaps.

Generate a coverage report and identify:
1. Files with 0% coverage (completely untested)
2. Functions with 0% coverage within partially-tested files
3. Branches with 0% coverage (the "else" paths, error handlers, edge cases)
4. The coverage percentage for each directory/module

Prioritize uncovered code by risk:
- HIGH RISK uncovered: auth, payments, data mutations, security checks
- MEDIUM RISK uncovered: business logic, data transformations, validations
- LOW RISK uncovered: formatters, helpers, UI components, logging

For each HIGH RISK gap, write the specific tests needed to cover it.
For MEDIUM RISK, create a test backlog with descriptions of what each test should verify.
For LOW RISK, acknowledge and move on (we'll get to these eventually).

Do NOT pursue 100% coverage as a goal. Diminishing returns hit hard after 80%.
Coverage of CRITICAL PATHS at 95%+ is worth more than overall coverage at 80%.
```

### Step 5.2 -- Test Quality Audit

```
Audit the existing tests for quality. Not all tests are created equal.

Check for these test anti-patterns and fix them:

1. **Tests that test nothing**: assertions that can never fail
   - Symptom: test passes even when you break the code it supposedly tests
   - Fix: add meaningful assertions or delete the test

2. **Tests that test the framework**: asserting that Express returns 200, not that YOUR code works
   - Fix: assert on business outcomes, not framework behavior

3. **Tightly coupled tests**: tests that break when implementation details change
   - Symptom: renaming a private function breaks tests
   - Fix: test through public interfaces only

4. **Flaky tests**: tests that sometimes pass and sometimes fail
   - Symptom: re-running the suite produces different results
   - Common causes: time-dependence, shared state, race conditions, external calls
   - Fix: isolate, make deterministic, or delete and rewrite

5. **Slow tests**: individual tests taking more than 2 seconds
   - Fix: mock external calls, use faster test database setup, parallelize

6. **Test pollution**: tests that affect other tests
   - Symptom: tests pass individually but fail when run together (or vice versa)
   - Fix: ensure complete cleanup in afterEach, no shared mutable state

7. **Duplicate tests**: multiple tests verifying the exact same behavior
   - Fix: keep the clearest one, delete the rest

For each issue found, fix it. Then re-run the suite to confirm all tests still pass.
Report the number of tests fixed, deleted, and added.
```

### Step 5.3 -- Set Coverage Thresholds

```
Now that we have a solid test baseline, configure coverage thresholds to prevent regression.

Look at the current coverage numbers and set thresholds at:
- Slightly below current coverage (so the build passes today)
- Plan to ratchet up by 5% per month

Configure the test runner to fail CI if coverage drops below thresholds:
- Overall line coverage: [CURRENT - 2]%
- Overall branch coverage: [CURRENT - 5]%
- Per-file minimum: 0% (do NOT set per-file minimums yet -- they create friction)

For critical modules (auth, billing, core business logic):
- Set module-specific higher thresholds
- These modules should be at 90%+ and never decrease

Add a coverage badge to the README.
Add a CI job that comments coverage changes on PRs (increase = green, decrease = red).

The goal is a ratchet: coverage only goes up, never down.
Every PR must either maintain or increase coverage.
```

---

## Ongoing Testing Practices

### For Every New Feature

```
Before implementing [FEATURE], write the test plan.

Answer these questions:
1. What are the happy path scenarios? (user does exactly the right thing)
2. What are the error scenarios? (invalid input, missing data, failed dependencies)
3. What are the edge cases? (empty state, max values, concurrent access, unicode, etc.)
4. What existing behavior might break? (regression risks)
5. What should NOT happen? (negative tests -- verify side effects don't occur)

Write tests BEFORE writing the implementation (TDD for the integration level):
1. Write an integration test for the happy path -- it should fail (the feature doesn't exist)
2. Implement just enough to make it pass
3. Write tests for error cases -- they should fail
4. Implement error handling to make them pass
5. Write tests for edge cases
6. Handle edge cases
7. Refactor the implementation (tests keep you safe)

This test-first approach catches design problems early.
If a feature is hard to test, the design probably needs to change.
```

### Monthly Test Health Check

```
Perform a monthly test suite health check.

Metrics to track:
1. Total test count (should increase with features)
2. Test suite run time (should stay under 5 minutes)
3. Flaky test count (should be zero)
4. Coverage trend (should be flat or increasing)
5. Tests per module (identify under-tested areas)

Actions:
- If run time exceeds 5 minutes: profile slow tests, parallelize, or move slow tests to a separate "full" suite
- If flaky tests appear: fix immediately -- flaky tests destroy trust in the suite
- If coverage decreased: investigate which PR reduced it and add missing tests
- If a module has zero tests and has changed recently: prioritize adding tests

The test suite is a product. It needs maintenance just like the application code.
A slow, flaky, or incomplete test suite is worse than no tests at all --
it gives false confidence while slowing everyone down.
```
