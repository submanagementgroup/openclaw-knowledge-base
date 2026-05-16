---
domain: plugins
topic: "Plugin SDK Runtime API Reference"
type: reference
keywords:
  - SDK runtime
  - plugin runtime
  - runtime API
  - context
  - tools API
  - SDK context
source: plugins/sdk-runtime.md
---

Reference for the `api.runtime` object injected into every plugin during registration. Use these helpers instead of importing host internals directly.

**Channel plugins:** Step-by-step guide that uses these helpers in context for channel plugins.
  

**Provider plugins:** Step-by-step guide that uses these helpers in context for provider plugins.
  

```typescript
register(api) {
  const runtime = api.runtime;
}
```

## Config loading and writes

Prefer config that was already passed into the active call path, for example `api.config` during registration or a `cfg` argument on channel/provider callbacks. This keeps one process snapshot flowing through the work instead of reparsing config on hot paths.

Use `api.runtime.config.current()` only when a long-lived handler needs the current process snapshot and no config was passed to that function. The returned value is readonly; clone or use a mutation helper before editing.

Tool factories receive `ctx.runtimeConfig` plus `ctx.getRuntimeConfig()`. Use the getter inside a long-lived tool's `execute` callback when config can change after the tool definition was created.

Persist changes with `api.runtime.config.mutateConfigFile(...)` or `api.runtime.config.replaceConfigFile(...)`. Each write must choose an explicit `afterWrite` policy:

- `afterWrite: { mode: "auto" }` lets the gateway reload planner decide.
- `afterWrite: { mode: "restart", reason: "..." }` forces a clean restart when the writer knows hot reload is unsafe.
- `afterWrite: { mode: "none", reason: "..." }` suppresses automatic reload/restart only when the caller owns the follow-up.

The mutation helpers return `afterWrite` plus a typed `followUp` summary so callers can log or test whether they requested a restart. The gateway still owns when that restart actually happens.

`api.runtime.config.loadConfig()` and `api.runtime.config.writeConfigFile(...)` are deprecated compatibility helpers under `runtime-config-load-write`. They warn once at runtime, and remain available for old external plugins during the migration window. Bundled plugins must not use them; the config boundary guards fail if plugin code calls them or imports those helpers from plugin SDK subpaths.

For direct SDK imports, use the focused config subpaths instead of the broad
`openclaw/plugin-sdk/config-runtime` compatibility barrel: `config-contracts` for
types, `plugin-config-runtime` for already-loaded config assertions and plugin
entry lookup, `runtime-config-snapshot` for current process snapshots, and
`config-mutation` for writes. Bundled plugin tests should mock these focused
subpaths directly instead of mocking the broad compatibility barrel.

Internal OpenClaw runtime code has the same direction: load config once at the CLI, gateway, or process boundary, then pass that value through. Successful mutation writes refresh the process runtime snapshot and advance its internal revision; long-lived caches should key off the runtime-owned cache key instead of serializing config locally. Long-lived runtime modules have a zero-tolerance scanner for ambient `loadConfig()` calls; use a passed `cfg`, a request `context.getRuntimeConfig()`, or `getRuntimeConfig()` at an explicit process boundary.

Provider and channel execution paths must use the active runtime config snapshot, not a file snapshot returned for config readback or editing. File snapshots preserve source values such as SecretRef markers for UI and writes; provider callbacks need the resolved runtime view. When a helper may be called with either the active source snapshot or the active runtime snapshot, route through `selectApplicableRuntimeConfig()` before reading credentials.

## Runtime namespaces

### api.runtime.agent

Agent identity, directories, and session management.

    ```typescript
    // Resolve the agent's working directory
    const agentDir = api.runtime.agent.resolveAgentDir(cfg);

    // Resolve agent workspace
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg);

    // Get agent identity
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // Get default thinking level
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // Validate a user-provided thinking level against the active provider profile
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // pass level to an embedded run
    }

    // Get agent timeout
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // Ensure workspace exists
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // Run an embedded agent turn
    const agentDir = api.runtime.agent.resolveAgentDir(cfg);
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      sessionFile: path.join(agentDir, "sessions", "my-plugin-task-1.jsonl"),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg),
      prompt: "Summarize the latest changes",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)` is the neutral helper for starting a normal OpenClaw agent turn from plugin code. It uses the same provider/model resolution and agent-harness selection as channel-triggered replies.

    `runEmbeddedPiAgent(...)` remains as a compatibility alias.

    `resolveThinkingPolicy(...)` returns the provider/model's supported thinking levels and optional default. Provider plugins own the model-specific profile through their thinking hooks, so tool plugins should call this runtime helper instead of importing or duplicating provider lists.

    `normalizeThinkingLevel(...)` converts user text such as `on`, `x-high`, or `extra high` to the canonical stored level before checking it against the resolved policy.

    **Session store helpers** are under `api.runtime.agent.session`:

    ```typescript
    const storePath = api.runtime.agent.session.resolveStorePath(cfg);
    const store = api.runtime.agent.session.loadSessionStore(storePath);
    await api.runtime.agent.session.updateSessionStore(storePath, (nextStore) => {
      // Patch one entry without replacing the whole file from stale state.
      nextStore[sessionKey] = { ...nextStore[sessionKey], thinkingLevel: "high" };
    });
    const filePath = api.runtime.agent.session.resolveSessionFilePath(cfg, sessionId);
    ```

    Prefer `updateSessionStore(...)` or `updateSessionStoreEntry(...)` for runtime writes. They route through the Gateway-owned session-store writer, preserve concurrent updates, and reuse the hot cache. `saveSessionStore(...)` remains available for compatibility and offline maintenance-style rewrites.

  ### api.runtime.agent.defaults

Default model and provider constants:

    ```typescript
    const model = api.runtime.agent.defaults.model; // e.g. "anthropic/claude-sonnet-4-6"
    const provider = api.runtime.agent.defaults.provider; // e.g. "anthropic"
    ```

  ### api.runtime.llm

Run a host-owned text completion without importing provider internals or
    duplicating OpenClaw model/auth/base URL preparation.

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "Summarize this transcript." }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
    });
    ```

    The helper uses the same simple-completion preparation path as OpenClaw's
    built-in runtime and the host-owned runtime config snapshot. Context engines
    receive a session-bound `llm.complete` capability, so model calls use the
    active session's agent and do not silently fall back to the default agent. The
    result includes provider/model/agent attribution plus normalized token,
    cache, and estimated cost usage when available.

    > **Note:** Model overrides require operator opt-in via `plugins.entries.<id>.llm.allowModelOverride: true` in config. Use `plugins.entries.<id>.llm.allowedModels` to restrict trusted plugins to specific canonical `provider/model` targets. Cross-agent completions require `plugins.entries.<id>.llm.allowAgentIdOverride: true`.
    

  ### api.runtime.subagent

Launch and manage background subagent runs.

    ```typescript
    // Start a subagent run
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "Expand this query into focused follow-up searches.",
      provider: "openai", // optional override
      model: "gpt-4.1-mini", // optional override
      deliver: false,
    });

    // Wait for completion
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // Read session messages
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // Delete a session
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    > **Note:** Model overrides (`provider`/`model`) require operator opt-in via `plugins.entries.<id>.subagent.allowModelOverride: true` in config. Untrusted plugins can still run subagents, but override requests are rejected.
    

    `deleteSession(...)` can delete sessions created by the same plugin through `api.runtime.subagent.run(...)`. Deleting arbitrary user or operator sessions still requires an admin-scoped Gateway request.

  ### api.runtime.nodes

List connected nodes and invoke a node-host command from Gateway-loaded plugin code or from plugin CLI commands. Use this when a plugin owns local work on a paired device, for example a browser or audio bridge on another Mac.

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    Inside the Gateway this runtime is in-process. In plugin CLI commands it calls the configured Gateway over RPC, so commands such as `openclaw googlemeet recover-tab` can inspect paired nodes from the terminal. Node commands still go through normal Gateway node pairing, command allowlists, plugin node-invoke policies, and node-local command handling.

    Plugins that expose dangerous node-host commands should register a node-invoke policy with `api.registerNodeInvokePolicy(...)`. The policy runs in the Gateway after command allowlist checks and before the command is forwarded to the node, so direct `node.invoke` calls and higher-level plugin tools share the same enforcement path.

  ### api.runtime.tasks.managedFlows

Bind a Task Flow runtime to an existing OpenClaw session key or trusted tool context, then create and manage Task Flows without passing an owner on every call.

    Task Flow tracks durable multi-step workflow state. It is not a scheduler:
    use Cron or `api.session.workflow.scheduleSessionTurn(...)` for future
    wakeups, then use `managedFlows` from the scheduled turn when that work
    needs flow state, child tasks, waits, or cancellation.

    ```typescript
    const taskFlow = api.runtime.tasks.managedFlows.fromToolContext(ctx);

    const created = taskFlow.createManaged({
      controllerId: "my-plugin/review-batch",
      goal: "Review new pull requests",
    });

    const child = taskFlow.runTask({
      flowId: created.flowId,
      runtime: "acp",
      childSessionKey: "agent:main:subagent:reviewer",
      task: "Review PR #123",
      status: "running",
      startedAt: Date.now(),
    });

    const waiting = taskFlow.setWaiting({
      flowId: created.flowId,
      expectedRevision: created.revision,
      currentStep: "await-hum