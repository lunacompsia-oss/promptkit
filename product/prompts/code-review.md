# Code Review Prompts

15+ battle-tested prompts for conducting thorough code reviews with AI. Each prompt is designed for a specific review angle -- use them individually or combine them for a comprehensive review.

---

### Security Review

**When to use:** Before merging any PR that touches authentication, authorization, user input handling, or data access.
**Tool:** Claude Code / Cursor / Any

```
Perform a security-focused review of this code. You are a security auditor
who has seen every OWASP Top 10 vulnerability in production.

Check for:
1. SQL injection: any string concatenation in queries? Parameterized queries used everywhere?
2. XSS: any user input rendered without escaping? dangerouslySetInnerHTML usage?
3. CSRF: are state-changing operations protected? Do forms include CSRF tokens?
4. Authentication bypass: can any protected endpoint be reached without valid auth?
5. Authorization flaws: can user A access user B's data? Are ownership checks on every query?
6. Secrets exposure: any hardcoded keys, tokens, or passwords? Any secrets in logs?
7. Mass assignment: can users set fields they shouldn't (role, permissions, pricing)?
8. Insecure deserialization: any eval(), new Function(), or unvalidated JSON.parse()?
9. Path traversal: any file operations using user-supplied paths?
10. Rate limiting: are brute-forceable endpoints (login, signup, password reset) rate-limited?

For each issue found, state:
- Severity: CRITICAL / HIGH / MEDIUM / LOW
- Location: exact file and line
- Attack scenario: how an attacker would exploit this
- Fix: the specific code change needed

If no issues found, explicitly state "No security issues found" with your confidence level.
```

---

### Performance Review

**When to use:** When reviewing code that handles data processing, database queries, or high-traffic endpoints.
**Tool:** Claude Code / Cursor / Any

```
Review this code for performance issues. Think like a developer who has debugged
slow production systems under heavy load.

Check for:
1. N+1 queries: loops that execute a database query per iteration
2. Missing indexes: queries that filter or sort on unindexed columns
3. Unbounded queries: SELECT without LIMIT, fetching all rows when only some are needed
4. Memory leaks: event listeners not removed, growing caches without eviction, unclosed resources
5. Blocking operations: synchronous I/O on the main thread, CPU-heavy computation without workers
6. Redundant computation: calculating the same value multiple times, missing memoization
7. Large payloads: sending more data than the client needs, missing pagination
8. Missing caching: data that rarely changes being fetched from DB on every request
9. Inefficient algorithms: O(n^2) where O(n) or O(n log n) is possible
10. Bundle size: importing entire libraries when only one function is needed

For each issue:
- Impact: estimated performance impact (e.g., "adds ~200ms per request at 1000 items")
- Location: file and line
- Fix: specific refactored code or approach
- Tradeoff: any downsides to the fix (added complexity, cache invalidation, etc.)
```

---

### Architecture Review

**When to use:** When reviewing a PR that introduces new modules, services, or significant structural changes.
**Tool:** Claude Code / Cursor / Any

```
Review the architecture and design decisions in this code change.

Evaluate:
1. Separation of concerns: does each module/function have a single responsibility?
2. Dependencies: does this introduce circular dependencies? Does it depend on the right abstractions?
3. Coupling: if I change module A, how many other modules break? (Lower is better)
4. Cohesion: do the things grouped together actually belong together?
5. Interface design: are public APIs minimal, clear, and hard to misuse?
6. Error handling strategy: is error handling consistent with the rest of the codebase?
7. Data flow: is it clear where data comes from, how it's transformed, and where it goes?
8. Extensibility: can this be extended for likely future requirements without refactoring?
9. Naming: do names communicate intent? Would a new team member understand this code?
10. Consistency: does this follow the patterns established in the rest of the codebase?

For each concern:
- What the code does now
- What the risk is
- What you'd recommend instead
- How confident you are (strong opinion vs. nitpick)

Distinguish between "must fix before merge" and "consider for future improvement."
```

---

### Pull Request Review (General)

**When to use:** For any pull request. A comprehensive general-purpose review.
**Tool:** Claude Code / Cursor / Any

```
Review this pull request as an experienced senior developer who cares about
code quality, maintainability, and correctness.

First, understand the intent:
- What problem does this PR solve?
- Is the approach reasonable for the problem size?

Then check:
1. Correctness: does the code do what it claims to do? Are there logical errors?
2. Edge cases: what happens with empty input, null values, concurrent access, very large data?
3. Error handling: are all failure modes handled? Are error messages helpful?
4. Tests: are the changes tested? Do tests cover the important cases? Are tests readable?
5. Readability: can I understand this code in 60 seconds? If not, what's confusing?
6. Naming: are variable/function names descriptive and consistent?
7. Duplication: does this duplicate logic that already exists elsewhere?
8. Complexity: is this more complex than it needs to be? Can it be simplified?
9. Dependencies: does this add new dependencies? Are they justified?
10. Backwards compatibility: does this break existing behavior or APIs?

Categorize feedback as:
- BLOCKER: must fix before merge
- SUGGESTION: would improve the code but not blocking
- QUESTION: need clarification to complete review
- PRAISE: something done particularly well (developers need positive feedback too)
```

---

### Database Query Review

**When to use:** When reviewing code that adds or modifies database queries.
**Tool:** Claude Code / Cursor / Any

```
Review all database queries in this code change.

For each query:
1. Run EXPLAIN (mentally) -- will this use an index or do a full table scan?
2. Is there a WHERE clause that limits results? Or does it fetch everything?
3. Does it SELECT only the columns needed, or SELECT *?
4. If it's in a loop, is this an N+1 problem? Should it be a JOIN or batch query?
5. Are there any raw SQL strings with string interpolation (SQL injection risk)?
6. For INSERT/UPDATE: is there a transaction where there should be one?
7. For DELETE: is this a soft delete or hard delete? Is that intentional?
8. For migrations: is this migration reversible? Will it lock the table? How long on production data?
9. Are there missing indexes for columns used in WHERE, ORDER BY, or JOIN conditions?
10. Could this query be replaced by a simpler ORM method?

Also check:
- Connection handling: are connections returned to the pool?
- Transaction scope: are transactions too broad (holding locks too long)?
- Deadlock potential: do transactions acquire locks in a consistent order?

For each issue, provide the corrected query.
```

---

### API Design Review

**When to use:** When reviewing new API endpoints or changes to existing ones.
**Tool:** Claude Code / Cursor / Any

```
Review this API for design quality, consistency, and developer experience.

Check:
1. URL structure: RESTful conventions followed? Nouns not verbs? Consistent pluralization?
2. HTTP methods: GET for reads, POST for creates, PUT/PATCH for updates, DELETE for deletes?
3. Status codes: correct codes for each scenario? (201 for create, 204 for delete, 422 for validation, etc.)
4. Request validation: all inputs validated? Validation errors include field names?
5. Response format: consistent envelope (or no envelope -- but consistent)?
6. Pagination: implemented for list endpoints? Cursor-based or offset-based? (Cursor is better)
7. Filtering and sorting: query parameters follow a consistent pattern?
8. Error responses: consistent format? Machine-readable error codes? Human-readable messages?
9. Versioning: is the API versioned? Is the version in the URL or header?
10. Rate limiting: are rate limit headers included? (X-RateLimit-Limit, X-RateLimit-Remaining)
11. Authentication: is the auth mechanism documented? Consistent across endpoints?
12. Idempotency: are POST/PUT endpoints idempotent or do they support idempotency keys?

For a good API:
- A developer should be able to guess the endpoint URL from the resource name
- Error messages should tell the developer exactly what to fix
- Response shape should be predictable (no surprises)

List any breaking changes this introduces to existing API consumers.
```

---

### Error Handling Review

**When to use:** When reviewing code that adds new error handling or failure modes.
**Tool:** Claude Code / Cursor / Any

```
Review the error handling in this code.

Check each error path:
1. Is the error caught at the right level? (Not too early, not too late)
2. Is the error logged with sufficient context to debug? (user ID, input data, stack trace)
3. Is the error response safe? (No internal details, no stack traces, no SQL in user-facing messages)
4. Are errors typed/categorized? (Validation error vs auth error vs server error)
5. Are errors recoverable or fatal? Is the distinction clear in the code?
6. After an error, is the system in a consistent state? (No partial writes, no leaked resources)
7. Are async errors handled? (Unhandled promise rejections, missing error events on streams)
8. Are errors in background jobs handled? (Retried? Dead-lettered? Logged?)
9. Are third-party errors wrapped? (Users should not see Stripe/AWS/etc error internals)
10. Is there a catch-all for unexpected errors? (Global error handler)

Anti-patterns to flag:
- Empty catch blocks (swallowing errors silently)
- catch(e) { throw e } (pointless re-throw)
- Logging the error and also throwing it (double-logging)
- Using exceptions for control flow (try/catch around expected conditions)
- Generic error messages that don't help the user ("Something went wrong")
```

---

### TypeScript Type Safety Review

**When to use:** When reviewing TypeScript code for type correctness and type design.
**Tool:** Claude Code / Cursor / Any

```
Review this TypeScript code for type safety and type design quality.

Check:
1. Any usage of `any`: each one is a potential runtime error. Can it be replaced with
   `unknown`, a generic, or a specific type?
2. Type assertions (`as Type`): each one bypasses the type checker. Is it justified?
   Is there a type guard alternative?
3. Non-null assertions (`value!`): each one claims a value is not null without proof.
   Is there a null check or optional chaining alternative?
4. Union types: are all variants handled? Check switch/if chains for exhaustive handling.
   Use `never` in default case to catch unhandled variants at compile time.
5. Null safety: does the code handle null/undefined at boundaries? (API responses,
   database results, URL parameters)
6. Generic constraints: are generics constrained appropriately? Too loose = useless.
   Too tight = inflexible.
7. Return types: are function return types explicit on public APIs? (Inferred is fine for internal)
8. Discriminated unions: are they used instead of optional properties where appropriate?
9. Readonly: are objects that shouldn't be mutated marked as Readonly<T>?
10. Type exports: are only the necessary types exported? (Don't pollute the public API)

Focus on types at module boundaries (function signatures, API contracts, database types).
Internal implementation types matter less.
```

---

### React Component Review

**When to use:** When reviewing React components, hooks, or UI code.
**Tool:** Claude Code / Cursor / Any

```
Review this React code for correctness, performance, and maintainability.

Check:
1. Rendering: are there unnecessary re-renders? Missing React.memo on expensive children?
2. Effects: does every useEffect have correct dependencies? Any missing cleanup functions?
   Any effects that should be event handlers instead?
3. State: is state at the right level? (Not too high = unnecessary re-renders.
   Not too low = prop drilling.) Could any state be derived instead of stored?
4. Keys: are list keys stable and unique? (NOT array index unless list is static and unordered)
5. Event handlers: are they defined outside render (useCallback) or recreated every render?
   (Only matters for memoized children)
6. Accessibility: do interactive elements have labels? Is keyboard navigation supported?
   Are ARIA attributes correct?
7. Loading states: is there a loading indicator? Does the UI avoid layout shift when data arrives?
8. Error states: what does the user see when data fails to load?
9. Empty states: what does the user see when there's no data?
10. Server vs Client: in Next.js, should this be a server component? Is "use client" necessary?

Also check:
- Props interface: is it minimal? Are optional props really optional?
- Component size: if over 200 lines, should it be split?
- Custom hooks: is reusable logic extracted into hooks?
- Conditional rendering: is complex conditional logic readable?
```

---

### Test Quality Review

**When to use:** When reviewing tests included in a PR.
**Tool:** Claude Code / Cursor / Any

```
Review the tests in this code change.

Check:
1. Do tests verify behavior, not implementation? (Will they break if we refactor internals?)
2. Is each test independent? (No test depends on another test's side effects)
3. Are test names descriptive? ("creates user with valid email" not "test case 1")
4. Is the arrange-act-assert pattern followed? (Setup, action, verification clearly separated)
5. Are assertions specific? (toBe(expected) not toBeTruthy())
6. Are error cases tested? (Not just happy path)
7. Are edge cases tested? (Empty input, null, boundaries, concurrent access)
8. Is the test data minimal? (Only includes data relevant to what's being tested)
9. Are there any commented-out tests? (Either fix them or delete them)
10. Could any tests be parameterized? (Same test logic with different inputs)

Missing tests:
- What scenarios are NOT covered by these tests but should be?
- Are there new code paths (if/else, switch cases) without corresponding tests?
- Are error handlers tested?

Test anti-patterns to flag:
- Testing private methods directly
- Asserting on exact error message strings (brittle)
- setTimeout/sleep in tests (indicates async handling issues)
- Tests that only pass when run in a specific order
```

---

### Dependency Review

**When to use:** When a PR adds, updates, or removes dependencies.
**Tool:** Claude Code / Cursor / Any

```
Review the dependency changes in this PR.

For each NEW dependency added:
1. What problem does it solve? Could we solve it without a dependency? (10 lines of code > 1 dependency)
2. Package health: how many weekly downloads? When was the last release? How many maintainers?
3. Size impact: what does it add to the bundle? (Check bundlephobia.com mentally)
4. License: is it compatible with our license? (MIT, Apache-2.0 = usually fine. GPL = check carefully)
5. Security: any known CVEs? Does it have a security policy?
6. Transitive dependencies: how many dependencies does IT pull in?
7. Alternative: is there a lighter/more maintained alternative?

For each UPDATED dependency:
1. Is this a major version bump? If so, what are the breaking changes?
2. Were the breaking changes addressed in the code?
3. Does the changelog mention any security fixes? (If so, this should be fast-tracked)

For each REMOVED dependency:
1. Is all usage actually removed? (Search the entire codebase)
2. Were replacement implementations added? Are they correct?

General check:
- Is package-lock.json / yarn.lock committed? (Must be)
- Any duplicate packages in the lock file?
- Any dev dependencies that should be dependencies or vice versa?
```

---

### Accessibility Review

**When to use:** When reviewing UI changes or new user-facing features.
**Tool:** Claude Code / Cursor / Any

```
Review this UI code for accessibility compliance.

Check WCAG 2.1 AA requirements:
1. Semantic HTML: are the right elements used? (<button> not <div onClick>, <nav>, <main>, etc.)
2. Heading hierarchy: is there a logical h1 > h2 > h3 order? No skipped levels?
3. Alt text: do all images have descriptive alt text? (Decorative images: alt="")
4. Color contrast: does text meet 4.5:1 ratio (normal) or 3:1 (large text)?
5. Keyboard navigation: can every interactive element be reached and activated with keyboard alone?
6. Focus management: is focus order logical? Is focus trapped in modals? Is focus restored when modals close?
7. Form labels: does every input have an associated <label>? Are required fields indicated?
8. Error messages: are form errors linked to their fields? Are they announced to screen readers?
9. ARIA usage: are ARIA attributes correct? (aria-label, aria-describedby, aria-expanded, role)
   Remember: no ARIA is better than wrong ARIA.
10. Motion: is there a prefers-reduced-motion check for animations?
11. Touch targets: are interactive elements at least 44x44 pixels on mobile?
12. Screen reader: will this make sense when read linearly without visual context?

For each issue, provide:
- The specific element that needs fixing
- The fix (exact HTML/attribute change)
- Which WCAG criterion it violates
```

---

### Concurrency and Race Condition Review

**When to use:** When reviewing code that handles concurrent operations, shared state, or parallel processing.
**Tool:** Claude Code / Cursor / Any

```
Review this code for concurrency issues and race conditions.

Check:
1. Shared mutable state: is any state accessed by multiple concurrent operations?
   (In-memory caches, global variables, static fields)
2. Database race conditions:
   - Read-then-write without transaction (TOCTOU)
   - Concurrent inserts violating unique constraints (handle with ON CONFLICT)
   - Lost updates (two users editing the same record simultaneously)
3. Distributed system issues:
   - Is this operation idempotent? What happens if it runs twice?
   - Can events arrive out of order? Is that handled?
   - Is there a distributed lock where needed? Could it deadlock?
4. Queue/worker issues:
   - What happens if a job fails midway? Is it retried? Is retry safe?
   - Can the same job be picked up by two workers? (At-least-once vs exactly-once)
5. Cache consistency:
   - When the source of truth changes, is the cache invalidated?
   - Can stale cache data cause incorrect behavior?
6. API race conditions:
   - Double-click on submit button: does it create two records?
   - Concurrent requests from the same user: handled correctly?

For each race condition found:
- Describe the exact sequence of events that triggers it
- Estimate the likelihood (common, occasional, rare but catastrophic)
- Provide the fix (transactions, locks, idempotency keys, optimistic locking, etc.)
```

---

### Logging and Observability Review

**When to use:** When reviewing code that adds logging, metrics, or monitoring.
**Tool:** Claude Code / Cursor / Any

```
Review the logging and observability in this code change.

Check:
1. Are important operations logged? (User actions, state changes, external calls)
2. Are errors logged with context? (Not just the error message, but user ID, input, request ID)
3. Are logs structured? (JSON format, consistent fields, not string concatenation)
4. Log levels correct? (ERROR for failures, WARN for degraded, INFO for operations, DEBUG for dev)
5. Are sensitive values redacted? (Passwords, tokens, credit card numbers, PII)
6. Is there too much logging? (Logging in tight loops, logging every cache hit)
7. Are request IDs/correlation IDs propagated across async operations?
8. Are external API calls logged with timing? (duration, status code, endpoint)
9. Are business metrics tracked? (Signups, purchases, feature usage -- not just technical metrics)
10. Can an on-call engineer debug a production issue using only these logs?

Anti-patterns to flag:
- console.log in production code (use a logger)
- Logging the full request/response body (too verbose, potential PII exposure)
- Logging in catch blocks that also re-throw (double logging)
- No log for failed operations (silent failures)
- Logging success but not failure (survivorship bias in logs)
```

---

### Migration and Backwards Compatibility Review

**When to use:** When reviewing database migrations, API changes, or any change that affects existing data or integrations.
**Tool:** Claude Code / Cursor / Any

```
Review this change for backwards compatibility and safe migration.

Database migrations:
1. Is the migration reversible? Is there a down/rollback migration?
2. Will this lock any tables? For how long? (ALTER TABLE on large tables can lock for minutes)
3. Is data backfill needed? Is it done in batches? Is it idempotent?
4. Can old code run against the new schema? (Critical for zero-downtime deploys)
5. Are there NOT NULL columns added without defaults? (This will fail if table has existing rows)

API changes:
1. Are any existing fields removed or renamed? (Breaking change)
2. Are any field types changed? (Breaking change)
3. Are any required parameters added? (Breaking change for existing clients)
4. Are response shapes changed? (Clients parsing the old shape will break)
5. Is there a deprecation period? Are deprecated fields marked in the response?

Configuration changes:
1. Are new environment variables required? (Will existing deployments break?)
2. Are default values sensible for existing installations?
3. Is there a migration guide for users updating from the previous version?

For each breaking change found:
- Who is affected (API consumers, existing users, other services)?
- Can it be made non-breaking? (Add new field, keep old field, deprecate later)
- If it must break, what's the migration path?
```

---

### Code Smell Detection

**When to use:** During regular code review to catch design problems early.
**Tool:** Claude Code / Cursor / Any

```
Scan this code for code smells -- patterns that indicate deeper design problems.

Check for:
1. Long method (>30 lines): should it be broken into smaller functions?
2. Long parameter list (>4 params): should parameters be grouped into an object?
3. Feature envy: a function that uses more of another class's data than its own
4. Shotgun surgery: a small change requires editing many files (high coupling)
5. Data clumps: the same group of variables always appears together (make a type)
6. Primitive obsession: using strings/numbers where a domain type would be clearer
7. Boolean parameters: function(true, false) is unreadable. Use named options.
8. Nested callbacks/promises: more than 2 levels deep indicates missing abstraction
9. Comments explaining what code does: the code should explain itself. Comments should explain WHY.
10. Dead code: unreachable code, unused variables, commented-out code blocks
11. Magic numbers/strings: literal values without explanation. Use named constants.
12. Copy-paste code: similar logic in multiple places. DRY it (but only if it's truly the same concept).

For each smell:
- Pattern name
- Location
- Why it's a problem (the deeper issue it indicates)
- Suggested refactoring (specific, not just "refactor this")
- Priority: fix now vs fix later
```
