# Nexora Architecture Summary

This is the concise map for contributors. The detailed milestones and rules are in [DEVELOPMENT_PLAN.md](DEVELOPMENT_PLAN.md).

```text
Desktop UI / Web UI
        ↓
Open WebUI Adapter + Agent Gateway
        ↓
Orchestrator and durable Task Journal
        ↓
Tool SDK / Security Policy
        ↓
Browser · Windows · Terminal · Voice · MCP workers
        ↓
Artifacts, verification, audit events
```

The core dependency direction is one-way:

```text
apps → services → packages
skills → packages/services
integrations → packages/services
```

Workers must not import UI components. Skills must not reach directly into another worker's private implementation; they use the Tool SDK and typed contracts.

