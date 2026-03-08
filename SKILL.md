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

| Category | Interface | Description |
|----------|-----------|-------------|
| Memory Hub REST API | `curl http://127.0.0.1:43123/v1/...` | Primary interface — reliable, works everywhere |
| Memory Hub MCP | `memory_write`, `memory_search`, etc. | Alternative — may not be available in all environments |
| Cherry Bridge REST | `curl http://127.0.0.1:43123/v1/cherry/...` | Agent management and categorized memory writes |

> **Transport Note**: Cherry Studio's MCP proxy has a known session caching bug that
> prevents external MCP servers from working reliably. Always prefer REST API calls
> via `Bash` + `curl`. See `reference/rest-api.md` for complete curl templates.

---

## Configuration Constants

```yaml
agent_id: agent_1772907543062_a6jc4lkaj
session_id: session_1772907543080_kglcx20le
model: cherryin:anthropic/claude-sonnet-4.6
memory_hub_base: http://127.0.0.1:43123
cherry_api_base: http://127.0.0.1:23333
cherry_api_key: cs-sk-f995f423-d32a-455e-b49f-66f288f60b12
memory_mcp_server_id: qhbCTgu1o7zIudmReObsF
```

---

## Lifecycle Overview

```
  [1. Bootstrap]  -->  [2. Resume]  -->  [3. Work]  -->  [4. Handoff]
       |                   |                |                |
  ensure_agent        load_context     remember_*      save_snapshot
  (REST API)          (REST API)       (REST API)      (REST API)
```

### Phase 1: Bootstrap (first time only)

Trigger: user asks to create a Cherry agent, or no agent exists yet.

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/cherry/agents/ensure \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Cherry Studio Dev Agent",
    "model": "cherryin:anthropic/claude-sonnet-4.6",
    "workspacePath": "/Users/apple/cherry-studio"
  }'
```

Store the returned `agent.id` and `session.id` for subsequent calls.

See: `workflows/agent-bootstrap.md`

### Phase 2: Resume (start of each session)

Trigger: beginning of any conversation that involves an existing Cherry agent.

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/cherry/handoff/context \
  -H "Content-Type: application/json" \
  -d '{"agentId": "agent_1772907543062_a6jc4lkaj"}'
```

Process the response:
1. `latest.snapshot` — high-level state (goal, status, decisions)
2. `semantic.items` — durable facts and preferences
3. `resource.items` — file paths, commands, endpoints
4. `episodic.items` — notable events and workarounds

See: `workflows/handoff-lifecycle.md`

### Phase 3: Work (during execution)

Write memories as they happen — do not batch them:

| What happened | REST endpoint | Layer |
|---------------|--------------|-------|
| Confirmed a technical decision | `POST /v1/cherry/memories/fact` | semantic |
| Discovered a user preference | `POST /v1/cherry/memories/fact` | semantic |
| Found an important file path | `POST /v1/cherry/memories/resource` | resource |
| Noted a useful command | `POST /v1/cherry/memories/resource` | resource |
| Encountered a failure/workaround | `POST /v1/cherry/memories/event` | episodic |
| Environment anomaly | `POST /v1/cherry/memories/event` | episodic |

See: `workflows/memory-strategy.md`

### Phase 4: Handoff (end of session or task boundary)

Trigger: task complete, context switch, user requests, or session ending.

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/cherry/handoff/snapshot \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "agent_1772907543062_a6jc4lkaj",
    "sessionId": "session_1772907543080_kglcx20le",
    "source": "claude-external",
    "goal": "<current goal>",
    "status": "<progress summary>",
    "decisions": ["<confirmed decisions>"],
    "nextSteps": ["<what should happen next>"]
  }'
```

See: `workflows/handoff-lifecycle.md`

---

## Decision Trees

### "What layer should this memory go to?"

```
Is this stable across sessions? (preferences, decisions, constraints)
  YES --> POST /v1/cherry/memories/fact (semantic)

Is this a reusable reference? (file path, command, port, endpoint)
  YES --> POST /v1/cherry/memories/resource (resource)

Is this a notable event? (failure, workaround, anomaly)
  YES --> POST /v1/cherry/memories/event (episodic)

Is this a full session state summary?
  YES --> POST /v1/cherry/handoff/snapshot (workspace_snapshot)
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

| Namespace | Purpose |
|-----------|---------|
| `cherry/agent/<agent_id>` | Long-lived semantic memory |
| `cherry/agent/<agent_id>/resource` | Reusable operational context |
| `cherry/agent/<agent_id>/session/<session_id>` | Per-session rolling snapshot |
| `cherry/agent/<agent_id>/latest` | Most recent handoff snapshot (fast resume) |

The bridge constructs these automatically.

---

## Known Issues

| Issue | Impact | Workaround |
|-------|--------|------------|
| Cherry MCP proxy caches stateful `Server` objects | MCP tools unavailable to agents after first session | Use REST API via `Bash` + `curl --noproxy '*'` |
| System HTTP proxy at `127.0.0.1:7897` | curl to localhost gets 502 errors | Always use `--noproxy '*'` flag |
| Claude Code MCP URL transport | memory-hub MCP tools may not load | Use REST API or `claude-mem` plugin for search |

---

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| `cherry_bridge_disabled` | memory-hub started without `CHERRY_*` env vars | Restart memory-hub with Cherry Bridge env vars |
| `Connection refused` on :43123 | memory-hub not running | `launchctl start com.cherrystudio.memory-hub` |
| `Connection refused` on :23333 | Cherry API server not enabled | Enable in Cherry Settings > API Server |
| curl returns `502` | HTTP proxy intercepting localhost | Add `--noproxy '*'` to curl command |

---

## File Index

```
mcp_skill/
  SKILL.md                              <-- You are here
  reference/
    rest-api.md                         <-- REST API curl templates (primary)
    cherry-bridge-tools.md              <-- Cherry Bridge MCP tools (reference)
    memory-hub-tools.md                 <-- Base MCP tools (reference)
    builtin-tools.md                    <-- Built-in agent tools
  workflows/
    agent-bootstrap.md                  <-- Step-by-step agent creation
    handoff-lifecycle.md                <-- Load/save handoff workflow
    memory-strategy.md                  <-- Memory classification rules
  templates/
    cherry-agent-prompt.md              <-- Precise prompt template for Cherry agents
```
