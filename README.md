# FastAPI Backend Architecture Skill

An AI coding agent skill (Claude Code, Codex, etc.) with FastAPI architecture rules.

## What it gives you

- **Layered architecture, enforced.** `route -> service -> repository -> db`, one-way dependencies, no shortcuts.
- **Service vs Flow distinction.** Services own a single unit of work and bind to one `AsyncSession`. Flows own multi-phase logic and short DB sessions across slow external calls.
- **Centralized composition root.** `build_*` functions live in `src/container/`. Routes and tasks consume them; nothing else defines its own wiring.
- **DTO-in / DTO-out service contracts.** Project-wide `BaseModel` / `ReadModel` bases, `model_dump(exclude_unset=True)` update semantics, no leaking ORM into routes.
- **Module isolation.** Apps never import apps. Cross-module work goes through service-to-service calls or shared protocols.
- **Protocol-based DI** for swappable infrastructure (LLM providers, payment gateways, auth backends, external APIs).
- **Transaction discipline.** Commit/rollback only at `get_db()` or a flow-local `sessionmanager.session()`. Services and repositories stay transaction-unaware.
- **Alembic migration rules.** Real-world traps covered: enum drops on downgrade, partial indexes, extension setup, autogenerate sanity checks.
- **Per-layer test strategy.** Routes mock services, services mock repositories, repositories hit a real test DB.
- **Mandatory reference reading.** A reference matrix and mid-turn re-check rule keep the agent honest when work expands into new layers.

## Combine it!

Pair this skill with [`fastapi-cookiecutter`](https://github.com/marcius-llmus/fastapi-cookiecutter), which ships the wired baseline (DB engine, session manager, container, settings profiles, health app) so the skill can focus on *how to use it* rather than how to scaffold it.

## Install

Clone into your agent's skills folder. Examples:

```bash
# Claude Code
git clone https://github.com/marcius-llmus/fastapi-skill.git ~/.claude/skills/fastapi-backend-architecture

# Codex
git clone https://github.com/marcius-llmus/fastapi-skill.git ~/.codex/skills/fastapi-backend-architecture
```

## Layout

- `SKILL.md` — non-negotiable rules and reference matrix (the entry point).
- `STRUCTURE.md` — project layout, four-layer contract, mandatory invocation matrix.
- `references/` — per-topic deep references (services, repositories, dependencies, flows, testing, logging, migrations, ...).