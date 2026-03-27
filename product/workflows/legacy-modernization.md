# Legacy Code Modernization

A systematic workflow for safely modernizing a legacy codebase using AI-assisted development. This workflow prioritizes safety at every step -- you should never break production during modernization.

**Core principle:** Never rewrite. Always migrate incrementally. The Strangler Fig pattern is your friend.

**Time estimate:** Varies wildly by codebase size. Budget 1-2 weeks for audit + test coverage, then ongoing incremental migration.

---

## Phase 1: Audit and Understanding (Day 1-2)

### Step 1.1 -- Codebase Health Assessment

**When to use:** First thing when you inherit or decide to modernize a codebase.

```
Perform a comprehensive health assessment of this codebase.

Analyze and report on:

1. **Architecture overview**: What patterns does this codebase follow? (MVC, layered, spaghetti, etc.)
   Map the major modules/packages and their dependencies.

2. **Dependency audit**:
   - List all dependencies with their current version vs latest version
   - Flag any dependencies with known CVEs (check npm audit / pip audit / bundle audit)
   - Identify abandoned dependencies (no updates in 2+ years)
   - Calculate the total dependency count and highlight bloat

3. **Code quality metrics**:
   - Largest files (top 20 by line count) -- these are likely god objects
   - Most complex functions (deeply nested, high cyclomatic complexity)
   - Duplicated code patterns
   - Dead code (exported functions with no importers)

4. **Technology assessment**:
   - Language/framework versions vs current LTS
   - Deprecated APIs or patterns in use
   - Build tooling age and complexity

5. **Test coverage**:
   - What percentage of code has tests?
   - What kind of tests exist? (unit, integration, e2e)
   - Are tests actually running in CI?
   - Are tests trustworthy? (check for skipped tests, flaky patterns)

Output a structured report with a severity rating for each finding:
- CRITICAL: Security vulnerabilities, data loss risks
- HIGH: Major framework version gaps, abandoned critical dependencies
- MEDIUM: Code quality issues, missing tests for core paths
- LOW: Style inconsistencies, non-critical dependency updates

End with a prioritized list of the top 10 things to fix first.
```

### Step 1.2 -- Dependency Graph Mapping

```
Map the internal dependency graph of this codebase.

For each module/package/directory:
1. What does it import from other internal modules?
2. What imports it?
3. What external dependencies does it use?

Identify:
- **Hub modules**: modules that everything depends on (changing these is high risk)
- **Leaf modules**: modules with no internal dependents (safest to refactor first)
- **Circular dependencies**: modules that depend on each other (must be broken)
- **God modules**: modules that know too much (depend on everything)

Create a text-based dependency diagram showing the relationships.
Mark each module with a risk level for refactoring: LOW (leaf), MEDIUM, HIGH (hub).

This map determines our migration order -- we always start with leaf modules and work inward.
```

### Step 1.3 -- Business Logic Extraction

```
Read through the codebase and identify all core business logic.

For each piece of business logic found:
1. Where does it live? (file, function, line numbers)
2. What does it do in plain English?
3. What are the inputs and outputs?
4. What are the edge cases and special conditions?
5. Is it tested? If so, do the tests actually verify the business rules?

Pay special attention to:
- Validation rules (what makes data "valid"?)
- Calculation logic (pricing, scoring, ranking, etc.)
- State machines (what states can an entity be in, what transitions are allowed?)
- Permission/authorization rules (who can do what?)
- Integration points (what external systems are called, with what contracts?)

Output this as a structured "Business Logic Inventory" document.
This becomes our source of truth for verifying that modernization doesn't break anything.
```

---

## Phase 2: Safety Net (Day 3-5)

### Step 2.1 -- Characterization Tests

**When to use:** Before changing any code. These tests capture current behavior, including bugs.

```
Write characterization tests for the existing codebase. These tests document what the code
CURRENTLY does, not what it SHOULD do. If there's a bug, the test should assert the buggy behavior
(we'll fix bugs separately, after the safety net is in place).

Priority order:
1. Revenue-critical paths (signup, payment, core product action)
2. Data mutation endpoints (anything that writes to the database)
3. Authentication and authorization flows
4. Integration points with external services

For each path, write tests that:
- Call the function/endpoint with realistic inputs
- Assert on the exact output (status codes, response shapes, side effects)
- Cover the happy path AND the top 3 most likely error cases
- Use the existing test framework if one exists; if not, set up the simplest option

Test naming convention: "characterization: [module] - [behavior description]"

These tests are our safety net. They MUST pass before and after every refactoring step.
If a characterization test fails after a change, the change broke existing behavior.

Start with the 10 most critical paths. We'll expand coverage iteratively.
```

### Step 2.2 -- Integration Test Harness

```
Set up an integration test harness that tests the application through its public interfaces.

Requirements:
1. Test database: create a separate test database that gets reset between test runs
2. Test fixtures: seed data that represents realistic scenarios
3. API tests: if this is a web app, write tests that make actual HTTP requests
4. Database assertions: verify that mutations produce the expected database state

The harness should:
- Run in CI on every pull request
- Complete in under 5 minutes (fast feedback)
- Be independent of external services (mock/stub external APIs)
- Clean up after itself (no test pollution between runs)

Write integration tests for these critical flows:
[LIST YOUR TOP 5 MOST IMPORTANT USER FLOWS]

Each test should be a complete user story:
"As [user type], when I [action], then [expected outcome]"
```

### Step 2.3 -- Snapshot Current Behavior

```
Create API contract snapshots for all public endpoints/interfaces.

For each endpoint/public function:
1. Document the request format (method, path, headers, body schema)
2. Document all possible response formats (status codes, body schemas)
3. Create snapshot tests that will fail if the contract changes

If this is a REST API:
- Generate an OpenAPI spec from the existing code if possible
- If not, manually document each endpoint by reading the route handlers
- Create contract tests that validate request/response shapes

If this is a library/module:
- Document all public function signatures
- Document return types and thrown exceptions
- Create type-level tests (TypeScript) or contract tests

These snapshots become our regression detection system.
Any modernization step that changes a public contract is flagged immediately.
```

---

## Phase 3: Incremental Migration (Ongoing)

### Step 3.1 -- Strangler Fig Setup

**When to use:** When you're ready to start replacing old code with new code, one piece at a time.

```
Set up the Strangler Fig pattern for incremental migration.

The strategy:
1. New code goes in a clearly separated directory/module (e.g., src/v2/ or src/modern/)
2. A routing/dispatch layer decides whether to use old or new code
3. We migrate one module at a time, with old and new running side by side
4. Once a module is fully migrated and verified, we remove the old version

Implementation steps:
1. Create the new module structure with [TARGET FRAMEWORK/PATTERNS]
2. Set up a feature flag system (simple env var based) that routes to old vs new code
3. Create an adapter/anti-corruption layer between old and new code
   - The new code should NOT import from old code directly
   - Shared data goes through the database or a clean interface
4. Write a migration checklist template:
   - [ ] New implementation written
   - [ ] Characterization tests pass against new implementation
   - [ ] New unit tests written for new implementation
   - [ ] Performance comparison (new should not be slower)
   - [ ] Feature flag tested in both positions
   - [ ] Deployed with flag OFF (old code still active)
   - [ ] Flag turned ON for internal testing
   - [ ] Flag turned ON for 10% of traffic
   - [ ] Flag turned ON for 100% of traffic
   - [ ] Old code removed
   - [ ] Feature flag removed

Start with: [IDENTIFY THE LOWEST-RISK, HIGHEST-VALUE MODULE TO MIGRATE FIRST]
```

### Step 3.2 -- Module Migration Template

**When to use:** For each module you migrate. Repeat this step for every module.

```
Migrate the [MODULE NAME] module from legacy to modern implementation.

Current state:
- Location: [FILE PATH]
- What it does: [DESCRIPTION]
- Dependencies: [WHAT IT IMPORTS]
- Dependents: [WHAT IMPORTS IT]
- Test coverage: [EXISTING TESTS]

Target state:
- Framework/patterns: [TARGET PATTERNS, e.g., "TypeScript with strict types, async/await instead of callbacks"]
- Location: [NEW FILE PATH]

Migration steps:
1. Read the existing implementation and all its tests thoroughly
2. Write the new implementation following modern patterns:
   - Strong typing (no `any`, no implicit types)
   - Error handling with typed errors (no throwing strings)
   - Pure functions where possible (easier to test)
   - Clear separation of I/O from logic
3. Ensure ALL existing characterization tests pass against the new implementation
   (swap the import in tests, run them, they must pass identically)
4. Write additional unit tests for the new implementation covering:
   - Edge cases the old tests missed
   - New patterns that need verification
   - Error paths
5. Create the adapter that the rest of the codebase uses:
   - Same interface as the old module
   - Delegates to new implementation internally
   - Can be swapped back to old implementation via feature flag

Do NOT change the public interface. The rest of the codebase should not know anything changed.
```

### Step 3.3 -- Database Schema Migration

**When to use:** When the modernization requires schema changes.

```
Plan and execute a safe database schema migration.

Current schema issues:
[DESCRIBE WHAT NEEDS TO CHANGE AND WHY]

Rules for safe schema migration:
1. NEVER rename or drop columns in a single step
2. NEVER change column types in a single step
3. All migrations must be backward-compatible (old code must still work)

Use the expand-contract pattern:
- EXPAND: Add new columns/tables alongside old ones
- MIGRATE: Backfill data from old columns to new columns (in batches, not one big UPDATE)
- CONTRACT: Once all code uses new columns, remove old ones (weeks later)

For this migration:
1. Write the EXPAND migration (additive only -- new columns, new tables, new indexes)
2. Write the data backfill script that runs in batches of 1000 rows
   - Must be idempotent (safe to run multiple times)
   - Must not lock tables (use WHERE clauses to process batches)
   - Must log progress
3. Write the code changes that read from new columns (with fallback to old)
4. Write the CONTRACT migration (to be run weeks after everything is stable)

Generate the migration files and the backfill script.
Include a rollback plan for each step.
```

---

## Phase 4: Validation (After Each Migration Step)

### Step 4.1 -- Regression Verification

```
Run a full regression verification after the [MODULE NAME] migration.

Checklist:
1. All characterization tests pass (zero failures, zero skips)
2. All integration tests pass
3. API contract snapshots are unchanged (unless intentionally modified)
4. No new TypeScript/linting errors introduced
5. Bundle size has not increased by more than 5%
6. Database query performance: run EXPLAIN on the top 10 queries, compare with baseline
7. Memory usage: compare before/after under load

If any check fails:
- Do NOT proceed to the next module
- Fix the regression
- Re-run all checks
- Document what went wrong in the migration log

Report format:
- Module migrated: [name]
- Tests: [pass count] / [total count]
- Performance: [better/same/worse with numbers]
- Issues found: [list]
- Decision: [proceed / rollback / fix and re-verify]
```

### Step 4.2 -- Performance Comparison

```
Compare the performance of the new [MODULE NAME] implementation against the old one.

Tests to run:
1. Response time: measure p50, p95, p99 latency for the 5 most common operations
2. Throughput: requests per second under the same load
3. Memory: RSS and heap usage under sustained load
4. Database: total query count and total query time per operation
5. Startup time: how long does the application take to become ready?

Method:
- Use the existing test fixtures to generate realistic load
- Run each benchmark 3 times and take the median
- Compare against the baseline numbers from the old implementation

Acceptable thresholds:
- Latency: new must not be more than 10% slower at p95
- Throughput: new must not be more than 5% lower
- Memory: new must not use more than 20% more memory

Present results in a table with old vs new vs threshold columns.
Flag any metric that exceeds the threshold.
```

---

## Phase 5: Cleanup (After Full Migration)

### Step 5.1 -- Dead Code Removal

```
Now that the migration of [MODULE/FEATURE] is complete and verified, remove the old code.

Steps:
1. Remove the old implementation files
2. Remove the feature flag that toggled between old and new
3. Remove the adapter/anti-corruption layer (the new code is now called directly)
4. Remove any compatibility shims or polyfills that were only needed for the old code
5. Remove old dependencies that are no longer imported anywhere
6. Update the characterization tests:
   - Remove tests that tested migration-specific behavior (flag toggling, adapter)
   - Keep tests that verify business logic (rename from "characterization:" to proper names)
7. Run the full test suite to confirm nothing is broken

After cleanup, verify:
- No references to old file paths remain in the codebase
- No unused imports
- No orphaned test fixtures
- Dependency count has decreased (or at least not increased)
- The build still succeeds
```

### Step 5.2 -- Documentation Update

```
Update all documentation to reflect the modernized state of [MODULE/FEATURE].

Update or create:
1. Architecture documentation: how does the system work now?
2. API documentation: any changes to public interfaces
3. Setup instructions: any new environment variables, dependencies, or steps
4. Migration log entry:
   - What was migrated
   - Why (the problems with the old approach)
   - What changed (technical summary)
   - Performance impact (from our benchmarks)
   - Lessons learned
5. Remove any documentation that references the old implementation
6. Update README if the tech stack or setup process changed

The migration log is permanent institutional knowledge.
Future developers should understand what happened and why without asking anyone.
```

---

## Migration Priority Framework

When deciding what to modernize next, score each module on:

| Factor | Weight | Score 1-5 |
|--------|--------|-----------|
| Security risk (CVEs, outdated auth) | 5x | ___ |
| Revenue impact (affects payments, signups) | 4x | ___ |
| Developer friction (slows down feature work) | 3x | ___ |
| User-facing bugs caused by legacy code | 3x | ___ |
| Dependency on abandoned libraries | 2x | ___ |
| Code complexity (hard to understand/modify) | 1x | ___ |

Migrate highest total score first. Always.

---

## Anti-Patterns to Avoid

1. **The Big Bang Rewrite** -- Never rewrite everything at once. You will lose.
2. **Scope Creep** -- Modernize one module at a time. Do not "while we're at it" adjacent modules.
3. **Premature Abstraction** -- Do not add new abstractions during migration. Match existing behavior first, then improve separately.
4. **Skipping Tests** -- If you cannot write characterization tests for a module, you cannot safely migrate it. Full stop.
5. **Changing Behavior During Migration** -- Bug fixes and feature changes happen in separate commits from migration commits. Always.
6. **Ignoring the Database** -- Schema migrations are the hardest part. Do not leave them for last.
