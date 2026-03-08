# memory-hub-skill

LLM skill definitions for [memory-hub](https://github.com/AlataChan/memory-hub) — a local MCP service that provides shared durable memory for cross-tool handoff between Claude Code, Cherry Studio agents, and other AI coding tools.

## What is this?

This repository contains structured instructions (a "skill") that teach LLM agents how to use memory-hub's 16 MCP tools effectively. It covers:

- **Agent lifecycle**: Bootstrap, resume, work, handoff
- **Memory classification**: When to use semantic, resource, or episodic memory
- **Cherry Bridge integration**: Managing Cherry Studio agents and sessions
- **Handoff protocol**: Saving and loading structured state across tools

## Structure

```
SKILL.md                           # Main skill document (start here)
reference/
  memory-hub-tools.md              # 8 base MCP tools (params, examples)
  cherry-bridge-tools.md           # 8 Cherry Bridge tools (params, examples)
  builtin-tools.md                 # 13 built-in agent tools
workflows/
  agent-bootstrap.md               # Step-by-step agent creation
  handoff-lifecycle.md             # Load/save handoff workflow
  memory-strategy.md               # Memory classification rules
```

## Prerequisites

- [memory-hub](https://github.com/AlataChan/memory-hub) running locally (default: `http://127.0.0.1:43123`)
- [Cherry Studio](https://github.com/CherryHQ/cherry-studio) with API server enabled (for Cherry Bridge features)

## Usage

Point your LLM agent's skill/prompt configuration at `SKILL.md`. The skill document references the other files as needed.

For Claude Code, you can include this as a skill in your project configuration.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
