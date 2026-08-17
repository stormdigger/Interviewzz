# ⚡ FastAPI

> FastAPI is Starlette (ASGI) + Pydantic (validation) + a dependency injection system, tied together by Python type hints. Understanding those three pieces explains everything the framework does.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [ASGI and the Async Foundation](#2-asgi)
3. [Request Lifecycle](#3-request-lifecycle)
4. [Pydantic](#4-pydantic)
5. [Dependency Injection](#5-dependency-injection)
6. [Routing and Path Operations](#6-routing)
7. [Async Correctness](#7-async-correctness)
8. [Database Integration](#8-database-integration)
9. [Authentication](#9-authentication)
10. [Error Handling](#10-error-handling)
11. [Middleware](#11-middleware)
12. [Background Work](#12-background-work)
13. [Testing](#13-testing)
14. [Project Structure](#14-project-structure)
15. [Production Deployment](#15-production)
16. [Performance](#16-performance)
17. [Interview Section](#17-interview-section)
18. [Cheat Sheet](#18-cheat-sheet)

---

## 1. Mental Model

```
   ┌──────────────────────────────────────────────────────────────┐
   │                        FastAPI                               │
   │                                                              │
   │   ┌────────────────┐  ┌──────────────┐  ┌────────────────┐   │
   │   │   STARLETTE    │  │   PYDANTIC   │  │  DEPENDENCY    │   │
   │   │                │  │              │  │   INJECTION    │   │
   │   │ ASGI app       │  │ Validation   │  │                │   │
   │   │ Routing        │  │ Serialization│  │ Resolution     │   │
   │   │ Middleware     │  │ Schema gen   │  │ Caching        │   │
   │   │ WebSockets     │  │ (Rust core)  │  │ Sub-deps       │   │
   │   │ Background     │  │              │  │ Yield cleanup  │   │
   │   └────────────────┘  └──────────────┘  └────────────────┘   │
   │            │                  │                  │           │
   │            └──────────────────┴──────────────────┘           │
   │                    Python TYPE HINTS drive everything        │
   │                    → OpenAPI docs generated for free         │
   └──────────────────────────────────────────────────────────────┘
```

The core idea: **a type annotation is a declaration of intent** that FastAPI uses for parsing, validation, serialization, documentation, and editor support simultaneously.

```python
@app.post("/items/{item_id}", response_model=ItemOut, status_code=201)
async def create(
    item_id: int,                      # ← path param, coerced + validated as int
    item: ItemIn,                      # ← request body, validated against the model
    q: str | None = None,              # ← optional query param
    user: User = Depends(current_user) # ← injected dependency
) -> ItemOut:
    ...
```

From that one signature FastAPI derives: where each value comes from, how to validate it, what errors to return, the OpenAPI schema, and the interactive docs.

---

## 2. ASGI

### 2.1 WSGI vs ASGI

```
   WSGI (Flask, Django ≤2)             ASGI (FastAPI, Starlette, Django 3+)
   ───────────────────────             ────────────────────────────────────
   def app(environ, start_response):   async def app(scope, receive, send):
       return [b"body"]                    await send({...})

   • One request per worker thread     • Many concurrent requests per worker
   • Blocking                          • Async native
   • No WebSockets/SSE                 • WebSockets, SSE, HTTP/2 push
   • Simple                            • Long-lived connections
```

```python
# The raw ASGI interface — everything else is built on this
async def app(scope, receive, send):
    assert scope['type'] == 'http'
    await send({'type': 'http.response.start', 'status': 200,
                'headers': [(b'content-type', b'text/plain')]})
    await send({'type': 'http.response.body', 'body': b'Hello'})
```

`scope` holds connection metadata, `receive` pulls incoming events, `send` pushes outgoing ones. Middleware is just a callable that wraps another ASGI app.

### 2.2 Why async matters here

```
   BLOCKING (WSGI, 4 workers)          ASYNC (1 worker, event loop)

   W1 [══ DB 50ms ══]                  ┌─ req1 ──await──┐
   W2 [══ DB 50ms ══]                  ├─ req2 ──await──┤  all waiting
   W3 [══ DB 50ms ══]                  ├─ req3 ──await──┤  concurrently
   W4 [══ DB 50ms ══]                  ├─ ...           │
   → 4 concurrent requests             └─ req1000 ──────┘
                                       → thousands concurrent
```

The win only materializes for **I/O-bound** work. CPU-bound work still blocks the loop, and async gives you nothing there.

---

## 3. Request Lifecycle

```
   HTTP request
       │
       ▼
   ┌──────────────────────────────────────────────┐
   │ 1. ASGI server (uvicorn) parses the request  │
   ├──────────────────────────────────────────────┤
   │ 2. MIDDLEWARE stack — outermost first        │
   │      CORS → GZip → Auth → custom → ...       │
   ├──────────────────────────────────────────────┤
   │ 3. ROUTING — match path + method             │
   ├──────────────────────────────────────────────┤
   │ 4. DEPENDENCIES resolved (depth-first)       │
   │      • cached per request by default         │
   │      • `yield` deps run setup here           │
   ├──────────────────────────────────────────────┤
   │ 5. VALIDATION — path, query, headers,        │
   │      cookies, body → Pydantic models         │
   │      ✗ failure → 422 with field-level errors │
   ├──────────────────────────────────────────────┤
   │ 6. PATH OPERATION runs                       │
   │      async def → on the event loop           │
   │      def       → in a THREADPOOL             │
   ├──────────────────────────────────────────────┤
   │ 7. RESPONSE MODEL filters + serializes       │
   │      ⭐ fields not in response_model are      │
   │        REMOVED — key security property       │
   ├──────────────────────────────────────────────┤
   │ 8. `yield` dependency CLEANUP runs           │
   ├──────────────────────────────────────────────┤
   │ 9. Middleware unwinds (innermost first)      │
   ├──────────────────────────────────────────────┤
   │ 10. BackgroundTasks run AFTER the response   │
   └──────────────────────────────────────────────┘
```

---

## 4. Pydantic

### 4.1 Models

```python
from pydantic import BaseModel, Field, EmailStr, field_validator, model_validator, ConfigDict
from typing import Annotated, Literal
from datetime import datetime
from decimal import Decimal

class UserCreate(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,
        extra="forbid",                      # ⭐ reject unknown fields
        frozen=False,
    )

    email: EmailStr
    username: Annotated[str, Field(min_length=3, max_length=32, pattern=r"^[a-z0-9_]+$")]
    password: Annotated[str, Field(min_length=12, max_length=128)]
    age: Annotated[int, Field(ge=13, le=120)]
    role: Literal["admin", "user", "guest"] = "user"
    tags: list[str] = Field(default_factory=list, max_length=10)

    @field_validator("password")
    @classmethod
    def strong_password(cls, v: str) -> str:
        if not any(c.isupper() for c in v) or not any(c.isdigit() for c in v):
            raise ValueError("must contain an uppercase letter and a digit")
        return v

    @model_validator(mode="after")
    def check_consistency(self):
        if self.role == "admin" and self.age < 18:
            raise ValueError("admins must be 18+")
        return self


class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)   # read from ORM objects
    id: int
    email: EmailStr
    username: str
    created_at: datetime
    # ⭐ NOTE: no password field. It literally cannot be serialized out.
```

### 4.2 The response_model security property

```python
@app.get("/users/{id}", response_model=UserOut)
async def get_user(id: int) -> Any:
    return await db.get_user(id)     # a full ORM object with password_hash, tokens...
```

FastAPI filters the output through `UserOut`, so `password_hash` cannot leak even if you forget to strip it. This is a genuine defense-in-depth mechanism, not just documentation.

```python
# Additional controls
response_model_exclude_unset=True     # omit fields the client didn't set
response_model_exclude_none=True      # omit nulls
response_model_by_alias=True          # use field aliases in output
```

### 4.3 Model composition

```python
class UserBase(BaseModel):
    email: EmailStr
    username: str

class UserCreate(UserBase):
    password: str

class UserUpdate(BaseModel):                     # all optional for PATCH
    email: EmailStr | None = None
    username: str | None = None

class UserInDB(UserBase):
    id: int
    hashed_password: str

class UserOut(UserBase):
    model_config = ConfigDict(from_attributes=True)
    id: int
    created_at: datetime
```

🏭 **Never reuse one model for input and output.** Input models accept a password; output models must not have one. Separate models make it impossible to confuse them.

### 4.4 Settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import PostgresDsn, RedisDsn, SecretStr
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_nested_delimiter="__")

    app_name: str = "api"
    environment: Literal["dev", "staging", "prod"] = "dev"
    debug: bool = False

    database_url: PostgresDsn
    redis_url: RedisDsn
    secret_key: SecretStr                   # ⭐ won't appear in repr/logs
    access_token_expire_minutes: int = 15
    cors_origins: list[str] = []

@lru_cache
def get_settings() -> Settings:
    return Settings()                        # validated once at startup
```

Configuration errors fail at startup with a clear message, rather than at 3 a.m. with a `KeyError`.

### 4.5 Pydantic v1 → v2

| v1 | v2 |
|---|---|
| `@validator` | `@field_validator` |
| `@root_validator` | `@model_validator` |
| `.dict()` | `.model_dump()` |
| `.json()` | `.model_dump_json()` |
| `parse_obj()` | `model_validate()` |
| `class Config` | `model_config = ConfigDict(...)` |
| `orm_mode` | `from_attributes` |
| Pure Python | **Rust core (pydantic-core) — 5-50× faster** |

---

## 5. Dependency Injection

### 5.1 The basics

```python
from typing import Annotated
from fastapi import Depends

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with SessionLocal() as session:
        try:
            yield session                    # ← everything before yield = setup
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()            # ← after yield = teardown

DB = Annotated[AsyncSession, Depends(get_db)]   # ⭐ reusable alias

@app.get("/users")
async def list_users(db: DB):
    return await db.execute(select(User))
```

### 5.2 Sub-dependencies and caching

```python
async def get_token(authorization: Annotated[str | None, Header()] = None) -> str:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(401, "Missing bearer token")
    return authorization[7:]

async def get_current_user(token: Annotated[str, Depends(get_token)], db: DB) -> User:
    payload = decode_jwt(token)
    user = await db.get(User, payload["sub"])
    if not user or not user.is_active:
        raise HTTPException(401, "Invalid credentials")
    return user

def require_role(*roles: str):
    async def checker(user: Annotated[User, Depends(get_current_user)]) -> User:
        if user.role not in roles:
            raise HTTPException(403, "Insufficient permissions")
        return user
    return checker

CurrentUser = Annotated[User, Depends(get_current_user)]
AdminUser = Annotated[User, Depends(require_role("admin"))]

@app.delete("/users/{id}")
async def delete_user(id: int, admin: AdminUser, db: DB): ...
```

```
   DEPENDENCY GRAPH — resolved depth-first, each cached per request

        delete_user
        ├── require_role("admin")
        │   └── get_current_user
        │       ├── get_token
        │       │   └── Header(authorization)
        │       └── get_db  ────────┐
        └── get_db  ────────────────┴──▶ SAME session instance
                                          (cached by default)
```

`use_cache=False` disables caching if you genuinely need a fresh instance per use.

### 5.3 Router- and app-level dependencies

```python
# Applied to every route in the router — the return value is discarded
router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(require_role("admin")), Depends(rate_limit)],
    responses={403: {"model": ErrorResponse}},
)

app = FastAPI(dependencies=[Depends(verify_api_key)])
```

🏭 This is the right place for cross-cutting authorization — it can't be forgotten on a new endpoint.

### 5.4 Overriding for tests

```python
app.dependency_overrides[get_db] = lambda: test_session
app.dependency_overrides[get_current_user] = lambda: fake_user
# Clear after each test
app.dependency_overrides.clear()
```

This is the single best feature for testability — you replace the dependency graph without touching application code.

---

## 6. Routing

```python
from fastapi import APIRouter, Query, Path, Body, Header, Cookie, Form, File, UploadFile

router = APIRouter(prefix="/api/v1/items", tags=["items"])

@router.get(
    "/{item_id}",
    response_model=ItemOut,
    responses={404: {"model": ErrorResponse, "description": "Item not found"}},
    summary="Get an item",
    description="Retrieve a single item by its ID.",
)
async def get_item(
    item_id: Annotated[int, Path(ge=1, description="Item ID")],
    q: Annotated[str | None, Query(max_length=50)] = None,
    x_request_id: Annotated[str | None, Header()] = None,
    session: Annotated[str | None, Cookie()] = None,
): ...

# Complex query params via a model (FastAPI 0.115+)
class ItemFilters(BaseModel):
    model_config = ConfigDict(extra="forbid")
    status: list[Literal["active", "archived"]] = []
    min_price: Decimal | None = None
    limit: Annotated[int, Field(ge=1, le=100)] = 20
    cursor: str | None = None

@router.get("/")
async def list_items(filters: Annotated[ItemFilters, Query()]): ...

# File upload
@router.post("/upload")
async def upload(
    file: Annotated[UploadFile, File()],
    description: Annotated[str, Form()],
):
    if file.size > 10 * 1024 * 1024:
        raise HTTPException(413, "File too large")
    while chunk := await file.read(1024 * 1024):     # ⭐ stream, don't load it all
        await storage.write(chunk)
```

⚠️ **Route order matters** — the first match wins:

```python
@app.get("/users/me")       # ✅ must come FIRST
@app.get("/users/{id}")     # otherwise "me" is parsed as an id and fails validation
```

---

## 7. Async Correctness

### 7.1 The rule

```python
# ✅ async def + async libraries — runs on the event loop
@app.get("/a")
async def a():
    return await async_db.fetch(...)

# ✅ def (sync) — FastAPI runs it in a THREADPOOL automatically
@app.get("/b")
def b():
    return sync_db.query(...)          # blocking is fine here

# ❌❌ THE CARDINAL SIN — blocking inside async
@app.get("/c")
async def c():
    time.sleep(1)                      # blocks the ENTIRE event loop
    requests.get(url)                  # every other request freezes
    return sync_db.query(...)
```

```
   One blocking call in an async endpoint blocks ALL concurrent requests
   on that worker. 100 concurrent users → 100 × the latency.

   This is the #1 FastAPI production bug.
```

### 7.2 Escaping blocking code

```python
from fastapi.concurrency import run_in_threadpool
import asyncio

@app.get("/mixed")
async def mixed():
    result = await run_in_threadpool(blocking_library_call, arg)   # I/O-bound
    # or: result = await asyncio.to_thread(blocking_call, arg)
    return result

# CPU-bound work → a process pool, not a thread (the GIL blocks threads)
executor = ProcessPoolExecutor(max_workers=4)

@app.post("/render")
async def render(data: RenderIn):
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(executor, heavy_cpu_work, data.payload)
```

### 7.3 Concurrency within a request

```python
@app.get("/dashboard")
async def dashboard(user: CurrentUser):
    # ❌ sequential: 300 ms
    # profile = await get_profile(user.id)
    # orders  = await get_orders(user.id)
    # recs    = await get_recs(user.id)

    # ✅ parallel: 100 ms
    profile, orders, recs = await asyncio.gather(
        get_profile(user.id), get_orders(user.id), get_recs(user.id),
    )

    # ✅ with per-call resilience
    async with asyncio.TaskGroup() as tg:          # Python 3.11+
        p = tg.create_task(get_profile(user.id))
        o = tg.create_task(get_orders(user.id))
    return {"profile": p.result(), "orders": o.result()}
```

### 7.4 Detecting blocked loops

```python
# Enable asyncio debug mode in staging — warns on callbacks over 100ms
import asyncio
asyncio.get_event_loop().set_debug(True)

# Or a watchdog
import time, threading
def monitor(loop, threshold=0.1):
    while True:
        start = time.perf_counter()
        future = asyncio.run_coroutine_threadsafe(asyncio.sleep(0), loop)
        future.result(timeout=5)
        if (lag := time.perf_counter() - start) > threshold:
            logger.warning("event loop blocked for %.3fs", lag)
        time.sleep(1)
```

---

## 8. Database Integration

### 8.1 Async SQLAlchemy 2.0

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, selectinload
from sqlalchemy import select, func, ForeignKey

engine = create_async_engine(
    settings.database_url,
    pool_size=20,                  # steady-state connections
    max_overflow=10,               # burst capacity
    pool_pre_ping=True,            # ⭐ validate before use — survives DB restarts
    pool_recycle=1800,             # avoid stale connections behind proxies
    echo=settings.debug,
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)

class Base(DeclarativeBase): pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True, index=True)
    orders: Mapped[list["Order"]] = relationship(back_populates="user", lazy="raise")
    #                                            ⭐ lazy="raise" makes N+1 a LOUD ERROR
```

### 8.2 Avoiding N+1

```python
# ❌ 1 + N queries — and with lazy="raise" it raises instead of silently doing this
users = (await db.execute(select(User))).scalars().all()
for u in users:
    print(u.orders)

# ✅ selectinload — 2 queries (IN clause), best for one-to-many
stmt = select(User).options(selectinload(User.orders)).limit(20)

# ✅ joinedload — 1 query with a JOIN, best for many-to-one
stmt = select(Order).options(joinedload(Order.user))

# ✅ Nested
stmt = select(User).options(selectinload(User.orders).selectinload(Order.items))
```

### 8.3 Repository pattern

```python
class UserRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get(self, user_id: int) -> User | None:
        return await self.db.get(User, user_id)

    async def list_paginated(self, cursor: str | None, limit: int) -> tuple[list[User], str | None]:
        stmt = select(User).order_by(User.created_at.desc(), User.id.desc()).limit(limit + 1)
        if cursor:
            created, uid = decode_cursor(cursor)
            stmt = stmt.where(tuple_(User.created_at, User.id) < (created, uid))
        rows = (await self.db.execute(stmt)).scalars().all()
        has_more = len(rows) > limit
        rows = rows[:limit]
        return rows, encode_cursor(rows[-1]) if has_more else None

def get_user_repo(db: DB) -> UserRepository:
    return UserRepository(db)

UserRepo = Annotated[UserRepository, Depends(get_user_repo)]
```

⚠️ **The session is per-request, not global.** A module-level session shared across requests causes cross-request data leakage and unpredictable transaction boundaries.

---

## 9. Authentication

```python
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from passlib.context import CryptContext
from jose import jwt, JWTError
from datetime import datetime, timedelta, timezone

pwd = CryptContext(schemes=["argon2"], deprecated="auto")   # ⭐ argon2 > bcrypt
oauth2 = OAuth2PasswordBearer(tokenUrl="/auth/token")

def create_access_token(sub: str, extra: dict | None = None) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": sub, "iat": now, "jti": str(uuid4()),
        "exp": now + timedelta(minutes=settings.access_token_expire_minutes),
        "iss": settings.app_name, "aud": "api",
        **(extra or {}),
    }
    return jwt.encode(payload, settings.secret_key.get_secret_value(), algorithm="HS256")

async def get_current_user(token: Annotated[str, Depends(oauth2)], db: DB) -> User:
    creds_exc = HTTPException(401, "Could not validate credentials",
                              headers={"WWW-Authenticate": "Bearer"})
    try:
        payload = jwt.decode(
            token, settings.secret_key.get_secret_value(),
            algorithms=["HS256"],           # ⭐ PINNED — never trust the alg header
            audience="api", issuer=settings.app_name,
        )
    except JWTError:
        raise creds_exc

    if await redis.sismember("revoked_jti", payload["jti"]):   # logout support
        raise creds_exc

    user = await db.get(User, int(payload["sub"]))
    if not user or not user.is_active:
        raise creds_exc
    return user

@app.post("/auth/token")
async def login(form: Annotated[OAuth2PasswordRequestForm, Depends()], db: DB):
    user = await db.scalar(select(User).where(User.email == form.username))
    # ⭐ Verify a dummy hash even when the user doesn't exist —
    #    prevents timing-based username enumeration
    valid = pwd.verify(form.password, user.hashed_password if user else DUMMY_HASH)
    if not user or not valid:
        raise HTTPException(401, "Incorrect email or password")
    return {"access_token": create_access_token(str(user.id)), "token_type": "bearer"}
```

See [API Design §10](api-design.md#10-auth) and [AppSec](../07-security/appsec.md) for the full auth model.

---

## 10. Error Handling

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

class AppError(Exception):
    status_code = 500
    code = "internal_error"
    def __init__(self, detail: str, **extra):
        self.detail, self.extra = detail, extra

class NotFoundError(AppError):
    status_code, code = 404, "not_found"

class ConflictError(AppError):
    status_code, code = 409, "conflict"

@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "type": f"https://api.example.com/errors/{exc.code}",
            "title": exc.code.replace("_", " ").title(),
            "status": exc.status_code,
            "detail": exc.detail,
            "instance": str(request.url.path),
            "request_id": request.state.request_id,
            **exc.extra,
        },
    )

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(status_code=422, content={
        "type": "https://api.example.com/errors/validation_failed",
        "title": "Validation failed", "status": 422,
        "request_id": request.state.request_id,
        "errors": [
            {"field": ".".join(str(x) for x in e["loc"][1:]),
             "code": e["type"], "message": e["msg"]}
            for e in exc.errors()
        ],
    })

@app.exception_handler(Exception)
async def unhandled_handler(request: Request, exc: Exception):
    logger.exception("Unhandled", extra={"request_id": request.state.request_id})
    return JSONResponse(status_code=500, content={
        "type": "https://api.example.com/errors/internal_error",
        "title": "Internal Server Error", "status": 500,
        "detail": "An unexpected error occurred.",     # ⭐ never leak the traceback
        "request_id": request.state.request_id,
    })
```

---

## 11. Middleware

```python
from starlette.middleware.base import BaseHTTPMiddleware
import time, uuid

@app.middleware("http")
async def request_context(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    request.state.request_id = request_id
    structlog.contextvars.bind_contextvars(request_id=request_id,
                                           path=request.url.path,
                                           method=request.method)
    start = time.perf_counter()
    try:
        response = await call_next(request)
    finally:
        duration = time.perf_counter() - start
        REQUEST_DURATION.labels(request.method, request.url.path).observe(duration)
        structlog.contextvars.clear_contextvars()
    response.headers["X-Request-ID"] = request_id
    response.headers["X-Process-Time"] = f"{duration:.4f}"
    return response

# Built-in middleware — order matters (added LAST = outermost)
app.add_middleware(GZipMiddleware, minimum_size=1000)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["api.example.com"])
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,   # ⭐ NEVER ["*"] with credentials
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=600,
)
```

⚠️ Middleware runs for **every** request including 404s and static files. Keep it cheap. For per-route logic, use dependencies instead — they're skipped when the route doesn't match.

---

## 12. Background Work

```python
from fastapi import BackgroundTasks

# ── In-process: runs AFTER the response is sent ──────────────
@app.post("/signup")
async def signup(user: UserCreate, bg: BackgroundTasks, db: DB):
    created = await create_user(db, user)
    bg.add_task(send_welcome_email, created.email)     # response returns immediately
    return created
```

⚠️ `BackgroundTasks` runs **in the same process**. If the process dies or restarts, the task is lost. It's fine for best-effort work (analytics, non-critical email); it is not a job queue.

```python
# ── Durable: Celery / ARQ / Dramatiq ─────────────────────────
from arq import create_pool
from arq.connections import RedisSettings

async def send_email(ctx, to: str, subject: str, body: str):
    await mailer.send(to, subject, body)

class WorkerSettings:
    functions = [send_email]
    redis_settings = RedisSettings.from_dsn(settings.redis_url)
    max_tries = 5
    job_timeout = 300

@app.post("/signup")
async def signup(user: UserCreate, db: DB):
    created = await create_user(db, user)
    await app.state.arq.enqueue_job("send_email", created.email, "Welcome", body)
    return created
```

### Lifespan

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    app.state.redis = await aioredis.from_url(settings.redis_url)
    app.state.http = httpx.AsyncClient(timeout=10.0)    # ⭐ ONE reusable client
    app.state.arq = await create_pool(RedisSettings.from_dsn(settings.redis_url))
    yield
    # shutdown
    await app.state.http.aclose()
    await app.state.redis.aclose()
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

⚠️ **Creating an `httpx.AsyncClient` per request** is a common performance bug — you lose connection pooling and pay a full TLS handshake every time. Create it once in lifespan.

---

## 13. Testing

```python
import pytest
from httpx import AsyncClient, ASGITransport

@pytest.fixture(scope="session")
def anyio_backend(): return "asyncio"

@pytest.fixture
async def db_session():
    """Each test runs inside a transaction that is rolled back."""
    async with engine.connect() as conn:
        trans = await conn.begin()
        session = AsyncSession(bind=conn, expire_on_commit=False)
        yield session
        await session.close()
        await trans.rollback()          # ⭐ perfect isolation, no cleanup needed

@pytest.fixture
async def client(db_session):
    app.dependency_overrides[get_db] = lambda: db_session
    async with AsyncClient(transport=ASGITransport(app=app),
                           base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()

async def test_create_user(client, db_session):
    r = await client.post("/users", json={
        "email": "a@b.com", "username": "ada", "password": "Str0ngPassword!", "age": 30,
    })
    assert r.status_code == 201
    assert "password" not in r.json()             # ⭐ response_model filtering works
    assert r.json()["email"] == "a@b.com"

async def test_validation_error(client):
    r = await client.post("/users", json={"email": "not-an-email"})
    assert r.status_code == 422
    fields = {e["field"] for e in r.json()["errors"]}
    assert "email" in fields

@pytest.mark.parametrize("age,expected", [(12, 422), (13, 201), (120, 201), (121, 422)])
async def test_age_boundaries(client, age, expected): ...
```

---

## 14. Project Structure

```
app/
├── main.py                  # app creation, lifespan, middleware, router mounting
├── config.py                # Settings
├── dependencies.py          # shared Annotated deps
├── exceptions.py            # AppError hierarchy + handlers
├── db/
│   ├── base.py              # DeclarativeBase, engine, session factory
│   ├── models/              # SQLAlchemy models
│   └── migrations/          # Alembic
├── api/
│   └── v1/
│       ├── router.py        # aggregates all v1 routers
│       ├── users.py
│       └── orders.py
├── schemas/                 # Pydantic models (In/Out/Update per entity)
├── services/                # ⭐ business logic — framework-independent
├── repositories/            # data access
├── workers/                 # background job definitions
└── core/
    ├── security.py
    └── logging.py
tests/
├── conftest.py
├── unit/                    # services, no DB
└── integration/             # full request cycle
```

🏭 **Keep business logic in `services/`, free of FastAPI imports.** Routers should parse input, call a service, and shape the response. That makes the logic testable without HTTP and portable if you ever move off FastAPI.

---

## 15. Production

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv export --frozen --no-dev > requirements.txt \
 && uv pip install --system --no-cache -r requirements.txt

FROM python:3.12-slim
RUN useradd -m -u 1000 app
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY --chown=app:app ./app ./app
USER app
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", \
     "--workers", "4", "--proxy-headers", "--forwarded-allow-ips", "*"]
```

```
   WORKERS = (2 × CPU cores) + 1  as a starting point, then measure.

   ⚠️ Each worker is a SEPARATE PROCESS with its own memory,
      connection pool, and in-process cache.
      4 workers × pool_size 20 = 80 DB connections. Size accordingly.
```

```python
# Health checks — distinguish liveness from readiness
@app.get("/health/live", include_in_schema=False)
async def live():
    return {"status": "ok"}                      # is the process alive?

@app.get("/health/ready", include_in_schema=False)
async def ready(db: DB):
    try:
        await db.execute(text("SELECT 1"))
        await app.state.redis.ping()
    except Exception as e:
        raise HTTPException(503, f"not ready: {e}")
    return {"status": "ready"}                   # can it serve traffic?
```

```python
# Graceful shutdown — uvicorn waits for in-flight requests on SIGTERM.
# Ensure your Kubernetes terminationGracePeriodSeconds exceeds your
# longest request, and add a preStop sleep so the load balancer
# stops sending new traffic before the process exits.
```

```python
# Disable docs in production if the API isn't public
app = FastAPI(
    docs_url=None if settings.environment == "prod" else "/docs",
    redoc_url=None,
    openapi_url=None if settings.environment == "prod" else "/openapi.json",
)
```

---

## 16. Performance

| Lever | Impact |
|---|---|
| Never block the event loop | Largest single factor |
| `orjson` / `ORJSONResponse` | 2-5× faster serialization |
| `uvloop` + `httptools` (default in uvicorn[standard]) | ~2× throughput |
| Connection pooling (DB, HTTP, Redis) | Avoids handshake per request |
| `asyncio.gather` for independent I/O | Latency ÷ number of calls |
| `response_model_exclude_unset` | Smaller payloads |
| Pagination limits enforced server-side | Prevents accidental full scans |
| `selectinload` / `joinedload` | Kills N+1 |

```python
from fastapi.responses import ORJSONResponse
app = FastAPI(default_response_class=ORJSONResponse)

# Streaming for large responses — constant memory
from fastapi.responses import StreamingResponse

@app.get("/export")
async def export(db: DB):
    async def rows():
        yield "id,email\n"
        async for user in db.stream_scalars(select(User)):
            yield f"{user.id},{user.email}\n"
    return StreamingResponse(rows(), media_type="text/csv")
```

---

## 17. Interview Section

<details>
<summary><b>Q1. What is ASGI and why does FastAPI use it?</b></summary>

ASGI is the async successor to WSGI. Where a WSGI application is a synchronous callable handling one request per worker thread, an ASGI application is an async callable receiving a scope plus receive and send channels, so one worker can handle thousands of concurrent connections.

FastAPI needs it for two reasons. First, concurrency: most API work is I/O-bound — waiting on a database or an upstream service — and async lets one process hold thousands of those waits simultaneously instead of dedicating a thread to each. Second, ASGI supports long-lived connections, so WebSockets and server-sent events work natively rather than needing a separate server.

The important caveat is that this only helps I/O-bound work. CPU-bound work still blocks the loop, and async gives no benefit there.
</details>

<details>
<summary><b>Q2. What happens if you call a blocking function in an async endpoint?</b></summary>

It blocks the entire event loop, so every concurrent request on that worker stalls until it returns. One `time.sleep(1)` or a synchronous `requests.get` turns a server handling thousands of concurrent requests into one handling one at a time.

This is the most common FastAPI production bug, and it's insidious because it doesn't show up under light load — with one user everything looks fine.

The fixes: use async libraries throughout, `httpx` instead of `requests`, async SQLAlchemy instead of sync. If a library has no async version, wrap it with `run_in_threadpool` or `asyncio.to_thread`. For CPU-bound work use a process pool, because threads don't help against the GIL.

There's also a shortcut people forget: if you define the endpoint as a plain `def` rather than `async def`, FastAPI runs it in a threadpool automatically, so blocking is safe. Mixing the two knowingly is fine — mixing them accidentally is the bug.
</details>

<details>
<summary><b>Q3. Explain FastAPI's dependency injection.</b></summary>

A dependency is any callable declared with `Depends`. FastAPI builds a graph from the type annotations, resolves it depth-first before the endpoint runs, and caches each dependency's result per request so a shared dependency like the database session is instantiated once even if five things need it.

Dependencies can use `yield`, which makes them context managers — code before the yield is setup, code after is teardown that runs after the response is generated. That's how session lifecycle, transactions, and resource cleanup are handled.

They compose: a dependency can depend on other dependencies, so you build `get_token` → `get_current_user` → `require_role("admin")` as layers. And you can attach them at router or app level so authorization can't be forgotten on a new endpoint.

The feature I value most is `dependency_overrides` — in tests you swap the database session or the current user without touching application code. That makes the whole application testable by construction.
</details>

<details>
<summary><b>Q4. How does Pydantic fit in, and what's the security benefit?</b></summary>

Pydantic does validation, parsing, and serialization, driven by type annotations. FastAPI uses the same models to generate the OpenAPI schema, so documentation can't drift from behavior.

The security benefit is `response_model`. FastAPI serializes the return value *through* that model, so any field not declared on it is removed. If your ORM object has a `password_hash` or an internal token and your output model doesn't declare it, it cannot leak — even if you forget to strip it manually.

The complementary control is `extra="forbid"` on input models, which rejects unknown fields rather than silently ignoring them. That prevents mass-assignment, where an attacker adds `"role": "admin"` to a signup payload.

The practice that makes both work is never reusing one model for input and output. Separate `UserCreate`, `UserUpdate`, and `UserOut` models make it structurally impossible to accept a field you shouldn't or return one you shouldn't.
</details>

<details>
<summary><b>Q5. How do you handle database sessions?</b></summary>

Per request, via a `yield` dependency. The dependency opens a session, yields it, and in the teardown commits on success or rolls back on exception, then closes.

Per-request is essential — a module-level session shared across requests causes cross-request data leakage and unpredictable transaction boundaries, and with async it's outright unsafe.

For the engine I'd configure `pool_pre_ping=True` so connections are validated before use, which makes the app survive database restarts and proxy timeouts, and `pool_recycle` to avoid stale connections. And I'd size the pool with worker count in mind — four uvicorn workers each with a pool of twenty is eighty database connections, which needs to fit within the database's limit.

On the ORM side, I set `lazy="raise"` on relationships so an accidental lazy load raises loudly rather than silently generating an N+1, then use `selectinload` or `joinedload` explicitly where relations are needed.
</details>

<details>
<summary><b>Q6. `BackgroundTasks` vs Celery/ARQ?</b></summary>

`BackgroundTasks` runs in the same process after the response is sent. It's zero-infrastructure and fine for best-effort work — analytics pings, non-critical notifications.

But it has no durability. If the process restarts or crashes, the task is gone with no record and no retry. There's also no visibility, no rate limiting, and no way to spread load across machines. And since it shares the process, a slow task consumes capacity that should be serving requests.

A real job queue — Celery, ARQ, Dramatiq — persists jobs in Redis or a broker, so they survive restarts, retry with backoff, run on dedicated workers you can scale independently, and give you a dead letter queue and monitoring.

My rule: if losing the task would be a bug someone files, it needs a real queue. If nobody would notice, `BackgroundTasks` is fine.
</details>

<details>
<summary><b>Q7. How do you test a FastAPI application?</b></summary>

`httpx.AsyncClient` with `ASGITransport` exercises the full stack — middleware, dependencies, validation, serialization — without starting a server or opening a socket.

For the database, each test runs inside a transaction that's rolled back afterward. That gives perfect isolation with no cleanup code and no cross-test contamination, and it's much faster than recreating schema per test.

Dependencies get replaced through `dependency_overrides` — a test session instead of the real one, a fake current user instead of real JWT validation. That's the mechanism that makes everything else testable.

I'd structure tests in two layers: unit tests against services with no HTTP and no database, since business logic shouldn't need either; and integration tests through the client covering the real request cycle, including the failure paths — validation errors, authorization failures, and confirming that sensitive fields don't appear in responses.
</details>

<details>
<summary><b>Q8. FastAPI vs Django vs Flask.</b></summary>

Django is a full framework: ORM, admin, auth, migrations, templates, forms — all integrated and conventional. It's the fastest path to a complete web application with server-rendered pages and an admin interface, and its ecosystem is enormous. The cost is that it's opinionated and heavier, and async support, while present since 3.0, is still partial in places.

Flask is a microframework — routing and templating, and you assemble everything else. Maximum flexibility, minimum structure, which means every team's Flask app looks different and validation and documentation are your problem.

FastAPI sits between them for API work specifically. You get async-native performance, automatic validation and OpenAPI documentation from type hints, and a dependency injection system that makes testing straightforward. What you don't get is an admin, an ORM, or auth — you assemble those.

So: Django for a full application with server-rendered pages and an admin. FastAPI for an API-only service, especially async-heavy or high-concurrency. Flask if you want minimal and have strong conventions already. For a new API service today, FastAPI is usually the right default in Python.
</details>

<details>
<summary><b>Q9. How do you deploy FastAPI in production?</b></summary>

Uvicorn with multiple workers behind a reverse proxy, in a container. Worker count starts around two times CPU cores plus one, then gets tuned by measurement.

The detail that matters: each worker is a separate process with its own connection pools and in-process caches. Four workers with a pool of twenty is eighty database connections, so pool sizing has to account for it — and any in-process caching is per-worker, not shared.

Operationally: separate liveness and readiness probes, since liveness only asks whether the process is alive while readiness checks dependencies. Graceful shutdown with a termination grace period longer than the longest request, plus a preStop delay so the load balancer stops routing before the process exits. Structured JSON logging with a request ID propagated through every log line. And docs disabled in production if the API isn't public.

For configuration, Pydantic Settings validates everything at startup, so a missing or malformed environment variable fails immediately with a clear message rather than at runtime.
</details>

<details>
<summary><b>Q10. How would you speed up a slow FastAPI endpoint?</b></summary>

First, measure — where is the time actually going? Usually it's a downstream call, not framework overhead.

The checks I'd run in order. Is anything blocking the event loop? That's the biggest single factor and it's easy to introduce accidentally. Are independent I/O calls running sequentially when `asyncio.gather` would parallelize them? Is there an N+1 in the ORM — the fastest way to find out is setting `lazy="raise"` and seeing what breaks. Are connection pools being reused, or is a new HTTP client created per request?

Then the cheaper wins: `orjson` for serialization, which is several times faster than the standard library; `response_model_exclude_unset` to shrink payloads; server-enforced pagination limits so a client can't request everything.

Beyond that it stops being a FastAPI question and becomes a database or caching question — the right index, or a cache in front of an expensive computation. The framework is rarely the bottleneck; what it's waiting on usually is.
</details>

---

## 18. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                         FASTAPI — ONE PAGE                           ║
╠══════════════════════════════════════════════════════════════════════╣
║ = Starlette(ASGI) + Pydantic(validation) + DI, driven by TYPE HINTS  ║
║ async def → event loop   |   def → threadpool (blocking is OK there) ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ #1 BUG: blocking call inside async def → freezes ALL requests      ║
║   fix: async libs · run_in_threadpool · ProcessPool for CPU          ║
╠══════════════════════════════════════════════════════════════════════╣
║ DEPENDENCIES: Depends() · cached per request · yield = setup/teardown║
║   Annotated[X, Depends(f)] aliases · router-level deps for authz     ║
║   dependency_overrides = the testing superpower                      ║
╠══════════════════════════════════════════════════════════════════════╣
║ PYDANTIC: response_model FILTERS OUTPUT → passwords can't leak ⭐     ║
║   extra="forbid" on input → blocks mass assignment                   ║
║   separate Create/Update/Out models — never reuse one                ║
║   Settings validated at STARTUP                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║ DB: session per REQUEST via yield dep · pool_pre_ping=True           ║
║   lazy="raise" makes N+1 loud · selectinload/joinedload to fix       ║
║   workers × pool_size = total connections — size for it              ║
╠══════════════════════════════════════════════════════════════════════╣
║ CONCURRENCY: asyncio.gather / TaskGroup for independent I/O          ║
║ LIFESPAN: create httpx.AsyncClient/redis ONCE, not per request       ║
║ BackgroundTasks = best-effort only (lost on restart) → use ARQ/Celery║
╠══════════════════════════════════════════════════════════════════════╣
║ ROUTES: /users/me BEFORE /users/{id} — first match wins              ║
║ ERRORS: custom AppError + handlers · RFC9457 shape · never leak      ║
║   tracebacks · request_id on every response and log line             ║
╠══════════════════════════════════════════════════════════════════════╣
║ PROD: uvicorn --workers · liveness ≠ readiness · graceful shutdown   ║
║   ORJSONResponse · docs_url=None in prod · CORS never ["*"]+creds    ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Python](../01-languages/python.md) · [API Design](api-design.md) · [Databases](databases.md) · [AppSec](../07-security/appsec.md)
