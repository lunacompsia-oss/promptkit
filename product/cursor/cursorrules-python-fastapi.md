# Python FastAPI — Cursor Rules

You are an expert Python developer building production APIs with FastAPI.

## Project Structure

```
app/
  api/
    routes/           # Route modules grouped by resource
      users.py
      orders.py
    deps.py           # Shared dependencies (get_db, get_current_user)
  core/
    config.py         # Settings via pydantic-settings
    security.py       # JWT, password hashing, auth utilities
    exceptions.py     # Custom exception classes and handlers
  models/             # SQLAlchemy ORM models
    user.py
    order.py
  schemas/            # Pydantic request/response schemas
    user.py
    order.py
  services/           # Business logic layer
    user_service.py
    order_service.py
  repositories/       # Data access layer (queries)
    user_repo.py
  db/
    session.py        # Engine, SessionLocal, get_db dependency
    migrations/       # Alembic migrations
  main.py             # App factory, middleware, router registration
tests/
  conftest.py         # Shared fixtures (client, db session, test user)
  test_users.py
  test_orders.py
```

## Naming Conventions

- Files and directories: `snake_case` exclusively. No exceptions.
- Classes: `PascalCase` — `UserService`, `CreateUserRequest`, `UserModel`.
- Functions and methods: `snake_case` — `create_user`, `get_by_email`.
- Pydantic schemas: `{Action}{Resource}{Request|Response}` — `CreateUserRequest`, `UserResponse`, `ListOrdersResponse`.
- SQLAlchemy models: singular nouns with `Model` suffix or just the noun — `User`, `Order`. Table names are plural: `users`, `orders`.
- Route functions: verb_noun — `create_user`, `get_user`, `list_orders`, `delete_order`.
- Constants: `SCREAMING_SNAKE_CASE`.

## Pydantic Schemas

- Every endpoint has explicit request and response schemas. Never return raw ORM objects.
- Use `model_config = ConfigDict(from_attributes=True)` for ORM-compatible response schemas.
- Separate Create, Update, and Response schemas. Do not reuse one for all.
- Use `Field()` for validation constraints with explicit examples:
  ```python
  email: str = Field(..., max_length=255, examples=["user@example.com"])
  ```
- Partial updates: use `Optional` fields with `exclude_unset=True` in the update logic.
- Never expose internal fields (hashed_password, internal IDs) in response schemas. Whitelist, do not blacklist.

## Dependency Injection

FastAPI's `Depends()` is the core pattern for composition:
```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    ...

@router.get("/me")
async def read_current_user(user: User = Depends(get_current_user)):
    return UserResponse.model_validate(user)
```

- Database session: always injected via `Depends(get_db)`, never imported globally.
- Auth: `get_current_user` dependency. Layer permissions on top: `get_admin_user`.
- Services: inject via dependencies, not as module-level singletons.

## Error Handling

- Define custom exceptions in `core/exceptions.py`:
  ```python
  class NotFoundError(Exception):
      def __init__(self, resource: str, id: str):
          self.resource = resource
          self.id = id
  ```
- Register exception handlers in `main.py` that return consistent JSON:
  ```json
  {"error": {"code": "USER_NOT_FOUND", "message": "User abc123 not found"}}
  ```
- Use `HTTPException` only in route functions for simple cases. Services raise domain exceptions.
- Never catch broad `Exception` silently. Log unexpected errors and re-raise or return 500.
- Validation errors (422) are handled automatically by FastAPI. Customize the format if needed.

## Database (SQLAlchemy + Alembic)

- Use async SQLAlchemy (`AsyncSession`, `create_async_engine`) with `asyncpg`.
- Models use `DeclarativeBase` (SQLAlchemy 2.0 style), never the legacy `declarative_base()`.
- Relationships: always set `lazy="selectin"` or explicitly load with `selectinload()`. Never use lazy loading with async — it raises errors.
- Every table has: `id` (UUID primary key), `created_at`, `updated_at` (server-side defaults).
- Alembic migrations: auto-generate with `alembic revision --autogenerate -m "description"`. Always review the generated migration before applying.
- Use transactions in services. The `get_db` dependency should NOT auto-commit — the service decides when to commit.

## Anti-Patterns to Avoid

BAD: Business logic in route functions
```python
@router.post("/users")
async def create_user(body: CreateUserRequest, db: AsyncSession = Depends(get_db)):
    user = User(**body.model_dump())
    db.add(user)
    await db.commit()
    await send_welcome_email(user.email)
    return user
```
GOOD: Route calls service, service contains logic
```python
@router.post("/users", response_model=UserResponse, status_code=201)
async def create_user(body: CreateUserRequest, service: UserService = Depends()):
    return await service.create(body)
```

BAD: Mixing sync and async database operations
GOOD: Use async throughout. If a library is sync-only, run it in a thread pool with `run_in_executor`.

BAD: Returning ORM objects directly as responses
GOOD: Always convert through a Pydantic response schema

BAD: Using `*` imports
GOOD: Explicit imports only. Every name is traceable.

## Async Rules

- All route handlers must be `async def` when doing I/O (database, HTTP calls).
- Use `def` (sync) only for pure CPU-bound routes — FastAPI runs them in a thread pool automatically.
- HTTP clients: use `httpx.AsyncClient` with a shared instance, not `requests`.
- Never call `asyncio.run()` inside an async context. Never use `time.sleep()` — use `asyncio.sleep()`.
- Background tasks: use FastAPI's `BackgroundTasks` for fire-and-forget work (sending emails, logging analytics).

## Testing

- Framework: `pytest` with `pytest-asyncio` and `httpx.AsyncClient`.
- Use `app.dependency_overrides` to swap real dependencies for test doubles.
- Test database: use a separate test database, apply migrations in `conftest.py`, wrap each test in a transaction that rolls back.
- Fixtures in `conftest.py`: `client` (AsyncClient), `db_session`, `test_user`, `auth_headers`.
- Test the HTTP layer: `response = await client.post("/users", json={...})` — assert status codes and response bodies.
- Test services independently by injecting mock repositories.
- Cover the sad paths: 400 for bad input, 401 for missing auth, 403 for wrong role, 404 for missing resource, 409 for conflicts.

## Import Order

```python
# 1. Standard library
from datetime import datetime, timezone
from uuid import UUID

# 2. Third-party
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, Field
from sqlalchemy.ext.asyncio import AsyncSession

# 3. Local application
from app.core.config import settings
from app.db.session import get_db
from app.schemas.user import CreateUserRequest, UserResponse

# 4. Relative (within same package)
from .deps import get_current_user
```

Use `isort` with profile `black`. Run `ruff` for linting — it replaces flake8, isort, and pyupgrade.

## Security

- Password hashing: use `passlib` with `bcrypt`. Never store plaintext passwords.
- JWT: `python-jose` or `PyJWT`. Short-lived access tokens (15min), longer refresh tokens (7d) in httpOnly cookies.
- Rate limiting: use `slowapi` middleware on auth endpoints.
- SQL injection: impossible with SQLAlchemy's parameterized queries. Never use `text()` with f-strings.
- CORS: configure in `main.py` with explicit allowed origins. No wildcards in production.
- Input size: limit request body size in the reverse proxy (nginx) or middleware.
- Dependencies: run `pip-audit` in CI to catch known vulnerabilities.

## Performance

- Use `selectinload` to avoid N+1 queries. Profile with `echo=True` on the engine during development.
- Pagination: implement cursor-based pagination for list endpoints. Return `next_cursor` in responses.
- Caching: use Redis with `redis.asyncio` for frequently read, rarely written data.
- Connection pool: configure `pool_size=20`, `max_overflow=10` for production.
- Startup: use `@app.on_event("startup")` (or lifespan) to initialize DB pool, Redis connection, and HTTP clients.
- Use `response_model_exclude_none=True` to reduce payload sizes.

## Configuration

Use pydantic-settings for type-safe configuration:
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    DATABASE_URL: str
    JWT_SECRET: str = Field(min_length=32)
    PORT: int = 8000
    ENV: Literal["development", "production", "test"] = "development"
    CORS_ORIGINS: list[str] = ["http://localhost:3000"]

settings = Settings()
```

Crash at startup if required variables are missing. Do not silently default critical config.
