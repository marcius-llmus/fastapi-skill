# Capability Lifecycle

## The Three Stages

Capabilities graduate through stages as they grow. This prevents premature abstraction — start simple, promote when evidence demands it.

## Stage 1: App-Internal

Used by only one app. Lives inside that app. No protocol needed.

```
apps/orders/
├── repositories/
│   └── items.py       # only orders uses it
└── services/
    └── processing.py  # only orders uses it
```

No shared code, no protocols. Keep it simple.

This includes app-local data/state collaborators such as Redis-backed counters,
denylist stores, rate-limit stores, or other non-ORM persistence helpers. If
only one app uses them and there is only one implementation, keep them as
concrete app-owned classes first. Do not extract a shared protocol just because
the backing store could change later.

## Stage 2: Shared Capability

Used by 2+ apps, no API needed. Extract the contract to `shared/`.

```
shared/embedding/
├── protocols.py       # EmbeddingProvider contract
└── service.py         # EmbeddingService (retries, logging)
```

Both consuming apps depend on the protocol. The container wires the implementation.

## Stage 3: Graduated App

Needs its own API, business logic, persistence. Becomes a full app. Its protocol stays in `shared/` so consumers keep working.

```
shared/monitoring/         # contract stays here
├── protocols.py           # MonitoringObserver
└── schemas.py             # MonitoringEvent DTO

apps/monitoring/           # implementation moves here
├── models.py              # internal ORM models
├── repositories/
│   └── events.py          # EventRepository
├── services/
│   └── monitor.py         # MonitoringService (implements MonitoringObserver)
├── routes.py              # HTTP endpoints
└── tasks.py               # optional background/task entrypoints
```

## Walkthrough: Monitoring Example

### Stage 1 — App-internal

Orders logs events internally. No protocol, no shared code.

```python
# apps/orders/services/order.py
class OrderService:
    def _log_operation(self, op: str, result: bool) -> None:
        self._logs.append({"op": op, "success": result, "ts": time.time()})
```

### Stage 2 — Extract to shared

Payments also needs event logging. Extract the contract:

```python
# shared/monitoring/protocols.py
class MonitoringObserver(Protocol):
    async def record(self, event: MonitoringEvent) -> None: ...

# shared/monitoring/schemas.py
from src.core.schemas import BaseModel


class MonitoringEvent(BaseModel):
    source: str
    action: str
    timestamp: int
```

Both apps depend on `MonitoringObserver` (injected).

### Stage 3 — Promote to app

Monitoring needs its own API and database. Promote to app:

```python
# apps/monitoring/services/monitor.py
class MonitoringService:
    def __init__(self, repository: EventRepository):
        self._repository = repository

    async def record(self, event: MonitoringEvent) -> None:
        await self._repository.save(event)

# wiring
order_service = OrderService(observer=monitoring_service)
payment_service = PaymentService(observer=monitoring_service)
```

Orders and payments never imported `apps/monitoring/`. The protocol is the boundary.

## Decision Criteria

| Signal | Action |
|---|---|
| Only one app uses it | Keep app-internal (Stage 1) |
| 2+ apps need it, no persistence/API needed | Extract to `shared/` (Stage 2) |
| Needs its own routes, DB, or complex logic | Graduate to app (Stage 3) |
| "Maybe we'll need it later" | Do nothing. Don't abstract until evidence demands it |
| "This Redis/data helper might be swapped later" | Keep the concrete app-local class until a second implementation or shared boundary appears |

## Rules

- Never promote speculatively. Promote when the second consumer appears.
- Protocol and DTOs always live in `shared/` (even at Stage 3) — they're the contract.
- Implementation always lives in `apps/` or `infrastructure/` — never in `shared/`.
- When in doubt, keep it app-internal. You can always promote later.
