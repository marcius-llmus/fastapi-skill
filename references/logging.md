# Logging Rules

## Purpose

Every layer produces logs. Without rules, logging becomes inconsistent — some handlers log raw dicts, others print, services swallow errors silently, and production debugging becomes guesswork. This reference defines what to log, where, how, and what never to log.

## Dependencies

```
structlog
asgi-correlation-id
```

## Logger Setup

Use `structlog` for all application logging. One logger per module:

```python
import structlog

logger = structlog.get_logger()
```

Never use `print()` for operational output. Never use `logging.getLogger()` directly.

### Structured Output Configuration

Configure once at the application entry point (`main.py` or `config.py`). Never configure logging inside individual modules.

```python
import logging

import structlog


def configure_logging(
    *,
    renderer: structlog.types.Processor = structlog.processors.JSONRenderer(),
    level: int = logging.INFO,
) -> None:
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.stdlib.add_log_level,
            structlog.stdlib.add_logger_name,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
        ],
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

    formatter = structlog.stdlib.ProcessorFormatter(
        processors=[
            structlog.stdlib.ProcessorFormatter.remove_processors_meta,
            renderer,
        ],
    )

    handler = logging.StreamHandler()
    handler.setFormatter(formatter)

    root = logging.getLogger()
    root.handlers.clear()
    root.addHandler(handler)
    root.setLevel(level)
```

## Correlation IDs

Use `asgi-correlation-id` for automatic request ID generation and `structlog.contextvars` to propagate it through all log entries.

## Access Log Middleware

Access logging belongs in a small ASGI/FastAPI middleware. Keep it mechanical: measure elapsed time, call the downstream app once, log the final status, and do not inspect raw ASGI scope strings inside `dispatch`.

```python
import time

import structlog
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware, RequestResponseEndpoint

access_logger = structlog.get_logger("access")


class AccessLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(
        self,
        request: Request,
        call_next: RequestResponseEndpoint,
    ) -> Response:
        start = time.perf_counter()
        status_code = 500
        try:
            response = await call_next(request)
            status_code = response.status_code
            return response
        finally:
            duration_ms = (time.perf_counter() - start) * 1000
            access_logger.info(
                "request completed",
                method=request.method,
                path=request.url.path,
                status_code=status_code,
                duration_ms=round(duration_ms, 2),
            )
```

Use lower-level ASGI middleware only when the middleware must handle non-HTTP scopes or streaming edge cases. In that case, make the ASGI scope branching explicit in a dedicated ASGI middleware class rather than mixing low-level scope checks into `BaseHTTPMiddleware.dispatch`.

## What to Log at Each Layer

### Routes

Log inbound request context and outbound errors. Routes are the HTTP boundary.

```python
import structlog

from fastapi import Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from src.container.core.dependencies import get_db
from src.container.wallets import build_wallet_service

logger = structlog.get_logger()


@router.put("/{wallet_id}", response_model=WalletRead)
async def update_wallet(
    wallet_id: UUID,
    wallet_in: WalletUpdate,
    db: AsyncSession = Depends(get_db),
) -> WalletRead:
    service = build_wallet_service(db=db)
    logger.info(
        "update_wallet called",
        wallet_id=str(wallet_id),
        fields=list(wallet_in.model_fields_set),
    )
    try:
        result = await service.update_wallet(wallet_id=wallet_id, wallet_in=wallet_in)
        logger.info("update_wallet succeeded", wallet_id=str(wallet_id))
        return result
    except WalletNotFoundException as exc:
        logger.warning("wallet not found", wallet_id=str(wallet_id))
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=str(exc)) from exc
```

### Services

Log business decisions, state transitions, and meaningful cross-service calls.

```python
import structlog

logger = structlog.get_logger()


class WalletService:
    async def update_wallet(
        self, *, wallet_id: UUID, wallet_in: WalletUpdate
    ) -> WalletRead:
        db_obj = await self._get_wallet_or_raise(wallet_id)

        if wallet_in.status and wallet_in.status != db_obj.status:
            logger.info(
                "wallet status transition",
                wallet_id=str(wallet_id),
                from_status=db_obj.status,
                to_status=wallet_in.status,
            )

        updated = await self.wallet_repository.update(db_obj=db_obj, obj_in=wallet_in)
        return WalletRead.model_validate(updated)
```

### Repositories

Log persistence-level events sparingly: slow queries, unexpected empty results, and batch sizes.

```python
import structlog

logger = structlog.get_logger()


class WalletRepository:
    async def get_by_id(self, wallet_id: UUID) -> Wallet | None:
        result = await self.db.get(Wallet, wallet_id)
        if result is None:
            logger.debug("wallet not found in DB", wallet_id=str(wallet_id))
        return result

    async def bulk_archive(self, wallet_ids: list[UUID]) -> int:
        count = await self._archive_batch(wallet_ids)
        logger.info("bulk archive completed", requested=len(wallet_ids), archived=count)
        return count
```

### Infrastructure Adapters

Log external system interactions. This is the most critical layer for logging.

```python
import structlog

logger = structlog.get_logger()


class LLMProviderAdapter:
    async def classify_reply(self, *, request: ClassificationRequest) -> ClassificationResult:
        logger.info("llm classify started", provider="openai")
        try:
            result = await self._client.classify_reply(request=request)
            logger.info("llm classify completed", provider="openai")
            return result
        except Exception:
            logger.exception("llm classify failed", provider="openai")
            raise
```

### Flows

Flows should log phase boundaries around slow external work so it is obvious where time is spent.

```python
class ClassifyInboundMessageFlow:
    async def execute(self, *, message_id: int) -> None:
        logger.info("classification prepare started", message_id=message_id)
        async with self._session_manager.session() as db:
            service = build_inbound_message_classification_service(db=db)
            request = await service.prepare_existing_message_classification(
                message_id=message_id,
            )

        logger.info("classification provider started", message_id=message_id)
        result = await self._llm_provider.classify_reply(request=request)

        logger.info("classification persist started", message_id=message_id)
        async with self._session_manager.session() as db:
            service = build_inbound_message_classification_service(db=db)
            await service.persist_classification_result(
                message_id=message_id,
                result=result,
            )
```

## Never Log

- access tokens, refresh tokens, password hashes, raw secrets
- full request/response bodies containing PII unless explicitly sanitized
- raw SQL parameters
- provider payloads that contain customer secrets or private message bodies unless sanitized

## Good Structured Fields

| Field | Type | Meaning |
|---|---|---|
| `correlation_id` | `str` | Request/job trace id |
| `message_id` | `int` | Domain object id |
| `wallet_id` | `str` | Entity id |
| `provider` | `str` | External system name |
| `duration_ms` | `int` | Timing for operation |
| `attempt` | `int` | Retry attempt |

Prefer stable field names across modules so logs are queryable.
