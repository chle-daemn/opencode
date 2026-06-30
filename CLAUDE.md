# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

OpenCode is an open-source AI coding agent. This is a Bun monorepo using Bun workspaces and Turbo, with TypeScript throughout and Effect as the runtime backbone for the core/server. The TUI and web/desktop UIs are written in SolidJS.

## Commands

Requirements: Bun 1.3+ (`packageManager` pins the exact version).

```bash
bun install              # install all workspace deps (postinstall patches node-pty)
bun dev                  # run OpenCode against packages/opencode (default dir)
bun dev <directory>      # run against another directory; `bun dev .` targets this repo
bun dev serve            # headless API server (default port 4096, --port to change)
bun dev web              # server + web UI
bun dev:desktop          # Electron desktop app (wraps packages/app)
bun dev:web              # web UI only (needs a server running)
```

`bun dev` is the local equivalent of the built `opencode` binary and exposes the same CLI (`bun dev --help`).

### Lint, typecheck, test

```bash
bun lint                 # oxlint (config in .oxlintrc.json)
bun typecheck            # turbo typecheck across all packages
```

- **Typecheck uses `tsgo --noEmit`** (the TypeScript native preview compiler), never plain `tsc`. Run it per package from the package directory (e.g. `cd packages/opencode && bun typecheck`) or all at once with `bun turbo typecheck` from root.
- **Tests cannot run from the repo root** — the root `test` script intentionally fails (guard `do-not-run-tests-from-root`). Run tests from a package directory:

  ```bash
  cd packages/opencode && bun test                       # all tests in the package
  cd packages/opencode && bun test path/to/file.test.ts  # a single test file
  cd packages/opencode && bun test -t "name pattern"      # tests matching a name
  ```

  Most test scripts pass `--only-failures`; the opencode package uses a 30s timeout.

### Code generation (do not hand-edit generated output)

- After changing the public Protocol or Server `HttpApi`, run `bun run generate` from `packages/client` (or `./script/generate.ts` from root). Never edit `packages/client/src/generated` or `src/generated-effect` directly.
- To regenerate the legacy JS SDK, run `./packages/sdk/js/script/build.ts`.

### Building a standalone binary

```bash
./packages/opencode/script/build.ts --single
# then: ./packages/opencode/dist/opencode-<platform>/bin/opencode
```

## Architecture

### Package layering and dependency direction

The monorepo is layered, and runtime dependencies flow in one direction. **Respect this when adding imports:**

- `packages/schema` (`@opencode-ai/schema`) — lightweight Zod/Effect schema leaf for values that mean the same thing internally and on the wire. Depended on by everything; depends on nothing heavy.
- `packages/protocol` (`@opencode-ai/protocol`) — composes Schema values into the public HTTP API: paths, payloads, envelopes, errors, cursors, streams. Owns Session endpoint construction and middleware placement.
- `packages/core` (`@opencode-ai/core`) — business logic: sessions, providers, tools, permissions, filesystem, MCP, LSP, storage (SQLite via Drizzle). Consumes Schema.
- `packages/llm` (`@opencode-ai/llm`) — provider protocol adapters and model routing; owns provider wire encoding.
- `packages/server` (`@opencode-ai/server`) — hosts Protocol's groups over Hono, owns protocol/domain adaptation. Imports Schema, Protocol, and Core.
- `packages/client` (`@opencode-ai/client`) — generated Promise + Effect network clients. The root export is **zero-Effect**; the `/effect` export depends only on Effect, Schema, and Protocol. Never imports Core or Server.
- `packages/sdk-next` (`@opencode-ai/sdk`, in transition) — composes Client + Core + Server into an embedded in-process host that runs Server's router in memory with no network I/O.

The rule of thumb (from `AGENTS.md`): dependencies go **Schema → Core/Protocol → Server**. Client code may depend on Schema and Protocol but never Core or Server. Schema and Protocol must never transitively load databases, Drizzle, session execution, providers, watchers, native modules, or WASM.

### Apps and surfaces

- `packages/opencode` — the CLI entrypoint (`src/index.ts`) and command implementations (`src/cli/cmd/`). Wires the core, server, and TUI together. Versioned and published as `opencode`.
- `packages/tui` (`@opencode-ai/tui`) — the terminal UI, SolidJS on top of [opentui](https://github.com/sst/opentui). (Note: `CONTRIBUTING.md` still references the old `packages/opencode/src/cli/cmd/tui/` location.)
- `packages/app` — shared SolidJS web UI components; `packages/desktop` — Electron app wrapping it; `packages/web` — marketing/docs site.
- `packages/plugin` (`@opencode-ai/plugin`) — the plugin SDK surface.
- `packages/console`, `packages/stats` — separate sub-apps (each is its own workspace group with `app`/`core`/`function`).
- `infra/`, `sst.config.ts` — SST infrastructure (AWS). `script/` — repo-wide dev/build scripts.

### V2 Session runtime (read before touching session/context code)

The session execution model is documented in depth in `CONTEXT.md` (terminology) and the `V2 Session Core` section of `AGENTS.md`. Core invariants:

- **Durable prompt admission is separate from model execution.** `SessionV2.prompt(...)` admits one durable `session_input` row, then schedules an advisory `SessionExecution.wake(sessionID)` (unless `resume: false` for admit-only). The serialized runner promotes admitted inputs into visible user messages at safe provider-turn boundaries.
- **System Context** (the structured facts shown to the model) is assembled from independently-observed **Context Sources** via a Location-scoped registry. Changes are sampled lazily at a Safe Provider-Turn Boundary and emitted as durable **Mid-Conversation System Messages** — never pushed asynchronously, never used to wake idle sessions. Compaction starts a new **Context Epoch** with a fresh baseline.
- `SessionExecution` is process-global and Session-ID based; `SessionRunner`, model resolution, tool registry, permissions, and filesystem are Location-scoped. Session drains are process-local coordination, not durable entities.
- Keep one explicit `llm.stream(request)` per provider turn; do not route through the legacy `SessionPrompt.loop(...)`.
- The System Context algebra, registry, and built-ins live in `packages/core/src/system-context`.

## Conventions

The authoritative style guide is **`AGENTS.md`** — read it before writing code. Highlights that bite if missed:

- **No aliased imports, no star imports.** Use a module's own exported namespace by name (e.g. `import { Project } from "@opencode-ai/core/project"`, then `Project.ID`).
- Prefer `const`, early returns over `else`, ternaries over reassignment. Avoid `try`/`catch` (prefer `.catch(...)`) and avoid `any`.
- Use Bun APIs (`Bun.file()` etc.) where they fit. Rely on type inference; avoid needless explicit annotations and destructuring. Inline single-use values and helpers.
- Drizzle schema fields use `snake_case` so column names need not be redefined as strings.
- In `src/config`, follow the self-export pattern (`export * as ConfigAgent from "./agent"`).
- In Effect generators, bind services to named variables first; no nested service yields like `yield* (yield* Foo.Service).bar()`.

### Git and PRs

- Default branch is `dev`. There may be no local `main`; diff against `dev` or `origin/dev`.
- Branch names: at most three hyphen-separated words, no slashes or type prefixes (e.g. `session-recovery`, `fix-scroll-state`).
- Commits and PR titles use conventional style `type(scope): summary`. Valid types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`. Scopes: `core`, `opencode`, `tui`, `app`, `desktop`, `sdk`, `plugin`, etc.
- Keep PRs small and focused; all PRs must reference an issue (`Fixes #123`).
