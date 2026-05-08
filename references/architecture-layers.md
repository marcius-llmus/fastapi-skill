# Architecture Layers

## The Four Layers

```
src/
├── core/
│   ├── config.py      # fixed settings and app-wide configuration
│   ├── schemas.py     # project Pydantic bases: BaseModel, ReadModel
│   ├── db/            # fixed DB primitives: Base, session manager, repository base
│   ├── redis/         # fixed Redis primitives when Redis is a chosen app primitive
│   └── celery/        # fixed task app when Celery is a chosen app primitive
├── domain/            # DOMAIN — pure models, logic, protocols (innermost ring)
├── shared/            # CONTRACTS — protocols, DTOs, shared services
├── infrastructure/    # REPLACEABLE OUTER IMPLEMENTATIONS — shared protocol adapters
├── apps/              # APPLICATIONS — each owns its full vertical slice
```

## Layer Responsibilities

| Layer | Contains | Imports |
|---|---|---|
| `core/` | Fixed app primitives and fixed technology choices. Config, schema bases, DB/Redis/Celery primitives. | Should stay low-level and stable |
| `domain/` | Pure models, value objects, protocols. No I/O. | NOTHING |
| `shared/` | Protocols, DTOs, thin wrapper services (retries, logging) | `domain/` only |
| `infrastructure/` | Replaceable protocol implementations and external adapters | `shared/` + `domain/` |
| `apps/<name>/` | Vertical slices — models, repos, services, flows, routes, tasks | All layers, NEVER other apps |

## Dependency Flow (NEVER violate)

```
core/           → foundational fixed primitives only.
domain/         → NOTHING. Pure inner ring.
shared/         → imports domain/ only.
                  NEVER imports infrastructure/ or apps/.
infrastructure/ → implements shared/ protocols. Can import domain/.
                  NEVER calls apps/.
apps/           → uses core/ + domain/ + shared/ + infrastructure/.
                  NEVER imports other apps.
```

## Concentric Ring Model

```
    +---------------------------------------+
    |  infrastructure/  (outermost ring)    |  replaceable outer adapters
    |  +-----------------------------+      |
    |  |  shared/  (contracts layer) |      |  protocols, DTOs, shared services
    |  |  +---------------------+    |      |
    |  |  |  domain/  (pure)    |    |      |  models, logic, protocols
    |  |  +---------------------+    |      |
    |  +-----------------------------+      |
    +---------------------------------------+

    core/ sits beside the rings as fixed app foundation.
    apps/ sits OUTSIDE the rings — each app composes
    core/ plus all layers for a specific use case.
```

## What Lives Where — Decision Table

| Question | Answer |
|---|---|
| Is it a fixed app primitive or fixed technology choice the whole backend is built on? | `core/` |
| Is it a pure domain model, value object, or protocol? | `domain/` |
| Is it a contract between layers (protocol + DTOs)? | `shared/` |
| Is it a replaceable external capability or protocol implementation? | `infrastructure/` |
| Is it business logic or app-specific? | `apps/<name>/` |
| Is it env vars or configuration? | `src/core/config.py` |
| Is it a project Pydantic base? | `src/core/schemas.py` |
| Is it the ORM declarative base or DB session/repository primitive? | `src/core/db/` |

Examples:

- `core/`: `settings`, `Base`, `DatabaseSessionManager`, Redis/Celery primitives
- `infrastructure/`: `AsaasPaymentAdapter`, `TwilioWhatsAppAdapter`, HTTP clients or queue adapters implementing shared protocols

If the project wants explicit startup/shutdown ownership for fixed primitives,
top-level lifecycle wrappers may live in `src/container/lifecycle.py`, while the
initialized singleton instances and runtime/build helpers may live in
`src/container/core/`. The primitive types/packages still live in `src/core/`.

## Domain and Shared Are Libraries, Not Services

`domain/` and `shared/` are **internal packages** — never deployed alone, never expose HTTP. They ship inside every app.

| Layer | Deployable? |
|---|---|
| `domain/` | No — ships inside apps |
| `shared/` | No — ships inside apps |
| `infrastructure/` | No — ships inside apps |
| `apps/<name>/` | **Yes** — this is the deployable unit |

## App Internal Structure

Each app owns its full vertical slice:

```
apps/<name>/
├── models.py              # ORM models, enums, app-specific types
├── schemas.py             # Pydantic DTOs (Create, Update, Internal, Read)
├── exceptions.py          # Domain exceptions
├── repositories/          # one module per model/aggregate
│   ├── items.py
│   └── categories.py
├── services/              # multiple services with distinct responsibilities
│   ├── processing.py
│   └── analysis.py
├── flows/                 # optional — only for non-unit-of-work processes
│   └── recovery.py
├── config.py              # app constants
├── routes.py              # FastAPI router
└── tasks.py               # optional — background/task entrypoints
```

Central composition does **not** live in each app module. Keep one project-wide
composition root such as a `src/container/` package, `src/container.py`, or
`src/composition/`.

## Apps Never Import Other Apps

Apps communicate through protocols defined in `shared/`. The container wires implementations.

```python
# shared/monitoring/protocols.py — the contract
class MonitoringObserver(Protocol):
    async def record(self, event: MonitoringEvent) -> None: ...

# apps/orders/services/order.py — uses the protocol
class OrderService:
    def __init__(self, observer: MonitoringObserver):  # injected
        self._observer = observer

# apps/monitoring/services/monitor.py — implements the protocol
class MonitoringService:
    async def record(self, event: MonitoringEvent) -> None:
        await self._repository.save(event)

# wiring — container
order_service = OrderService(observer=monitoring_service)
```

## Monolith to Microservices

When an app needs to become an independent service:

1. The app already owns its full vertical slice — nothing to extract
2. `shared/` protocols become the service contract (API boundary)
3. Swap local implementation for HTTP client — same protocol

```python
# Before (monolith): local call
order_service = OrderService(observer=monitoring_service)

# After (microservices): HTTP call through same protocol
order_service = OrderService(observer=MonitoringHttpClient("http://monitoring:8000"))
```

No business logic changes. The protocol is the seam.

## Schema Placement Rule

- Every project Pydantic model lives in a schema module.
- This includes `*Internal` DTOs used between service and repository boundaries.
- Do not declare Pydantic models in services, repositories, or utility modules just because the DTO is "internal".

## Core Foundations

`core/` is not "everything infrastructural". It is the fixed foundation of the
backend:

- app-wide primitives the whole codebase is built on
- fixed technology choices you are not modeling as protocol seams
- usually low-level connection or platform objects, plus project-wide bases

Put something in `core/` when all of these are true:

1. The whole backend relies on it as a foundational primitive.
2. You are not treating it as a replaceable capability behind a protocol.
3. It should stay stable, boring, and globally understandable.

Examples that fit:

- `src/core/config.py`: settings and runtime configuration
- `src/core/schemas.py`: project-wide Pydantic bases
- `src/core/db/`: DB primitives shared by the whole backend
- `src/core/redis/`: Redis primitives and Redis-owned helpers when Redis is a fixed app primitive
- `src/core/celery/`: Celery app when task execution is a fixed platform choice

When the project chooses explicit lifecycle ownership, `src/container/lifecycle.py`
may own app startup/shutdown wrappers, and `src/container/core/` may own the
initialized app-scoped instances plus `init_*` / `build_*` / `close_*` helpers
for those fixed primitives. That does not move the primitive types themselves
out of `core/`.

## Import Surface Policy

Container packages must not use `__init__.py` files as convenience export
barrels. Keep `src/container/__init__.py` and `src/container/core/__init__.py`
empty unless a narrow package-level concern requires otherwise. Routes,
lifespan code, flows, and tasks import concrete modules directly:

```python
from src.container.actions import build_execute_action_flow
from src.container.core.dependencies import get_db
from src.container.lifecycle import close_core, init_core
```

Stable `core` package exports are acceptable when they expose fixed primitives
or settings helpers, for example `Base`, `BaseRepository`,
`DatabaseSessionManager`, and `settings`. Do not export container builders,
runtime lifecycle wrappers, app services, or FastAPI dependency adapters from
`core` package `__init__.py` files.

Examples that do not fit:

- payment gateway implementations
- WhatsApp providers
- LLM provider adapters
- other replaceable external capability implementations

Those belong in `infrastructure/` or explicit container composition, not in
`core/`.

## FastAPI App Entrypoint

`src/main.py` defines `create_app()` and exposes the module-level ASGI
instance that runners target as `src.main:app`:

```python
def create_app() -> FastAPI:
    application = FastAPI(...)
    # middleware, routers, exception handlers...
    return application


app = create_app()
```

Use `application` (or any name other than `app`) as the local variable
inside `create_app()`. Reusing `app` shadows the module-level `app`
created by `app = create_app()` at the bottom of the file and produces
spurious "shadows name from outer scope" warnings.
