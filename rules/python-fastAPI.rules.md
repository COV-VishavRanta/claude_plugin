---
paths:
  - "**/*.py"
description: "Generic coding standards and best practices for Python FastAPI services (async SQLAlchemy, optional Strawberry GraphQL), aligned with FastAPI 0.128.x."
---

# FastAPI Backend — Copilot Instructions

Generic, reusable conventions for building Python **FastAPI** services. They assume an async stack — FastAPI + async SQLAlchemy (`asyncpg`/`aiomysql`) — and cover an optional **Strawberry GraphQL** layer. Adapt module/package names to your project; the _patterns_ are what matter.

Apply **DRY** throughout: extract shared logic into reusable functions, parameterize differences, and keep a single source of truth.

## Recommended Project Layout

```
app/
  main.py            # create_app() factory + lifespan; `app = create_app()`
  config.py          # pydantic-settings BaseSettings (loads .env)
  database.py        # engine, async_session_maker, get_db() dependency
  logging.py         # structlog setup + get_logger()
  middleware/        # auth, rate limiting, request-context middleware
  routes/            # REST APIRouter modules (mounted under /api)
  graphql/           # schema, types, mutations, queries, errors (if using GraphQL)
  models/            # SQLAlchemy models + enums
  schemas/           # Pydantic request/response models (REST)
  services/          # business logic (reused by REST + GraphQL)
  utils/             # helpers, validation, messages, errors, shared queries
```

- **Entry point**: a `create_app() -> FastAPI` factory configures middleware, routers, and the lifespan, returning the app; instantiate `app = create_app()` at module bottom.
- **Separation of concerns**: routes/resolvers stay thin and delegate to `services/`; services own business logic; models own persistence; utils hold cross-cutting helpers.

## Application Lifecycle (lifespan)

Use the `@asynccontextmanager` `lifespan` for startup/shutdown — not the deprecated `@app.on_event(...)` handlers. Code before `yield` runs at startup; after `yield` runs at shutdown.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup: open pools, load resources
    yield
    await close_db()  # shutdown: release resources

def create_app() -> FastAPI:
    setup_logging()
    app = FastAPI(title=settings.app_name, lifespan=lifespan)
    # add_middleware(...) in deliberate order, then include_router(...)
    return app

app = create_app()
```

## Configuration

Centralize config in a `pydantic-settings` `BaseSettings` class loaded from `.env`. Read config through the `settings` object — never `os.environ` directly.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    app_name: str = "My API"
    db_user: str
    db_pass: str
    db_name: str
    db_host: str
    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()
```

## Routing & Dependencies

- Group related REST endpoints on an `APIRouter(prefix=..., tags=...)` and mount them under a common prefix (e.g. `app.include_router(router, prefix="/api")`).
- Use `async def` path operations and `await` for async I/O (DB, HTTP, object storage). Never run blocking sync I/O inside an `async def` — it stalls the event loop.
- Manage resources with **dependencies that `yield`** (setup before `yield`, cleanup after). FastAPI caches a dependency's result within a single request.

```python
async def get_db():
    async with async_session_maker() as db:
        yield db
```

- **Prefer `Annotated` dependencies** for reusable, type-checkable injection:

```python
from typing import Annotated
from fastapi import Depends
DbSession = Annotated[AsyncSession, Depends(get_db)]

@router.get("/items")
async def list_items(db: DbSession): ...
```

## Middleware

- Register middleware in `create_app()` in a deliberate order; the order matters (auth, then CORS, then request-context/logging, then rate limiting is a common arrangement).
- Centralize cross-cutting concerns (authentication, request-id/context, rate limiting) in middleware rather than repeating them per route.
- For auth middleware, keep an explicit allowlist of public paths/prefixes/methods and update it when adding public endpoints — don't bypass the middleware per route.

## Error Handling

Define a small, explicit set of error codes mapped to HTTP statuses, and a single custom exception type. Raise it instead of returning ad-hoc error payloads.

```python
from enum import Enum
from typing import Optional

class ErrorCode(Enum):
    UNAUTHENTICATED = ("UNAUTHENTICATED", 401)
    UNAUTHORIZED    = ("UNAUTHORIZED", 403)   # permission failures
    BAD_REQUEST     = ("BAD_REQUEST", 400)
    NOT_FOUND       = ("NOT_FOUND", 404)
    CONFLICT        = ("CONFLICT", 409)
    VALIDATION_ERROR = ("VALIDATION_ERROR", 422)
    INTERNAL_ERROR  = ("INTERNAL_ERROR", 500)

    def __init__(self, code: str, status: int):
        self._code, self._status = code, status
    @property
    def code(self) -> str: return self._code
    @property
    def status(self) -> int: return self._status

class AppError(Exception):
    """Carries an opaque message code + an ErrorCode (with HTTP status) + optional human text."""
    def __init__(self, message: Optional[str] = None,
                 code: "ErrorCode | None" = ErrorCode.INTERNAL_ERROR,
                 readable_message: Optional[str] = None):
        self.message, self.code, self.readable_message = message, code, readable_message
        super().__init__(message)
```

- **For REST**, register `@app.exception_handler(AppError)` (and use FastAPI's `HTTPException` where appropriate) to translate exceptions into consistent JSON responses with the right status code.
- **For GraphQL**, raise the custom error from resolvers and translate it in the endpoint handler / schema error hook (see GraphQL section).
- Keep error-code definitions framework-agnostic (no GraphQL imports) so services and background workers can import them safely.

## Message Codes (Dual-Message Pattern)

Decouple **what the server logs** from **what the client receives**. Define `str`-backed enums for stable, opaque codes the frontend can map to localized text; log human-readable strings separately.

```python
from enum import Enum

class ErrorMessageCode(str, Enum):
    # ── User ──────────────────────────────────────────────────────────────────
    USER_NOT_FOUND = "USER_NOT_FOUND"          # User not found

class SuccessMessageCode(str, Enum):
    # ── User ──────────────────────────────────────────────────────────────────
    USER_ACTIVATED = "USER_ACTIVATED"          # User activated successfully
```

Conventions:

- Both enums subclass `str, Enum`; the value equals the key (`USER_NOT_FOUND = "USER_NOT_FOUND"`).
- Add an inline `# human-readable text` comment on each entry, grouped under `# ── Section ──` headers.
- **Send `message_code.value` to the client**; **log** the human-readable string via the logger (never `print()`).
- When adding a message, define the enum entry first, then reference it everywhere.

```python
# service returns both a human message (for logs) and a code (for the client):
return False, "User not found", ErrorMessageCode.USER_NOT_FOUND, ErrorCode.NOT_FOUND, None

# caller logs the human string, surfaces the code:
if not success:
    logger.error("operation failed", error=message)
    raise AppError(message_code.value, error_code)
```

## Service Layer Patterns

Put business logic in `services/` so it's reusable across REST and GraphQL, independently testable, and separated from transport concerns.

### Function Signatures & Return Tuples

Service functions take explicit dependencies (session, actor, validated input) and return a **multi-value tuple** with a consistent ordering — a clean alternative to raising for expected, recoverable outcomes:

```python
async def create_user(
    db: AsyncSession, actor: User, data: CreateUserInput,
) -> tuple[bool, Optional[str], Optional[ErrorMessageCode], Optional[ErrorCode], Optional[User]]:
    ...
```

Conventional ordering: `(success, human_message, error_message_code, error_code, data, *extras)`. List operations return the data plus a pagination/meta `dict`. Use full type hints throughout (`Optional[T]`, `list[T]`, `tuple[...]`).

### Multi-Value Return Statements

**Always format multi-value returns across lines** — wrap in parentheses, one value per line, trailing comma:

```python
return (
    False,
    "Unauthorized: insufficient permissions for this resource.",
    ErrorMessageCode.NOT_AUTHORIZED,
    ErrorCode.UNAUTHORIZED,
    None,
)
```

### Private Helper Functions

Extract logical sub-steps into private `_`-prefixed helpers to keep the orchestrating function short and focused. Extract when a step exceeds ~10 lines, has a single responsibility, or could be reused. Name them `_<verb>_<noun>` (e.g. `_validate_and_assign_roles`, `_build_response`). **Place helpers above** the public function so the file reads top-to-bottom. Review any function over ~80 lines for extraction.

### Transactions

- **The caller (route/resolver) owns `await db.commit()`** — call it after the service reports success. Services do their work (`add`, `flush`, queries) but leave the commit to the boundary, so a failure before commit rolls the whole unit back.
- Use `await db.flush()` to obtain generated IDs and surface constraint violations early; `await db.refresh(obj)` to reload server-side defaults.
- If an external call (payment, email provider, identity provider) fails before commit, return early and let the transaction roll back.

### Audit / Side-Effect Logging Isolation

Wrap non-critical side effects (audit logs, metrics) in their own `try/except Exception` and **log failures — never re-raise**. A failed audit write must not roll back or break the main operation.

```python
try:
    await create_audit_log(...)
except Exception as e:
    logger.error("Failed to create audit log", error=str(e))
```

## Input Validation (DRY)

Centralize reusable regex/constants and validation helpers in one module (e.g. `utils/validation.py`). Each helper returns `Optional[str]` — an error message if invalid, `None` if valid — and takes a `required` flag so the **same** function serves both create (required) and update (optional) paths.

```python
import re
from typing import Optional

EMAIL_PATTERN = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'

def validate_email(email: Optional[str], required: bool = True) -> Optional[str]:
    if email is None:
        return "Email is required" if required else None
    if not email.strip():
        return "Email cannot be empty"
    if not re.match(EMAIL_PATTERN, email.strip()):
        return "Invalid email address format"
    return None
```

Compose these helpers in a `validate()` method on input types (or in service-layer validation). Apply the same approach to business-rule validation, data transforms, and query filter/sort logic. Add new shared validators to the central module rather than inlining regex.

## Background Tasks

Run side effects (emails, file cleanup, downstream status updates) **after** all DB work — including `await db.commit()` — is complete. Never block the response on them.

- For response-scoped work in REST handlers, use FastAPI's `BackgroundTasks`.
- For fire-and-forget work elsewhere (e.g. GraphQL resolvers), use `asyncio.create_task(...)` — **not** the legacy `asyncio.ensure_future()`.

```python
# A) local closure for a quick fire-and-forget task
async def _send_invite_email() -> None:
    try:
        await send_invitation_email(user, ...)
    except Exception as e:
        logger.error("Failed to send invitation email", error=str(e))

await db.commit()
asyncio.create_task(_send_invite_email())

# B) longer background work that owns its OWN session
async def process_followup(snapshot: FollowupContext) -> None:
    try:
        async with async_session_maker() as db:
            ...
            await db.commit()
    except Exception as e:
        logger.error("Followup failed", error=str(e))
```

Rules for background helpers:

- Wrap the entire body in `try/except Exception` and **log** — never re-raise.
- Pass **plain-value snapshots** (e.g. a `TypedDict`) into the task and **re-query ORM objects** inside it. Never reuse session-bound objects from a completed transaction.
- A task that needs the DB opens its own session via `async_session_maker()` (don't reuse the request session after it's closed).

## SQLAlchemy Models & Database

Keep models in `models/`, extending a shared declarative `Base`. Pick one column style for the project and stay consistent — either classic `Column(...)` or the typed `Mapped[]` / `mapped_column()` (2.0) style.

### Standard Column Defaults

Set **both** a Python `default` and a `server_default` so values are consistent whether rows are created via the ORM or raw SQL (`from sqlalchemy import text`). Prefer UUID primary keys.

```python
import uuid
from datetime import datetime
from sqlalchemy import Column, Boolean, DateTime, String, text
from sqlalchemy.dialects.postgresql import UUID

id = Column(UUID(as_uuid=True), primary_key=True,
            default=uuid.uuid4, server_default=text("gen_random_uuid()"))
created_at = Column(DateTime, default=datetime.now, server_default=text("CURRENT_TIMESTAMP"))
is_active  = Column(Boolean, nullable=False, default=True, server_default=text("true"))
```

- Make boolean/state flags `nullable=False` with matching `default`/`server_default`.
- Add `updated_at` only where update tracking is needed.
- Register every new model in `models/__init__.py` (explicit import + `__all__`).
- Name tables plural snake_case (`users`, `order_items`).

### Enum Columns

Define a `str, Enum` for status-style values, then use it as the column type so the DB enforces valid values. For PostgreSQL native enums, create the type and set both defaults:

```python
from sqlalchemy import Enum
status_enum = Enum(OrderStatusEnum, name="order_statuses", create_type=True)
status = Column(status_enum, nullable=False,
                default=OrderStatusEnum.NEW,           # ORM default (member)
                server_default=OrderStatusEnum.NEW.value)  # SQL default (string)
```

### Relationships & Loading

- Declare FKs as `Column(..., ForeignKey("table.id"))` and `relationship(...)` with explicit `foreign_keys=[...]`; use `back_populates` for bidirectional links.
- Choose `lazy=` deliberately: `"select"` (lazy), `"selectin"`, `"joined"` — and in async code, **eager-load explicitly** at query time (`selectinload(...)` for collections, `joinedload(...)` for single relations). **Never lazy-load after `commit()`.**

### Soft Delete

For entities that should be recoverable, add `deleted_at = Column(DateTime, nullable=True)` (and optionally `deleted_by`). Filter active rows with `.where(Model.deleted_at.is_(None))`. Use SQL idioms `Col.is_(True)` / `Col.is_(None)` and `Col.in_([...])`.

### Constraints & Indexes

Declare composite constraints in `__table_args__` with explicit names (`uq_<table>_<cols>`); use inline `index=True` / `unique=True` for single columns.

```python
__table_args__ = (UniqueConstraint("name", "owner_id", name="uq_items_name_owner_id"),)
email = Column(String(255), nullable=False, index=True)
```

### Migrations (Alembic)

- Generate with `alembic revision --autogenerate -m "..."` then `alembic upgrade head`.
- **Guard against multiple heads** from parallel work: pull latest, check `alembic heads` (a single head), and `alembic merge heads -m "..."` if needed _before_ creating a new revision. Consider enforcing single-head in `env.py`, CI, and the container entrypoint.
- Never manually edit `down_revision` on someone else's migration.

## Strawberry GraphQL (if used)

When the project exposes GraphQL via Strawberry:

### Schema Composition

Root `Query`/`Mutation` are empty `@strawberry.type` classes that **inherit fields from per-domain mixin classes** (`UserQuery`, `UserMutation`, …). Consider a custom `Schema` subclass overriding `process_errors()` to log expected app errors cleanly and unexpected ones with a traceback.

### Types, Inputs, Payloads, Enums

```python
import strawberry
from typing import Annotated, Optional, List

@strawberry.type
class UserType:
    id: str                                # expose UUIDs as str
    email: str
    name: Optional[str] = None
    # strawberry.lazy + Annotated avoids circular imports across type modules:
    orders: Optional[List[Annotated["OrderType", strawberry.lazy("app.graphql.types.order")]]] = None

@strawberry.input
class UpdateUserInput:
    email: Optional[str] = None
    def validate(self) -> Optional[str]:    # each input validates itself
        return validate_email(self.email, required=False) if self.email is not None else None

@strawberry.type
class UpdateUserPayload:                     # standard payload shape
    success: bool
    message: str                            # a SuccessMessageCode .value
    user: Optional[UserType] = None
```

- Field names stay snake_case in Python; Strawberry converts to camelCase in the schema.
- Wrap DB enums with `strawberry.enum(SomeEnum)`, or declare GraphQL-only enums inline with `@strawberry.enum`.
- Convert ORM objects to GraphQL types via small `db_<entity>_to_type(...)` helpers in the service layer.

### Context

Build a context dict for the executor (`user`, `db`, `request`, `response`, anything request-scoped) and access it through small helpers (`get_db_from_context`, `get_current_user_from_context`) rather than reaching into `info.context[...]` everywhere. For cookies, append to a `context["cookies_to_set"]` list and apply them in the endpoint handler after execution — don't mutate the response inside resolvers.

### Mutation Flow

Validate → authorize → get session → call service → check result → `commit` → dispatch background tasks → return payload, all inside `try / except AppError: raise / except Exception`.

```python
@strawberry.mutation(description="Create a user.")
async def create_user(self, info: Info, input: CreateUserInput) -> CreateUserPayload:
    try:
        if (err := input.validate()):
            raise AppError(err, ErrorCode.VALIDATION_ERROR)
        actor = await require_roles_graphql(info, [...])      # coarse authz
        db = get_db_from_context(info)
        success, msg, msg_code, code, user = await create_user_service(db, actor, input)
        if not success:
            raise AppError(msg_code.value if msg_code else "", code)
        await db.commit()
        asyncio.create_task(_send_invite_email(user))
        return CreateUserPayload(success=True, message=SuccessMessageCode.USER_CREATED.value,
                                 user=db_user_to_type(user))
    except AppError:
        raise
    except Exception as e:
        logger.error("Unexpected error in create_user", error=str(e))
        raise AppError(ErrorMessageCode.USER_FAILED_CREATE.value, ErrorCode.INTERNAL_ERROR)
```

### Queries, Pagination & Filtering

Declare list queries with `@strawberry.field`, taking `page: int = 1`, `page_size: int = 20`, and optional `*FilterInput` / `*SortInput` objects; return a payload carrying a reusable `PaginationInfo` (`total_count`, `page`, `page_size`, `total_pages`, `has_next_page`, `has_previous_page`). Clamp/validate pagination args. A sentinel like `page_size == -1` for "fetch all" is a useful convention. Map sort fields via a `sort_column_map` dict with a safe default.

## Authorization

- **Two layers**: a coarse "does the user have one of these roles at all?" check at the route/resolver boundary, and a fine-grained "is that role scoped to _this_ resource?" check in the service layer. Don't rely on the coarse check alone for entity-scoped operations.
- Keep roles in a single `RoleEnum` (or role table) — **never hardcode role-name strings**. Reuse role groupings (e.g. `ADMIN_ROLES`) instead of re-listing roles.
- A reusable scoped check returns a small tuple like `(allowed: bool, error_msg, error_code)` and validates the target entity exists, is active, and isn't soft-deleted.

## Logging

- Use **structured logging** (e.g. `structlog`). At the top of every module: `logger = get_logger(__name__)`.
- **Never use `print()`** anywhere — including the human-readable side of the dual-message pattern. Log via `logger`.
- Use structured kwargs, not f-strings: `logger.info("Event", key=value)` — not `logger.info(f"Event: {value}")`.
- Levels: `debug` (diagnostic/high-frequency), `info` (notable operations), `warning` (recoverable problems), `error` (caught exceptions/failures).

## Testing

- Test the app in-process with `httpx.AsyncClient` (no running server needed).
- Disable the lifespan in tests; mock external auth/identity providers; override the DB dependency via `app.dependency_overrides[get_db] = _override`.
- Use `pytest` with async mode; for GraphQL, `POST` to the GraphQL endpoint with `{"query": "...", "variables": {...}}`.
- **DRY in tests**: extract shared setup, fixtures, and helpers into `conftest.py`.

## Keeping Up to Date

When you need current API details for FastAPI, Strawberry, SQLAlchemy, Pydantic, `pydantic-settings`, or any other library, consult the **context7** MCP docs tool rather than relying on memory — these libraries evolve and defaults change between versions.
