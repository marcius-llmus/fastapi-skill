# Scaffolds

One-time setup patterns for project bootstrap and new app creation. Not an ongoing reference.

## BaseRepository[T]

```python
# src/core/db/repository.py
from typing import Any, TypeVar

from sqlalchemy.ext.asyncio import AsyncSession

from src.core.db import Base
from src.core.schemas import BaseModel

ModelType = TypeVar("ModelType", bound=Base)


class BaseRepository[ModelType: Base]:
    model: type[ModelType]

    def __init__(self, db: AsyncSession):
        self.db = db

    async def get(self, pk: Any) -> ModelType | None:
        return await self.db.get(self.model, pk)

    async def create(self, obj_in: BaseModel) -> ModelType:
        db_obj = self.model(**obj_in.model_dump())
        db_obj = await self.db.merge(db_obj)
        await self.db.flush()
        return db_obj

    async def update(self, *, db_obj: ModelType, obj_in: BaseModel) -> ModelType:
        update_data = obj_in.model_dump(exclude_unset=True)
        for key, value in update_data.items():
            setattr(db_obj, key, value)
        await self.db.flush()
        return db_obj

    async def delete(self, *, pk: Any) -> ModelType | None:
        db_obj = await self.db.get(self.model, pk)
        if db_obj:
            await self.db.delete(db_obj)
            await self.db.flush()
        return db_obj
```

## Service Skeleton

```python
class OrderService:
    def __init__(self, order_repository: OrderRepository, item_repository: ItemRepository):
        self._order_repository = order_repository
        self._item_repository = item_repository

    async def create(self, *, order_in: OrderCreate) -> OrderRead:
        db_obj = await self._order_repository.create(obj_in=order_in)
        return OrderRead.model_validate(db_obj)
```

## Repository Skeleton

```python
class EntityRepository(BaseRepository):
    async def get(self, pk: UUID) -> Entity | None:
        return await self.db.get(Entity, pk)

    async def create(self, obj_in: EntityCreate) -> Entity:
        db_obj = Entity(**obj_in.model_dump())
        db_obj = await self.db.merge(db_obj)
        await self.db.flush()
        return db_obj

    async def update(self, *, db_obj: Entity, obj_in: EntityUpdate) -> Entity:
        update_data = obj_in.model_dump(exclude_unset=True)
        for key, value in update_data.items():
            setattr(db_obj, key, value)
        await self.db.flush()
        return db_obj
```

## DB Dependency Skeleton

```python
# src/container/core/dependencies.py
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncSession

from src.container.core.db import build_session_manager


async def get_db() -> AsyncIterator[AsyncSession]:
    sessionmanager = build_session_manager()
    async with sessionmanager.session() as db:
        yield db
```

## Composition Skeleton

```python
# src/container/entities.py
def _build_entity_repository(*, db: AsyncSession) -> EntityRepository:
    return EntityRepository(db=db)


def build_entity_service(*, db: AsyncSession) -> EntityService:
    return EntityService(
        entity_repository=_build_entity_repository(db=db),
    )
```

## Flow Skeleton

```python
class RecoverEntitiesFlow:
    def __init__(self, *, session_manager: DatabaseSessionManager) -> None:
        self._session_manager = session_manager

    async def execute(self) -> None:
        async with self._session_manager.session() as db:
            service = build_entity_service(db=db)
            entity_ids = await service.list_pending_ids()

        for entity_id in entity_ids:
            async with self._session_manager.session() as db:
                service = build_entity_service(db=db)
                await service.recover(entity_id=entity_id)
```

```python
# src/container/entities.py
from src.container.core.db import build_session_manager


def build_recover_entities_flow() -> RecoverEntitiesFlow:
    return RecoverEntitiesFlow(session_manager=build_session_manager())
```

```python
# apps/entities/tasks.py
async def _run() -> None:
    flow = build_recover_entities_flow()
    await flow.execute()
```

## Route Skeleton

```python
@router.put("/{entity_id}", response_model=EntityRead)
async def update_entity(
    entity_id: UUID,
    entity_in: EntityUpdate,
    db: AsyncSession = Depends(get_db),
) -> EntityRead:
    service = build_entity_service(db=db)
    try:
        return await service.update(entity_id=entity_id, entity_in=entity_in)
    except EntityNotFoundException as exc:
        raise HTTPException(status_code=404, detail=str(exc)) from exc
```
