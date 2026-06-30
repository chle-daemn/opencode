> **Confidentiality:** INTERNAL — Daemon Solutions
> **Document Status:** DRAFT - UNREVIEWED

# OpenCode — Container Diagram (C4 Level 2)

OpenCode ships as a Bun + Turborepo monorepo. At runtime the clients (terminal, web, desktop) talk to an HTTP **Server** that hosts the wire **Protocol** over the **Core** runtime. Core owns the session runtime, providers, the SQLite database and the `system-context` algebra.

```mermaid
C4Container
  title Container Diagram - OpenCode

  Person(developer, "Developer", "Writes and edits code")

  System_Ext(llmapi, "LLM Provider APIs", "Anthropic, OpenAI, etc.")
  System_Ext(modelsdev, "models.dev", "Model metadata catalogue")
  System_Ext(mcp, "MCP Servers", "External tools and context")
  System_Ext(workspace, "Workspace & Filesystem", "Local code and git repo")

  Container_Boundary(opencode, "OpenCode") {
    Container(cli, "opencode CLI", "Bun, TypeScript", "Entrypoint: runs commands, launches the server and UIs")
    Container(tui, "Terminal UI", "SolidJS, opentui", "Interactive terminal client")
    Container(webapp, "Web UI", "SolidJS SPA", "Browser client")
    Container(desktop, "Desktop App", "Electron", "Wraps the web UI")
    Container(server, "HTTP Server", "Hono, Effect HttpApi", "Hosts the Protocol over Core; per-request directory instances")
    Container(core, "Core Runtime", "Bun, Effect", "Session runtime, providers, system-context, domain logic")
    Container(llm, "LLM Adapters", "TypeScript", "Provider-neutral protocol adapters")
    ContainerDb(db, "Session Store", "SQLite, Drizzle", "Durable inputs, projected history, events")
  }

  Rel(developer, tui, "Uses")
  Rel(developer, webapp, "Uses")
  Rel(developer, cli, "Runs commands")

  Rel(cli, server, "Starts (serve / web) and spawns")
  Rel(cli, tui, "Launches")
  Rel(tui, server, "Calls", "HTTP + SSE via @opencode-ai/client")
  Rel(webapp, server, "Calls", "HTTP + SSE via @opencode-ai/client")
  Rel(desktop, webapp, "Embeds")

  Rel(server, core, "Invokes", "in-process")
  Rel(core, db, "Reads and writes", "Drizzle / SQLite")
  Rel(core, llm, "Runs one stream per provider turn")
  Rel(llm, llmapi, "Streams completions", "HTTPS / SSE")
  Rel(core, mcp, "Invokes tools, pulls context", "stdio / HTTP")
  Rel(core, modelsdev, "Loads model metadata", "HTTPS")
  Rel(core, workspace, "Reads and writes files, runs commands")

  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## How the clients connect

- The **TUI** spawns a worker that runs the **Server** in-process, then connects to it over HTTP plus a Server-Sent Events stream using the generated `@opencode-ai/client`.
- `opencode serve` runs the **Server** headless (default port 4096), and `opencode web` additionally opens the **Web UI**. The Server loads a per-directory instance from the `x-opencode-directory` request header, so one server can host multiple workspaces.
- The **Desktop App** is an Electron shell around the same Web UI.

## Package dependency direction (the core invariant)

Runtime dependencies point one way; breaking this is the most common way to break the bundle boundaries:

```
schema ─┬─> protocol ─┐
        └─> core ─────┴─> server ──> sdk-next (+ client + core)
client ──> schema, protocol   (NEVER core or server)
```

- **`@opencode-ai/schema` / `@opencode-ai/protocol`** are compile-time shared libraries (Effect `Schema` leaves and the HTTP wire contract), not runtime containers, so they are omitted from the diagram above.
- **`@opencode-ai/client`** is browser-safe and must never transitively load Core, Server, the DB, providers or native modules — it appears here only as the transport label on the UI-to-Server calls.
- **`@opencode-ai/sdk-next`** ("Embedded OpenCode") is an alternative host that runs the Server's router *in memory* with no network I/O, composing Client + Core + Server. It is the embedding path rather than a separately-deployed process.
