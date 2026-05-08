# Testing Rules by Layer

## Scope

Use this reference for all test creation/refactors under `tests/`.

## Directory and Fixture Structure

Required project structure:
- Tests split by type first (`tests/unit/`, `tests/integration/`), then by app module.
- One `conftest.py` at root — shared fixtures (`db_session`, `make_claim`, etc.) and `pytest_plugins`.
- Per-app `fixtures.py` files — app-specific fixtures, registered globally via `pytest_plugins`.
- Fixtures are global — any app can use fixtures from another app's `fixtures.py`.
- Test file names must match the service/class under test (e.g., `test_resolver_service.py` for `ResolverService`).
- Shared test data (gold standard pairs, constants used across unit and integration) belongs in `tests/data.py`.

### Good

```
tests/
├── conftest.py                    # shared fixtures + pytest_plugins
├── data.py                        # shared test data (constants, gold standard pairs)
├── unit/
│   └── ingestion/
│       ├── fixtures.py            # unit-specific fixtures (mocks)
│       └── test_resolver_service.py
└── integration/
    └── ingestion/
        ├── fixtures.py            # integration-specific fixtures (real DB, fake APIs)
        └── test_resolver_service.py
```

```python
# tests/conftest.py — one conftest, all fixture files registered globally
pytest_plugins = [
    "tests.unit.ingestion.fixtures",
    "tests.integration.ingestion.fixtures",
]
```

### Bad

```python
# conftest.py per app directory — causes scoping issues
tests/unit/ingestion/conftest.py   # bad: use fixtures.py instead

# scattered ad-hoc imports in individual tests
from tests.projects.fixtures import *   # bad

# flat test structure without unit/integration separation
tests/
├── test_resolver.py   # bad: no layer separation, name doesn't match class
```

## Fixture Naming

- Fixtures registered globally must have unique names across all `fixtures.py` files.
- Prefix fixtures by layer to avoid collisions when multiple layers define the same concept:
  - Unit: `mock_atom_service`, `mock_llm_service`, `unit_resolver`
  - Integration: `integration_atom_service`, `fake_llm_service`, `integration_resolver`
- Shared fixtures in `conftest.py` use plain names: `db_session`, `db_session_mock`, `make_claim`.

### Good

```python
# tests/unit/ingestion/fixtures.py
@pytest.fixture
def mock_atom_service() -> AsyncMock: ...

@pytest.fixture
def unit_resolver(mock_llm_service, mock_embedding_service, mock_atom_service) -> ResolverService: ...

# tests/integration/ingestion/fixtures.py
@pytest.fixture
def integration_atom_service(db_session, memory_vector_store) -> AtomService: ...

@pytest.fixture
def integration_resolver(fake_llm_service, fake_embedding_service, integration_atom_service) -> ResolverService: ...
```

### Bad

```python
# Both files define `atom_service` — name collision when loaded globally
# tests/unit/ingestion/fixtures.py
@pytest.fixture
def atom_service() -> AsyncMock: ...    # bad: collides with integration

# tests/integration/ingestion/fixtures.py
@pytest.fixture
def atom_service(db_session) -> AtomService: ...   # bad: same name
```

## Fixture Rules

- Shared fixtures (`db_session`, `make_claim`) belong in root `conftest.py`.
- App-specific fixtures belong in `fixtures.py` per app per layer.
- Test files do NOT import shared fixtures — conftest injects them automatically.
- Constants shared across layers (e.g., gold standard pairs) belong in `tests/data.py`.
- Constants specific to one layer (e.g., `FAKE_EMBEDDING`) belong in that layer's `fixtures.py`.

## DB Fixtures in conftest.py

Session-scoped engine with per-test rollback:

```python
@pytest.fixture(scope="session")
async def engine() -> AsyncGenerator[AsyncEngine]:
    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    yield engine
    await engine.dispose()

@pytest.fixture(scope="session", autouse=True)
async def setup_db(engine: AsyncEngine) -> AsyncGenerator[None]:
    from src.apps.ingestion import models as ingestion_models  # noqa
    async with engine.begin() as conn:
        await conn.execute(text("PRAGMA foreign_keys=ON"))
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

@pytest.fixture(scope="session")
def sessionmaker(engine: AsyncEngine) -> async_sessionmaker[AsyncSession]:
    return async_sessionmaker(engine, autocommit=False)

@pytest.fixture(scope="function")
async def db_session(sessionmaker: async_sessionmaker[AsyncSession]) -> AsyncGenerator[AsyncSession]:
    async with sessionmaker() as session:
        try:
            yield session
        finally:
            await session.rollback()

@pytest.fixture(scope="function")
def db_session_mock() -> AsyncSession:
    return create_autospec(AsyncSession, instance=True)
```

## Mandatory Mocking Matrix

1. Router tests mock services.
2. Service tests mock repositories and downstream services.
3. Repository tests use real DB interactions (no DB mocking).

This is mandatory and non-optional for standard tests.

## Router Tests

### Good

- Use `TestClient`.
- Override DI providers (`app.dependency_overrides`).
- Assert HTTP status/body/headers and service calls.

```python
def test_clear_chat(client, override_get_chat_service, chat_service_mock):
    response = client.post("/chat/clear")
    assert response.status_code == 200
    chat_service_mock.clear_session_messages.assert_awaited_once()
```

### Bad

- Calling route handlers directly for endpoint behavior coverage:

```python
response = await cancel_turn.__wrapped__(...)   # bad for route behavior tests
```

Direct invocation is only acceptable for narrow unit tests of pure helper behavior, not route integration behavior.

## Service Tests

### Good

- Instantiate real service with mocked repositories/services.
- Validate business rules, DTO shaping, and downstream calls.
- Group tests by method under test (one test class per method or logical group).

```python
message_repository_mock.create = AsyncMock()
await chat_service.add_user_message(content="hi", session_id=1, turn_id="t1")
message_repository_mock.create.assert_awaited_once()
```

### Bad

```python
# service test hitting real DB for repository behavior
await real_repo.create(...)   # bad: belongs to repository tests
```

## Repository Tests

### Good

- Use real `db_session` fixture (test DB).
- Create rows, flush, query, verify persistence and SQL behavior.

```python
db_session.add(LLMSettings(...))
await db_session.flush()
result = await llm_settings_repository.get_by_model_name("gpt-4.1-mini")
assert result is not None
```

### Bad

```python
llm_settings_repository.get_by_model_name = AsyncMock(...)  # bad in repository tests
```

## Assertion Quality Rules

- Assert behavior, not implementation trivia.
- For routers: status code + response payload + relevant headers.
- For services: domain output + called dependencies.
- For repositories: query semantics, ordering, filtering, and persistence effects.
