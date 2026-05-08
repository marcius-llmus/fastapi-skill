# Scoped Resources and Service Construction

## Purpose

Use this reference when you need to reason about any resource whose lifetime
must stay open across a route call or a task step.

Typical examples:
- `AsyncSession`
- a per-request audit buffer
- a scoped unit-of-work helper
- any future request/job-scoped collaborator that is not app-scoped

This reference separates three concerns that often get blurred together:

1. object graph composition
2. scoped resource lifetime
3. FastAPI dependency adaptation

## Default Rule

If a service receives an `AsyncSession`, the caller owns that session lifetime.

That means:
- routes get `db` from `Depends(get_db)`
- tasks and flows open `async with sessionmanager.session() as db:` after
  fetching the runtime-managed session manager from the central composition
  root when the project uses explicit lifecycle ownership
- public service builders accept `db` and return normally

This is the default pattern because it keeps route and task composition symmetric.

## File-by-File Example: Route-Scoped Service Plus Task Usage

### `src/container/core/dependencies.py`

```python
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncSession
from src.container.core.db import build_session_manager


async def get_db() -> AsyncIterator[AsyncSession]:
    # FastAPI request DB lifetime belongs with container-owned core runtime wiring.
    sessionmanager = build_session_manager()
    async with sessionmanager.session() as db:
        yield db
```

### `src/container/notifications.py`

```python
from sqlalchemy.ext.asyncio import AsyncSession

from src.apps.notifications.repositories.emails import EmailRepository
from src.apps.notifications.services.emails import EmailService


def _build_email_repository(*, db: AsyncSession) -> EmailRepository:
    return EmailRepository(db=db)


def build_email_service(*, db: AsyncSession) -> EmailService:
    return EmailService(
        email_repository=_build_email_repository(db=db),
    )
```

What this means:
- `_build_email_repository` is internal composition only
- `build_email_service` is the public service builder
- neither function owns DB lifetime

### `src/apps/notifications/routes.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from src.apps.notifications.exceptions import EmailNotFound
from src.apps.notifications.schemas import EmailRead, SendEmailRequest
from src.container.core.dependencies import get_db
from src.container.notifications import build_email_service

router = APIRouter(prefix="/notifications", tags=["notifications"])


@router.post("/send", response_model=EmailRead)
async def send_email(
    payload: SendEmailRequest,
    db: AsyncSession = Depends(get_db),
) -> EmailRead:
    service = build_email_service(db=db)
    try:
        return await service.send_email(payload=payload)
    except EmailNotFound as exc:
        raise HTTPException(status_code=404, detail=str(exc)) from exc
```

### `src/apps/notifications/tasks.py`

```python
import asyncio

from celery import Task

from src.container.notifications import build_email_service
from src.container.core.db import build_session_manager
from src.core.celery.app import celery_app


@celery_app.task(name="src.apps.notifications.tasks.send_saved_email", bind=True)
def send_saved_email(self: Task, email_id: int) -> None:
    async def _run() -> None:
        sessionmanager = build_session_manager()
        async with sessionmanager.session() as db:
            service = build_email_service(db=db)
            await service.send_saved_email(email_id=email_id)

    asyncio.run(_run())
```

The service stays DB-bound in both places. The only difference is who opens and closes the session.

## Why a Direct Service Call From a Sync Task Needs a Coroutine Boundary

Celery task bodies are synchronous in this pattern:

```python
@celery_app.task(...)
def send_saved_email(...) -> None:
    ...
```

But both of these are async:
- `async with sessionmanager.session() as db:`
- `await service.send_saved_email(...)`

So a sync task cannot execute the service call directly. It needs a coroutine
for `asyncio.run(...)` to drive. A small nested `async def _run(): ...` is the
minimal shape when the task is calling a service directly.

This is different from a task that calls a flow:

```python
@celery_app.task(...)
def classify_inbound_message(...) -> None:
    asyncio.run(build_classify_inbound_message_flow().execute(message_id=message_id))
```

No intermediary helper is needed there because `flow.execute(...)` is already
the coroutine passed to `asyncio.run(...)`.

Use these rules:
- direct async service call from sync task: needs a coroutine boundary
- flow call from sync task: call `asyncio.run(flow.execute(...))` directly
- if the task system itself supports async task bodies, neither wrapper is needed

## Why `get_db()` Uses `yield`

`get_db()` is the DB lifetime owner for the request:

```python
async def get_db() -> AsyncIterator[AsyncSession]:
    # FastAPI request DB lifetime belongs with container-owned core runtime wiring.
    sessionmanager = build_session_manager()
    async with sessionmanager.session() as db:
        yield db
```

FastAPI opens the session before the route body runs, pauses at `yield`, lets the route and service use `db`, then resumes and closes the session after the route returns or raises.

The service builder does **not** need `yield`, because the builder is not the lifetime owner anymore.

## When to Use a Flow Instead

If the use case includes slow external I/O, do not keep the request `db` open across that call.

Use a flow/orchestrator with explicit phases:

```python
class GenerateArtifactFlow:
    def __init__(
        self,
        *,
        session_manager: DatabaseSessionManager,
        payment_provider: PaymentProvider,
    ) -> None:
        self._session_manager = session_manager
        self._payment_provider = payment_provider

    async def execute(self, *, payment_id: int) -> ArtifactRead:
        async with self._session_manager.session() as db:
            service = build_payment_artifact_service(db=db)
            request = await service.prepare(payment_id=payment_id)

        artifact = await self._payment_provider.generate_artifact(request=request)

        async with self._session_manager.session() as db:
            service = build_payment_artifact_service(db=db)
            return await service.persist_result(
                payment_id=payment_id,
                artifact=artifact,
            )
```

That is the correct pattern when the operation crosses DB state and external systems.

## Optional Shared Scope Pattern

If you later introduce a second request/job-scoped collaborator and do not want
routes and jobs to nest multiple context managers manually, promote the lifetime
handling into one reusable scope object.

### `src/container/core/scope.py`

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from sqlalchemy.ext.asyncio import AsyncSession

from src.apps.audit.buffer import AuditBuffer, audit_buffer_scope
from src.container.core.db import build_session_manager


class RequestScope:
    def __init__(self, *, db: AsyncSession, audit: AuditBuffer) -> None:
        self.db = db
        self.audit = audit


@asynccontextmanager
async def request_scope() -> AsyncIterator[RequestScope]:
    sessionmanager = build_session_manager()
    async with sessionmanager.session() as db:
        async with audit_buffer_scope() as audit:
            yield RequestScope(db=db, audit=audit)


async def get_request_scope() -> AsyncIterator[RequestScope]:
    async with request_scope() as scope:
        yield scope
```

### `src/container/notifications.py`

```python
def build_email_service(*, scope: RequestScope) -> EmailService:
    return EmailService(
        email_repository=EmailRepository(db=scope.db),
        audit_buffer=scope.audit,
    )
```

### `src/apps/notifications/routes.py`

```python
@router.post("/send")
async def send_email(scope: RequestScope = Depends(get_request_scope)) -> None:
    service = build_email_service(scope=scope)
    await service.send_email(...)
```

### `src/apps/notifications/tasks.py`

```python
async def _run_send_email(*, email_id: int) -> None:
    async with request_scope() as scope:
        service = build_email_service(scope=scope)
        await service.send_saved_email(email_id=email_id)
```

## When to Introduce a Shared Scope

Do not start with a `RequestScope` just because it is possible.

Start with the default pattern when:
- `AsyncSession` is the only scoped dependency
- flows/tasks can cleanly open one local session
- `build_*(db=db)` is still easy to read

Introduce a shared scope when:
- you have two or more scoped resources that always travel together
- route and task composition would otherwise duplicate nested context managers
- the scope object is mechanical and not carrying business rules

## Anti-Patterns

- public service builders that secretly open/close the DB session
- routes using `Depends(build_service)` for DB-bound services
- tasks reaching into repositories directly instead of going through builders
- service methods that keep DB open while waiting on provider/LLM/network calls
