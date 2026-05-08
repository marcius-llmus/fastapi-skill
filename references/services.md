# Services Layer Rules

## Responsibility

Services contain business logic and orchestration only. They are the only layer allowed to coordinate repositories and other services inside one active unit of work.

This reference distinguishes between two application-layer shapes:

- **Service**: unit-of-work logic. A service operates correctly inside one active `AsyncSession`.
- **Flow**: non-unit-of-work logic. A flow exists only when the operation cannot safely run inside one active `AsyncSession` and therefore must own `DatabaseSessionManager` and open short sessions internally.

## Required Constraints

- No HTTP/web-layer imports (`fastapi`, `starlette`, `HTTPException`).
- Dependencies are constructor-injected.
- Raise domain exceptions from local `exceptions.py`.
- Services must receive Pydantic DTOs for create/update inputs.
- Services must return Pydantic DTOs from public methods.
- Public service methods return DTOs on success and raise domain exceptions on error.
- CRUD-style entity services should prefer `list(...)` for the primary collection read. Use `list_*` only when there are multiple list operations or the behavior needs a real qualifier such as `for_scope`, `for_receivable`, or `pending`.
- When a class defines a `list(...)` method, avoid bare `list[...]` annotations later in that same class body. Prefer `builtins.list[...]` or `Sequence[...]` for later annotations so IDEs do not resolve `list` to the method symbol.
- Avoid `_assert_*` helper names for business-rule validation in services. Prefer `require_*` for mandatory scope/resource checks and `validate_*` for consistency checks that may raise domain exceptions.
- Do not force private helpers to be `@staticmethod` when that makes normal service code call them through the class name. Prefer instance methods and `self._helper(...)` unless static binding clearly improves clarity.
- Stay transaction-unaware: never call `commit` or `rollback`.
- Do not inject `DatabaseSessionManager` into a normal service just because the service coordinates multiple collaborators.
- A flow must not receive a session-bound service instance in its constructor.

## Inter-Service Rules

- Service-to-service calls are allowed through public methods only.
- Never import another app's repository directly.
- For complex multi-service workflows that still fit one unit of work, add a service/facade service to keep dependency flow clear.
- Promote logic to a flow only when the operation spans multiple transactions, waits on external systems, retries, or otherwise should not hold one `AsyncSession` for the whole run.
- Inside a flow, after opening a short-lived session, obtain session-bound services through central builders instead of instantiating repositories or service graphs inline.
- Same-app service-to-service calls are allowed directly. Cross-app calls still go through `shared/` protocols injected by the composition root.

## Flow Rules

Flows are top-level process coordinators:

- a flow may call services
- a flow should not call another flow in normal design
- a flow should be fully constructed by central `build_*_flow()` functions
- task entrypoints should be thin pass-throughs that call the built flow
- if a route must synchronously coordinate DB work around slow external I/O, it may call a flow explicitly, but not through `Depends(...)`

## External I/O Boundary Rule

If a use case includes slow provider, network, or LLM calls, do not hold the DB session open across that call.

Use this shape:

1. open DB session
2. prepare request / load DB state
3. close DB session
4. call external system
5. open DB session
6. persist result or failure

That split belongs in a flow/orchestrator, not in a DB-bound service method.

## Tracked Async Intent Pattern

Use this when the domain object should exist before the external side effect is attempted.

- the request-scoped service creates the tracked intent inside the current unit of work and returns its DTO
- a worker/flow performs the external call later
- the worker/flow updates the existing domain object to `sent` / `failed`
- do not mix "persist failed state" and "raise request-scoped provider error" in the same direct-send path for the same tracked domain object

### Good — Request Creates Queued Intent

```python
class OutboundMessageService:
    async def queue_dispatch(
        self, *, input_: OutboundMessageDispatchInput
    ) -> MessageRead:
        prepared = await self._prepare_dispatch(input_=input_)
        message = await self._message_repository.create(
            MessageCreate(
                company_id=prepared.company_id,
                receivable_id=prepared.receivable_id,
                contact_id=prepared.contact_id,
                template_id=prepared.template_id,
                sent_by_user_id=input_.sent_by_user_id,
                direction=MessageDirection.outbound,
                channel=MessageChannel.whatsapp,
                status=MessageStatus.queued,
                request_key=input_.idempotency_key,
                provider_message_sid=None,
                body_snapshot=prepared.rendered_body,
            )
        )
        await self._outbound_message_job_repository.create_job(
            input_=OutboundMessageJobCreate(
                message_id=message.id,
                idempotency_key=input_.idempotency_key,
            )
        )
        return MessageRead.model_validate(message)
```

### Good — Flow Performs External I/O Outside DB

```python
class DispatchPendingOutboundJobsFlow:
    def __init__(
        self,
        *,
        session_manager: DatabaseSessionManager,
        whatsapp_provider: WhatsAppProvider,
    ) -> None:
        self._session_manager = session_manager
        self._whatsapp_provider = whatsapp_provider

    async def execute(self, *, job_id: int) -> MessageRead | None:
        async with self._session_manager.session() as db:
            service = build_outbound_job_dispatch_service(db=db)
            request = await service.prepare_dispatch(job_id=job_id)
        if request is None:
            return None

        sent = await self._whatsapp_provider.send_template(request=request.provider_input)

        async with self._session_manager.session() as db:
            service = build_outbound_job_dispatch_service(db=db)
            return await service.persist_success(
                job_id=job_id,
                provider_result=sent,
            )
```

### Bad — DB-Bound Service Waiting on Provider

```python
class OutboundMessageService:
    async def dispatch_job(self, *, job_id: int) -> MessageRead | None:
        job = await self._outbound_message_job_repository.get_for_update(job_id=job_id)
        sent = await self._whatsapp_provider.send_template(...)
        ...
```

Bad because the service now holds one DB session open while waiting on external I/O.

## Prepare + Observe Pattern for Idempotent Transitions

Use this when a service method transitions a state machine (for example: job → dispatched → completed) and the flow must also be able to observe the terminal state of records it did not transition itself (race conditions, jobs already completed, or entries invalidated during prep).

Do not model this as a union return such as `Pending* | *Result | None`. That is a "success/failure wrapper" (violates rule 7) and forces `isinstance` branching in the caller.

Split into two methods on the same service:

- `prepare_*(id) -> Pending* | None` — transitions state inside one unit of work. Returns the pending handoff DTO when the caller has work to perform. Returns `None` in every non-dispatchable case, including cases where the service itself persists a terminal failure during prep (missing refs, integration gone).
- `load_*_result(id) -> *Result | None` — observational read. Returns the current terminal DTO when one exists. The flow calls this after `prepare_*` returns `None` whenever it still needs to observe the outcome (e.g. to finalize a reminder, emit a metric, log a completion).

The prepare method still raises domain exceptions for true errors. `None` from prepare means "no pending work for the caller", not "error".

### Good — Split Prepare and Observe

```python
class OutboundMessageJobDispatchService:
    async def prepare_dispatch_job(
        self, *, job_id: int
    ) -> PendingOutboundJobDispatch | None:
        job = await self._jobs.get_for_update(job_id=job_id)
        if job is None or job.completed_at is not None:
            return None
        await self._jobs.mark_dispatched(row=job)
        message = await self._messages.get_for_update(message_id=job.message_id)
        if message is None or not self._is_dispatchable(message):
            await self._jobs.mark_completed(row=job)
            return None
        # ... build and return PendingOutboundJobDispatch

    async def load_dispatch_result(
        self, *, job_id: int
    ) -> OutboundJobDispatchResult | None:
        job = await self._jobs.get(job_id)
        if job is None:
            return None
        message = await self._messages.get(job.message_id)
        if message is None:
            return None
        return self._to_result(message=message, job=job)
```

```python
class DispatchPendingOutboundJobsFlow:
    async def _dispatch(self, *, job_id: int) -> OutboundJobDispatchResult | None:
        async with self._session_manager.session() as db:
            prepared = await build_dispatch_service(db=db).prepare_dispatch_job(
                job_id=job_id,
            )
        if prepared is None:
            async with self._session_manager.session() as db:
                return await build_dispatch_service(db=db).load_dispatch_result(
                    job_id=job_id,
                )
        return await self._send_and_persist(pending=prepared)
```

### Bad — Union Return Mixing Pending and Terminal

```python
class OutboundMessageJobDispatchService:
    async def prepare_dispatch_job(
        self, *, job_id: int,
    ) -> PendingOutboundJobDispatch | OutboundJobDispatchResult | None:
        ...
```

Bad because:
- the caller must `isinstance` branch on the return
- "pending work" and "terminal result" are two different contracts collapsed into one signature
- rule 7 prohibits returning a failed/terminal DTO from the success path

### Don't carry DTOs on exceptions as a workaround

Raising `OutboundJobNotDispatchable(result=...)` to carry the terminal DTO re-introduces the wrapper problem on the exception side. Prefer the two-method split.

## Good Service vs Flow Composition

### Service: one unit of work

```python
class LinkInboundMessageService:
    def __init__(
        self,
        *,
        inbound_message_service: InboundMessageService,
        inbox_service: InboxService,
    ) -> None:
        self._inbound_message_service = inbound_message_service
        self._inbox_service = inbox_service

    async def execute(
        self,
        *,
        message_id: int,
        contact_id: int,
        receivable_id: int | None,
        current_user: CurrentUser,
    ) -> InboxItem:
        await self._inbound_message_service.link_unlinked(
            message_id=message_id,
            contact_id=contact_id,
            receivable_id=receivable_id,
            current_user=current_user,
        )
        item = await self._inbox_service.get_item(
            message_id=message_id,
            current_user=current_user,
        )
        if item is None:
            raise InboundMessageNotFound(
                f"inbound message id={message_id} not found in scope"
            )
        return item
```

This is still a **service**, not a flow, because the whole operation fits one unit of work even though it coordinates multiple services.

### Flow: multiple units of work over time

```python
class ClassifyInboundMessageFlow:
    def __init__(
        self,
        *,
        session_manager: DatabaseSessionManager,
        llm_provider: LLMProvider,
    ) -> None:
        self._session_manager = session_manager
        self._llm_provider = llm_provider

    async def execute(self, *, message_id: int) -> None:
        async with self._session_manager.session() as db:
            service = build_inbound_message_classification_service(db=db)
            request = await service.prepare_existing_message_classification(
                message_id=message_id,
            )

        result = await self._llm_provider.classify_reply(request=request)

        async with self._session_manager.session() as db:
            service = build_inbound_message_classification_service(db=db)
            await service.persist_classification_result(
                message_id=message_id,
                result=result,
            )
```

This is a **flow** because one `AsyncSession` for the whole process would be the wrong lifetime.

## Good Task Boundary for a Flow

```python
async def _run() -> None:
    flow = build_classify_inbound_message_flow()
    await flow.execute(message_id=message_id)
```

The task boundary stays thin. It does not manually instantiate the flow or rebuild its collaborators.
