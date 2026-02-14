# OpenClaw Architecture Deep Dive

## System overview

OpenClaw is a multi-channel AI gateway and agent runtime. A long-running Gateway process owns messaging/channel connections, exposes a typed WebSocket/HTTP control plane, and orchestrates agent runs, sessions, cron jobs, plugin extensions, and device nodes.

Core execution shape:

1. Inbound event arrives from a channel plugin.
2. Route resolver selects `agentId` + canonical `sessionKey`.
3. Auto-reply pipeline applies commands/directives/policies.
4. Agent runtime executes model/tool loop (streamed events).
5. Session metadata/transcript is updated.
6. Output is delivered to target channel(s) via outbound adapters.

---

## Repository structure

- `src/`
  - `gateway/`: WS/HTTP control plane, auth, method handlers, protocol schemas.
  - `agents/`: model runtime, tool policies, embedded PI runner, fallbacks, auth profiles.
  - `auto-reply/`: inbound message orchestration, directives, queueing, typing, run dispatch.
  - `channels/`: channel abstractions, docking metadata, channel plugin interfaces.
  - `config/`: config IO, validation, defaults, migration, includes/env substitution.
  - `cron/`: persistent scheduler and isolated/main session execution flows.
  - `memory/`: memory index/search managers and embedding backends.
  - `routing/`: agent/session key derivation and binding-based route resolution.
  - `sessions/`: session policy helpers and metadata behavior.
  - `plugins/`: plugin discovery, manifest validation, runtime registry, hook wiring.
- `extensions/*`: installable/bundled plugin packages (channels, memory, auth providers, etc.).
- `ui/`: Control UI web client.
- `apps/{macos,ios,android,shared}`: native clients/nodes.
- `packages/{clawdbot,moltbot}`: compatibility CLI shims forwarding to OpenClaw.

---

## Startup and CLI flow

Primary bootstrap path:

- `openclaw.mjs` → loads compiled entrypoint (`dist/entry.*`).
- `src/entry.ts` normalizes env/argv and calls CLI runtime.
- `src/cli/run-main.ts` executes route-first command handling for faster startup.
- `src/cli/program/*` registers command tree with lazy sub-CLI loading.

Design intent:

- Keep startup fast for frequent operations (`status`, `health`, `sessions`).
- Avoid loading heavyweight modules unless needed.

---

## Gateway control plane

Key files:

- `src/gateway/server.impl.ts`
- `src/gateway/server-http.ts`
- `src/gateway/server/ws-connection.ts`
- `src/gateway/server/ws-connection/message-handler.ts`
- `src/gateway/server-methods.ts`

Gateway responsibilities:

- Own channel connectivity and runtime state.
- Serve typed WebSocket RPC/event transport.
- Serve HTTP APIs (OpenAI-compatible, OpenResponses, tools invoke, hooks, Control UI).
- Manage cron, heartbeat, node registry, pairing workflows, and plugin-provided handlers.
- Broadcast presence/health/agent/chat events to connected clients.

Protocol model:

- First WS frame must be `connect`.
- Request frames (`req`) get typed response frames (`res`).
- Event frames push async state (`agent`, `chat`, `presence`, `health`, etc.).
- Schema validation enforced via TypeBox + AJV (`src/gateway/protocol/*`).

---

## Authentication, authorization, pairing

Core auth path:

- `src/gateway/auth.ts`

Supported gateway auth modes:

- Token
- Password
- Tailscale identity-based path (config-gated)

Connection hardening:

- Device identity signatures verified at connect.
- Nonce challenge required for non-local contexts.
- Pairing approval required for new device identities.
- Role/scope checks enforced per method (`src/gateway/server-methods.ts`).

Role model:

- `operator` clients: control APIs with scope gates (`operator.read`, `operator.write`, `operator.admin`, etc.).
- `node` clients: restricted to node-specific event/result surfaces.

Browser-side protections:

- Origin checks for Control UI/WebChat (`src/gateway/origin-check.ts`).

---

## Agent runtime and execution lanes

Key files:

- `src/commands/agent.ts`
- `src/agents/pi-embedded-runner/run.ts`
- `src/agents/pi-embedded-subscribe.ts`
- `src/process/command-queue.ts`

Execution characteristics:

- Session-aware lane serialization prevents race conditions in shared context.
- Optional global lane concurrency controls runtime throughput.
- Model selection supports provider/model overrides and fallback chains.
- Auth profile resolution includes cooldown/rotation behavior.
- Agent stream emits lifecycle + assistant + tool events.

Output + persistence:

- Session usage/model metadata is written back after runs.
- Delivery layer optionally sends response payloads via channel adapters.

---

## Session model

Key files:

- `src/config/sessions/store.ts`
- `src/config/sessions/metadata.ts`
- `src/routing/session-key.ts`
- `src/routing/resolve-route.ts`

Session identity:

- Canonical key format anchored by `agent:<agentId>:...`.
- DM scope can collapse or isolate DM context (`main`, `per-peer`, `per-channel-peer`, `per-account-channel-peer`).
- Group/channel/thread sessions remain isolated by conversation identity.

Persistence:

- Session map: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Transcript logs: `~/.openclaw/agents/<agentId>/sessions/*.jsonl`

Important behavior:

- Route/binding logic can map by channel/account/peer/guild/team/roles.
- Session metadata includes delivery/origin fields for explainability and routing continuity.

---

## Auto-reply orchestration

Key file:

- `src/auto-reply/reply/get-reply.ts`

Pipeline stages:

- Finalize inbound context.
- Resolve command authorization and command/directive semantics.
- Apply reset/model/thinking/verbose and queue controls.
- Execute inline actions when applicable.
- Run agent path with typing/streaming controls.

This layer is the high-level “message brain” connecting raw inbound messages to agent execution.

---

## Channels and extension architecture

Channel abstraction:

- `src/channels/plugins/types.core.ts`
- `src/channels/dock.ts`
- `src/channels/registry.ts`

Plugin system:

- `src/plugins/discovery.ts`
- `src/plugins/loader.ts`
- `src/plugins/registry.ts`
- `src/plugins/runtime.ts`

How extensions participate:

- Register channel implementations, tools, hooks, gateway methods, HTTP routes, CLI hooks.
- `extensions/*` packages provide concrete integrations (telegram, whatsapp, matrix, memory-core, etc.).

Trust model:

- Plugins execute in-process; they are trusted code at runtime.
- Loader validates schema and registration conflicts but does not sandbox plugin code.

---

## Cron and automations

Key files:

- `src/cron/service.ts`
- `src/cron/service/ops.ts`
- `src/cron/isolated-agent/run.ts`

Modes:

- Main-session wake/system-event jobs.
- Isolated agent-turn jobs in dedicated cron sessions.

Scheduler state is persistent and survives restarts.

---

## Node/device subsystem

Key files:

- `src/gateway/node-registry.ts`
- `src/gateway/server-methods/nodes.ts`
- `src/gateway/server-node-events.ts`

Capabilities:

- Paired node clients (macOS/iOS/Android/headless) connect as `role: node`.
- Gateway can invoke node commands (`node.invoke`) and receive results/events.
- Node events can feed session context and trigger heartbeat/system events.

Security:

- Pairing/token validation required.
- Allowed command filtering enforced at connect/runtime.

---

## HTTP surfaces

Key files:

- `src/gateway/openai-http.ts`
- `src/gateway/openresponses-http.ts`
- `src/gateway/tools-invoke-http.ts`
- `src/gateway/server-http.ts`

Provides:

- OpenAI-style chat completions endpoint.
- OpenResponses endpoint with tool/file/image flows.
- Direct tools invoke endpoint.
- Hook ingestion with token-based auth.

Controls:

- Shared auth checks via gateway auth.
- Request validation and body limits.
- URL/file/media policy controls for ingestion paths.

---

## Memory subsystem

Key files:

- `src/memory/manager.ts`
- `src/memory/search-manager.ts`
- `extensions/memory-core/index.ts`

Model:

- Markdown files are source of truth.
- Optional vector retrieval/indexing backends provide semantic recall.
- Memory tools are exposed via plugin slot (`memory_search`, `memory_get`).

---

## Security posture summary

Primary controls implemented in code:

- Strict WS handshake + protocol schema validation.
- Gateway auth modes + optional tailscale identity path.
- Device signature + nonce + pairing gates.
- Role/scope authorization for gateway method access.
- Browser origin checks for UI clients.
- Session/send policy constraints.
- Plugin registration validation and conflict detection.
- Security auditing/fix CLI (`openclaw security audit`).

Residual risk categories:

- In-process plugin trust boundary.
- Prompt/tool abuse if policies are too permissive.
- Misconfigured DM scope in multi-user environments.
- External data ingress via web/file tools without restrictive policy.

---

## Operationally important invariants

- One gateway instance is the session/control authority on a host.
- Session keys are canonicalized; routing controls context boundaries.
- Agent runs are lane-serialized per session for consistency.
- Events are transient stream state; clients should refresh on reconnect.

---

## Practical reading order for contributors

1. `src/entry.ts`
2. `src/cli/run-main.ts`
3. `src/gateway/server.impl.ts`
4. `src/gateway/server/ws-connection/message-handler.ts`
5. `src/gateway/server-methods.ts`
6. `src/auto-reply/reply/get-reply.ts`
7. `src/commands/agent.ts`
8. `src/agents/pi-embedded-runner/run.ts`
9. `src/routing/resolve-route.ts`
10. `src/config/sessions/store.ts`
11. `src/plugins/loader.ts`
12. `docs/concepts/architecture.md`
