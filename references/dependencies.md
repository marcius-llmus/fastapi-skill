# FastAPI Dependencies Boundary

## Core Point

In this project, `dependencies.py` files are **FastAPI-facing adapter modules**.

They are not the general composition root.

Current project shape:
- `src/container/core/dependencies.py` owns request-scoped DB access
- `src/shared/auth/dependencies.py` adapts auth resolution into `Depends(...)`
- app-local dependency modules such as `src/apps/webhooks/dependencies.py` parse
  request data, verify transport-level inputs, and return typed values for routes
- `src/container/...` is the actual composition root for services, flows, and
  app-scoped collaborators

If you confuse these two roles, you end up pushing container wiring into
`dependencies.py` or pushing request parsing into the container.

## What Dependency Modules Are For

Dependency modules exist to adapt FastAPI request handling into typed values
that routes can use.

Good uses:
- yield the request-scoped `AsyncSession`
- resolve `CurrentUser`
- verify signatures / parse headers / read request body
- return typed request-derived DTOs
- map dependency-level failures to `HTTPException`

They are the boundary between FastAPI transport machinery and the route body.

## What Dependency Modules Are Not For

Dependency modules are not the place for:
- central service graph wiring
- app-wide `build_*` composition functions
- ad hoc repository construction
- use-case orchestration
- business workflows that belong in services or flows

That work belongs in `src/container/...`, services, or flows depending on the
concern.

## Actual Project Pattern

### 1. Request DB Dependency

`src/container/core/dependencies.py`

```python
async def get_db() -> AsyncIterator[AsyncSession]:
    sessionmanager = build_session_manager()
    async with sessionmanager.session() as session:
        yield session
```

Role:
- FastAPI entrypoint for request DB lifetime
- runtime wiring around the container-owned session manager
- nothing else

### 2. Shared Auth Dependency

`src/shared/auth/dependencies.py`

```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> CurrentUser:
    auth = build_authenticator(db=db)
    return await resolve_current_user(token=token, auth=auth)
```

Role:
- adapt auth into `Depends(...)`
- optionally obtain request-scoped collaborators from the container
- return a typed auth DTO

This is still a FastAPI adapter. It is not a second composition root.

### 3. App Request Parsing Dependency

`src/apps/webhooks/dependencies.py`

```python
async def require_valid_asaas_webhook(
    request: Request,
    asaas_signature: Annotated[str | None, Header(alias="asaas-signature")] = None,
) -> VerifiedAsaasWebhook:
    ...
```

Role:
- read request-level transport data
- validate/verify it
- return a typed DTO for the route

## Dependency Module Rules

- Dependency functions may use FastAPI `Depends`, `Request`, `Header`, and
  `HTTPException`.
- Dependency functions may depend on `get_db()` when they need request-scoped
  DB access.
- Dependency functions may build one request-scoped collaborator from the
  container when that collaborator is needed to resolve the dependency itself
  (for example auth).
- Dependency functions should return typed DTOs or primitives that the route can
  consume directly.
- Dependency functions should keep business logic out of the dependency body.
- Dependency functions must not define `build_*` functions.
- Dependency functions must not become alternate service-graph construction
  paths.

## Relationship to the Container

Keep the split explicit:

- `dependencies.py`
  - FastAPI adapter layer
  - request parsing / auth resolution / request-scoped DB handoff

- `src/container/...`
  - composition root
  - `build_*(db=db)` and `build_*_flow()`
  - app-scoped provider wiring

`src/container/core/dependencies.py` is a narrow FastAPI adapter colocated with
container-owned core runtime lifecycle. It must stay limited to request-scoped
primitive handoff such as `get_db()` and must not become a service builder
module.

Dependency modules may call container builders in limited cases, but that does
not make them the composition root.

## Good Patterns

### Good: DB Access Dependency

```python
async def get_db() -> AsyncIterator[AsyncSession]:
    sessionmanager = build_session_manager()
    async with sessionmanager.session() as db:
        yield db
```

### Good: Auth Dependency Using Request-Scoped Builder

```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> CurrentUser:
    auth = build_authenticator(db=db)
    return await resolve_current_user(token=token, auth=auth)
```

### Good: Request-Derived DTO

```python
async def require_valid_webhook(request: Request) -> VerifiedWebhook:
    raw_body = (await request.body()).decode("utf-8")
    ...
    return VerifiedWebhook(
        provider_event_id=provider_event_id,
        raw_body=raw_body,
        raw_headers=dict(request.headers.items()),
    )
```

This keeps transport parsing in the dependency and keeps the route body clean.

## Bad Patterns

### Bad: Service Builder Hidden Behind `Depends(...)`

```python
async def build_wallet_service_dep(
    db: AsyncSession = Depends(get_db),
) -> WalletService:
    return build_wallet_service(db=db)
```

Bad because:
- it hides composition inside a dependency wrapper for no benefit
- routes stop showing their true service boundary
- it blurs FastAPI adaptation with composition

### Bad: Dependency Doing Use-Case Work

```python
async def require_linked_message(... ) -> InboxItem:
    service = build_link_inbound_message_service(db=db)
    return await service.execute(...)
```

Bad because:
- dependencies should resolve inputs, not execute the use case
- the route loses its explicit control over the service/flow call

### Bad: Dependency Module as Builder Island

```python
def build_webhook_service(...): ...
```

Bad because:
- `dependencies.py` is not a second container
- it creates competing construction paths

## When To Read This Reference

Read this file when the change involves:
- `dependencies.py`
- `Depends(...)`
- `Request`, `Header`, or request-derived DTO parsing
- auth/user resolution
- deciding whether something belongs in a FastAPI dependency or in the container

If the change is really about:
- transaction lifetime → also read `scoped-resources.md` and `database-setup.md`
- central wiring / `build_*` placement → also read `STRUCTURE.md` and the
  composition rules in `SKILL.md`
