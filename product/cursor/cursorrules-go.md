# Go — Cursor Rules

You are an expert Go developer building production services and CLI tools.

## Project Structure

```
cmd/
  server/             # Main entry point for the API server
    main.go
  worker/             # Main entry point for background worker (if needed)
    main.go
internal/             # Private application code (not importable by others)
  handler/            # HTTP handlers
    user_handler.go
    order_handler.go
  service/            # Business logic
    user_service.go
  repository/         # Data access
    user_repo.go
  middleware/          # HTTP middleware (auth, logging, recovery)
  model/              # Domain types and database models
  config/             # Configuration loading and validation
  pkg/                # Internal shared utilities (logger, errors, validation)
pkg/                  # Public library code (only if building a reusable library)
migrations/           # SQL migration files
```

Follow the standard Go project layout. `internal/` enforces that other modules cannot import your application code. Use `cmd/` for each binary entry point.

## Naming Conventions

- Packages: short, lowercase, single-word. `handler`, `service`, `model`. Never `utils`, `helpers`, `common` — be specific.
- Files: `snake_case.go`. Test files: `snake_case_test.go`.
- Exported types: `PascalCase` — `UserService`, `CreateUserRequest`.
- Unexported: `camelCase` — `validateEmail`, `dbConn`.
- Interfaces: describe behavior, not the thing. `Reader`, `UserStore`, `Notifier`. Never `IUserService`.
- Interface naming: single-method interfaces use the method name + `er`: `type Reader interface { Read(p []byte) (n int, err error) }`.
- Constructors: `New` prefix — `NewUserService(repo UserRepository) *UserService`.
- HTTP handlers: `Handle` prefix — `HandleCreateUser`, `HandleListOrders`.
- Getters: no `Get` prefix. Use `user.Name()`, not `user.GetName()`.
- Acronyms are all-caps: `ID`, `HTTP`, `URL`, `JSON`. Not `Id`, `Http`, `Url`.

## Error Handling

- Always check errors immediately. Never use `_` to discard errors unless the function literally cannot fail for your input.
- Use `fmt.Errorf("doing X: %w", err)` to wrap errors with context. Every wrap adds what the current function was trying to do.
- Use `errors.Is()` for sentinel error comparison, never `==`.
- Use `errors.As()` to extract typed errors, never type assertion on errors.
- Define domain errors as package-level sentinels: `var ErrUserNotFound = errors.New("user not found")`.
- For errors that carry data, define a struct type:
  ```go
  type ValidationError struct {
      Field   string
      Message string
  }
  func (e *ValidationError) Error() string { return fmt.Sprintf("%s: %s", e.Field, e.Message) }
  ```
- HTTP handlers translate domain errors to status codes. The service layer never knows about HTTP.
- Never panic in library code. Reserve `panic` for truly unrecoverable programmer errors (nil map assignment is a bug, not an expected error).

## Interface Design

- Accept interfaces, return structs. Functions take the narrowest interface they need.
- Define interfaces where they are USED, not where they are implemented. The `service` package defines `UserRepository` interface. The `repository` package implements it.
- Keep interfaces small. 1-3 methods is ideal. If an interface has 10 methods, it is doing too much.
- Never define an interface before you have two implementations or need it for testing. Concrete types are fine.

## Anti-Patterns to Avoid

BAD: Naked returns in non-trivial functions
```go
func getUser(id string) (user User, err error) {
    // ... 20 lines ...
    return // What is being returned? Unreadable.
}
```
GOOD: Explicit returns always

BAD: `init()` functions for anything beyond package-level registration
GOOD: Explicit initialization in `main()` or constructors

BAD: Global mutable state (`var db *sql.DB` at package level)
GOOD: Pass dependencies through constructors. Wire everything in `main()`.

BAD: Over-using channels for things a mutex handles better
GOOD: Use `sync.Mutex` for protecting shared state. Channels are for communication between goroutines, not for guarding data.

BAD: Empty interface `interface{}` or `any` as function parameters
GOOD: Define specific types. Use generics if you need polymorphism.

BAD: Returning concrete types from constructors that could break tests
```go
func NewService() *Service { return &Service{db: realDB()} }
```
GOOD: Accept dependencies as parameters
```go
func NewService(repo UserRepository) *Service { return &Service{repo: repo} }
```

## Concurrency

- Never start a goroutine without a plan for how it will stop. Use `context.Context` for cancellation.
- Always pass `context.Context` as the first parameter of functions that do I/O.
- Use `errgroup.Group` for running concurrent operations with error collection.
- Protect shared state with `sync.Mutex` or `sync.RWMutex`. Document what the mutex guards.
- Never read and write to a map from multiple goroutines without synchronization. Use `sync.Map` only when the key set is append-only.
- Channels: prefer buffered channels with known capacity. Unbuffered channels couple sender and receiver timing.

## HTTP Handlers

Use the standard library `net/http` or a minimal router (`chi`). Avoid heavy frameworks.

```go
func (h *UserHandler) HandleCreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "invalid request body")
        return
    }
    if err := req.Validate(); err != nil {
        respondError(w, http.StatusUnprocessableEntity, err.Error())
        return
    }
    user, err := h.service.Create(r.Context(), req)
    if err != nil {
        handleServiceError(w, err) // Maps domain errors -> HTTP status codes
        return
    }
    respondJSON(w, http.StatusCreated, user)
}
```

- Use `r.Context()` and pass it downstream for cancellation and deadlines.
- Middleware chains: logging -> recovery -> auth -> rate-limit -> handler.
- Response helpers: create `respondJSON` and `respondError` functions. Never write raw `w.Write` with manual Content-Type everywhere.

## Database

- Use `database/sql` with `pgx` driver, or `sqlc` for type-safe query generation.
- `sqlc` is strongly preferred: write SQL, generate Go code. No ORM magic.
- Always use parameterized queries. The `$1, $2` placeholder style with pgx. Never `fmt.Sprintf` into SQL.
- Use `context.Context` with all database calls for timeout control.
- Connection pool: `sql.DB` manages it. Set `SetMaxOpenConns(25)`, `SetMaxIdleConns(5)`, `SetConnMaxLifetime(5 * time.Minute)`.
- Transactions: pass `*sql.Tx` through the call chain or use a transaction-scoped repository.

## Testing

- Use the standard `testing` package. No assertion libraries needed — `if got != want { t.Errorf(...) }` is clear.
- Table-driven tests for functions with multiple input/output cases:
  ```go
  tests := []struct {
      name    string
      input   string
      want    int
      wantErr bool
  }{ ... }
  for _, tt := range tests {
      t.Run(tt.name, func(t *testing.T) { ... })
  }
  ```
- Use `t.Helper()` in test helper functions so error line numbers point to the caller.
- Mock interfaces with hand-written fakes or `gomock`. Prefer hand-written fakes for small interfaces.
- HTTP handler tests: use `httptest.NewRecorder()` and `httptest.NewRequest()`.
- Integration tests: use build tags (`//go:build integration`) and a real database.
- Test file is in the same package for white-box testing, or `_test` package for black-box.

## Import Order

```go
import (
    // 1. Standard library
    "context"
    "fmt"
    "net/http"

    // 2. Third-party packages
    "github.com/go-chi/chi/v5"
    "github.com/jackc/pgx/v5"

    // 3. Internal project packages
    "github.com/yourorg/yourapp/internal/model"
    "github.com/yourorg/yourapp/internal/service"
)
```

Use `goimports` to enforce grouping automatically. Three groups separated by blank lines.

## Security

- SQL injection: parameterized queries with `pgx`. Never build SQL with string concatenation.
- Input validation: validate and sanitize all user input in the handler layer before passing to services.
- Auth: use middleware to extract and verify JWT. Pass user identity through `context.Context`.
- Secrets: read from environment variables. Use `os.Getenv` in config, nowhere else.
- HTTPS: terminate TLS at the load balancer. The Go server listens on plain HTTP behind it.
- Rate limiting: use `golang.org/x/time/rate` with per-IP limiters.
- Timeouts: set `http.Server{ReadTimeout: 5s, WriteTimeout: 10s, IdleTimeout: 120s}`. Never run with zero timeouts.

## Performance

- Profile before optimizing. Use `pprof` to identify actual bottlenecks.
- Allocations: reuse slices with `sync.Pool` only in hot paths confirmed by benchmarks. Premature pooling adds complexity.
- JSON: `encoding/json` is fine for most cases. Switch to `github.com/goccy/go-json` only if marshaling shows up in profiles.
- Database: batch inserts with `pgx.CopyFrom` for bulk operations. Use `EXPLAIN ANALYZE` on slow queries.
- Benchmark tests: use `func BenchmarkX(b *testing.B)` to measure before and after optimization.
- Binary size: use `-ldflags="-s -w"` for production builds to strip debug info.

## Configuration

```go
type Config struct {
    Port        int    `env:"PORT" envDefault:"8080"`
    DatabaseURL string `env:"DATABASE_URL,required"`
    JWTSecret   string `env:"JWT_SECRET,required"`
    Env         string `env:"APP_ENV" envDefault:"development"`
}
```

Use `github.com/caarlos0/env/v11` or similar. Validate at startup, crash on missing required vars. Pass `Config` (or relevant parts) into constructors — never read env vars deep in the code.
