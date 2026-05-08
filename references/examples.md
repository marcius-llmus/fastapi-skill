# Examples

## Route to Service to Repository Update Flow

1. Route validates `WalletUpdate` DTO.
2. Route gets `db` from `Depends(get_db)`.
3. Route builds `WalletService` with `build_wallet_service(db=db)`.
4. Service fetches ORM object from repository and applies business rules.
5. Service passes `(db_obj, wallet_in)` to repository update.
6. Repository uses `model_dump(exclude_unset=True)` + `setattr`, then `flush`.
7. Service converts updated ORM object to `WalletRead` DTO.
8. Route returns DTO and maps domain exceptions to HTTP status codes.

## Inter-Service Call

Consumer service imports provider service's DTO and constructs it explicitly:

```python
wallet_in = WalletCreate(currency="USD", balance=0)
wallet = await wallet_service.create_wallet_for_user(
    user_id=user.id,
    wallet_in=wallet_in,
)
```

## Cross-Module Isolation Example

### Good

```python
class SettingsService:
    async def update_settings(self, settings_in: SettingsUpdate) -> SettingsRead:
        if settings_in.coding_llm_settings:
            await self.llm_service.update_coding_llm(...)
        updated = await self.settings_repo.update(db_obj=db_obj, obj_in=settings_in)
        return SettingsRead.model_validate(updated)
```

### Bad

```python
class SettingsService:
    async def _to_public_llm_settings(self, llm_settings):
        return await self.llm_service.llm_settings_repo.get_api_key_for_provider(...)
```

## Non-HTTP Context

Background task/worker code should either build a service through the central
composition root inside a short-lived session, or call a flow if the process
does not fit one unit of work:

```python
sessionmanager = build_session_manager()
async with sessionmanager.session() as db:
    wallet_service = build_wallet_service(db=db)
    try:
        await wallet_service.archive_wallet(wallet_id=wallet_id)
    except WalletNotFoundException:
        logger.warning("wallet not found", wallet_id=str(wallet_id))
```

If that code lives under a sync Celery task, wrap it in a coroutine for
`asyncio.run(...)`:

```python
@celery_app.task(name="src.apps.wallets.tasks.archive_wallet", bind=True)
def archive_wallet(self: Task, wallet_id: str) -> None:
    async def _run() -> None:
        sessionmanager = build_session_manager()
        async with sessionmanager.session() as db:
            wallet_service = build_wallet_service(db=db)
            await wallet_service.archive_wallet(wallet_id=wallet_id)

    asyncio.run(_run())
```

If the task calls a flow instead, the flow's `execute(...)` coroutine is the
thing passed directly to `asyncio.run(...)`.

## Tracked Async Intent Example

### Request Side

```python
message = await outbound_message_service.queue_dispatch(input_=dispatch_input)
return message
```

The request returns the queued/accepted DTO for the tracked domain object. It
does not attempt the long-running provider call inline.

### Worker Side

```python
flow = build_dispatch_pending_outbound_jobs_flow()
await flow.execute(limit=50)
```

The worker flow opens short DB sessions, performs the provider call outside DB,
and updates the existing tracked object to `sent` or `failed`.

### Later Read Side

```python
@router.get("/messages/{message_id}", response_model=MessageRead)
async def get_message(
    message_id: int,
    db: AsyncSession = Depends(get_db),
) -> MessageRead:
    service = build_message_history_service(db=db)
    return await service.get(message_id=message_id)
```

The same tracked object can later be read back as `queued`, `sent`, or
`failed`. A queued request is still a successful request outcome; later worker
state transitions do not change that original request contract.

## Synchronous External-I/O Example

```python
@router.post("/payments/{payment_id}/artifact", response_model=PaymentArtifactRead)
async def generate_artifact(payment_id: int) -> PaymentArtifactRead:
    flow = build_generate_payment_artifact_flow()
    return await flow.execute(payment_id=payment_id)
```

The flow is correct here because it can do:
- DB prepare in one short session
- provider call outside DB
- DB persist in a second short session

## Test Layering Examples

### Router Test (Good)

```python
def test_update_settings(client, override_get_settings_service, settings_service_mock):
    response = client.post("/settings/", json={...})
    assert response.status_code == 204
    settings_service_mock.update_settings.assert_awaited_once()
```

### Service Test (Good)

```python
settings_repo_mock.update = AsyncMock(return_value=settings_obj)
result = await settings_service.update_settings(settings_in=settings_update)
assert result.id == settings_obj.id
settings_repo_mock.update.assert_awaited_once()
```

### Repository Test (Good)

```python
db_session.add(LLMSettings(...))
await db_session.flush()
result = await llm_settings_repository.get_by_model_name("gpt-4.1-mini")
assert result is not None
```
