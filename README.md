# Nexora

Nexora is an execution-first personal AI desktop platform. It combines conversation, memory, knowledge, browser automation, Windows desktop control, terminal execution, voice interaction, skills, and resumable background tasks.

Nexora is intentionally separate from the local Open WebUI Fork at `D:/AManager/code/open-webui`.

## Project Position

- **Open WebUI Fork**: conversation, knowledge, notes, calendar, memory, model access, users, and base tool management.
- **Nexora**: the product layer for task orchestration, desktop execution, browser workers, voice, skills, artifacts, approvals, and the Electron desktop app.

The source of truth for architecture and implementation priorities is [docs/DEVELOPMENT_PLAN.md](docs/DEVELOPMENT_PLAN.md).

## Repository Layout

```text
Nexora/
├─ apps/          # Electron desktop and standalone web surfaces
├─ services/      # Agent gateway, orchestration, workers, voice, scheduler
├─ packages/      # Shared protocols, SDKs, policies, and types
├─ skills/        # User-facing business capabilities
├─ integrations/  # Open WebUI, LiteLLM, MCP, and external systems
├─ infra/         # Local and production deployment assets
├─ scripts/       # Developer and release automation
└─ docs/          # Architecture, decisions, API, and security documentation
```

## Current Status

The repository currently contains the initial architecture and development plan. Runtime services will be added incrementally according to the plan.

## Important Rule

Do not put all new functionality into Open WebUI. Add new capabilities to the appropriate Nexora application, service, package, skill, or integration. Modify the Open WebUI Fork only when its UI or base platform contract must change.

