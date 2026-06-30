# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

OpenCode is an AI coding agent. This is a Bun + Turborepo monorepo. Read `AGENTS.md` for the authoritative style guide and `CONTEXT.md` for the V2 session-runtime domain language — both are required reading before non-trivial work.

## Commands

Run from the repo root unless noted:

```bash
bun install            # install workspace deps (postinstall patches node-pty)
bun dev                # run OpenCode TUI against packages/opencode
bun dev <directory>    # run against another directory; `bun dev .` targets this repo
bun dev serve          # headless API server (port 4096; --port to change)
bun dev web            # server + web UI
bun lint               # oxlint across the repo
bun typecheck          # turbo-orchestrated typecheck of all packages
```

- **Requirements:** Bun. The root `packageManager` field pins the exact version (`bun@1.3.14`); the `.husky/pre-push` hook derives a caret range from it (`^1.3.14`), enforces that Bun satisfies the range, and runs `bun typecheck` on every push — so a failing typecheck blocks pushing.
- **Branch:** the default and main branch is `dev`; local `main` may not exist, so diff against `dev`/`origin/dev`.
- **Typecheck:** always run `bun typecheck` (or `bun typecheck` from a package dir, which runs a `tsgo` typecheck — `tsgo --noEmit` in most packages, `tsgo -b` in a few such as `app`). Never invoke `tsc` directly.
- **Build a standalone binary:** `./packages/opencode/script/build.ts --single`.

### Tests

- **Tests cannot run from the repo root** — a guard (`do-not-run-tests-from-root`) makes `bun test` fail there. `cd` into a package (e.g. `packages/opencode`, `packages/core`, `packages/llm`) first.
- Run a package's suite with `bun test` from that package dir; run a single file with `bun test test/foo.test.ts`, and a single case with `bun test -t "name"`.
- Most suites use `bun test --only-failures`; `packages/llm` uses recorded HTTP fixtures (`bun run setup:recording-env`).
- HTTP API coverage: `bun run test:httpapi` from `packages/opencode`.
- Testing rules (from `AGENTS.md`): avoid mocks and `globalThis.*`; test the real implementation rather than duplicating its logic into the test.

### Code generation (do not hand-edit generated output)

- After changing the public Protocol or Server `HttpApi`, run `bun run generate` from `packages/client`. Never edit `src/generated` or `src/generated-effect`.
- `./script/generate.ts` (repo root) regenerates the SDK after API/SDK changes.
- Regenerate the legacy JS SDK with `./packages/sdk/js/script/build.ts`.

## Architecture

### Package dependency direction (the core invariant)

Runtime dependencies must point in one direction. Violating this is the most common way to break the build/bundle boundaries. Each arrow points from a package to what it depends on (`A ──> B` means A imports B):

```
protocol ──> schema
core     ──> schema
server   ──> core, protocol
client   ──> schema, protocol          (NEVER core or server)
sdk-next ──> client, core, server
```

- `@opencode-ai/schema` — lightweight Effect `Schema` leaf for values that mean the same thing internally and on the wire. Depends on nothing but `effect`.
- `@opencode-ai/protocol` — composes Schema into HTTP paths, payloads, envelopes, errors, cursors, streams. Depends only on schema.
- `@opencode-ai/core` — domain/business logic, providers, DB (Drizzle), session execution, native modules.
- `@opencode-ai/server` — hosts Protocol's groups over Core; owns protocol/domain adaptation.
- `@opencode-ai/client` — generated Promise + Effect HTTP clients. Browser-safe; must never transitively load Core, Server, DB/Drizzle, providers, watchers, native modules, or WASM. Root export is zero-Effect; `/effect` adds Effect + Schema + Protocol only.
- `@opencode-ai/sdk-next` — Effect-native in-process ("Embedded OpenCode") host composed over Client + Core + Server, running Server's router in memory with no network I/O.
- `opencode` (`packages/opencode`) — the CLI/server entrypoint and most product logic; depends on core, server, tui, sdk, llm, plugin.
- `@opencode-ai/tui` — SolidJS terminal UI built on `@opentui/*`.
- `@opencode-ai/app` (SolidJS web UI), `@opencode-ai/desktop` (Electron wrapping `app`), `@opencode-ai/plugin` (public plugin API), `@opencode-ai/llm` (provider-neutral LLM protocol adapters using recorded-HTTP tests).

### V2 session runtime

The session model is the deepest part of the system; `CONTEXT.md` defines the precise vocabulary (System Context, Context Source, Context Epoch, Session Drain, Provider Turn, Admitted/Promoted Prompt, etc.). Key constraints when touching session code (see the "V2 Session Core" section of `AGENTS.md`):

- Keep durable prompt admission separate from model execution: `SessionV2.prompt(...)` admits a durable `session_input` row, then schedules an advisory `SessionExecution.wake(sessionID)` unless `resume: false`.
- `SessionExecution` is process-global and Session-ID based; `SessionRunner`, model resolution, tool registry, permissions, and filesystem are Location-scoped.
- One explicit `llm.stream(request)` per provider turn; reload projected history before durable continuation. Do not route through legacy `SessionPrompt.loop(...)`.
- System Context algebra, registry, and built-ins live in `src/system-context`; context changes are admitted lazily at a Safe Provider-Turn Boundary, never pushed when a source changes.

### Provider model

New LLM providers should require little or no code change here — model metadata comes from `models.dev`. To add a provider, PR `https://github.com/anomalyco/models.dev` first.

## Conventions

`AGENTS.md` is authoritative; highlights that bite most often:

- **Imports:** never alias (`import { x as y }`) and never star-import (`import * as Foo`). For namespace style, import a module's own exported namespace by name (e.g. `import { Project } from "@opencode-ai/core/project"`). Prefer dynamic imports for heavy modules in startup-sensitive paths.
- **Control flow:** avoid `else` (use early returns); prefer `const` over `let`; prefer `.catch(...)` over `try`/`catch`; avoid `any`; rely on type inference.
- **Effect generators:** bind services to named variables before calling methods — no `yield* (yield* Foo.Service).bar()`.
- **Drizzle schemas:** use `snake_case` field names so column names need not be redeclared as strings.
- **Inline single-use values/helpers** rather than extracting preemptively; keep supporting helpers below the happy-path function.

### Branches, commits, PRs

- Branch names: at most three hyphen-separated words, no slashes or `feat/` prefixes (e.g. `fix-scroll-state`).
- Conventional commits/PR titles: `type(scope): summary` where type ∈ `feat|fix|docs|chore|refactor|test`; scope is a package/area (`core`, `tui`, `app`, `desktop`, `sdk`, `plugin`, …).
- PRs must reference an existing issue (`Fixes #123`); keep them small and explain how the change was verified. Avoid AI-generated walls of text in descriptions.
