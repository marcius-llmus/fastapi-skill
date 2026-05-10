# Pagination, Filtering, and Text Search

## Responsibility Split

Pagination cuts across all four layers. Each layer owns exactly one concern:

| Layer | Owns | Forbidden |
|-------|------|-----------|
| Repository | `Select` construction, count + slice SQL | Pydantic, schemas, HTTP echo |
| Service | Filter selection, DTO conversion, multi-step enrichment | Raw SQL execution, HTTP echo |
| Route | Query parameter parsing, `Page` envelope construction | Filtering rules, count queries |
| Schemas | `PaginatedResult[T]`, `Page[T]`, default/max limits | None — pure data |

## Schemas (live in `src/core/schemas.py`)

```python
PAGE_DEFAULT_LIMIT = 50
PAGE_MAX_LIMIT = 200


class PaginatedResult[T](BaseModel):
    """Domain pagination result returned by services. No HTTP echo."""

    items: list[T]
    total: int

    @classmethod
    def from_rows[U: BaseModel](
        cls,
        schema: type[U],
        rows: Iterable[object],
        total: int,
    ) -> "PaginatedResult[U]":
        return PaginatedResult[schema](
            items=[schema.model_validate(row) for row in rows],
            total=total,
        )


class Page[T](PaginatedResult[T]):
    """HTTP pagination envelope returned by routes. Adds limit/offset echo."""

    limit: int
    offset: int
```

Rules:
- `PaginatedResult[T]` is the **service-layer DTO**. Services return this.
- `Page[T]` is the **HTTP envelope**. Only routes construct this.
- `from_rows` is the only sanctioned validator-construction call site. Repositories never call it.

## Repository: SQL Only

Repositories expose `list_*` methods that build a `Select`, plus the inherited `paginate()` that runs count + slice queries. **No Pydantic anywhere.**

`BaseRepository.paginate` lives in `src/core/db/repository.py`:

```python
async def paginate(
    self,
    stmt: Select,
    *,
    limit: int,
    offset: int,
) -> tuple[Sequence[T], int]:
    total_stmt = select(func.count()).select_from(stmt.subquery())
    total = int((await self.db.execute(total_stmt)).scalar_one())
    page_stmt = stmt.limit(limit).offset(offset)
    rows = (await self.db.execute(page_stmt)).scalars().all()
    return rows, total
```

Why a method on `BaseRepository` and not a free function:
- Reuses `self.db` — services don't reach into `repo.db`.
- Keeps the SQL count/slice idiom in one place.
- Repository remains DTO-unaware: returns ORM rows + an `int`.

### Filtered list method

`list_*` returns a `Select`. It applies `WHERE` clauses for filters and text search but **does not** apply limit/offset:

```python
def list_for_engagement(
    self,
    engagement_id: uuid.UUID,
    *,
    status: TurnStatus | None = None,
    q: str | None = None,
) -> Select:
    stmt = (
        select(self.model)
        .where(self.model.engagement_id == engagement_id)
        .order_by(self.model.created_at.desc())
    )
    if status is not None:
        stmt = stmt.where(self.model.status == status)
    return self._with_text_search(stmt, q, self.model.title, self.model.description)
```

`_with_text_search` is a `BaseRepository` static helper that ORs `ILIKE %q%` over the supplied columns when `q` is non-empty. Use it whenever a list endpoint takes a free-text `q`.

### Repository contract

- Returns `Select`, never `PaginatedResult`.
- Never imports anything from `src.core.schemas` related to pagination.
- Never calls `model_validate` or constructs DTOs.
- `ORDER BY` belongs in the repo's `Select` (it's part of the query).

## Service: Compose Repo Output Into DTO

The service is the only layer allowed to call both `repo.paginate` and `PaginatedResult.from_rows`:

```python
async def list_turns_page_for_engagement(
    self,
    *,
    engagement_id: uuid.UUID,
    status: TurnStatus | None = None,
    q: str | None = None,
    limit: int,
    offset: int,
) -> PaginatedResult[OrchestratorTurnRead]:
    await self._require_engagement(engagement_id=engagement_id)
    stmt = self._turn_repository.list_for_engagement(engagement_id, status=status, q=q)
    rows, total = await self._turn_repository.paginate(stmt, limit=limit, offset=offset)
    return PaginatedResult.from_rows(OrchestratorTurnRead, rows, total)
```

Naming: when a service has both a paginated and a non-paginated reader, suffix the paginated one with `_page`. Single-reader services may use plain `list_*`.

### Post-paginate enrichment

If a paginated DTO needs async enrichment that doesn't fit one query (cross-module lookup, batch ref resolution), enrich `result.items` after `from_rows`:

```python
rows, total = await self._world_event_repository.paginate(stmt, limit=limit, offset=offset)
result = PaginatedResult.from_rows(WorldEventRead, rows, total)
await self._attach_entity_refs_to_reads(result.items)
return result
```

`BaseModel` is mutable by default, so post-construction field assignment is fine. Keep enrichment in a private helper on the service.

### Service contract

- Returns `PaginatedResult[Schema]`, never `Page[Schema]`.
- Never constructs DTOs from rows manually — always go through `PaginatedResult.from_rows`.
- Never calls `db.execute`, `select(func.count())`, or `.subquery()` directly.

## Route: HTTP Echo + Defaults

Routes own query-parameter parsing, the `limit`/`offset` defaults, and the `Page` envelope:

```python
@router.get(
    "/engagements/{engagement_id}/turns",
    response_model=Page[OrchestratorTurnRead],
)
async def list_turns(
    engagement_id: uuid.UUID,
    db: AsyncSession = Depends(get_db),
    status: Annotated[TurnStatus | None, Query()] = None,
    q: Annotated[str | None, Query()] = None,
    limit: Annotated[int, Query(ge=1, le=PAGE_MAX_LIMIT)] = PAGE_DEFAULT_LIMIT,
    offset: Annotated[int, Query(ge=0)] = 0,
) -> Page[OrchestratorTurnRead]:
    service = build_orchestration_service(db=db)
    try:
        result = await service.list_turns_page_for_engagement(
            engagement_id=engagement_id,
            status=status,
            q=q,
            limit=limit,
            offset=offset,
        )
    except EngagementsException as exc:
        raise _to_http(exc) from exc
    return Page[OrchestratorTurnRead](
        items=result.items,
        total=result.total,
        limit=limit,
        offset=offset,
    )
```

Rules:
- `Query(ge=1, le=PAGE_MAX_LIMIT)` and `Query(ge=0)` are the canonical bounds.
- Default `limit` is `PAGE_DEFAULT_LIMIT`; default `offset` is `0`.
- `response_model` is `Page[Schema]` so OpenAPI shows the full envelope.
- Construct `Page[Schema]` explicitly with the four fields. Do not subclass.
- Echo `limit` / `offset` from the route's parameters, not from any service-side computation.

## Anti-Patterns

```python
# bad: Pydantic in repository
async def paginate(self, ...) -> PaginatedResult[Schema]:
    return PaginatedResult.from_rows(Schema, rows, total)
```

```python
# bad: SQL in service
total = (await self.db.execute(select(func.count())...)).scalar_one()
```

```python
# bad: route building DTOs from ORM rows
return Page[TurnRead](
    items=[TurnRead.model_validate(r) for r in await repo.list(...)],
    total=...,
)
```

```python
# bad: service returning Page (HTTP envelope leaks downward)
async def list_turns_page(...) -> Page[TurnRead]: ...
```

```python
# bad: repository applying limit/offset itself
return stmt.limit(limit).offset(offset)
```

```python
# bad: free-function paginate that takes db as a parameter
await paginate(repo.db, stmt, limit=..., offset=..., schema=Schema)
```

## Summary

```
repo.list_for_*(filters)      -> Select                       # SQL build
repo.paginate(stmt, ...)      -> (Sequence[ORM], int)         # SQL count+slice
PaginatedResult.from_rows(...) -> PaginatedResult[Schema]     # DTO conversion
service.list_*_page(...)      -> PaginatedResult[Schema]      # composes the above
Page[Schema](items, total, limit, offset)                      # HTTP envelope
```

One direction, no leakage: SQL stays in repo, DTOs in service, HTTP echo in route.
