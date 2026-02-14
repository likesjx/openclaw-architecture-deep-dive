# OpenClaw Architecture Reference (Full)

This document is a practical, implementation-oriented architecture guide for the OpenClaw repository.
It is written to help you understand how the system actually works in code: startup, gateway transport,
routing/session boundaries, agent execution, channels/plugins, cron, memory, nodes, and security controls.

## 1. System model

OpenClaw is a long-running AI control plane (Gateway) plus channel adapters and agent runtime.
One gateway instance on a host owns channel connectivity and session state.

High-level data flow:

1. Inbound message/event arrives from channel runtime/plugin.
2. Route resolution maps inbound context to `agentId` and canonical `sessionKey`.
3. Auto-reply pipeline parses directives/commands/policies.
4. Agent runtime executes model+tool loop with per-session queueing.
5. Session metadata + transcript are persisted.
6. Outbound delivery sends final payloads to channel destinations.
7. Gateway broadcasts runtime events (`agent`, `chat`, `presence`, etc.) to clients.

## 2. Repository topology

- Core runtime: [`src/`](https://github.com/openclaw/openclaw/tree/main/src/)
- Extension packages: [`extensions/*`](https://github.com/openclaw/openclaw/tree/main/extensions)
- Control UI: [`ui/`](https://github.com/openclaw/openclaw/tree/main/ui/)
- Native clients/nodes: [`apps/{macos,ios,android,shared}`](https://github.com/openclaw/openclaw/tree/main/apps)
- Compatibility shims: [`packages/{clawdbot,moltbot}`](https://github.com/openclaw/openclaw/tree/main/packages)
- Documentation: [`docs/`](https://github.com/openclaw/openclaw/tree/main/docs/)

Largest implementation areas (by file count):

- [`src/agents`](https://github.com/openclaw/openclaw/tree/main/src/agents)
- [`src/commands`](https://github.com/openclaw/openclaw/tree/main/src/commands)
- [`src/auto-reply`](https://github.com/openclaw/openclaw/tree/main/src/auto-reply)
- [`src/gateway`](https://github.com/openclaw/openclaw/tree/main/src/gateway)
- [`src/infra`](https://github.com/openclaw/openclaw/tree/main/src/infra)
- [`src/cli`](https://github.com/openclaw/openclaw/tree/main/src/cli)
- [`src/config`](https://github.com/openclaw/openclaw/tree/main/src/config)

## 3. Startup and CLI bootstrap

### 3.1 Entrypoints

- CLI bin: [`openclaw.mjs`](https://github.com/openclaw/openclaw/blob/main/openclaw.mjs)
- Runtime entry: [`src/entry.ts`](https://github.com/openclaw/openclaw/blob/main/src/entry.ts)
- CLI orchestration: [`src/cli/run-main.ts`](https://github.com/openclaw/openclaw/blob/main/src/cli/run-main.ts)
- Command registration: [`src/cli/program/*`](https://github.com/openclaw/openclaw/tree/main/src/cli/program)

### 3.2 Behavior

- Environment normalization + warning filters are installed early.
- Route-first execution path handles common commands quickly.
- Full command tree is lazily loaded to reduce startup overhead.

### 3.3 Command surface

Core command families include gateway control, status/health/sessions, channels, models, cron,
security, skills/plugins, docs/helpers, device/node control.

## 4. Gateway architecture

### 4.1 Core gateway files

- Server lifecycle: [`src/gateway/server.impl.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server.impl.ts)
- HTTP serving + hooks/UI/API wiring: [`src/gateway/server-http.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-http.ts)
- WS attach and connection handling:
  - [`src/gateway/server-ws-runtime.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-ws-runtime.ts)
  - [`src/gateway/server/ws-connection.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server/ws-connection.ts)
  - [`src/gateway/server/ws-connection/message-handler.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server/ws-connection/message-handler.ts)
- Method registry/dispatch:
  - [`src/gateway/server-methods-list.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods-list.ts)
  - [`src/gateway/server-methods.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods.ts)

### 4.2 Responsibilities

- Starts WS + HTTP servers.
- Loads and validates config (with migration/auto-enable behavior).
- Initializes channel manager, cron service, heartbeat, discovery, node registry.
- Loads plugin-provided methods/channels/http routes.
- Maintains global runtime state and broadcasts events.

### 4.3 Protocol

- First WS request must be `connect`.
- Request/response/event frames are schema-validated.
- Features/methods/events are declared in handshake payload.
- Gateway emits snapshots (`presence`, `health`) and state versions.

Protocol schema and validation:

- [`src/gateway/protocol/index.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/protocol/index.ts)
- [`src/gateway/protocol/schema/*`](https://github.com/openclaw/openclaw/tree/main/src/gateway/protocol/schema)

## 5. Authentication, pairing, and authorization

### 5.1 Gateway auth

Implementation: [`src/gateway/auth.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/auth.ts)

Auth modes:

- token
- password
- tailscale-derived path (config gated)

### 5.2 WS connect hardening

In [`src/gateway/server/ws-connection/message-handler.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server/ws-connection/message-handler.ts):

- validates connect frame and protocol versions
- verifies role and scopes
- checks origin for Control UI/WebChat
- verifies device identity/signature/timestamp/nonce
- enforces pairing requirement for unknown or upgraded permission sets
- issues/rotates device tokens

### 5.3 Method authorization

[`src/gateway/server-methods.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods.ts) applies role/scope checks before method handler execution.

Common scope levels:

- `operator.read`
- `operator.write`
- `operator.admin`
- pairing/approval specialized scopes

### 5.4 Browser origin control

- [`src/gateway/origin-check.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/origin-check.ts) enforces host/origin allow rules for browser clients.

## 6. Agent execution model

### 6.1 Main path

- Front door: [`src/commands/agent.ts`](https://github.com/openclaw/openclaw/blob/main/src/commands/agent.ts)
- Embedded runner: [`src/agents/pi-embedded-runner/run.ts`](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run.ts)
- Streaming bridge: [`src/agents/pi-embedded-subscribe.ts`](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-subscribe.ts)

### 6.2 Steps per run

1. Resolve session, workspace, agent scope, run context.
2. Resolve model/provider defaults + overrides + fallback plan.
3. Resolve auth profile and provider credentials.
4. Queue execution on session lane (and optional global lane).
5. Execute embedded model/tool loop.
6. Stream lifecycle/assistant/tool events.
7. Persist session metadata/tokens/model state.
8. Deliver payloads (if deliver enabled).

### 6.3 Lane/queueing model

- queue engine: [`src/process/command-queue.ts`](https://github.com/openclaw/openclaw/blob/main/src/process/command-queue.ts)
- lane IDs: [`src/process/lanes.ts`](https://github.com/openclaw/openclaw/blob/main/src/process/lanes.ts)
- gateway lane tuning: [`src/gateway/server-lanes.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-lanes.ts)

Goal: deterministic session consistency under concurrent inputs.

## 7. Sessions and routing

### 7.1 Routing

- route resolver: [`src/routing/resolve-route.ts`](https://github.com/openclaw/openclaw/blob/main/src/routing/resolve-route.ts)
- key helpers: [`src/routing/session-key.ts`](https://github.com/openclaw/openclaw/blob/main/src/routing/session-key.ts)

Inputs considered:

- channel, account, peer, parent peer, guild/team, role IDs
- configured bindings and default agent selection

### 7.2 Session key model

Canonical namespace is agent-scoped (`agent:<agentId>:...`).

DM scope controls context isolation:

- `main`
- `per-peer`
- `per-channel-peer`
- `per-account-channel-peer`

### 7.3 Store/transcripts

Session store and metadata:

- [`src/config/sessions/store.ts`](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store.ts)
- [`src/config/sessions/metadata.ts`](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/metadata.ts)

On disk (gateway host):

- `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- `~/.openclaw/agents/<agentId>/sessions/*.jsonl`

### 7.4 Security implications

- Shared DM scope in multi-user ingress can leak context.
- Send policy and route metadata are important guardrails.

## 8. Auto-reply orchestration

Main orchestrator: [`src/auto-reply/reply/get-reply.ts`](https://github.com/openclaw/openclaw/blob/main/src/auto-reply/reply/get-reply.ts)

Subsystem responsibilities:

- finalize inbound context
- command/directive parsing
- model/thinking/verbose/session reset handling
- queue and follow-up behavior
- typing and block streaming controls
- run dispatch into agent runtime

This layer is the glue between channel envelopes and executable agent runs.

## 9. Channels and plugin architecture

### 9.1 Channel abstraction

- Types: [`src/channels/plugins/types.core.ts`](https://github.com/openclaw/openclaw/blob/main/src/channels/plugins/types.core.ts)
- Dock behavior map: [`src/channels/dock.ts`](https://github.com/openclaw/openclaw/blob/main/src/channels/dock.ts)
- Registry and normalization: [`src/channels/registry.ts`](https://github.com/openclaw/openclaw/blob/main/src/channels/registry.ts)

### 9.2 Plugin runtime

- discovery: [`src/plugins/discovery.ts`](https://github.com/openclaw/openclaw/blob/main/src/plugins/discovery.ts)
- loader: [`src/plugins/loader.ts`](https://github.com/openclaw/openclaw/blob/main/src/plugins/loader.ts)
- registry: [`src/plugins/registry.ts`](https://github.com/openclaw/openclaw/blob/main/src/plugins/registry.ts)
- active runtime state: [`src/plugins/runtime.ts`](https://github.com/openclaw/openclaw/blob/main/src/plugins/runtime.ts)

### 9.3 Extension packages

Examples:

- [`extensions/telegram`](https://github.com/openclaw/openclaw/tree/main/extensions/telegram)
- [`extensions/whatsapp`](https://github.com/openclaw/openclaw/tree/main/extensions/whatsapp)
- [`extensions/slack`](https://github.com/openclaw/openclaw/tree/main/extensions/slack)
- [`extensions/discord`](https://github.com/openclaw/openclaw/tree/main/extensions/discord)
- [`extensions/memory-core`](https://github.com/openclaw/openclaw/tree/main/extensions/memory-core)
- many additional channels/providers/features

### 9.4 Trust model

Plugins are in-process and should be treated as trusted runtime code.
Validation protects registration/schema consistency, not behavioral sandboxing.

## 10. HTTP APIs

- OpenAI-compatible endpoint: [`src/gateway/openai-http.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/openai-http.ts)
- OpenResponses endpoint: [`src/gateway/openresponses-http.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/openresponses-http.ts)
- Direct tools invoke: [`src/gateway/tools-invoke-http.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/tools-invoke-http.ts)
- Shared HTTP mux + hooks/UI: [`src/gateway/server-http.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-http.ts)

Controls used:

- shared auth checks
- body-size and parse guards
- schema checks and policy checks
- hook auth throttling and strict token checks

## 11. Cron subsystem

- service wrapper: [`src/cron/service.ts`](https://github.com/openclaw/openclaw/blob/main/src/cron/service.ts)
- operation logic: [`src/cron/service/ops.ts`](https://github.com/openclaw/openclaw/blob/main/src/cron/service/ops.ts)
- isolated agent turns: [`src/cron/isolated-agent/run.ts`](https://github.com/openclaw/openclaw/blob/main/src/cron/isolated-agent/run.ts)

Modes:

- main-session system-event wake jobs
- isolated agent-turn jobs in `cron:<jobId>` session space

State is persisted and survives restarts.

## 12. Nodes and device runtime

- node registry: [`src/gateway/node-registry.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/node-registry.ts)
- node methods: [`src/gateway/server-methods/nodes.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods/nodes.ts)
- node event bridge: [`src/gateway/server-node-events.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-node-events.ts)

Node clients connect with role `node`, advertise caps/commands, and support `node.invoke` workflows.
Pairing and tokens gate trust and command surfaces.

## 13. Memory subsystem

- manager/search/index files in [`src/memory/*`](https://github.com/openclaw/openclaw/tree/main/src/memory)
- plugin exposure in [`extensions/memory-core/index.ts`](https://github.com/openclaw/openclaw/blob/main/extensions/memory-core/index.ts)

Memory model:

- markdown files are durable source of truth
- retrieval layer can use vector backends for semantic lookup
- memory tools are exposed through plugin slot system

## 14. Config system

- config IO: [`src/config/io.ts`](https://github.com/openclaw/openclaw/blob/main/src/config/io.ts)
- validation: [`src/config/validation.ts`](https://github.com/openclaw/openclaw/blob/main/src/config/validation.ts)
- migration: [`src/config/legacy-migrate.ts`](https://github.com/openclaw/openclaw/blob/main/src/config/legacy-migrate.ts)

Features:

- JSON5 parsing
- include expansion
- env substitution
- defaults application
- runtime overrides
- duplicate agent dir checks

## 15. UI and apps

- Control UI web app: `ui/`
- macOS app: [`apps/macos`](https://github.com/openclaw/openclaw/tree/main/apps/macos)
- iOS app: [`apps/ios`](https://github.com/openclaw/openclaw/tree/main/apps/ios)
- Android app: [`apps/android`](https://github.com/openclaw/openclaw/tree/main/apps/android)
- shared SDK/protocol: [`apps/shared/OpenClawKit`](https://github.com/openclaw/openclaw/tree/main/apps/shared/OpenClawKit)

All clients integrate through gateway protocol semantics and pairing/auth models.

## 16. Security controls and residual risks

Implemented controls:

- strict WS handshake + typed schema validation
- gateway token/password auth options
- device signature + nonce + pairing checks
- role/scope method authorization
- browser origin checks
- tool/send/session policy layers
- security audit command surface

Residual risks to manage operationally:

- plugin trust boundary (in-process code)
- permissive tool policies + untrusted content
- DM scope misconfiguration in multi-user deployments
- external ingress tools without restrictive allowlists

## 17. Fast code-reading path

Recommended file order for onboarding:

1. [`src/entry.ts`](https://github.com/openclaw/openclaw/blob/main/src/entry.ts)
2. [`src/cli/run-main.ts`](https://github.com/openclaw/openclaw/blob/main/src/cli/run-main.ts)
3. [`src/gateway/server.impl.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server.impl.ts)
4. [`src/gateway/server/ws-connection/message-handler.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server/ws-connection/message-handler.ts)
5. [`src/gateway/server-methods.ts`](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods.ts)
6. [`src/auto-reply/reply/get-reply.ts`](https://github.com/openclaw/openclaw/blob/main/src/auto-reply/reply/get-reply.ts)
7. [`src/commands/agent.ts`](https://github.com/openclaw/openclaw/blob/main/src/commands/agent.ts)
8. [`src/agents/pi-embedded-runner/run.ts`](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run.ts)
9. [`src/routing/resolve-route.ts`](https://github.com/openclaw/openclaw/blob/main/src/routing/resolve-route.ts)
10. [`src/config/sessions/store.ts`](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store.ts)
11. [`src/plugins/loader.ts`](https://github.com/openclaw/openclaw/blob/main/src/plugins/loader.ts)
12. [`docs/concepts/architecture.md`](https://github.com/openclaw/openclaw/blob/main/docs/concepts/architecture.md)

## 18. Public export note

This document is intended as the public, usable architecture artifact from this thread.
No private keys/tokens/system prompts are included; content is limited to open-source architecture analysis.
