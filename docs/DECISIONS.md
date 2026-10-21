# Architecture Decisions

## ADR-001: Nexora is separate from Open WebUI

**Decision:** Maintain the Open WebUI Fork and Nexora as separate repositories.

**Reason:** Open WebUI provides valuable chat and knowledge functionality, while Nexora needs an independent execution core, desktop shell, workers, and release lifecycle. This keeps upstream synchronization possible and prevents the Open WebUI source tree becoming a catch-all.

## ADR-002: Open WebUI is not a runtime GitHub dependency

**Decision:** Releases package the required Open WebUI build and Nexora services. End users do not clone GitHub.

**Reason:** Users may not have GitHub access and should not need development tooling.

## ADR-003: LiteLLM is the provider boundary

**Decision:** Product code uses logical model roles and delegates provider-specific configuration to LiteLLM.

**Reason:** Users and deployments may use OpenAI, DeepSeek, Alibaba/Qwen, Ollama, or other compatible providers without code changes.

## ADR-004: Execution is task-oriented

**Decision:** Durable tasks, events, approvals, artifacts, and verification are first-class objects.

**Reason:** A chat response alone cannot represent long-running, resumable, multi-tool desktop work.

