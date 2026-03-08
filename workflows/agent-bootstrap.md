# Agent Bootstrap Workflow

This workflow covers creating a new Cherry agent and preparing it for
memory-hub handoff collaboration.

---

## Prerequisites

Before bootstrapping, verify:

1. **memory-hub is running**: `launchctl list | grep memory-hub` (PID should be a number)
2. **Cherry API is accessible**: `curl --noproxy '*' http://127.0.0.1:23333/v1/models`
3. **Cherry Bridge is enabled**: `curl --noproxy '*' http://127.0.0.1:43123/v1/cherry/status`

If any check fails, see troubleshooting in `SKILL.md`.

---

## Step-by-Step: Create a New Agent

### Step 1: Determine agent parameters

Gather from the user or infer from context:

| Parameter | Source | Default |
|-----------|--------|---------|
| `name` | User provides or infer from project | Required |
| `model` | Fixed | `"cherryin:anthropic/claude-sonnet-4.6"` |
| `workspacePath` | Current project directory | Required |
| `description` | Infer from task | Optional |
| `instructions` | User's special instructions | Optional |

### Step 2: Call ensure_cherry_agent

```json
{
  "name": "Cherry Studio Dev Agent",
  "model": "cherryin:anthropic/claude-sonnet-4.6",
  "workspacePath": "/path/to/your/project",
  "description": "Development agent for Cherry Studio codebase"
}
```

### Step 3: Record the IDs

From the response, extract and remember:

```
agent_id:  agent_abc123_example
session_id: session_def456_example
```

Store these with `remember_resource`:

```json
{
  "agentId": "agent_abc123_example",
  "title": "Agent and session IDs",
  "text": "agent_id: agent_abc123_example\nsession_id: session_def456_example\nworkspace: /path/to/your/project\nmodel: anthropic/claude-sonnet-4.6",
  "tags": ["agent-config"]
}
```

### Step 4: Verify in Cherry UI

The agent should now appear in Cherry's Agent page. Confirm:
- Name matches
- MCP server `memory-hub` is listed
- Instructions include the handoff prompt appendix

### Step 5: Save initial snapshot

```json
{
  "agentId": "agent_xxx",
  "sessionId": "session_xxx",
  "source": "claude-external",
  "goal": "Initial agent setup",
  "status": "Agent created and configured with memory-hub",
  "decisions": [],
  "facts": [],
  "resources": [],
  "openQuestions": [],
  "nextSteps": ["User will interact with agent in Cherry UI"]
}
```

---

## Step-by-Step: Reuse an Existing Agent

### Option A: You know the agent_id

```json
{
  "agentId": "agent_xxx",
  "name": "Cherry Studio Dev Agent",
  "model": "cherryin:anthropic/claude-sonnet-4.6",
  "workspacePath": "/path/to/your/project"
}
```

The bridge finds the agent by ID, patches it to ensure config is current.

### Option B: You know the agent name

```json
{
  "name": "Cherry Studio Dev Agent",
  "model": "cherryin:anthropic/claude-sonnet-4.6",
  "workspacePath": "/path/to/your/project"
}
```

The bridge searches by name. If found, reuses it. If not, creates new.

### Option C: Load from memory

1. Call `memory_search` with `query: "agent-config"` and `tags: ["agent-config"]`
2. Extract the stored `agent_id`
3. Call `ensure_cherry_agent` with that `agentId`

---

## Creating a New Session for an Existing Agent

When you need a fresh session (e.g., new task context):

```json
// get_or_create_cherry_session
{
  "agentId": "agent_xxx"
  // omit sessionId and reuseLatest to create new
}
```

When you want to continue the latest session:

```json
{
  "agentId": "agent_xxx",
  "reuseLatest": true
}
```

---

## Complete Bootstrap Example

```
User: "Create a Cherry agent for working on the memory-hub project"

Claude:
1. ensure_cherry_agent({
     name: "Memory Hub Dev Agent",
     model: "cherryin:anthropic/claude-sonnet-4.6",
     workspacePath: "/path/to/memory-hub",
     description: "Development agent for memory-hub package"
   })
   -> agent_id: agent_xxx, session_id: session_xxx

2. remember_resource({
     agentId: "agent_xxx",
     title: "Memory Hub agent config",
     text: "agent_id: agent_xxx\nsession_id: session_xxx\nworkspace: /path/to/memory-hub",
     tags: ["agent-config"]
   })

3. save_handoff_snapshot({
     agentId: "agent_xxx",
     sessionId: "session_xxx",
     source: "claude-external",
     goal: "Development agent bootstrapped for memory-hub",
     status: "Agent created, ready for user interaction in Cherry UI",
     resources: ["/path/to/memory-hub"],
     nextSteps: ["Open agent in Cherry UI", "Start coding task"]
   })

4. Report to user:
   "Agent 'Memory Hub Dev Agent' created. You can find it in Cherry's
    Agent page. The agent has memory-hub MCP configured and will
    share handoff state with me."
```
