# Authentication Pattern

## Actual Project Shape

In this project, authentication is a shared FastAPI-facing capability with one
app-owned implementation.

Current layout:

```text
shared/auth/
├── protocols.py        # Authenticator + auxiliary auth protocols
├── schemas.py          # CurrentUser, UserScope
├── resolver.py         # OAuth scheme + auth/admin HTTP mapping helpers
└── dependencies.py     # FastAPI Depends(...) wrappers

apps/auth/
├── repositories/
├── services/
└── routes.py

container/auth.py       # concrete wiring
```

The important split is:
- `shared/auth/*` exposes the auth-facing contract used across the app
- `apps/auth/*` owns the implementation details
- `container/auth.py` wires the concrete auth collaborators

## What Exists Today

### Protocols

Current shared auth protocol surface includes:
- `Authenticator`
- `UserScopeResolver`
- `AuthRateLimiter`
- `RefreshTokenStore`
- `AccessTokenDenylist`

There is **no** `Authorizer` protocol in the current project.

Admin authorization is currently resolved from `CurrentUser` directly in
`shared/auth/resolver.py`.

### Shared Resolver

In this project, `shared/auth/resolver.py` is **not** a pure helper layer. It
is already FastAPI-facing:
- defines `oauth2_scheme`
- catches auth exceptions
- raises `HTTPException`
- applies the admin-role check

So do not assume a clean split where `resolver.py` is framework-free and only
`dependencies.py` touches FastAPI. That is not the current codebase shape.

### Shared Dependencies

`shared/auth/dependencies.py` is intentionally thin:
- `get_current_user(...)` gets `db` from `Depends(get_db)`
- builds the `Authenticator` from `container/auth.py`
- calls `resolve_current_user(...)`
- `require_admin_user(...)` only depends on `get_current_user(...)` and calls
  `resolve_admin_user(...)`

It does **not** build an `Authorizer`.

## Current Contract

```python
class Authenticator(Protocol):
    async def get_current_user(self, token: str) -> CurrentUser: ...
```

`CurrentUser` is the cross-app auth DTO used by routes and services.

## Current Resolution Flow

```text
route
  -> Depends(get_current_user)
  -> shared/auth/dependencies.py
  -> Depends(get_db)
  -> build_authenticator(db=db)
  -> Authenticator protocol
  -> apps/auth implementation
```

Admin-only routes add:

```text
route
  -> Depends(require_admin_user)
  -> shared/auth/dependencies.py
  -> resolve_admin_user(current_user=...)
```

## Good Patterns

### Good: Current User Dependency

```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> CurrentUser:
    auth = build_authenticator(db=db)
    return await resolve_current_user(token=token, auth=auth)
```

### Good: Admin Gate

```python
async def require_admin_user(
    current_user: CurrentUser = Depends(get_current_user),
) -> CurrentUser:
    return await resolve_admin_user(current_user=current_user)
```

### Good: Protected Route

```python
@router.get("/orders")
async def list_orders(
    current_user: CurrentUser = Depends(get_current_user),
) -> list[OrderRead]:
    ...
```

### Good: Admin Route

```python
@router.delete("/users/{user_id}")
async def delete_user(
    current_user: CurrentUser = Depends(require_admin_user),
) -> None:
    ...
```

## Container Role

`container/auth.py` is the composition root for auth wiring.

Current public builders include:
- `build_auth_service(*, db)`
- `build_authenticator(*, db)`
- `build_lockout_service(*, db)`

Protected routes and auth dependencies should use these builders rather than
instantiating auth services directly.

## What To Avoid

- do not document or introduce `Authorizer` unless the codebase actually adds it
- do not add `build_authorizer(...)` unless the auth design really changes
- do not claim `resolver.py` is framework-free; it currently raises
  `HTTPException`
- do not use `Depends(build_authenticator)` for a DB-bound authenticator
- do not redefine `get_current_user` / `require_admin_user` inside app routes
- do not push app-auth wiring into `shared/auth/dependencies.py`

## When To Read This Reference

Read this file when the change involves:
- protected routes
- `get_current_user` / `require_admin_user`
- auth protocol changes
- auth wiring in `container/auth.py`
- auth exception-to-HTTP mapping

If the change also touches general FastAPI dependency behavior, read
`references/dependencies.md` too.
