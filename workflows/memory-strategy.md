# Memory Classification Strategy

This document defines what to store, where to store it, and when to store it.
The goal is to build a useful, compact memory base — not an exhaustive log.

---

## Three Memory Layers

| Layer | Tool | Namespace | Purpose | Lifespan |
|-------|------|-----------|---------|----------|
| **Semantic** | `remember_fact` | `cherry/agent/<id>` | Stable knowledge | Permanent |
| **Resource** | `remember_resource` | `cherry/agent/<id>/resource` | Operational references | Permanent |
| **Episodic** | `remember_event` | `cherry/agent/<id>` | Notable events | Session-linked |

Plus **Workspace Snapshot** (`save_handoff_snapshot`) which is a structured summary
of current state, not a memory category.

---

## Semantic Layer: Facts

### What belongs here

Stable information that is true across sessions and tasks:

- **User preferences**
  - "User prefers Biome over Prettier for formatting"
  - "User wants all commits signed with --signoff"
  - "User communicates in Chinese, code comments in English"

- **Technical decisions**
  - "Chose last-writer-wins for concurrent snapshot writes"
  - "Using CSS custom properties for theming, not CSS-in-JS"
  - "memory-hub stores data as JSON-per-namespace flat files"

- **Constraints**
  - "Redux data model changes blocked until v2.0.0"
  - "Never use console.log — always use loggerService"
  - "Cherry standalone must not be modified"

- **Recurring goals**
  - "Primary project: Cherry Studio desktop AI assistant"
  - "Building memory-hub for cross-tool handoff"

### What does NOT belong here

- Temporary state ("currently working on X")
- Session-specific context (use snapshot instead)
- File contents (use resource to reference paths)
- Debugging steps (use episodic for notable ones)

### Importance guidance

| Importance | When to use |
|------------|-------------|
| 0.9 - 1.0 | Core constraint that must never be forgotten |
| 0.7 - 0.8 | Strong preference or confirmed decision |
| 0.5 - 0.6 | Useful convention or observation |
| 0.3 - 0.4 | Minor preference, might change |

### Example

```json
{
  "agentId": "agent_xxx",
  "title": "Cherry API mcps persistence gap",
  "text": "POST /v1/agents does not persist mcps and allowed_tools fields in the database insert. The sidecar must use a create-then-patch workaround: POST to create, then PATCH to add mcps/allowed_tools/instructions.",
  "tags": ["cherry-api", "workaround", "architecture"],
  "importance": 0.9
}
```

---

## Resource Layer: Operational References

### What belongs here

Reusable references that help navigate and operate:

- **File paths**
  - "Main bridge service: /path/to/memory-hub/src/cherry/CherryBridgeService.ts"
  - "Launchd plist: ~/Library/LaunchAgents/com.cherrystudio.memory-hub.plist"

- **Commands**
  - "Start memory-hub: cd /path/to/memory-hub && pnpm dev"
  - "Run tests: pnpm test"
  - "Restart launchd: launchctl stop/start com.cherrystudio.memory-hub"

- **Endpoints**
  - "memory-hub HTTP: http://127.0.0.1:43123"
  - "memory-hub MCP: http://127.0.0.1:43123/mcp"
  - "Cherry API: http://127.0.0.1:23333"

- **Conventions**
  - "Branch naming: feature/<task-name>"
  - "Commit format: <type>: <description> (feat/fix/docs/refactor/chore)"

- **IDs and configuration**
  - "memory-hub MCP server ID in Cherry: <your-mcp-server-id>"
  - "Default model: anthropic/claude-sonnet-4.6"

### What does NOT belong here

- Explanations of why (use semantic facts for decisions)
- Narrative descriptions (keep resources terse and scannable)
- Temporary paths (only store paths that will persist)

### Importance guidance

| Importance | When to use |
|------------|-------------|
| 0.8 - 1.0 | Critical operational reference (API keys, endpoints) |
| 0.6 - 0.7 | Frequently used path or command |
| 0.4 - 0.5 | Occasionally useful reference |

### Example

```json
{
  "agentId": "agent_xxx",
  "title": "Memory-hub project paths and commands",
  "text": "Source: /path/to/memory-hub/\nGitHub: https://github.com/<owner>/memory-hub\nDev: pnpm dev\nTest: pnpm test\nTypecheck: pnpm typecheck\nHTTP API: http://127.0.0.1:43123\nMCP endpoint: http://127.0.0.1:43123/mcp",
  "tags": ["paths", "commands", "memory-hub"],
  "importance": 0.7
}
```

---

## Episodic Layer: Notable Events

### What belongs here

Events worth remembering for future reference:

- **Failures and root causes**
  - "curl to localhost returned 502 because http_proxy was set to 127.0.0.1:7897"
  - "pnpm install failed due to Node version mismatch (needed >=24, had 20)"

- **Workarounds**
  - "Use --noproxy '*' with curl for all localhost requests on this machine"
  - "Must use nanoid server ID, not display name, for CHERRY_MCP_SERVER_ID"

- **Environment anomalies**
  - "System has HTTP proxy configured globally via http_proxy env var"
  - "Node version is 20.12.2 but package.json requires >=24"

- **Debugging insights**
  - "Cherry Bridge MCP tools only register when cherryBridge option is provided"
  - "The default session created by POST /v1/agents inherits stale config"

- **Timeline markers**
  - "2026-03-07: Initial memory-hub release with 8 base tools"
  - "2026-03-07: Cherry Bridge added with 8 additional tools"

### What does NOT belong here

- Every error message encountered (only store notable ones)
- Step-by-step debugging logs (summarize the insight)
- Routine operations that succeeded normally

### Importance guidance

| Importance | When to use |
|------------|-------------|
| 0.7 - 0.9 | Critical workaround that prevents recurring failures |
| 0.5 - 0.6 | Useful debugging insight |
| 0.3 - 0.4 | Minor observation, context-specific |

### Example

```json
{
  "agentId": "agent_xxx",
  "sessionId": "session_xxx",
  "title": "MCP server ID must be nanoid, not name",
  "text": "Cherry getMcpServerInfo API at /v1/mcps/{id} requires the nanoid ID (e.g., <your-mcp-server-id>), not the display name. Although the internal find function supports name matching, the HTTP route does not. Always use the actual ID for MEMORY_HUB_CHERRY_MCP_SERVER_ID.",
  "tags": ["debugging", "cherry-api", "mcp"],
  "importance": 0.8
}
```

---

## Before Writing: The Deduplication Check

Before creating any new memory entry:

1. **Search first**: `memory_search({ query: "<topic>", limit: 5 })`
2. **If a similar entry exists**:
   - Same topic, needs update → `memory_update` the existing entry
   - Same topic, still correct → Skip, do not create duplicate
   - Related but different → Create new, but reference the related entry
3. **If no match**: Create new entry

---

## Decision Flowchart

```
Something worth remembering?
  |
  YES
  |
  Is it stable across sessions? ──YES──> remember_fact
  |                                        tags: kind:semantic
  NO
  |
  Is it a reusable reference? ──YES──> remember_resource
  |                                     tags: kind:resource
  NO
  |
  Is it a notable event? ──YES──> remember_event
  |                                tags: kind:episodic
  NO
  |
  Is it a full state summary? ──YES──> save_handoff_snapshot
  |
  NO
  |
  Don't store it.
```

---

## Anti-Patterns

| Anti-pattern | Why it's bad | Do instead |
|-------------|-------------|------------|
| Storing raw transcripts | Bloats memory, hard to search | Extract key decisions and facts |
| Storing chain-of-thought | Ephemeral, not useful later | Store the conclusion only |
| Storing every file read | Too noisy, already in filesystem | Store only notable paths |
| Duplicate entries | Degrades search quality | Search before writing |
| Vague descriptions | Not actionable | Be specific and operational |
| Storing temporary state as fact | Becomes stale | Use snapshot for current state |
