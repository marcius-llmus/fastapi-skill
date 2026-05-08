# Routes Layer Rules

## Responsibility

Routes are the HTTP translation layer. A route should:
1. parse path/query/body inputs
2. rely on FastAPI/Pydantic validation
3. obtain the right request-scoped dependencies (`db`, auth, etc.)
4. build one service or invoke one flow/use-case
5. map domain exceptions to `HTTPException`
6. return DTOs

## Do

- Keep handlers thin and orchestration-free.
- Use `db: AsyncSession = Depends(get_db)` for DB-bound request work.
- Build services through public container `build_*(db=db)` functions.
- Build one service or invoke one flow per route use case. If a request needs behavior from multiple services in one unit of work, add a facade service and build that facade instead of assembling multiple services in the route.
- If a route needs "do A, then do B" across two services to satisfy one request, that orchestration belongs in a request-scoped facade service, not in the handler body.
- If a request must commit DB state before enqueueing worker work, use a flow that owns the session boundary and enqueue step after commit.
- Use `response_model` for output shape control.
- Translate domain exceptions explicitly.
- Return successful DTOs exactly as the service or flow returned them.

## Do Not

- Do not implement business rules in routes.
- Do not import repositories in routes.
- Do not raise domain exceptions from routes.
- Do not manually compose repositories/adapters in routes.
- Do not build multiple services in one route just to coordinate a single use case.
- Do not reinterpret service return values as hidden error wrappers.
- Do not put flows in `Depends(...)`.
- Do not use FastAPI `BackgroundTasks` to hide domain orchestration when the project already has a proper worker/task path.
- Do not keep request `db` open across slow external/provider/LLM calls.

## Standard DB-Only Pattern

```python
@router.put("/{wallet_id}", response_model=WalletRead)
async def update_wallet(
    wallet_id: UUID,
    wallet_in: WalletUpdate,
    db: AsyncSession = Depends(get_db),
) -> WalletRead:
    service = build_wallet_service(db=db)
    try:
        return await service.update_wallet(wallet_id=wallet_id, wallet_in=wallet_in)
    except WalletNotFoundException as exc:
        raise HTTPException(status_code=404, detail=str(exc)) from exc
```

## Good Route Triggering Background Work Indirectly

If the operation cannot safely run inside one request unit of work and does not
need to finish synchronously, the route should persist intent and hand work off
to a worker path:

```python
@router.post("/webhook-recovery", response_model=JobAccepted)
async def recover_webhooks(
    db: AsyncSession = Depends(get_db),
) -> JobAccepted:
    service = build_webhook_recovery_request_service(db=db)
    return await service.enqueue()
```

The route still depends on a **service**. The long-running **flow** runs later in the task/worker boundary.

## Good Route Invoking a Flow Explicitly

Sometimes the request must synchronously coordinate DB state around slow
external I/O. In that case, let a flow own the short DB phases:

```python
@router.post("/payments/{payment_id}/artifact", response_model=PaymentArtifactRead)
async def generate_artifact(payment_id: int) -> PaymentArtifactRead:
    flow = build_generate_payment_artifact_flow()
    try:
        return await flow.execute(payment_id=payment_id)
    except PaymentNotFound as exc:
        raise HTTPException(status_code=404, detail=str(exc)) from exc
```

This is acceptable because the flow, not the route, owns the DB session boundaries around the external provider call.

The same pattern applies when the request must:
1. persist state
2. commit
3. enqueue async work

That sequence belongs in a flow, not in the route and not in a request-scoped service that runs before the route-owned transaction commits.

## Good Route Returning Tracked Async Intent

If the domain object should exist before the background work happens, the route
still returns the successful DTO created by the request-scoped service:

```python
@router.post(
    "/receivables/{receivable_id}/cobranca",
    response_model=MessageRead,
    status_code=status.HTTP_202_ACCEPTED,
)
async def send_cobranca(
    receivable_id: int,
    payload: SendCobrancaRequest,
    current_user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> MessageRead:
    service = build_outbound_message_service(db=db)
    try:
        return await service.send_cobranca(
            receivable_id=receivable_id,
            payload=payload,
            current_user=current_user,
        )
    except TemplateNotFound as exc:
        raise HTTPException(status_code=404, detail=str(exc)) from exc
```

The route returns the queued message DTO. The actual provider send happens later in the worker/flow path.

## Bad Route Patterns

### Bad: Flow in `Depends(...)`

```python
@router.post("/webhook-recovery")
async def recover_webhooks(
    flow: RecoverWebhookEventsFlow = Depends(build_recover_webhook_events_flow),
) -> None:
    await flow.execute(older_than_minutes=30, limit=50)
```

### Bad: Request DB Held Across Slow External I/O

```python
@router.post("/payments/{payment_id}/artifact")
async def generate_artifact(
    payment_id: int,
    db: AsyncSession = Depends(get_db),
) -> PaymentArtifactRead:
    service = build_payment_artifact_service(db=db)
    return await service.generate(payment_id=payment_id)
```

Bad because the route-bound DB lifetime is now coupled to a slow provider call inside the service method.
