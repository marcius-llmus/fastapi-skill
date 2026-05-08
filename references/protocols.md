# Protocols and Provider Swapping

## Why Protocols

Protocols define contracts between layers. They enable swapping providers without touching consumers.

Without protocols, a service imports a concrete provider directly. Swapping means editing business logic for an infrastructure concern. Protocols break this coupling.

## Implementing a Protocol

Every protocol is a `typing.Protocol` class:

```python
# shared/llm/protocols.py
from typing import Protocol

class LLMProvider(Protocol):
    async def complete(self, prompt: str) -> str: ...
```

Any class with a matching method signature satisfies the protocol — no inheritance required (structural subtyping).

```python
# infrastructure/anthropic/completer.py
class AnthropicCompleter:
    def __init__(self, client: anthropic.AsyncAnthropic, model: str):
        self._client = client
        self._model = model

    async def complete(self, prompt: str) -> str:
        message = await self._client.messages.create(
            model=self._model,
            max_tokens=200,
            messages=[{"role": "user", "content": prompt}],
        )
        return message.content[0].text
```

## The Chain

```
apps/<name>/service.py  →  LLMService (shared)  →  LLMProvider (protocol)  →  AnthropicCompleter (infra)
```

The service depends on `LLMService`, which depends on `LLMProvider` protocol. The actual provider is injected at the composition root.

## Shared Services Are Optional

The shared service layer adds cross-cutting concerns (retries, logging). It's not mandatory:

```
# With shared service (adds retries, logging):
app service  →  LLMService (shared)  →  LLMProvider (protocol)  →  Provider (infra)

# Without shared service (raw capability):
app service  →  LLMProvider (protocol)  →  Provider (infra)
```

| Pattern | When to use |
|---|---|
| Service → **protocol** → infrastructure | Raw capability is enough |
| Service → **shared service** → protocol → infrastructure | Shared service adds value (retries, caching, logging) |

## Swapping a Provider

**Step 1** — Create the new adapter in `infrastructure/`:

```python
# infrastructure/openai/completer.py
class OpenAICompleter:
    async def complete(self, prompt: str) -> str: ...
```

**Step 2** — Change one line in wiring:

```python
# Before
llm = LLMService(provider=AnthropicCompleter(client, model="claude-sonnet-4-20250514"))

# After
llm = LLMService(provider=OpenAICompleter(client, model="gpt-4.1"))
```

**Step 3** — Nothing else changes. Not `domain/`, not `shared/`, not `apps/`.

## When to Use Protocols vs Skip Them

**Use protocols for things you actually swap:**
- LLM providers (OpenAI, Anthropic, Google)
- Embedding providers
- Notification channels (email, Slack, webhook)
- External APIs that might change
- Authentication backends (JWT, OAuth, API keys)

**Skip protocols for things where the ORM is the abstraction:**
- Repositories over SQLAlchemy (SQLite/Postgres handled by connection string)
- File storage where S3 SDK is already the interface

**Also skip protocols for app-local data/state collaborators with one implementation:**
- Redis-backed counters, denylist stores, rate-limit stores, cache-backed lookups
- App-owned non-ORM persistence helpers used by one app only

If the only argument for a protocol is "we might swap Redis for DB later", keep
the concrete class until that second implementation is real.

The pragmatic rule: **don't abstract until evidence demands it.**

## Protocol Location Rules

| Protocol type | Location |
|---|---|
| Cross-app capability (LLM, embedding, notification) | `shared/<capability>/protocols.py` |
| Domain protocol (apps implement it) | `domain/protocols.py` |
| App-internal protocol | Inside the app (rare — prefer explicit classes) |

For app-internal data/state collaborators, the default is usually no protocol at
all. Keep the concrete implementation in the owning app and let the container
wire it directly. Naming (`repository`, `store`, etc.) depends on project
conventions; the important boundary is app-owned concrete class first, protocol
later only when evidence demands it.

## Testing with Protocols

Protocols make testing trivial — inject a mock that satisfies the protocol:

```python
class FakeLLMProvider:
    async def complete(self, prompt: str) -> str:
        return "fake response"

service = MyService(llm=FakeLLMProvider())
```

No patching, no monkeypatching, no complex mock setup.
