---
name: hexagonal-ports
description: >
  Rules and patterns for Hexagonal Architecture (Ports & Adapters) by Alistair Cockburn.
  Defines how to separate the application core from driving (primary) and driven (secondary)
  adapters via port interfaces. Use when designing adapters for external systems, testing
  strategies, or validating that the domain is framework-agnostic. Required by Dev Agents
  on infrastructure tasks and the Architecture Analyst (D02).
applies-to: [dev-agent, architecture-analyst, software-architect]
stack: none
layer: none
version: 1.0
---

# Hexagonal Architecture (Ports & Adapters)

<context>
This skill applies when structuring the boundary between an application core and any
external system: databases, HTTP clients, message brokers, UI frameworks, or CLI runners.
The core principle: the application core defines Ports (interfaces expressing its needs);
external systems connect through Adapters (implementations of those interfaces). The core
is always testable without real infrastructure — adapters are swappable.
</context>

<rules>
## Core Isolation
- The application core MUST NOT import any framework, database driver, HTTP client, or messaging library.
- The application core MUST express all external dependencies as Port interfaces.
- The application core MUST be fully testable by replacing adapters with in-memory or test doubles.

## Ports
- Driving ports (primary) MUST be interfaces that external actors (UI, API, CLI) call into the core.
- Driven ports (secondary) MUST be interfaces that the core calls to reach external systems.
- Port interfaces MUST be defined inside the application core, not in adapter packages.
- Port methods MUST use domain language — no persistence or transport terminology in signatures.
- Port interfaces SHOULD be narrow (1–3 methods); broad ports SHOULD be split by use case.

## Adapters
- A driving adapter (e.g., REST controller) MUST translate HTTP requests into core method calls.
- A driven adapter (e.g., SQL repository) MUST implement a driven port interface.
- Adapters MUST NOT contain business logic — only translation between external format and core types.
- Adapter naming SHOULD follow the pattern: `{Technology}{Port}Adapter` (e.g., `PostgresOrderRepository`, `SmtpEmailAdapter`).
- Each adapter MUST be independently replaceable without touching the core.

## Dependency Direction
- Driving adapters MUST depend on the core (call into it).
- The core MUST depend on driven port interfaces, not on adapter implementations.
- Driven adapters MUST depend on the core port interface they implement.
- Dependency injection MUST wire adapters to ports outside the core (Composition Root).
</rules>

<patterns>
## Port definitions (inside application core)

```python
# driven port — core defines what it needs from persistence
from abc import ABC, abstractmethod
from typing import Optional
from domain.order import Order, OrderId

class OrderRepository(ABC):          # driven port
    @abstractmethod
    async def find_by_id(self, order_id: OrderId) -> Optional[Order]: ...

    @abstractmethod
    async def save(self, order: Order) -> None: ...

# driving port — the core exposes this to adapters (e.g., REST)
class OrderService(ABC):             # driving port
    @abstractmethod
    async def place_order(self, customer_id: str, items: list[dict]) -> str: ...
```

## Driven adapter (SQL implementation)

```python
# adapter lives OUTSIDE the core — depends on the port interface
from sqlalchemy.ext.asyncio import AsyncSession
from core.ports import OrderRepository
from domain.order import Order, OrderId
from infrastructure.models import OrderModel

class SqlAlchemyOrderRepository(OrderRepository):   # driven adapter
    def __init__(self, session: AsyncSession):
        self._session = session

    async def find_by_id(self, order_id: OrderId) -> Order | None:
        model = await self._session.get(OrderModel, order_id.value)
        return _to_domain(model) if model else None

    async def save(self, order: Order) -> None:
        model = _to_model(order)
        self._session.add(model)
        await self._session.flush()
```

## Driving adapter (FastAPI controller)

```python
# REST adapter — translates HTTP into core calls
from fastapi import APIRouter, Depends
from core.ports import OrderService

router = APIRouter(prefix="/orders")

@router.post("/", status_code=201)
async def place_order(payload: PlaceOrderRequest, svc: OrderService = Depends()):
    order_id = await svc.place_order(payload.customer_id, payload.items)
    return {"order_id": order_id}
```

## Test with in-memory adapter (no database required)

```python
class InMemoryOrderRepository(OrderRepository):
    def __init__(self):
        self._store: dict[str, Order] = {}

    async def find_by_id(self, order_id: OrderId) -> Order | None:
        return self._store.get(order_id.value)

    async def save(self, order: Order) -> None:
        self._store[order.id.value] = order

# Test — core logic tested with no real DB
async def test_place_order_creates_pending_order():
    repo = InMemoryOrderRepository()
    svc = OrderServiceImpl(repo)
    order_id = await svc.place_order("cust-1", [{"product": "A", "qty": 2}])
    order = await repo.find_by_id(OrderId(order_id))
    assert order.status == "pending"
```
</patterns>

<anti-patterns>
- **Core imports SQLAlchemy / EF Core / Prisma**: the core becomes untestable without a real database; adapters are no longer swappable.
- **Business logic in adapters**: validation, invariant checks, or domain calculations in a REST controller or SQL adapter violates separation; logic cannot be reused or tested without HTTP/DB.
- **Fat ports with 10+ methods**: a port covering all CRUD operations for an entity mixes concerns; split into `OrderReader` and `OrderWriter` or per-use-case ports.
- **Adapter instantiated inside core**: using `new SqlRepository()` inside a use case defeats inversion; always inject through the port interface.
- **Port interface in adapter package**: if the interface lives in the adapter, the core must import from the adapter — direction reversed, hexagon broken.
- **Skipping driving ports**: directly calling core service classes without a port interface prevents swapping the entry point (e.g., adding a CLI or message queue consumer).
</anti-patterns>

<checklist>
- [ ] All port interfaces are declared inside the application core package.
- [ ] No infrastructure library (ORM, HTTP client, broker SDK) is imported in the core.
- [ ] Each driven adapter implements exactly one driven port interface.
- [ ] Core use cases can be tested end-to-end using in-memory adapter stubs (no real I/O).
- [ ] Adapters contain no business logic — only format translation.
- [ ] Dependency injection wiring (adapter → port) occurs only in the Composition Root.
- [ ] Adapter names follow the `{Technology}{Port}` naming convention.
</checklist>
