# Nexora Architecture and Development Plan

Status: baseline architecture, 2026-10-21

This document is the long-term source of truth for Nexora. It exists so the product direction remains stable even when implementation continues in a different chat or by multiple contributors.

## 1. Product Objective

Nexora is an execution-first personal AI desktop platform. A user should be able to state a goal in natural language or voice, and Nexora should plan, execute, verify, and report the work across:

- Windows applications and windows
- Browser pages and web services
- Terminal, Python, Git, and local files
- Documents, spreadsheets, presentations, and generated artifacts
- MCP services and external integrations
- Scheduled and background tasks

The product is not only a chat client. Conversation is the entry point; a durable task, its actions, approvals, and artifacts are the primary product objects.

## 2. Product Principles

1. **Execution over explanation**: a response is incomplete when the requested work can be safely performed but was not performed.
2. **Observable execution**: every task exposes its plan, current step, tool calls, progress, and final artifacts.
3. **Resumability**: tasks can pause, wait for approval/input, resume after restart, retry safe steps, and recover from partial failure.
4. **Least privilege**: tools receive only the directories, applications, credentials, and actions they need.
5. **Verification**: a successful tool call is not the same as a successful business outcome; each important action has a verification step.
6. **Provider neutrality**: OpenAI, DeepSeek, Alibaba/Qwen, local Ollama, and other OpenAI-compatible providers are selected through LiteLLM and logical model roles.
7. **Modularity**: new capability belongs in a worker, package, skill, or integration rather than an ever-growing core.
8. **Offline-capable distribution**: end users receive an installer or offline bundle and do not need GitHub to run the product.

## 3. Repository Boundaries

The Open WebUI Fork and Nexora are separate repositories:

```text
D:/AManager/code/open-webui/   # fork of Open WebUI
D:/AManager/code/Nexora/        # Nexora product and execution system
```

### Open WebUI Fork owns

- Chat UI and chat history
- Knowledge/RAG, Notes, Calendar, Memory
- Users, groups, permissions, and base settings
- Model/provider configuration surfaces
- Base MCP/tool/skill management where already provided

### Nexora owns

- Electron desktop shell
- Agent Gateway and task API
- Orchestrator and task state machine
- Browser, Windows, terminal, and voice workers
- Unified tool protocol and SDKs
- Skills and domain workflows
- Approval/risk policy and audit events
- Artifact lifecycle and previews
- Desktop-specific pages and background execution

Nexora may adapt or extend Open WebUI, but must not place all workers and orchestration logic inside the Open WebUI source tree.

## 4. Target Architecture

```text
┌──────────────────────────────────────────────────────────────┐
│ Nexora Desktop (Electron)                                    │
│ chat surface · task board · approvals · browser preview      │
│ tray · global hotkeys · voice · notifications · artifacts    │
└─────────────────────────────┬────────────────────────────────┘
                              │ HTTP/WebSocket
┌─────────────────────────────▼────────────────────────────────┐
│ Open WebUI Fork                                               │
│ conversation · knowledge · memory · notes · calendar          │
└─────────────────────────────┬────────────────────────────────┘
                              │ Agent API
┌─────────────────────────────▼────────────────────────────────┐
│ Nexora Agent Gateway                                          │
│ auth · task creation · event streaming · approvals            │
└─────────────────────────────┬────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────┐
│ Orchestrator                                                  │
│ planning · execution loop · state · retries · verification    │
│ pause/resume · background tasks · task journal                │
└───────┬────────────┬────────────┬─────────────┬──────────────┘
        │            │            │             │
  Browser Worker  Windows Worker Terminal      Voice Worker
  Playwright      UIA/Win32     PowerShell     ASR/TTS/VAD
  browser-use     input/capture Python/Git     realtime audio
```

## 5. Planned Directory Structure

```text
Nexora/
├─ apps/
│  ├─ desktop/                 # Electron shell and desktop UI
│  └─ web/                     # optional independent Nexora web UI
├─ services/
│  ├─ agent-gateway/           # public API and WebSocket events
│  ├─ orchestrator/            # task planning and state machine
│  ├─ browser-worker/           # Playwright + browser-use
│  ├─ windows-worker/           # Windows UI Automation, Win32, input, capture
│  ├─ terminal-worker/          # PowerShell, Python, Git, file operations
│  ├─ voice-worker/             # ASR, TTS, VAD, realtime sessions
│  ├─ scheduler/                # recurring and delayed jobs
│  └─ memory-service/           # durable user/project/task memory
├─ packages/
│  ├─ task-protocol/            # task, step, event, approval, artifact schemas
│  ├─ tool-sdk/                 # tool lifecycle and validation interface
│  ├─ skill-sdk/                # skill manifest and execution interface
│  ├─ security-policy/          # risk levels and authorization decisions
│  ├─ artifact-store/           # files, previews, checksums, retention
│  └─ shared-types/             # cross-service TypeScript/Python contracts
├─ skills/
│  ├─ file-organizer/
│  ├─ excel-assistant/
│  ├─ coding-agent/
│  ├─ browser-research/
│  └─ report-generator/
├─ integrations/
│  ├─ open-webui-adapter/
│  ├─ litellm/
│  ├─ mcp/
│  └─ external/                # WeChat, Feishu, email, databases, etc.
├─ infra/
│  ├─ docker/
│  ├─ postgres/
│  ├─ redis/
│  └─ observability/
├─ scripts/
└─ docs/
```

## 6. Core Runtime Contracts

### 6.1 Task lifecycle

```text
CREATED → PLANNING → WAITING_APPROVAL → EXECUTING
        → WAITING_INPUT → VERIFYING → COMPLETED
```

Terminal states and recovery states:

```text
PAUSED · FAILED · RETRYING · CANCELLED · EXPIRED
```

Every task records its goal, plan, current step, worker calls, approvals, risk level, errors, and artifacts.

### 6.2 Tool lifecycle

Every tool should provide:

```text
validate(input)
authorize(context)
execute(input)
verify(result)
rollback(result)       # when a reversible operation is possible
```

The tool result must distinguish `accepted`, `executed`, and `verified`. A process accepting an input does not prove that the requested business outcome occurred.

### 6.3 Event stream

The Agent Gateway will stream typed events such as:

```text
task.created
task.plan.created
task.step.started
tool.call.started
tool.call.output
approval.required
task.progress
artifact.created
task.step.verified
task.completed
task.failed
```

The Electron UI, Open WebUI adapter, logs, and future mobile clients consume the same event contract.

## 7. Execution Domains

### Browser

- Playwright provides deterministic browser control.
- browser-use provides model-driven action selection.
- Prefer DOM/accessibility state over visual clicks.
- Keep browser profiles, downloads, uploads, cookies, and target identities isolated.
- Verify business outcomes after navigation, submission, or download.

### Windows desktop

- Windows UI Automation/pywinauto for semantic controls.
- Win32 for bounded window and process operations.
- SendInput only through a validated desktop session.
- Windows Graphics Capture or an equivalent bounded capture path for screenshots.
- Maintain window identity, focus, geometry, and revision checks.
- Require confirmation for destructive, external, privileged, or irreversible actions.

### Terminal and files

- PowerShell, Python, Git, and file tools run through a controlled worker.
- Workspaces and allowed paths are explicit.
- Commands are represented as structured tool calls, not raw model-generated shell strings.
- Output is bounded, streamed, redacted, and attached to the task journal.

### Voice

- VAD → streaming ASR → conversation/task runtime → streaming TTS.
- Support interruption, cancellation, turn IDs, and voice confirmation.
- Credentials remain in backend/OS secret storage and never in renderer-visible configuration.

## 8. Model and Provider Strategy

LiteLLM is the provider boundary. Nexora uses logical roles:

```text
fast-chat · planner · executor · coding · vision · reviewer · voice
```

Each deployment may map roles to OpenAI, DeepSeek, Alibaba/Qwen, Ollama, or another OpenAI-compatible endpoint. Provider keys are user-scoped or admin-scoped, encrypted at rest, excluded from logs, and never hard-coded in source.

## 9. Security and Approval Model

Risk levels:

| Level | Examples | Default |
|---|---|---|
| Low | read public page, inspect workspace | automatic |
| Medium | create files, run ordinary scripts | configurable |
| High | delete/overwrite, send message, upload | confirmation |
| Critical | admin commands, payments, secrets, registry | mandatory confirmation |

The execution path is:

```text
tool request → policy evaluation → approval if needed
→ bounded execution → verification → audit event
```

The system must protect its own processes, configuration, credentials, and data directories from tool calls that attempt to disable or alter the platform.

## 10. Phased Development Plan

### Phase 0: Foundation and contracts

- Create repository and documentation baseline.
- Define task, step, tool, approval, artifact, and event schemas.
- Define local configuration and secret-storage rules.
- Establish CI, formatting, linting, and test conventions.

**Exit condition:** a versioned protocol package exists and all later services can depend on it.

### Phase 1: Executable assistant MVP

- Agent Gateway API.
- Orchestrator with task state machine.
- Terminal worker for PowerShell, Python, Git, and bounded file operations.
- LiteLLM provider routing.
- Task journal and streamed progress.
- Approval prompts for destructive actions.

**Exit condition:** a user can ask Nexora to inspect files, modify a project, run tests, and return verified artifacts.

### Phase 2: Open WebUI integration

- Open WebUI adapter.
- Task creation from chat.
- Task progress embedded in conversation.
- Shared authentication and user identity.
- Knowledge/context injection into task planning.

**Exit condition:** Open WebUI can start, observe, approve, pause, resume, and receive results from Nexora tasks.

### Phase 3: Browser execution

- Playwright worker.
- browser-use action loop.
- Browser profiles and target registration.
- Page preview, screenshots, downloads, uploads, and navigation history.
- Browser task verification and recovery.

**Exit condition:** Nexora can complete a bounded multi-step website workflow and provide evidence/artifacts.

### Phase 4: Windows execution

- Electron shell.
- Windows worker service.
- UI Automation and window discovery.
- Mouse/keyboard input with focus and identity checks.
- Bounded screenshots and application launching.
- Safety policy for privileged/destructive actions.

**Exit condition:** Nexora can complete controlled workflows in selected Windows applications such as Notepad and Excel.

### Phase 5: Voice and desktop experience

- Voice worker with streaming ASR/TTS/VAD.
- Interruptible voice turns.
- Tray, global hotkey, floating assistant, notifications.
- Task status and approval UI in Electron.

**Exit condition:** a user can start, monitor, interrupt, and approve tasks by voice and desktop UI.

### Phase 6: Skills, integrations, and background execution

- Skill SDK and manifest format.
- Skill validation and tests.
- MCP registry and permission declarations.
- Scheduler, recurring tasks, background reports.
- Artifact previews and retention.

**Exit condition:** new capabilities can be added as skills or integrations without modifying the orchestrator core.

### Phase 7: Distribution and scale

- Windows installer and offline bundle.
- Optional Docker deployment for server components.
- Auto-update or controlled offline update.
- Database migrations and rollback.
- Telemetry/diagnostics with privacy controls.
- Multi-user and multi-device access.

**Exit condition:** a user can install and operate Nexora without cloning GitHub or manually installing development dependencies.

## 11. Definition of Done for a Capability

A new capability is complete only when it has:

- a clear owner module;
- a typed tool or service contract;
- input validation;
- permission and risk classification;
- progress and error events;
- verification behavior;
- tests for normal, failure, and boundary cases;
- documentation and an example;
- no secrets committed to the repository;
- an explicit installation/deployment path.

## 12. Git and Release Strategy

- `open-webui` remains a separately maintained Fork.
- `Nexora` is the product repository.
- Development can run the two repositories side by side.
- The final installer packages the required Open WebUI build and Nexora services; end users do not need GitHub.
- A future release pipeline may vendor or pin the Open WebUI Fork at a known commit, but runtime must not require a live GitHub connection.
- Preserve Open WebUI and third-party license notices in every distribution.

## 13. Immediate Next Steps

1. Add the initial `packages/task-protocol` schemas.
2. Create the Agent Gateway skeleton with health and task endpoints.
3. Create the Orchestrator skeleton with durable task states.
4. Add a safe terminal/file worker for a development workspace only.
5. Add a local LiteLLM configuration with logical model roles.
6. Build the first Open WebUI adapter endpoint.
7. Add integration tests for task creation, progress, approval, completion, and failure.

Do not implement Windows desktop control or unrestricted command execution before the task protocol and approval model exist.

