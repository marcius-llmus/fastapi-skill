# FastAPI Backend Architecture Skill

A Claude Code skill that enforces a strict, batteries-included architecture for FastAPI backends. It turns "vague best practices" into hard contracts the agent will read and apply on every change.

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

## Pairs well with

[`fastapi-cookiecutter`](https://github.com/marcius-llmus/fastapi-cookiecutter) — ships the wired baseline (DB engine, session manager, container, settings profiles, health app) so the skill governs *how to use it*, not how to scaffold it.

## Install

Clone into your Claude Code skills folder:

```bash
git clone <this-repo-url> ~/.claude/skills/fastapi-backend-architecture
```

That's it. Claude Code auto-discovers `SKILL.md` and activates the skill whenever a FastAPI footprint is detected (imports, dependencies, or layer-shaped paths like `routes.py`, `services/`, `repositories/`, `container/`).

## Layout

- `SKILL.md` — non-negotiable rules and reference matrix (the entry point).
- `STRUCTURE.md` — project layout, four-layer contract, mandatory invocation matrix.
- `references/` — per-topic deep references (services, repositories, dependencies, flows, testing, logging, migrations, ...).
- `scripts/` — standalone setup baselines.