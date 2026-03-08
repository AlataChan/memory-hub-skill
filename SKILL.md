---
name: cherry-memory-bridge
description: |
  Orchestrate Cherry Studio agents and shared memory through memory-hub.
  Use when creating/managing Cherry agents, saving/loading handoff state,
  or writing durable memories (facts, resources, events) for cross-tool collaboration.
  Covers the full lifecycle: bootstrap, work, handoff, resume.
---

# Cherry Memory Bridge Skill

This skill defines how Claude (external) orchestrates Cherry Studio agents and shares
durable state through memory-hub. It is the **Skill Layer** described in the
Cherry Claude Memory Handoff Design.

## Quick Reference

| Category | Tools |
|----------|-------|
| Memory Hub (base) | `memory_write`, `memory_search`, `memory_update`, `memory_delete`, `memory_pin`, `workspace_snapshot_save`, `workspace_snapshot_get`, `memory_status` |
| Cherry Bridge | `ensure_cherry_agent`, `get_or_create_cherry_session`, `get_session_locator`, `load_handoff_context`, `save_handoff_snapshot`, `remember_fact`, `remember_resource`, `remember_event` |
| Agent Built-in | `Bash`, `Edit`, `Write`, `Read`, `Glob`, `Grep`, `MultiEdit`, `NotebookEdit`, `NotebookRead`, `Task`, `TodoWrite`, `WebFetch`, `WebSearch` |

Detailed parameter reference for each tool is in `reference/`.

---

## Configuration Constants

```yaml
model: cherryin:anthropic/claude-sonnet-4.6
agent_type: claude-code
permission_mode: bypassPermissions
default_mcp_servers:
  - "@cherry/browser"
  - "@cherry/fetch"
  - "@cherry/memory"
memory_hub_base: http://127.0.0.1:43123
cherry_api_base: http://127.0.0.1:23333
memory_mcp_server_id: <your-mcp-server-id>
```

When calling `ensure_cherry_agent`, always use `model: "cherryin:anthropic/claude-sonnet-4.6"`
unless the user explicitly requests a different model.

The bridge automatically applies these defaults:
- **Permission mode**: `bypassPermissions` (全自动模式) — agents work autonomously without confirmations
- **MCP servers**: `@cherry/browser`, `@cherry/fetch`, `@cherry/memory` plus the memory-hub MCP server
- These can be overridden via `configuration` and `mcpServerIds` parameters

---

## Lifecycle Overview

```
  [1. Bootstrap]  -->  [2. Resume]  -->  [3. Work]  -->  [4. Handoff]
       |                   |                |                |
  ensure_agent        load_handoff     remember_*      save_snapshot
  get_session         context          memory_write
```

### Phase 1: Bootstrap (first time only)

Trigger: user asks to create a Cherry agent, or no agent exists yet.

1. Call `ensure_cherry_agent` with:
   - `name`: descriptive agent name (e.g., "Cherry Studio Dev Agent")
   - `model`: `"cherryin:anthropic/claude-sonnet-4.6"`
   - `workspacePath`: the project directory the agent will work in
   - `description`: optional, what this agent does
2. The bridge automatically:
   - Creates the agent (type: claude-code) with `bypassPermissions` mode
   - Patches in memory-hub MCP server plus `@cherry/browser`, `@cherry/fetch`, `@cherry/memory`
   - Injects handoff prompt appendix into instructions
   - Creates a default session with matching config
3. Store the returned `agent.id` and `session.id` for subsequent calls.

See: `workflows/agent-bootstrap.md`

### Phase 2: Resume (start of each session)

Trigger: beginning of any conversation that involves an existing Cherry agent.

1. Call `load_handoff_context` with `agentId` (and optionally `sessionId`).
2. Read the returned:
   - `latest` — most recent handoff snapshot (goal, status, decisions, etc.)
   - `semantic` — durable facts and preferences
   - `resource` — file paths, commands, endpoints
   - `episodic` — notable events and workarounds
3. Use this context to resume work without asking the user to repeat themselves.

See: `workflows/handoff-lifecycle.md`

### Phase 3: Work (during execution)

Trigger: any meaningful work product, discovery, or decision during the session.

Write memories as they happen — do not batch them:

| What happened | Tool to call | Layer |
|---------------|-------------|-------|
| Confirmed a technical decision | `remember_fact` | semantic |
| Discovered a user preference | `remember_fact` | semantic |
| Found an important file path | `remember_resource` | resource |
| Noted a useful command | `remember_resource` | resource |
| Encountered a failure/workaround | `remember_event` | episodic |
| Environment anomaly | `remember_event` | episodic |

See: `workflows/memory-strategy.md`

### Phase 4: Handoff (end of session or task boundary)

Trigger: any of these conditions:
- Task is complete or a meaningful milestone is reached
- About to switch to a different task
- User explicitly requests handoff
- Session is ending
- Before context compaction (if applicable)

1. Call `save_handoff_snapshot` with:
   - `agentId`, `sessionId`, `source`: `"claude-external"` (or `"cherry"` if from Cherry agent)
   - `goal`: current user goal
   - `status`: progress summary
   - `answerSummary`: latest useful answer
   - `decisions`: list of confirmed decisions
   - `facts`: durable facts
   - `resources`: useful file paths, commands, endpoints
   - `openQuestions`: unresolved issues
   - `nextSteps`: recommended next actions
2. The bridge writes to both `cherry/agent/<id>/latest` and `cherry/agent/<id>/session/<sid>`.

See: `workflows/handoff-lifecycle.md`

---

## Decision Trees

### "Should I create an agent or reuse one?"

```
Has the user provided an agent_id?
  YES --> Call ensure_cherry_agent with agentId (reuse)
  NO  --> Does an agent with the requested name exist?
          YES --> Bridge auto-reuses it
          NO  --> Bridge auto-creates it
```

`ensure_cherry_agent` handles both cases — just always call it.

### "What layer should this memory go to?"

```
Is this stable across sessions? (preferences, decisions, constraints)
  YES --> remember_fact (semantic)

Is this a reusable reference? (file path, command, port, endpoint, convention)
  YES --> remember_resource (resource)

Is this a notable event? (failure, workaround, anomaly, debugging insight)
  YES --> remember_event (episodic)

Is this a full session state summary?
  YES --> save_handoff_snapshot (workspace_snapshot)
```

### "Should I save a snapshot now?"

```
Am I done with a task or subtask?           --> YES, save
Am I about to switch context?               --> YES, save
Has the user asked to hand off?             --> YES, save
Has 15+ minutes passed since last snapshot? --> YES, save
Am I just mid-thought with no conclusion?   --> NO, wait
```

---

## What NOT to Store

- Raw chain-of-thought
- Full conversation transcripts
- Speculative or unverified conclusions
- Temporary debugging state (unless it's a notable workaround)
- Duplicate information already in an existing entry

---

## Namespace Protocol

All namespaces follow this pattern:

| Namespace | Purpose |
|-----------|---------|
| `cherry/agent/<agent_id>` | Long-lived semantic memory |
| `cherry/agent/<agent_id>/resource` | Reusable operational context |
| `cherry/agent/<agent_id>/session/<session_id>` | Per-session rolling snapshot |
| `cherry/agent/<agent_id>/latest` | Most recent handoff snapshot (fast resume) |

The bridge constructs these automatically — you do not need to build them manually.

---

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| `MCP server not found` | memory-hub not registered in Cherry UI | Ask user to check Cherry Settings > MCP Servers |
| `Validation failed` | Missing required field (usually `model`) | Ensure `model` uses `provider:model_id` format |
| `cherry_bridge_disabled` | memory-hub started without `CHERRY_*` env vars | Restart memory-hub with Cherry Bridge env vars |
| `Connection refused` on :43123 | memory-hub not running | `launchctl start com.cherrystudio.memory-hub` |
| `Connection refused` on :23333 | Cherry API server not enabled | Enable in Cherry Settings > API Server |

---

## File Index

```
mcp_skill/
  SKILL.md                              <-- You are here
  reference/
    memory-hub-tools.md                 <-- 8 base MCP tools (params, examples)
    cherry-bridge-tools.md              <-- 8 Cherry Bridge tools (params, examples)
    builtin-tools.md                    <-- 13 built-in agent tools
  workflows/
    agent-bootstrap.md                  <-- Step-by-step agent creation
    handoff-lifecycle.md                <-- Load/save handoff workflow
    memory-strategy.md                  <-- Memory classification rules
```
