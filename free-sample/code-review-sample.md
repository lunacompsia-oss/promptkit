# Code Review Prompts — Sample (5 of 16)

5 battle-tested prompts for conducting thorough code reviews with AI. Each prompt is designed for a specific review angle -- use them individually or combine them for a comprehensive review.

---

### 1. Security Review

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

### 2. Performance Review

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

### 3. Pull Request Review (General)

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

### 4. Concurrency and Race Condition Review

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

### 5. Code Smell Detection

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

---

**This is a sample.** The full collection includes 16 prompts covering: Architecture Review, Database Query Review, API Design Review, Error Handling Review, TypeScript Type Safety Review, React Component Review, Test Quality Review, Dependency Review, Accessibility Review, Logging and Observability Review, and Migration and Backwards Compatibility Review.

Get the complete collection at: **https://m3phist0s.github.io/promptkit/**
