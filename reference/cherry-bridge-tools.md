# Cherry Bridge Tools Reference

These 8 tools extend memory-hub with Cherry Studio integration. They manage
Cherry agents, sessions, handoff snapshots, and categorized memory writes.
All tools require memory-hub to be running with Cherry Bridge enabled.

---

## 1. ensure_cherry_agent

Create or reuse a Cherry `claude-code` agent and patch it with memory-hub
handoff configuration.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `agentId` | string | no | Existing agent ID to reuse. Omit to search by name or create new. |
| `name` | string | **yes** | Agent display name. Used to find existing agent if `agentId` is omitted. |
| `description` | string | no | Agent description |
| `model` | string | **yes** | Model in `provider:model_id` format. Default: `"cherryin:anthropic/claude-sonnet-4.6"` |
| `workspacePath` | string | **yes** | Project directory the agent can access |
| `instructions` | string | no | Base instructions (handoff prompt is auto-appended) |
| `mcpServerIds` | string[] | no | Additional MCP server IDs to include |
| `allowedTools` | string[] | no | Additional tools to allow |
| `configuration` | object | no | Agent configuration (see below) |

### Configuration

The `configuration` object controls agent behavior:

| Field | Type | Values | Default |
|-------|------|--------|---------|
| `permission_mode` | string | `"default"`, `"acceptEdits"`, `"bypassPermissions"`, `"plan"` | `"bypassPermissions"` |
| `max_turns` | number | Max conversation turns | (Cherry default) |

Permission modes:
- `default` — Normal mode, asks for confirmation on each tool use
- `acceptEdits` — Auto-approves file edits, asks for other tools
- `bypassPermissions` — **Full auto mode**, no confirmations needed (recommended for agents)
- `plan` — Plan mode, requires plan approval before execution

The bridge defaults to `bypassPermissions` so agents can work autonomously. Override by passing a different `configuration.permission_mode`.

### Default MCP Servers

The bridge automatically includes these built-in MCP servers:
- `@cherry/browser` — Web browsing capability
- `@cherry/fetch` — HTTP fetch capability
- `@cherry/memory` — Cherry's built-in memory

Plus the memory-hub MCP server specified in the bridge configuration. You can add more via `mcpServerIds`.

### What it does (3-step workaround)

1. **Create or find** the agent (by `agentId` or by `name` match)
2. **Patch agent** with memory-hub MCP, allowed tools, merged instructions
3. **Patch or create default session** with matching configuration

### Response

```json
{
  "created": true,
  "agent": { "id": "agent_xxx", "type": "claude-code", ... },
  "session": { "id": "session_xxx", "agent_id": "agent_xxx", ... }
}
```

### Example

```json
{
  "name": "Cherry Studio Dev Agent",
  "model": "cherryin:anthropic/claude-sonnet-4.6",
  "workspacePath": "/path/to/your/project",
  "description": "Development agent for Cherry Studio codebase"
}
```

### Rules

- Always use `model: "cherryin:anthropic/claude-sonnet-4.6"` unless user specifies otherwise.
- The bridge merges your `instructions` with the handoff prompt appendix. Do not include the handoff prompt yourself.
- If the agent already exists, the bridge patches it to ensure MCP and tools are current.
- The bridge auto-includes `@cherry/browser`, `@cherry/fetch`, `@cherry/memory` MCP servers and sets `permission_mode: "bypassPermissions"` by default.
- Store the returned `agent.id` and `session.id` for all subsequent calls.

---

## 2. get_or_create_cherry_session

Reuse or create a Cherry session for an existing agent and align it with
current handoff config.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `agentId` | string | **yes** | The agent's ID |
| `sessionId` | string | no | Specific session to reuse |
| `reuseLatest` | boolean | no | If true and no `sessionId`, reuse the most recent session |

### Response

```json
{
  "created": false,
  "session": { "id": "session_xxx", ... }
}
```

### Example

```json
{
  "agentId": "agent_abc123_example",
  "reuseLatest": true
}
```

### Rules

- Always patches the reused session to ensure it has current MCP, tools, and instructions.
- Use `reuseLatest: true` to continue the most recent session.
- Create a new session (omit both `sessionId` and `reuseLatest`) for a fresh context.

---

## 3. get_session_locator

Return the current Cherry agent/session locator metadata for external coordination.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `agentId` | string | **yes** | The agent's ID |
| `sessionId` | string | no | Specific session (default: latest) |

### Response

```json
{
  "agentId": "agent_xxx",
  "sessionId": "session_xxx",
  "agentUpdatedAt": "2026-03-07T18:19:03.102Z",
  "sessionUpdatedAt": "2026-03-07T18:19:03.164Z"
}
```

### When to use

Use to verify that agent and session exist and get their timestamps.
Useful for coordination between external Claude and Cherry.

---

## 4. load_handoff_context

Load the latest Cherry handoff snapshot plus all categorized memory from memory-hub.
This is the primary **resume** tool.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `agentId` | string | **yes** | The agent's ID |
| `sessionId` | string | no | Specific session (loads session snapshot if provided) |
| `limit` | integer | no | Max entries per category, 1-100 (default: 20) |

### Response

```json
{
  "namespaces": {
    "agent": "cherry/agent/agent_xxx",
    "resource": "cherry/agent/agent_xxx/resource",
    "latest": "cherry/agent/agent_xxx/latest",
    "session": "cherry/agent/agent_xxx/session/session_xxx"
  },
  "latest": {
    "entry": { "id": "...", "layer": "workspace_snapshot", ... },
    "snapshot": {
      "agent_id": "agent_xxx",
      "session_id": "session_xxx",
      "revision": 3,
      "goal": "Add dark mode support",
      "status": "CSS variables defined, toggle pending",
      "decisions": ["Use CSS custom properties"],
      "facts": ["User prefers system theme detection"],
      "resources": ["/src/theme/variables.css"],
      "open_questions": ["Should we support custom themes?"],
      "next_steps": ["Build ThemeToggle component"]
    }
  },
  "session": null,
  "semantic": { "items": [...], "total": 5 },
  "resource": { "items": [...], "total": 3 },
  "episodic": { "items": [...], "total": 1 }
}
```

### Example

```json
{
  "agentId": "agent_abc123_example"
}
```

### Rules

- Call this at the **start of every session** that involves an existing agent.
- Read `latest.snapshot` first for the high-level state.
- Then scan `semantic`, `resource`, and `episodic` for detailed context.
- If `latest` is null, this is a fresh agent with no prior work.

---

## 5. save_handoff_snapshot

Write a normalized Cherry handoff snapshot to both `latest` and `session` namespaces.
This is the primary **handoff** tool.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `agentId` | string | **yes** | The agent's ID |
| `sessionId` | string | **yes** | The session's ID |
| `source` | string | **yes** | Who wrote this: `"claude-external"` or `"cherry"` |
| `title` | string | no | Snapshot title (auto-generated if omitted) |
| `goal` | string | **yes** | Current user goal |
| `status` | string | **yes** | Progress summary |
| `answerSummary` | string | no | Latest useful answer from the session |
| `decisions` | string[] | no | Confirmed decisions |
| `facts` | string[] | no | Durable facts discovered |
| `resources` | string[] | no | Useful paths, commands, endpoints |
| `openQuestions` | string[] | no | Unresolved issues |
| `nextSteps` | string[] | no | Recommended next actions |
| `importance` | number | no | 0.0-1.0 (default: 0.9 for snapshots) |
| `tags` | string[] | no | Additional tags |

### Response

```json
{
  "snapshot": {
    "agent_id": "agent_xxx",
    "session_id": "session_xxx",
    "revision": 4,
    "source": "claude-external",
    "goal": "...",
    "status": "...",
    ...
  },
  "latestEntry": { "id": "...", "namespace": "cherry/agent/agent_xxx/latest", ... },
  "sessionEntry": { "id": "...", "namespace": "cherry/agent/agent_xxx/session/session_xxx", ... }
}
```

### Example

```json
{
  "agentId": "agent_abc123_example",
  "sessionId": "session_def456_example",
  "source": "claude-external",
  "goal": "Implement memory-hub Cherry Bridge",
  "status": "All 14 files implemented, tests passing, pushed to GitHub",
  "answerSummary": "Cherry Bridge enables cross-tool memory sharing between Claude Code and Cherry agents",
  "decisions": [
    "Use last-writer-wins for concurrent snapshots",
    "Advisory revision counter only in phase 1"
  ],
  "facts": [
    "Cherry API does not persist mcps/allowed_tools on POST, requires PATCH workaround"
  ],
  "resources": [
    "src/cherry/CherryBridgeService.ts",
    "http://127.0.0.1:43123/v1/cherry/agents/ensure"
  ],
  "openQuestions": [
    "Should the sidecar auto-detect Cherry's default model?"
  ],
  "nextSteps": [
    "Create skill for LLM workflow orchestration",
    "Test end-to-end handoff between Cherry and external Claude"
  ]
}
```

### Rules

- **Always fill `goal` and `status`** — these are the minimum for a useful snapshot.
- Be concise. Each field should be operational, not narrative.
- Do not store raw chain-of-thought or full transcripts.
- The bridge auto-increments `revision` based on the previous snapshot.

---

## 6. remember_fact

Write durable semantic memory for a Cherry agent. Auto-constructs the namespace
`cherry/agent/<agentId>`.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `agentId` | string | **yes** | The agent's ID |
| `sessionId` | string | no | Session ID (added as tag if provided) |
| `title` | string | **yes** | Short fact title |
| `text` | string | **yes** | Full fact content |
| `tags` | string[] | no | Additional tags (auto-tagged `kind:semantic`) |
| `importance` | number | no | 0.0-1.0 |

### Example

```json
{
  "agentId": "agent_xxx",
  "title": "User prefers conventional commits",
  "text": "All commits must follow Conventional Commit format: feat:, fix:, refactor:, docs:, chore:. Use --signoff flag.",
  "tags": ["preference", "git"],
  "importance": 0.8
}
```

### What to store as facts

- User preferences and workflow habits
- Long-lived technical constraints
- Confirmed architectural decisions
- Project conventions and standards
- Recurring goals or requirements

---

## 7. remember_resource

Write reusable resource memory for a Cherry agent. Auto-constructs the namespace
`cherry/agent/<agentId>/resource`.

### Parameters

Same as `remember_fact`.

### Example

```json
{
  "agentId": "agent_xxx",
  "title": "Memory-hub HTTP API base",
  "text": "memory-hub HTTP API: http://127.0.0.1:43123\nCherry API: http://127.0.0.1:23333\nMCP endpoint: http://127.0.0.1:43123/mcp",
  "tags": ["endpoint", "memory-hub"],
  "importance": 0.7
}
```

### What to store as resources

- File paths (important source files, configs)
- Commands (build, test, deploy, restart)
- Ports and API endpoints
- Repository conventions (branch naming, directory structure)
- Environment variables and their purposes

---

## 8. remember_event

Write episodic Cherry session memory. Auto-constructs the namespace
`cherry/agent/<agentId>` with `kind:episodic` tag.

### Parameters

Same as `remember_fact`.

### Example

```json
{
  "agentId": "agent_xxx",
  "sessionId": "session_xxx",
  "title": "HTTP proxy caused curl failures",
  "text": "curl to localhost was routed through http_proxy=127.0.0.1:7897, causing 502 errors. Fix: use --noproxy '*' or unset http_proxy for local requests.",
  "tags": ["debugging", "proxy"],
  "importance": 0.6
}
```

### What to store as events

- Failures and their root causes
- Workarounds discovered during debugging
- Environment anomalies
- Notable breakthroughs or unexpected behaviors
- Timeline markers for complex multi-session tasks
