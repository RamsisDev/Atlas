---
name: python-fastapi
description: >
  Python FastAPI patterns for building production-grade APIs: Pydantic v2 models,
  dependency injection, router organization, async patterns, lifespan events, and
  OpenAPI generation. Based on FastAPI official documentation and the Full Stack FastAPI
  Template. Use for any task creating or modifying FastAPI endpoints, dependencies,
  schemas, or application startup. Required by Dev Agents on Python API-layer tasks.
applies-to: [dev-agent, qa-agent]
stack: python
layer: api
version: 1.0
---

# Python FastAPI

<context>
This skill applies when implementing a Python API using FastAPI. It covers the
standard patterns: Pydantic v2 for request/response schemas, FastAPI's dependency
injection for services and auth, APIRouter for modular route organization, async
database access, lifespan for startup/shutdown, and automatic OpenAPI generation.
</context>

<rules>
## Routing and Organization
- Routes MUST be organized into `APIRouter` instances, one per domain area, and included in `app` via `include_router`.
- Router files MUST NOT contain business logic — dispatch to service classes or use cases.
- Route path parameters MUST be typed: `item_id: int`, `order_id: UUID` — FastAPI validates and rejects malformed input automatically.
- HTTP methods MUST follow REST semantics: GET (read), POST (create), PUT/PATCH (update), DELETE (remove).
- Response models MUST be declared on every endpoint via `response_model=` to strip sensitive fields from output.

## Pydantic v2 Schemas
- All request bodies MUST be Pydantic `BaseModel` subclasses with explicit field types and constraints.
- All response bodies MUST be Pydantic `BaseModel` subclasses; never return raw dicts from endpoints.
- Request and response schemas MUST be separate classes — never reuse the same model for both.
- Optional fields MUST use `Optional[T]` or `T | None` with a `default=None`; required fields have no default.
- Validators MUST use `@field_validator` (v2 API) — not `@validator` (deprecated v1 API).
- `model_config = {"strict": True}` SHOULD be set on request schemas to prevent silent coercions.

## Dependency Injection
- Database sessions MUST be provided via `Depends(get_db)` — never instantiated inside endpoint functions.
- Authentication MUST be implemented as a FastAPI dependency (`Depends(get_current_user)`).
- Services and repositories MUST be injected via `Depends()` — not imported and called as module-level singletons.
- Dependencies MUST be async if they perform I/O.

## Async
- All endpoint functions MUST be declared `async def` when performing I/O (database, HTTP calls, file I/O).
- Synchronous blocking calls MUST NOT be made inside `async def` endpoints — use `run_in_executor` or an async library.
- Database access MUST use an async driver: `asyncpg`, `aiosqlite`, `motor`, or SQLAlchemy with `AsyncSession`.

## Error Handling
- HTTP errors MUST be raised with `HTTPException(status_code=..., detail=...)`.
- Domain exceptions MUST be caught in a global exception handler registered with `app.add_exception_handler()`.
- Unhandled exceptions MUST return a generic 500 response — never expose tracebacks in production.
- Validation errors (Pydantic `RequestValidationError`) return 422 automatically — do not re-raise.

## Lifespan
- Startup and shutdown logic MUST use the `@asynccontextmanager` lifespan pattern — not `@app.on_event("startup")` (deprecated).
</rules>

<patterns>
## Router Organization

```python
# api/routers/orders.py
from fastapi import APIRouter, Depends, HTTPException, status
from uuid import UUID
from app.schemas.order import PlaceOrderRequest, OrderDetailResponse
from app.services.order_service import OrderService
from app.dependencies import get_order_service, get_current_user

router = APIRouter(prefix="/orders", tags=["Orders"])

@router.post("/", response_model=dict, status_code=status.HTTP_201_CREATED)
async def place_order(
    body: PlaceOrderRequest,
    service: OrderService = Depends(get_order_service),
    user: dict = Depends(get_current_user),
):
    order_id = await service.place_order(user["sub"], body)
    return {"id": str(order_id)}

@router.get("/{order_id}", response_model=OrderDetailResponse)
async def get_order(
    order_id: UUID,
    service: OrderService = Depends(get_order_service),
    user: dict = Depends(get_current_user),
):
    order = await service.get_order(order_id, user["sub"])
    if order is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Order not found.")
    return order
```

## Pydantic v2 Schemas

```python
# schemas/order.py
from pydantic import BaseModel, Field, field_validator
from uuid import UUID
from decimal import Decimal
from typing import Annotated

class OrderLineRequest(BaseModel):
    model_config = {"strict": True}
    product_id: UUID
    quantity: Annotated[int, Field(gt=0, le=1000)]
    unit_price: Annotated[Decimal, Field(gt=0, decimal_places=2)]

class PlaceOrderRequest(BaseModel):
    model_config = {"strict": True}
    lines: list[OrderLineRequest] = Field(min_length=1)

class OrderLineResponse(BaseModel):
    product_id: UUID
    quantity: int
    unit_price: Decimal
    subtotal: Decimal

class OrderDetailResponse(BaseModel):
    id: UUID
    status: str
    total: Decimal
    lines: list[OrderLineResponse]
    created_at: str
```

## Dependency Injection

```python
# dependencies.py
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_async_session
from app.repositories.order_repository import SqlAlchemyOrderRepository
from app.services.order_service import OrderService

async def get_order_service(
    db: AsyncSession = Depends(get_async_session)
) -> OrderService:
    repo = SqlAlchemyOrderRepository(db)
    return OrderService(repo)

# database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

engine = create_async_engine(settings.DATABASE_URL, echo=False)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_async_session():
    async with AsyncSessionLocal() as session:
        yield session
```

## Lifespan (startup/shutdown)

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)  # or run alembic
    yield
    # Shutdown
    await engine.dispose()

app = FastAPI(title="My API", lifespan=lifespan)
app.include_router(orders_router)
app.include_router(auth_router)
```

## Global Exception Handler

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from app.domain.exceptions import DomainException, NotFoundException

@app.exception_handler(DomainException)
async def domain_exception_handler(request: Request, exc: DomainException):
    return JSONResponse(status_code=422, content={
        "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
        "title": "Business Rule Violation",
        "status": 422,
        "detail": str(exc),
    })

@app.exception_handler(NotFoundException)
async def not_found_handler(request: Request, exc: NotFoundException):
    return JSONResponse(status_code=404, content={"detail": str(exc)})
```
</patterns>

<anti-patterns>
- **Business logic in endpoint functions**: computing discounts, enforcing domain rules inside `@router.post()` — endpoint must only validate, dispatch, and map response.
- **Raw dict return from endpoint**: `return {"id": id, "status": status}` without `response_model` — sensitive fields may leak; OpenAPI schema is not generated.
- **Reusing request schema as response schema**: `response_model=PlaceOrderRequest` — request fields (passwords, internal IDs) appear in responses.
- **Synchronous DB call inside async endpoint**: `session.execute(query)` (sync) inside `async def` — blocks the event loop, serializes all requests.
- **Module-level DB session**: `db = SessionLocal()` at module import time — sessions are not thread-safe and leak across requests.
- **`@validator` instead of `@field_validator`**: Pydantic v1 API deprecated in v2; v1 validators silently misbehave with v2 models.
- **`@app.on_event("startup")` deprecated handler**: use `lifespan` context manager for startup/shutdown — `on_event` is removed in FastAPI future versions.
- **Missing `response_model` stripping**: internal fields (password_hash, internal_id) returned to client because response model is not declared.
</anti-patterns>

<checklist>
- [ ] All endpoint functions are `async def`.
- [ ] All request bodies are `BaseModel` with typed, constrained fields.
- [ ] Separate request and response schemas — not reused.
- [ ] `response_model=` declared on all endpoints.
- [ ] Database session provided via `Depends(get_async_session)`.
- [ ] Authentication via `Depends(get_current_user)` — not ad-hoc token parsing in each endpoint.
- [ ] Domain exceptions handled by global exception handler → structured JSON response.
- [ ] No synchronous I/O in `async def` functions.
- [ ] Lifespan context manager used for startup/shutdown logic.
</checklist>
