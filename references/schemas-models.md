# Schemas and Models Rules

## Responsibility Split

- `models.py`: ORM/table definitions.
- `schemas.py`: Pydantic DTOs for create/update/internal/read contracts.
- `src/core/schemas.py`: the only place project Pydantic bases are defined.

## DTO Contract Rules

- Define explicit DTOs per operation (`EntityCreate`, `EntityUpdate`, `EntityRead`).
- Keep all Pydantic DTO declarations inside schema modules, including `*Internal` DTOs.
- Use update DTOs with omittable fields for partial updates.
- Services consume and return these DTOs as stable contracts.
- Route inputs/outputs should be DTO-driven; do not leak ORM models as public contracts.
- Project DTOs must inherit from `src.core.schemas.BaseModel` or `src.core.schemas.ReadModel`, not directly from Pydantic `BaseModel`.
- Service handoff types (`Pending*`, `Prepared*`, and similar intermediate contracts passed between services and flows) are DTOs. Do not declare them with `@dataclass`; use `BaseModel` in a schema module like any other DTO.

## Default Text and Patch Semantics

For non-null DB text columns that are optional in the create/edit UX, prefer an
empty-string sentinel in the DTO:

```python
class CampaignCreate(BaseModel):
    scope_rule: str = ""
```

Do not write `str | None = ""`, and do not add validators or service-side
`model_dump()` rewriting to turn `None` into `""`. If clients should clear a
field, require an explicit empty string. Explicit `null` should be invalid
unless `NULL` is a real domain/database value.

For patch DTOs, a field can stay omittable without using `None`:

```python
class CampaignUpdate(BaseModel):
    scope_rule: str = ""
```

With repository updates using `model_dump(exclude_unset=True)`, an omitted field
is not written, while an explicit `""` clears the text field.

## Required DTO Set per Entity

- `EntityCreate`: required fields for create
- `EntityUpdate`: optional fields for partial patch
- `EntityCreateInternal`: optional, when service-enriched data differs from route-entry create data
- `EntityRead`: response contract, usually inheriting `ReadModel`
- Optional `EntityReadPublic` or `EntityReadBasic` for sanitized/subset projections

When an internal create DTO only adds service-owned fields before persistence,
prefer a flat DTO that extends the route-entry create DTO. The dumped field names
must match ORM constructor keyword arguments so `BaseRepository.create()` can
remain generic.

### Good

```python
from src.core.schemas import ReadModel


class EntityRead(ReadModel):
    id: int
    name: str
```

```python
from src.core.schemas import BaseModel
from src.domain.enums import HypothesisState


class HypothesisCreate(BaseModel):
    claim: str
    prior_confidence: float


class HypothesisCreateInternal(HypothesisCreate):
    state: HypothesisState = HypothesisState.generated


row = await self._hypothesis_repository.create(
    obj_in=HypothesisCreateInternal(
        **hypothesis_in.model_dump(),
        state=HypothesisState.generated,
    )
)
```

### Bad

```python
from pydantic import BaseModel


class EntityRead(BaseModel):
    __root__: dict   # bad: opaque response contracts
```

```python
class EntityCreateInternal(BaseModel):  # bad: internal DTO outside schemas.py
    ...
```

```python
class HypothesisCreateInternal(BaseModel):
    hypothesis: HypothesisCreate
    state: HypothesisState = HypothesisState.generated
```

Bad when passed to the generic repository because
`obj_in.model_dump()` produces `{"hypothesis": {...}, "state": ...}`,
but the ORM model expects flat column keyword arguments.

## Update DTO Rules (Critical)

Update DTOs define what generic repository updates may mutate.

- Fields handled by dedicated service/repository methods must use `Field(exclude=True)`.
- Keep update DTO fields omittable via defaults to support partial updates.
- For non-null text fields that use an empty-string sentinel, use `str = ""`
  rather than `str | None = None`.
- Reserve `T | None = None` for nullable/domain-null fields, or for fields
  excluded from generic repository update and explicitly handled elsewhere.
- Avoid nested mutable payloads unless service logic explicitly processes them.

### Good

```python
from pydantic import Field

from src.core.schemas import BaseModel


class SettingsUpdate(BaseModel):
    max_history_length: int | None = None
    coding_llm_settings_id: int = Field(exclude=True)
    coding_llm_settings: LLMSettingsUpdate | None = Field(default=None, exclude=True)
```

### Bad

```python
from src.core.schemas import BaseModel


class SettingsUpdate(BaseModel):
    max_history_length: int
    coding_llm_settings: dict | None = None  # bad: required field + untyped nested payload
```

## Circular Import Safety

- Prefer one-way nesting for parent-child read models.
- For reciprocal relations, use local `ReadBasic` models to avoid import cycles.
- Avoid direct model-module circular imports across applications.

## Cross-App Usage

- Construct provider-service DTOs explicitly in consumer services when needed.
- Keep contracts explicit rather than passing untyped dict payloads.

## ORM vs DTO Boundaries

- Repositories return ORM models to services.
- Services convert ORM -> DTO for public service contracts.
- `ReadModel.model_validate(obj)` is ideal when an ORM/object shape matches the DTO closely.
- Manually constructing a read DTO is still correct when the response is assembled, flattened, or derived from multiple sources.
- Do not use ORM types as external API contracts.

## URL/Field Naming Hygiene

- Keep schema field names aligned with domain terms, not transport quirks.
- Do not encode action semantics in entity field names (for example `toggle_*` boolean that represents an action call payload).
