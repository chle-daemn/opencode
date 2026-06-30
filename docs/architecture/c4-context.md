> **Confidentiality:** INTERNAL — Daemon Solutions
> **Document Status:** DRAFT - UNREVIEWED

# OpenCode — System Context (C4 Level 1)

OpenCode is an AI coding agent. A developer issues prompts; OpenCode drives a large language model to read, edit and run code in a local workspace, calling external tools and providers as needed.

```mermaid
C4Context
  title System Context - OpenCode AI Coding Agent

  Person(developer, "Developer", "Writes and edits code with help from an AI agent")

  System(opencode, "OpenCode", "AI coding agent: drives an LLM to read, edit and run code in a local workspace")

  System_Ext(llmapi, "LLM Provider APIs", "Anthropic, OpenAI, etc. Generate completions and tool calls")
  System_Ext(modelsdev, "models.dev", "Public catalogue of model metadata and capabilities")
  System_Ext(mcp, "MCP Servers", "External tools and context via Model Context Protocol")
  System_Ext(workspace, "Workspace & Filesystem", "The user's local code, files and git repository")
  System_Ext(github, "GitHub", "Hosting for pull requests and issues")

  Rel(developer, opencode, "Issues prompts, reviews edits", "TUI / web / CLI")
  Rel(opencode, llmapi, "Streams prompts and receives tool calls", "HTTPS / SSE")
  Rel(opencode, modelsdev, "Fetches model metadata", "HTTPS")
  Rel(opencode, mcp, "Invokes tools, pulls context", "stdio / HTTP")
  Rel(opencode, workspace, "Reads and writes files, runs commands")
  Rel(opencode, github, "Opens PRs, reads issues", "HTTPS")

  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Notes

- **Provider-neutral by design.** Model metadata is sourced from `models.dev`, so adding a provider usually needs no code change here — the catalogue is updated upstream.
- **Local-first.** The primary deployment is a developer's own machine: the workspace, filesystem and git repository are local, and an LLM provider is reached over the network.
- **MCP** lets OpenCode consume external tools and context from separately-run servers over stdio or HTTP.
