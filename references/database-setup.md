# Database Setup Rules

## Scope

Use this reference whenever creating or changing:
- DB engine/session setup
- `get_db()` or other DB-scoped dependencies
- transaction boundaries
- repository write behavior
- startup initialization that touches DB

## Canonical Setup

The wired baseline lives in [`fastapi-cookiecutter`](https://github.com/marcius-llmus/fastapi-cookiecutter): `src/core/db/{base,engine,repository}.py` for the `Base`, `DatabaseSessionManager`, and `BaseRepository`, plus `src/container/core/db.py` and `src/container/lifecycle.py` for the app-lifetime session manager. Treat those files as the implementation; this reference governs how to use them.

## Environment-Selected Database URL

Use one `DATABASE_URL` setting per environment profile. The default profile
should be development; `ENVIRONMENT=test` should load the test settings and its
test database URL. Alembic `env.py` should read the active settings profile and
use `settings.DATABASE_URL`.

Do not introduce a separate `MIGRATION_DATABASE_URL`, parallel local migration
URLs, or duplicate Makefile targets such as `migrate-test` /
`alembic-check-test` by default. Keep migration commands environment-agnostic:
`make migrate`, `make alembic-check`, and `make generate-migration`. To run
against another profile, set the environment explicitly, for example:
`ENVIRONMENT=test make migrate`.

If a project already has a separate migration URL convention, follow the
existing convention. If a new privilege split is required, stop and make that
security model explicit before adding new settings or Makefile variables.

## Transaction Boundary Contract

- Commit/rollback occurs at `get_db()` for request-scoped work or at a flow/task-local `sessionmanager.session()` block.
- Services are transaction-unaware.
- Repositories are transaction-unaware.
- Repositories use `flush()` inside the active transaction.
- Normal request-scoped services are built from one active `AsyncSession`.
- `DatabaseSessionManager` is for flows that cannot safely execute inside one unit of work and therefore must open short sessions internally.
- Public service builders accept the active `AsyncSession` explicitly and return normally.
- App-scoped infrastructure collaborators should be owned by the central composition root rather than threaded through every route manually.
- Slow provider/LLM/network calls must not happen while a DB session remains open.

## AsyncSession Concurrency Contract

Do not run concurrent repository or service calls over the same `AsyncSession`.

SQLAlchemy documents `AsyncSession` as a mutable, stateful transaction object. The safe model is one session per asyncio task. A request-scoped service built from `db: AsyncSession = Depends(get_db)` owns one transaction snapshot; repository reads inside that service should be sequential when they share that session.

FastAPI runs on Starlette's ASGI stack, where AnyIO is the native concurrency compatibility layer. For non-DB task fan-out inside flows, prefer `anyio.create_task_group()` over raw `asyncio.gather()` or direct `asyncio.TaskGroup`. Keep `asyncio` imports out of application flow code unless an underlying library explicitly requires an asyncio-only primitive.

### Good — One Request Session, Sequential Reads

```python
class RunProjectionService:
    async def read_world_map(self, *, run_id: UUID) -> RunWorldMapRead:
        subjects = await self._subject_repository.list_for_run(run_id=run_id)
        beliefs = await self._belief_repository.list_for_run(run_id=run_id)
        notes = await self._note_repository.list_for_run(run_id=run_id)
        return self._build_projection(
            subjects=subjects,
            beliefs=beliefs,
            notes=notes,
        )
```

### Bad — Shared AsyncSession In Concurrent Tasks

```python
subjects, beliefs, notes = await asyncio.gather(
    self._subject_repository.list_for_run(run_id=run_id),
    self._belief_repository.list_for_run(run_id=run_id),
    self._note_repository.list_for_run(run_id=run_id),
)
```

If true DB concurrency is required, each concurrent task must open and close its own short session through the session manager and accept that it may observe a different transaction snapshot. Do not add a flow or session manager to a normal request service just to parallelize ordinary repository reads.

### Good — Flow Fan-Out With AnyIO And Per-Task Sessions

```python
from anyio import create_task_group


async with create_task_group() as task_group:
    for worker in workers:
        async def run_worker(*, worker_id: UUID = worker.id) -> None:
            await execute_worker(worker_id=worker_id)

        task_group.start_soon(run_worker)
```

The worker function may call provider/network code and then open its own short `sessionmanager.session()` blocks for persistence. It must not share a request-scoped `AsyncSession` with sibling tasks.

## Request DB Dependency

`get_db()` lives in `src/container/core/dependencies.py` because it adapts FastAPI request scope to the container-owned session manager. Keep `src/core/db/` focused on DB primitives (`Base`, `DatabaseSessionManager`, repository bases) — never put FastAPI dependency adapters there.

Routes and FastAPI auth dependencies use `Depends(get_db)`. Tasks and flows use `async with sessionmanager.session() as db:` instead.

## Repository Write Contract

### Good

```python
async def create(self, obj_in: EntityCreate) -> Entity:
    db_obj = self.model(**obj_in.model_dump())
    self.db.add(db_obj)
    await self.db.flush()
    return db_obj
```

### Bad (Do Not Do This)

```python
async def create(self, obj_in: EntityCreate) -> Entity:
    db_obj = self.model(**obj_in.model_dump())
    self.db.add(db_obj)
    await self.db.commit()
    return db_obj
```

### Update Pattern

```python
async def update(self, *, db_obj: Entity, obj_in: EntityUpdate) -> Entity:
    update_data = obj_in.model_dump(exclude_unset=True)
    for key, value in update_data.items():
        setattr(db_obj, key, value)
    await self.db.flush()
    await self.db.refresh(db_obj)
    return db_obj
```

`refresh()` is the default so callers receive DB-generated fields (`id`, `created_at`, `updated_at`) without a second round trip in the service. Drop it only if profiling shows the extra `SELECT` is wasteful for a specific entity.

## Service Contract Around DB

### Good

```python
class EntityService:
    async def update(self, *, entity_id: int, entity_in: EntityUpdate) -> EntityRead:
        db_obj = await self.repo.get(entity_id)
        updated = await self.repo.update(db_obj=db_obj, obj_in=entity_in)
        return EntityRead.model_validate(updated)
```

### Bad

```python
class EntityService:
    async def update(self, *, entity_id: int, entity_in: EntityUpdate) -> EntityRead:
        updated = await self.repo.update(...)
        await self.repo.db.commit()
        return EntityRead.model_validate(updated)
```

## Startup/Lifespan DB Access

- Startup hooks may run initialization through an async session context.
- Prefer a stable app-lifetime session manager initialized from settings through `src/container/lifecycle.py` and accessed through `src/container/core/db.py` runtime helpers.
- Keep initialization logic in services/repositories where possible.
- Avoid bypassing service boundaries for cross-module writes.
- Do not inject `DatabaseSessionManager` into normal request-scoped services or routes. Reserve it for flows, workers, schedulers, and other long-running entrypoints.

## Test DB Expectations

- Repository tests run against real test DB/session fixtures.
- Do not mock repository DB calls in repository tests.
- Use transaction rollback fixture strategy per test function.
