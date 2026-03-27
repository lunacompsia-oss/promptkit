# Debugging Prompts

15+ prompts for systematic debugging with AI. Each prompt targets a specific class of bug and guides the AI to investigate methodically instead of guessing.

---

### General Bug Diagnosis

**When to use:** Starting point for any bug. Provides a systematic framework before diving into specifics.
**Tool:** Claude Code / Cursor / Any

```
I have a bug to diagnose. Here's what I know:

**What should happen:** [expected behavior]
**What actually happens:** [actual behavior]
**Steps to reproduce:** [step by step]
**When it started:** [always, after a deploy, after a specific change, intermittent]
**Environment:** [browser, OS, Node version, etc.]

Diagnose this systematically:
1. Based on the symptoms, list the 3 most likely root causes (ranked by probability)
2. For each candidate cause, describe:
   - What evidence would confirm it
   - What evidence would rule it out
   - The fastest way to check
3. Start with the highest-probability cause and investigate
4. If you need to see specific files or logs, ask for them explicitly

Do NOT guess and fix. Diagnose first, fix second.
A correct diagnosis with a surgical fix beats a shotgun approach every time.
```

---

### Stack Trace Analysis

**When to use:** You have an error stack trace and need to understand what went wrong.
**Tool:** Claude Code / Cursor / Any

```
Analyze this stack trace and help me find the root cause:

```
[PASTE FULL STACK TRACE HERE]
```

Analysis steps:
1. Read the error message. What does it literally say?
2. Identify the origin frame -- which line in OUR code (not library/framework code) triggered this?
3. Read that file and function. What is it trying to do?
4. Trace the data flow backward: what inputs led to this function being called with bad data?
5. Identify the root cause: where did the bad data originate?

For each frame in the stack:
- Is it our code or a dependency? (Focus on our code)
- What was the function trying to do?
- What preconditions does it assume?

Common patterns:
- "Cannot read property X of undefined" -> trace back to where the object should have been defined
- "Type error" -> trace back to where the wrong type was introduced
- "Connection refused" -> check service availability, connection strings, firewall
- "ENOENT" -> file/path doesn't exist, check the path construction

Tell me: what is the root cause, and what is the minimal fix?
```

---

### Race Condition Debugging

**When to use:** Bug that happens intermittently, depends on timing, or only under load.
**Tool:** Claude Code / Cursor / Any

```
I suspect a race condition. Here are the symptoms:

**Behavior:** [describe the intermittent bug]
**Frequency:** [how often does it occur -- 1 in 10 requests? Only under load?]
**Context:** [concurrent users, background jobs, multiple servers, etc.]

Investigate for race conditions:
1. Identify all shared state in the relevant code paths:
   - Database rows that multiple requests read and write
   - In-memory caches or global variables
   - File system resources
   - External service state
2. For each piece of shared state, map the sequence of operations:
   - Which operations READ it?
   - Which operations WRITE it?
   - Are read-then-write sequences atomic?
3. Construct the race scenario:
   - What happens if request A reads, then request B reads, then A writes, then B writes?
   - What happens if two identical requests arrive at the exact same time?
4. Identify the fix:
   - Database: use transactions with appropriate isolation level, or use SELECT FOR UPDATE
   - Application: use optimistic locking (version field), or pessimistic locking (mutex)
   - Distributed: use a distributed lock (Redis SETNX) or make the operation idempotent
5. Write a test that reproduces the race condition (even if it's probabilistic)

Show me the timeline of events that causes the bug, and the fix that makes it impossible.
```

---

### Memory Leak Investigation

**When to use:** Application memory usage grows over time, or OOM (Out of Memory) errors occur.
**Tool:** Claude Code / Cursor / Any

```
The application's memory usage is growing over time and I suspect a memory leak.

**Symptoms:**
- Memory at start: [e.g., 150MB]
- Memory after [time period]: [e.g., 2GB]
- Crash/OOM occurs after: [time or request count]

Investigate these common leak sources in the codebase:

1. Event listeners: search for addEventListener, .on(, .subscribe( without corresponding removal.
   Look for listeners added in loops or on every request.

2. Closures holding references: look for callbacks or closures that capture large objects
   (database results, request bodies, DOM nodes) and are never released.

3. Caches without eviction: search for Map, Set, object used as cache, arrays that only grow.
   Any in-memory cache MUST have a size limit or TTL.

4. Timers without cleanup: search for setInterval, setTimeout in request handlers.
   Are they cleared? What happens if the request ends before the timer fires?

5. Unclosed resources: database connections not returned to pool, file handles not closed,
   HTTP connections not terminated, streams not destroyed.

6. Global state accumulation: variables at module scope that grow with each request.
   Session stores, request logs, error arrays.

7. Circular references (in older JS engines): objects referencing each other preventing GC.

For each potential leak found:
- File and line number
- What is being allocated and never freed
- The fix (with code)
- How to verify the fix (what metric should stabilize)

Also suggest: how to add memory monitoring so we detect leaks early in the future.
```

---

### Slow Query Debugging

**When to use:** API endpoint or page is slow, and you suspect the database.
**Tool:** Claude Code / Cursor / Any

```
This endpoint/page is slow: [ENDPOINT OR PAGE NAME]
Response time: [current time, e.g., 3.5 seconds]
Expected: [target time, e.g., under 200ms]

Debug the database queries:

1. List every database query this endpoint executes (trace through the code)
2. For each query:
   - Write the actual SQL that gets generated
   - Run EXPLAIN ANALYZE (mentally or actually) and assess:
     - Is it doing a sequential scan where an index scan would work?
     - Is it scanning more rows than it returns?
     - Are there any nested loops with large row counts?
   - Check the WHERE clause: is every filtered column indexed?
   - Check JOINs: are join columns indexed on both sides?
3. Count the total number of queries. Is this an N+1 problem?
   (One query to get a list, then one query per item in the list)
4. Check for missing eager loading / includes / joins
5. Check if any query can be eliminated entirely via caching
6. Check if any query fetches more columns than needed (SELECT * vs SELECT specific columns)

For each slow query, provide:
- The current query and its estimated execution time
- The optimized query (with added indexes, restructured JOINs, etc.)
- The expected improvement

Also check: is the slow query actually the bottleneck, or is there application-level
processing (serialization, transformation) that's the real culprit?
```

---

### API Error Debugging

**When to use:** An API endpoint returns an error (4xx or 5xx) and you need to find out why.
**Tool:** Claude Code / Cursor / Any

```
An API endpoint is returning an error.

**Endpoint:** [METHOD /path]
**Request:** [headers, body]
**Response:** [status code, body]
**Expected:** [what it should return]

Debug step by step:
1. Trace the request through the code:
   - Which route handler catches this request?
   - What middleware runs before the handler? (auth, validation, rate limiting)
   - Does the request fail in middleware or in the handler itself?
2. If 401/403 (auth error):
   - Is the auth token present in the request?
   - Is the token valid and not expired?
   - Does the user exist in the database?
   - Does the user have permission for this operation?
3. If 404 (not found):
   - Does the route exist? (Check route definition)
   - Is the route mounted? (Check app/router setup)
   - Is the URL correct? (Trailing slashes, path parameters)
   - If looking up a resource by ID, does the resource exist?
4. If 422/400 (validation error):
   - What validation rules apply to this endpoint?
   - Which specific rule is the request violating?
   - Is the content-type header correct?
5. If 500 (server error):
   - Find the exact error being thrown (check logs/catch blocks)
   - Is it a database error? Network error? Logic error?
   - Is it reproducible or intermittent?

Show me the exact line of code that produces the error and why the request triggers it.
```

---

### Frontend Rendering Bug

**When to use:** UI doesn't look or behave as expected.
**Tool:** Claude Code / Cursor / Any

```
There's a rendering/UI bug:

**What I see:** [describe or screenshot]
**What I expect:** [describe or screenshot]
**Browser/device:** [Chrome, Safari, mobile, etc.]
**Occurs:** [always, on specific data, on specific screen size, after specific action]

Debug the rendering issue:
1. Component tree: which component renders the broken area?
   Trace from the page down to the specific component.
2. Data check: is the component receiving the correct data via props/context/state?
   Log the props at the render boundary.
3. CSS check:
   - Inspect the computed styles on the broken element
   - Check for: overflow hidden cutting content, z-index stacking issues,
     flexbox/grid layout miscalculation, position absolute without positioning context
   - Check media queries: does the breakpoint match the screen size?
4. State check: is React state correct at the time of render?
   Add temporary logging in the component to trace state changes.
5. Hydration check (SSR/Next.js):
   - Does the server-rendered HTML match the client-rendered HTML?
   - Is there a hydration mismatch warning in the console?
   - Is the component using browser-only APIs (window, document) during SSR?
6. Race condition check: is the bug caused by data loading order?
   (Component renders before data arrives, then doesn't re-render when it does)

Check the specific component, its parent, and its CSS. Show me the root cause.
```

---

### Authentication Flow Debugging

**When to use:** Users can't log in, sessions expire unexpectedly, or auth is inconsistent.
**Tool:** Claude Code / Cursor / Any

```
Users are experiencing authentication issues:

**Symptom:** [can't login, session expires, redirected unexpectedly, etc.]
**Auth system:** [NextAuth, custom JWT, session-based, OAuth, etc.]
**Frequency:** [all users, some users, intermittent]

Debug the authentication flow end-to-end:
1. Login request:
   - Is the login endpoint receiving the correct credentials?
   - Is the password comparison correct? (bcrypt compare, not string equality)
   - Is the user found in the database? (Case-sensitive email lookup?)
2. Token/session creation:
   - Is the JWT/session being created correctly?
   - Are the claims/data correct? (User ID, expiration, roles)
   - Is the signing key/secret correct? (Same key for signing and verifying?)
3. Token/session storage:
   - Is the cookie being set? (Check Set-Cookie header)
   - Cookie attributes: httpOnly, secure, sameSite, domain, path -- are they correct for the environment?
   - Is HTTPS required but the dev environment is HTTP? (secure flag issue)
   - Is the cookie domain correct? (subdomain mismatch?)
4. Token/session validation on subsequent requests:
   - Is the cookie/token being sent? (Check request headers)
   - Is the middleware/guard reading it from the correct location?
   - Is the token expired? (Check exp claim vs current time)
   - Is the token signature valid? (Check signing key consistency)
5. Session refresh/renewal:
   - Is the refresh mechanism working?
   - Are expired sessions being extended or rejected?
   - Is there a race condition in token refresh?
6. Logout:
   - Is the session/token actually invalidated?
   - Is the cookie actually cleared? (Same domain/path as when it was set?)

Trace through each step and identify where the flow breaks.
```

---

### Deployment Failure Debugging

**When to use:** The app works locally but fails in production/staging.
**Tool:** Claude Code / Cursor / Any

```
The application fails after deployment but works locally.

**Environment:** [Vercel, Railway, Docker, EC2, etc.]
**Error:** [error message, HTTP status, or behavior]
**Last working deploy:** [commit hash or date]
**Changes since last working deploy:** [PR or commit range]

Debug the environment difference:
1. Environment variables:
   - Are ALL required env vars set in production? Compare .env.example with deployed config.
   - Are any env vars different between local and production? (Database URL, API keys, feature flags)
   - Are there any env vars that reference localhost or 127.0.0.1?
2. Build vs runtime:
   - Does the build succeed? Check build logs for warnings treated as errors.
   - Is the build output different from local? (Different Node version, OS-specific dependencies)
3. Dependencies:
   - Is node_modules identical? (Check lockfile is committed)
   - Are there native dependencies that need compilation? (Different OS architecture)
   - Are dev dependencies excluded in production? (Missing dependency that's a devDependency)
4. Database:
   - Can the application connect to the production database?
   - Are migrations up to date in production?
   - Is the schema different between local and production?
5. Network:
   - Can the production server reach external services? (Firewall, security groups, egress rules)
   - Is DNS resolved correctly?
   - Are there CORS issues? (Different domain in production)
6. File system:
   - Is the code writing to the filesystem? (Serverless/containers have ephemeral storage)
   - Are file paths OS-specific? (Backslashes vs forward slashes)

Check each of these systematically. The bug is almost always an environment difference.
```

---

### Infinite Loop / Hang Debugging

**When to use:** Application hangs, CPU spikes to 100%, or a request never returns.
**Tool:** Claude Code / Cursor / Any

```
The application is hanging / stuck in an infinite loop.

**Where:** [which endpoint, function, or process]
**Symptoms:** [100% CPU, request timeout, process unresponsive]
**Reproducible:** [always, sometimes, under specific conditions]

Investigate:
1. Identify the hot path:
   - Which function is executing when the hang occurs?
   - Add console.log markers at the start and end of suspected functions.
   - The last log printed before the hang identifies the location.

2. Check for infinite loops:
   - While loops: is the loop condition guaranteed to become false? What if the data is unexpected?
   - For loops: is the iterator being modified inside the loop?
   - Recursive functions: is there a base case? Can the base case be unreachable for certain inputs?
   - Event loops: does an event handler trigger the same event it handles?

3. Check for deadlocks:
   - Are multiple database transactions waiting on each other?
   - Are multiple async operations waiting for each other? (Promise A awaits B, B awaits A)
   - Are mutex/locks acquired in different orders by different code paths?

4. Check for blocking operations:
   - Synchronous file reads on large files
   - Synchronous crypto operations
   - DNS resolution blocking the event loop
   - JSON.parse/stringify on very large objects

5. Reproduce minimally:
   - What is the smallest input that triggers the hang?
   - Remove code until the hang stops -- the last removed piece is the cause.

Show me the specific loop or blocking call and why it doesn't terminate.
```

---

### Data Inconsistency Debugging

**When to use:** Data in the database doesn't match what it should be, or different parts of the system show different data.
**Tool:** Claude Code / Cursor / Any

```
There's a data inconsistency in the system:

**What's wrong:** [describe the inconsistent data]
**Where:** [which tables/records are affected]
**Scale:** [one record, some records, widespread]

Investigate the data flow:
1. Trace writes: find every code path that creates or modifies the affected data.
   - Where is this data written? (List every INSERT/UPDATE touching these tables)
   - Are there background jobs that modify this data?
   - Are there webhooks or external triggers that modify this data?
   - Are there database triggers or cascading updates?

2. Check transaction boundaries:
   - Are related writes wrapped in a transaction?
   - Can a partial write succeed? (First INSERT succeeds, second fails, first is committed)
   - Are there writes outside transactions that should be inside one?

3. Check for race conditions:
   - Can two requests write conflicting data simultaneously?
   - Is there a read-modify-write pattern without locking?
   - Are counters incremented without atomic operations?

4. Check for migration issues:
   - Was a recent migration supposed to backfill data? Did it complete?
   - Are there NULL values where NOT NULL was recently added?
   - Are there orphaned foreign keys?

5. Check for caching issues:
   - Is stale cached data being served while the database has been updated?
   - Is the cache invalidated when the underlying data changes?

For the affected records:
- Write a SQL query that identifies all inconsistent records
- Write a SQL query that fixes the inconsistency (but DO NOT run it without review)
- Identify the code fix that prevents this from recurring
```

---

### Third-Party Integration Debugging

**When to use:** An integration with an external service (Stripe, SendGrid, AWS, etc.) is failing.
**Tool:** Claude Code / Cursor / Any

```
An integration with [SERVICE NAME] is failing.

**What's failing:** [describe the failure]
**Error message:** [exact error from the service]
**Was it working before:** [yes/no, when did it stop]

Debug the integration:
1. Authentication:
   - Is the API key/token valid? (Check expiration, correct environment: test vs live)
   - Is the correct key being used? (Not the publishable key where the secret key is needed)
   - Are there IP allowlist restrictions on the API key?

2. Request:
   - Log the exact request being sent (URL, method, headers, body)
   - Compare it against the API documentation
   - Is the content-type header correct?
   - Is the request body the correct format? (JSON vs form-encoded)
   - Are required fields present?
   - Are field values the correct type? (String "100" vs number 100)

3. Response:
   - What status code is returned?
   - What error code/message is in the response body?
   - Does the API documentation explain this error code?
   - Is the response shape what our code expects? (API version mismatch?)

4. Environment:
   - Are we hitting the correct API endpoint? (Sandbox vs production, region-specific)
   - Has the API version changed? (Check if we pin an API version)
   - Is there a service outage? (Check the service's status page)

5. Webhooks (if applicable):
   - Is our webhook endpoint reachable from the internet?
   - Is the webhook signature verification passing?
   - Are we responding with 200 fast enough? (Some services timeout at 5 seconds)

Reproduce the failing request with curl and show me the exact request and response.
```

---

### CSS/Layout Debugging

**When to use:** Layout is broken, elements are misaligned, or styling doesn't apply.
**Tool:** Claude Code / Cursor / Any

```
There's a CSS/layout issue:

**Element:** [which element is affected]
**Expected layout:** [what it should look like]
**Actual layout:** [what it looks like -- overflow, misalignment, wrong size, invisible, etc.]
**Breakpoint:** [does it happen at specific screen sizes?]

Debug the layout:
1. Box model check:
   - What are the computed width, height, padding, margin, border?
   - Is box-sizing: border-box set? (If not, padding adds to width)
   - Are there unexpected margins? (Margin collapse between siblings/parent-child)

2. Layout context:
   - Is the parent flexbox? Check: flex-direction, justify-content, align-items, flex-wrap
   - Is the parent grid? Check: grid-template-columns/rows, gap, grid areas
   - Is the element positioned? Check: position (static/relative/absolute/fixed/sticky)
     and its positioning context (nearest positioned ancestor)

3. Overflow and clipping:
   - Does any ancestor have overflow: hidden? (Content gets cut off)
   - Does any ancestor have a fixed height? (Content overflows)
   - Is there a z-index stacking context hiding the element?

4. Specificity and cascade:
   - Is the expected style actually being applied? (Check for overrides)
   - Is there a more specific selector overriding it?
   - Is there an !important somewhere?
   - With Tailwind: is there a conflicting utility class? (Later classes win)

5. Responsive issues:
   - At what exact pixel width does the layout break?
   - Is there a media query boundary that causes a jump?
   - Are there elements with fixed pixel widths that don't fit small screens?

6. Common gotchas:
   - Flexbox: min-width defaults to auto (text can't shrink below content width). Fix: min-width: 0
   - Grid: auto columns can grow unexpectedly. Fix: minmax(0, 1fr)
   - Images: without max-width: 100%, images overflow containers
   - White-space: nowrap preventing text wrap

Show me the CSS fix and explain why the current styles produce the wrong result.
```

---

### WebSocket / Real-time Debugging

**When to use:** WebSocket connections drop, messages are lost, or real-time features are unreliable.
**Tool:** Claude Code / Cursor / Any

```
Real-time functionality is broken:

**Protocol:** [WebSocket, SSE, polling]
**Symptom:** [connection drops, messages lost, duplicates, delayed, not connecting]
**Frequency:** [always, after X minutes, intermittent]

Debug the real-time connection:
1. Connection establishment:
   - Is the WebSocket URL correct? (ws:// vs wss://, correct port, correct path)
   - Is the upgrade handshake succeeding? (Check for 101 response)
   - Is there a proxy/load balancer that doesn't support WebSockets? (nginx needs explicit config)
   - Is CORS blocking the connection?

2. Connection maintenance:
   - Are ping/pong heartbeats implemented? (Detect dead connections)
   - What's the heartbeat interval? Is it shorter than the server/proxy timeout?
   - Is there automatic reconnection on disconnect?
   - Does reconnection use exponential backoff? (Don't hammer the server)

3. Message delivery:
   - Are messages being sent? (Log outgoing messages)
   - Are messages being received? (Log incoming messages)
   - Is the message format correct? (JSON parse errors?)
   - Are messages being processed in order?
   - Is there a queue for messages sent while disconnected?

4. Server-side:
   - How many concurrent connections can the server handle?
   - Is the server broadcasting to the correct rooms/channels?
   - Are disconnected clients cleaned up? (Memory leak from zombie connections)
   - Is there a connection limit being hit?

5. Infrastructure:
   - Do load balancers use sticky sessions? (WebSocket needs session affinity)
   - Is there a timeout on idle connections? (ALB default is 60 seconds)
   - Are there firewall rules blocking WebSocket traffic?

Trace a single message from sender to receiver and identify where it gets lost.
```

---

### Build / Compilation Error Debugging

**When to use:** The project won't build, compile, or bundle.
**Tool:** Claude Code / Cursor / Any

```
The build is failing:

**Build tool:** [Webpack, Vite, tsc, esbuild, Turbopack, etc.]
**Error:** [paste the build error]
**Context:** [did it work before? What changed?]

Debug the build failure:
1. Read the error message carefully:
   - What file is it complaining about?
   - What line number?
   - Is it a syntax error, type error, module resolution error, or plugin error?

2. Module resolution errors ("Cannot find module X"):
   - Is the package installed? (Check node_modules and package.json)
   - Is the import path correct? (Relative vs absolute, file extension needed?)
   - Is the package's types installed? (@types/X for TypeScript)
   - Is there a path alias that's not configured in the build tool?

3. Type errors (TypeScript):
   - Is the error in our code or in a dependency's types?
   - Did a dependency update change its types? (Check the lockfile diff)
   - Is there a version mismatch between @types/X and X?

4. Plugin/loader errors:
   - Is the plugin configured correctly in the build config?
   - Did the plugin version change? (Breaking change)
   - Is the plugin compatible with the build tool version?

5. Memory/performance:
   - Is the build running out of memory? (Increase with NODE_OPTIONS=--max-old-space-size=4096)
   - Is the build hanging? (Usually a circular dependency or infinite plugin loop)

6. Environment:
   - Does the build work on a clean install? (Delete node_modules, reinstall)
   - Is the Node version correct? (Check .nvmrc or package.json engines)
   - Are there OS-specific issues? (Case-sensitive vs case-insensitive file systems)

Fix the root cause, not the symptom. If the fix is adding a type assertion or @ts-ignore,
that's probably hiding a real problem.
```
