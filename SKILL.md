---
name: fastapi-backend-architecture
description: |
  MUST read this skill for ANY kind of interaction with a FastAPI project. Use for FastAPI backend projects and backend Python work that runs behind FastAPI. Trigger whenever the repo imports `fastapi` or `starlette`, lists FastAPI in dependencies, contains patterns like `routes.py`, `dependencies.py`, `services/`, `flows/`, `tasks.py`, `container/`, or `repositories/`, or when the user mentions FastAPI routes, services, repositories, flows, tasks, DTOs, schemas, dependency injection, transaction boundaries, or composition/container wiring. Trigger again whenever ongoing work expands into a new FastAPI layer or boundary that was not part of the initial read. Enforces layered architecture, DTO-based service contracts, centralized session and transaction ownership, module isolation, protocol-based DI, required reference usage, and layer-specific testing. Skip only when the project has no FastAPI footprint.
---

# FastAPI Backend Architecture

## Purpose

Apply a strict architecture for FastAPI backends where:
- routes translate HTTP input/output only
- services own business logic and single-unit-of-work orchestration
- flows own multi-phase application logic that cannot safely run inside one DB session
- repositories own persistence access only
- `get_db()` in `src/container/core/dependencies.py` and flow-local session contexts own DB lifetime
- project Pydantic bases live in `src/core/schemas.py`
- protocols define swappable contracts between layers
- apps are isolated vertical slices that never import each other
- tests validate each layer with the correct mocking boundary

## Non-Negotiable Rules

1. Keep dependencies one-way: `route -> service -> repository -> db`.
2. Services and repositories are transaction-unaware (`no commit/rollback`).
3. Services never import `fastapi`, `starlette`, or `HTTPException`.
4. Services raise domain exceptions; routes map them to HTTP exceptions.
5. Cross-module interactions must be service-to-service (or facade/orchestrator), never repository-to-repository.
6. Service methods must receive and return Pydantic DTOs (`DTO in -> DTO out`).
7. Public service methods return DTOs only for successful outcomes. Any error path must raise a domain exception instead of returning a success/failure wrapper or failed DTO result.
8. All project Pydantic models must live in schema modules. Do not define `BaseModel` classes in services, repositories, or other non-schema files, even for `*Internal` DTOs.
9. All project schemas must inherit from `src.core.schemas.BaseModel` or `src.core.schemas.ReadModel`, not directly from Pydantic `BaseModel`.
10. Use `ReadModel` only for read DTOs that benefit from `model_validate(obj)` against matching object attributes. Use `BaseModel` for create/update/internal DTOs and for assembled response payloads that are not direct object projections.
11. `core/` means fixed app primitives and fixed technology choices, not "all infrastructure". Put something in `core/` only when the whole app is built on it and you are not treating it as a swappable seam.
12. The database foundation lives in `src/core/db/` as a stable package. Do not place session/base/repository primitives under `infrastructure/sqlalchemy`.
13. Repository write methods must stay transaction-unaware and must not use `add()`/`refresh()` in the generic pattern.
14. Repository tests use real DB interaction; do not mock DB calls in repository tests.
15. Apps NEVER import other apps. Communication through `shared/` protocols only.
16. `domain/` imports NOTHING. `shared/` imports only `domain/`. `infrastructure/` implements `shared/` protocols. These dependency rules are never violated.
17. Use `typing.Protocol` for things you actually swap (LLM providers, auth backends, external APIs). Skip protocols where the ORM is already the abstraction.
18. Don't abstract until evidence demands it. Capabilities start app-internal and promote only when a second consumer appears.
19. Repository dependencies must use explicit names such as `customer_repository` or `payment_repository`. Never use generic constructor/provider names like `repository`.
20. Do not keep compatibility alias parameters such as both `repository` and `customer_repository` in the same constructor or function. Pick the explicit name and update every call site.
21. If a service can be built request-scoped or job-scoped from repositories and collaborators, build it at the composition boundary. Do not let the service own `DatabaseSessionManager` just for convenience.
22. App services must not call global `get_*()` protocol locators in normal business flow when that dependency can be constructor-injected by the composition root.
23. A `Service` is unit-of-work logic. It must operate correctly inside a single active `AsyncSession` and may orchestrate other services as long as the whole operation still fits one unit of work.
24. A `Flow` exists only when the operation cannot safely run inside a single unit of work. Flows own `DatabaseSessionManager`, open short sessions internally, and coordinate multiple phases over time.
25. Routes and FastAPI dependencies must not depend on `DatabaseSessionManager`. Normal request handling should build DB-bound services from `db: AsyncSession = Depends(get_db)`, where `get_db` is imported from `src.container.core.dependencies`.
26. Routes should not put flows in `Depends(...)`. If a request must synchronously coordinate DB work around slow external I/O, the route may build/invoke a flow explicitly, but that flow must own the short DB sessions itself.
27. Tasks and other long-running entrypoints may call services when they open a short-lived session locally, or call flows when the process spans multiple transactions, waits on external systems, retries, or should not keep one session open for the whole run.
28. Do not create a flow merely because code coordinates multiple services. If one `AsyncSession` is still the correct lifetime, it remains a service.
29. Composition must live in one central composition root (`src/container/`, `container.py`, or a dedicated `composition/` package). Do not scatter real wiring logic across app dependency modules, tasks, and ad hoc helpers.
30. Flows may call services, but must not call other flows in normal design. If logic is shared across flows, extract a service or helper instead of nesting flows.
31. After a flow opens a short-lived session, it must obtain session-bound services from central container composition helpers. Do not instantiate repositories or service graphs inline inside flow methods.
32. Public container `build_*` functions are the entrypoints for routes, flows, and task entrypoints.
33. Only the central composition root defines `build_*` functions. Routes, services, flows, repositories, and adapters must not define their own competing construction paths.
34. Adapter classes must adapt or implement a protocol boundary only. They must not construct inner service graphs inside request methods. If an adapter needs another service, inject that service from the composition root.
35. The composition root exists to encapsulate construction and remove boilerplate. It must not hold business rules or execute use-case decisions.
36. Public service builders accept `db: AsyncSession` and return normally. `get_db()` owns request DB lifetime; task/flow entrypoints own job-local DB lifetime.
37. Public flow builders should return a fully wired flow. Task entrypoints should not manually instantiate flows or pass app-scoped collaborators one by one.
38. Fixed low-level primitives such as DB session manager types, Redis primitives, and Celery primitives belong in `core/`. If the project uses explicit startup/shutdown ownership, app lifecycle wrappers may live in `src/container/lifecycle.py`, while runtime accessors and FastAPI request dependency adapters for those primitives may live under `src/container/core/`. The primitive types/packages stay in `core/`. Replaceable providers and external capability adapters do not become `core/` just because they are long-lived.
39. App-scoped infrastructure capabilities (providers, enqueuers, external clients, caches) are not application services. Build them in the central composition root and inject them where needed unless they are fixed `core/` primitives.
40. The central composition root may be split across multiple files by feature, but it is still one composition system. Do not replace one giant file with many app-local builder islands.
41. Builders receive infrastructure and wiring inputs such as `db` or app-scoped collaborators. Builders do not receive request actor data such as `current_user` or `user_id`.
42. Slow provider, network, or LLM calls must not run while a DB session is held open. Split those use cases into prepare/call/persist phases and let a flow own the session boundaries.
43. CRUD-style entity services should expose `list(...)` for their default collection read. Use `list_*` only when the service has multiple list variants or the method name needs a real qualifier such as parent resource, status, or queue state.
44. Container package `__init__.py` files are not barrel exports. Keep `src/container/__init__.py` and `src/container/core/__init__.py` empty unless there is a narrowly justified package-level concern. Import concrete container modules directly, such as `src.container.actions`, `src.container.core.dependencies`, and `src.container.lifecycle`.
45. Stable `core` package exports are allowed when they expose fixed primitives only, such as `Base`, `BaseRepository`, `DatabaseSessionManager`, or settings helpers. Do not export container builders, runtime lifecycle wrappers, app services, or FastAPI dependency adapters from `core` package `__init__.py` files.

## Migration Rules

When working on database schema changes in a FastAPI backend:
- Treat `alembic revision --autogenerate` output as a draft. Review and correct every revision by hand before considering it done.
- Alembic should use the active settings profile's `DATABASE_URL` by default. Do not invent `MIGRATION_DATABASE_URL` or per-environment migration targets unless the existing repo already uses that convention.
- Keep ORM metadata and migration DDL aligned. Partial indexes, unique constraints, computed columns, defaults, and enum usage must match in both places.
- Add required PostgreSQL extensions explicitly in migrations when runtime SQL depends on them.
  Example: `pgcrypto` for `pgp_sym_encrypt` / `pgp_sym_decrypt`.
- Custom SQLAlchemy `TypeDecorator` classes are runtime adapters, not migration DDL. In migrations, emit the concrete database storage type and any required SQL setup explicitly.
- Use named PostgreSQL partial indexes via `Index(..., unique=..., postgresql_where=...)` in models and verify the migration emits the same index.
- Review downgrade steps by hand, especially for partial indexes, enums, computed columns, and database-wide objects such as extensions.
- Run `alembic check` after model changes and test `alembic upgrade head` on a fresh database before treating the schema work as complete.
- Prefer fixing the model so `alembic revision --autogenerate` produces a runnable migration without manual surgery. Cyclic FK warnings ("Cannot correctly sort tables ... unresolvable cycles") usually point to a denormalized schema (a back-edge column whose only purpose is provenance and whose readers don't exist). Drop the smelly column rather than papering over it with `use_alter=True`. Use `use_alter=True` only when the cycle is genuinely intentional and both directions have real readers.
- PostgreSQL ENUM types created by `sa.Enum(MyEnum, name="x")` inside `op.create_table(...)` are NOT dropped by alembic's autogenerated `downgrade()` (alembic issue #886). A bare downgrade leaves orphan types and the next `upgrade head` fails with `type "x" already exists`. After every autogenerated migration that introduces a new enum, append the matching drop in `downgrade()`:
  ```python
  bind = op.get_bind()
  for enum_name in ("my_enum_1", "my_enum_2"):
      sa.Enum(name=enum_name).drop(bind, checkfirst=False)
  ```
  Place the loop AFTER the `# ### end Alembic commands ###` marker so it survives subsequent autogenerate edits. Do this only for enums that this specific migration creates — never for enums created by an earlier migration.
- Alembic also does not autogenerate `ALTER TYPE ... ADD VALUE` for enum value changes, nor `DROP TYPE` when the last column using an enum is removed. Hand-write both when needed.

## Reference Reading Protocol

The rules above are summaries. The references contain the exact patterns,
constraints, and code contracts. Code written without reading the matching
reference will have wrong patterns — wrong update semantics, broken transaction
boundaries, violated layer dependencies, incorrect DI wiring, wrong test mocking.

Every code change requires this workflow (MUST FOLLOW EXACTLY):

1. Read [STRUCTURE.md](STRUCTURE.md). Identify which layers your change touches.
2. Read **every matching reference** from the matrix below. Fully, not skimmed.
   If your change touches services and repositories, you read both references.
   If it touches routes and schemas, you read both. Read all that match — do
   not cherry-pick one and skip another that also applies.
   In doubt? READ THE reference.
3. State which references you are applying before proposing code.
4. Apply constraints exactly as written. Do not improvise patterns that differ from a reference.
5. If your implementation conflicts with a reference, resolve it explicitly. Never silently bypass.

### Mid-Turn Re-Check Rule

Do not treat the first reference read as sufficient for the whole turn.

You MUST re-run the invocation matrix and read any newly relevant references when:
- the edit expands into a new layer (`routes`, `services`, `repositories`, `flows`, `tasks`, `container`)
- the architecture shape changes (for example service -> flow, route -> facade service, route/task -> container wiring)
- you are about to add or move any `build_*` function
- you are about to introduce a new composition/wiring file
- transaction ownership changes (`get_db()` vs `sessionmanager.session()` vs worker/task boundary)

Stop and re-check before editing in those cases. Do not continue on memory or “close enough” pattern matching.

### Required Reference Matrix (FastAPI-first)

- **Architecture / layer decisions / new app or module**
  - [references/architecture-layers.md](references/architecture-layers.md) (required when creating a new app, deciding where code lives, or touching layer boundaries)
  - [references/capability-lifecycle.md](references/capability-lifecycle.md) (required when promoting capability from app-internal to shared or to a full app)
- **Protocols / provider swapping / infrastructure adapters**
  - [references/protocols.md](references/protocols.md) (required when adding a new infrastructure adapter, swapping a provider, or defining a shared contract)
- **Authentication / authorization / role-based access**
  - [references/authentication.md](references/authentication.md) (required when adding auth, protecting routes, or wiring auth dependencies)
- **Routes / URL design**
  - [references/routes.md](references/routes.md)
- **Services / orchestration / cross-module calls**
  - [references/services.md](references/services.md)
  - [references/module-isolation.md](references/module-isolation.md) (required when touching more than one module)
- **Repositories / query methods / update methods**
  - [references/repositories.md](references/repositories.md)
  - [references/database-setup.md](references/database-setup.md) (required when transaction behavior is relevant)
- **Schemas / DTO contracts / update payloads**
  - [references/schemas-models.md](references/schemas-models.md)
  - [references/repositories.md](references/repositories.md) (required for update semantics)
- **FastAPI dependency / request resolution / transaction boundaries**
  - [references/dependencies.md](references/dependencies.md)
  - [references/scoped-resources.md](references/scoped-resources.md) (required when choosing between `get_db()`, task-local session composition, or a shared scope object)
  - [references/database-setup.md](references/database-setup.md) (required when request DB access or commit timing is part of the change)
- **Logging / correlation IDs / structured output**
  - [references/logging.md](references/logging.md) (required when adding log statements, configuring logging, adding middleware, or handling sensitive data in logs)
- **Exceptions / enums**
  - [references/exceptions-enums.md](references/exceptions-enums.md)
- **Tests**
  - [references/testing.md](references/testing.md)
  - plus layer reference under test ([references/routes.md](references/routes.md), [references/services.md](references/services.md), or [references/repositories.md](references/repositories.md))
- **Starter scaffolds / implementation examples**
  - [references/scaffolds.md](references/scaffolds.md) (one-time bootstrap patterns for new apps/modules)
  - [references/examples.md](references/examples.md)
  - [scripts/database_setup.py](scripts/database_setup.py) (DB setup baseline)

## Output Expectations

When proposing or editing code with this skill:
- preserve strict layer boundaries (four-layer model + per-app route→service→repo→db)
- preserve the service/flow distinction:
  - services = single unit of work, `AsyncSession`-bound
  - flows = non-unit-of-work, `DatabaseSessionManager`-bound
- preserve the composition rule:
  - only the central composition module defines `build_*`
  - if you are about to add a `build_*` outside `src/container`, stop and re-read `STRUCTURE.md` and the composition rules in `SKILL.md`
  - routes normally inject `db` with `Depends(get_db)` from `src.container.core.dependencies` and then call `build_*(db=db)`
  - auth dependencies may also inject `db` with `Depends(get_db)` and build auth collaborators explicitly
  - task entrypoints fetch the runtime-managed session manager from the central composition root when needed, open `async with sessionmanager.session() as db:`, and call `build_*(db=db)` or a public flow builder
  - flows call private `_build_*` helpers or public `build_*(db=db)` helpers after opening a session
  - public service builders return normally; they do not own DB lifetime
  - public flow builders should fully wire the flow without caller-managed app-scoped infra
- preserve container import discipline:
  - keep `src/container/__init__.py` and `src/container/core/__init__.py` empty; do not use them as convenience export barrels
  - import builders, lifecycle functions, and runtime adapters from their concrete modules
  - stable `core` package barrels are acceptable only for fixed primitives/settings and must not re-export composition or FastAPI adapters
- eliminate duplicated wiring:
  - do not define one object graph in container/builders and a second version in route helpers or tasks
  - do not create app-local `builders.py` or similar wiring islands under `src/apps/...`
- keep composition root mechanical:
  - central builders remove construction boilerplate
  - central builders do not contain business decisions
- keep app-scoped infrastructure separate from application services:
  - providers, enqueuers, and other external clients belong to composition/infrastructure wiring
  - services and flows consume them through injected collaborators or central builders
- use explicit typed method signatures and DTO contracts
- use `src.core.schemas.BaseModel` / `ReadModel` as the only project Pydantic bases
- keep every Pydantic DTO in schema modules, including `*Internal` DTOs
- use explicit dependency names for repositories and collaborators; avoid generic names like `repository`
- keep business logic in services, not routes/repositories
- keep service outcome contracts simple:
  - return DTO on success
  - raise domain exception on error
  - do not introduce success/failure wrapper result objects for normal service APIs
- when a domain object represents tracked async intent (for example queued outbound work), create/return that DTO in the request unit of work and let a worker/flow transition it to `sent` / `failed` later
- keep transaction control in `get_db()` or a flow-local `sessionmanager.session()` context
- keep DB primitives in `src/core/db/`, not in infrastructure adapter namespaces; keep FastAPI request DB adapters such as `get_db` in `src/container/core/dependencies.py`
- treat `core/` as fixed foundation only:
  - config, schema bases, DB session manager types, Redis primitives, Celery primitives, and similar app-wide primitives fit there
  - when startup/shutdown explicitly owns the app-scoped singleton instance, `src/container/lifecycle.py` may hold app lifecycle wrappers and `src/container/core/` may hold init/build/close helpers for those fixed primitives
  - replaceable providers such as payment gateways, WhatsApp providers, LLM adapters, and other external capability implementations do not
- use `ReadModel.model_validate(obj)` when object shape already matches the read DTO; use manual DTO construction when the response is assembled or derived
- prefer explicit route/task DB ownership over services that own `DatabaseSessionManager`
- never inject `DatabaseSessionManager` into a normal request-scoped service; promote that logic to a flow instead
- never nest flows by having one flow build/call another flow in normal business design
- never instantiate repositories or concrete service graphs inline inside flow methods; route that composition through central builders
- never let adapters construct concrete services inside request methods; inject the already-built collaborator instead
- inject dependencies into services/adapters instead of calling global `get_*()` protocol locators inside normal service flow
- use `typing.Protocol` for swappable external dependencies
- never let apps import other apps — shared protocols are the seam
- keep slow external calls outside DB session scope by splitting prepare/call/persist phases
- apply test layering conventions (router mocks service, service mocks repo, repo uses real DB)
- include concise rationale when structural decisions are made
