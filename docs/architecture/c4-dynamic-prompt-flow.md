> **Confidentiality:** INTERNAL — Daemon Solutions
> **Document Status:** DRAFT - UNREVIEWED

# OpenCode — Prompt to Provider Turn (C4 Dynamic)

This sequence shows what happens between a developer submitting a prompt and the model producing a turn. It makes the admission-then-wake split concrete: the prompt is durably stored (step 3) before execution is advised (step 4), so a wake is always advisory over already-durable state.

```mermaid
C4Dynamic
  title Dynamic Diagram - Prompt to Provider Turn

  Container(ui, "UI", "TUI / Web", "Developer client")
  ContainerDb(db, "Session Store", "SQLite, Drizzle", "Durable inputs and projected history")
  Container_Ext(llmapi, "LLM Provider API", "HTTPS", "Generates completions and tool calls")

  Container_Boundary(opencode, "OpenCode") {
    Component(server, "HTTP Server", "Hono", "Protocol endpoint")
    Component(sessionv2, "SessionV2", "Module", "Prompt admission")
    Component(execution, "SessionExecution", "Process-global", "Wake and drain")
    Component(runner, "SessionRunner", "Location-scoped", "Provider turn")
  }

  Rel(ui, server, "1. Submit prompt", "JSON / HTTPS")
  Rel(server, sessionv2, "2. prompt(...)")
  Rel(sessionv2, db, "3. Admit durable session_input row")
  Rel(sessionv2, execution, "4. wake(sessionID)", "advisory")
  Rel(execution, runner, "5. Start drain at safe boundary")
  Rel(runner, db, "6. Reload projected history, promote inputs")
  Rel(runner, llmapi, "7. stream(request), one per provider turn", "HTTPS / SSE")
  Rel(runner, db, "8. Persist messages and tool results")

  UpdateRelStyle(ui, server, $offsetY="-20")
  UpdateRelStyle(runner, db, $offsetX="-40")
```

## Flow

1. The client submits a prompt to the Server over HTTP.
2. The Server calls `SessionV2.prompt(...)`.
3. `SessionV2` admits a single durable `session_input` row — the prompt now survives a crash.
4. `SessionV2` schedules an advisory `SessionExecution.wake(sessionID)` (skipped when `resume: false`).
5. `SessionExecution` (via the run coordinator) starts a drain at the next safe provider-turn boundary, coalescing any concurrent wakeups for the same session.
6. The `SessionRunner` reloads projected history and promotes admitted inputs into visible user messages.
7. The runner issues exactly one `llm.stream(request)` for the provider turn; completions and tool calls stream back.
8. Resulting messages and tool results are persisted, after which the runner re-evaluates whether continuation is required before the next turn.
