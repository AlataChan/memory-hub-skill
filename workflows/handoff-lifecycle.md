# Handoff Lifecycle Workflow

This workflow defines when and how to load context (resume) and save snapshots
(handoff) during the collaboration between external Claude and Cherry agents.

---

## Core Principle

> memory-hub is the durable handoff bus, not the raw transcript store.

Claude resumes by reading structured handoff state, not by replaying transcripts.
Cherry agent writes structured memory during execution. External Claude reads
that memory at resume time.

---

## Loading Context (Resume)

### When to load

- **Start of every conversation** that involves an existing Cherry agent
- **After context compaction** (conversation was summarized)
- **When user mentions prior Cherry work** ("remember what we did in Cherry?")

### How to load

```
1. Determine agent_id
   - From conversation history
   - From memory_search({ query: "agent-config", tags: ["agent-config"] })

2. load_handoff_context({ agentId: "agent_xxx" })

3. Process response in this order:
   a. latest.snapshot  -> high-level state (goal, status, decisions)
   b. semantic.items   -> durable facts and preferences
   c. resource.items   -> file paths, commands, endpoints
   d. episodic.items   -> notable events, workarounds
   e. session.snapshot  -> previous session state (if sessionId provided)
```

### How to use loaded context

After loading, integrate the context into your working knowledge:

```
From latest snapshot:
- Resume the stated goal
- Continue from the stated status
- Respect all listed decisions
- Address open questions if relevant
- Follow next steps if still applicable

From semantic memory:
- Apply user preferences
- Respect constraints
- Maintain confirmed conventions

From resource memory:
- Use the correct file paths
- Run the correct commands
- Call the correct endpoints

From episodic memory:
- Avoid known pitfalls
- Apply known workarounds
```

### What if there's no context?

If `load_handoff_context` returns `latest: null` and empty search results:
- This is a fresh agent with no prior work
- Proceed normally and start building memory from scratch

---

## Saving Snapshots (Handoff)

### When to save

| Trigger | Priority | Example |
|---------|----------|---------|
| Task complete | **Required** | Feature implemented and tested |
| Meaningful milestone | **Required** | Phase 1 done, moving to phase 2 |
| Before task switch | **Required** | "Let's work on something else" |
| User requests handoff | **Required** | "Save progress", "I'll continue in Cherry" |
| Session ending | **Required** | User says goodbye, conversation wrapping up |
| Before context compaction | **Recommended** | Long conversation approaching limits |
| After significant discovery | **Recommended** | Found root cause of a bug |
| 15+ minutes since last snapshot | **Nice to have** | Periodic checkpoint |

### What NOT to trigger a save

- Mid-thought with no conclusion yet
- Simple Q&A that doesn't change project state
- Reading files without making changes
- Exploring without reaching a decision

### How to save

```json
// save_handoff_snapshot
{
  "agentId": "agent_xxx",
  "sessionId": "session_xxx",
  "source": "claude-external",
  "goal": "<what the user is trying to achieve>",
  "status": "<where we are now>",
  "answerSummary": "<the key answer or result from this session>",
  "decisions": [
    "<confirmed decision 1>",
    "<confirmed decision 2>"
  ],
  "facts": [
    "<durable fact discovered>"
  ],
  "resources": [
    "<important file path>",
    "<useful command>"
  ],
  "openQuestions": [
    "<unresolved question>"
  ],
  "nextSteps": [
    "<what should happen next>"
  ]
}
```

### Quality criteria for a good snapshot

| Field | Good | Bad |
|-------|------|-----|
| `goal` | "Add Cherry Bridge MCP tools to memory-hub" | "Working on stuff" |
| `status` | "14 files implemented, all 18 tests passing" | "Made some progress" |
| `decisions` | "Use last-writer-wins for phase 1" | "Decided some things" |
| `facts` | "Cherry POST /v1/agents does not persist mcps field" | "Cherry API has bugs" |
| `resources` | "src/cherry/CherryBridgeService.ts" | "Some file in cherry folder" |
| `nextSteps` | "Create SKILL.md for LLM workflow orchestration" | "Do more work" |

### Revision tracking

The bridge auto-increments the `revision` field by reading the previous snapshot.
In phase 1, revision is advisory only (no compare-and-swap). Two concurrent writers
may produce the same revision number — this is accepted.

---

## Handoff Patterns

### Pattern 1: Claude-to-Cherry

```
External Claude works on a task
  -> save_handoff_snapshot(source: "claude-external")
  -> User opens Cherry agent in UI
  -> Cherry agent calls load_handoff_context (via MCP)
  -> Cherry agent continues from the snapshot
```

### Pattern 2: Cherry-to-Claude

```
User works with Cherry agent in UI
  -> Cherry agent saves snapshot (via MCP, guided by prompt appendix)
  -> User returns to external Claude
  -> External Claude calls load_handoff_context
  -> External Claude continues from the snapshot
```

### Pattern 3: Periodic checkpoints

```
During a long session:
  -> After milestone 1: save_handoff_snapshot
  -> After milestone 2: save_handoff_snapshot
  -> After milestone 3: save_handoff_snapshot
  -> Each snapshot overwrites the previous in "latest" namespace
  -> Each also creates a per-session copy for history
```

---

## Complete Resume-and-Handoff Example

```
Session start:

1. memory_search({ query: "agent-config", tags: ["agent-config"] })
   -> Found: agent_id = agent_xxx

2. load_handoff_context({ agentId: "agent_xxx" })
   -> latest.snapshot:
      goal: "Implement memory-hub Cherry Bridge"
      status: "All 14 files implemented, tests passing"
      nextSteps: ["Create skill for LLM workflow"]

3. Tell user: "I see we completed the Cherry Bridge implementation.
   The next step was creating the skill. Shall I proceed?"

... work happens ...

Session end:

4. save_handoff_snapshot({
     agentId: "agent_xxx",
     sessionId: "session_xxx",
     source: "claude-external",
     goal: "Create Cherry Memory Bridge skill",
     status: "Skill created with 7 files covering all 29 tools",
     decisions: ["Use cherryin:anthropic/claude-sonnet-4.6 as default model"],
     resources: ["SKILL.md"],
     nextSteps: ["Test skill in real Cherry agent session"]
   })

5. remember_fact({
     agentId: "agent_xxx",
     title: "Default model for Cherry agents",
     text: "cherryin:anthropic/claude-sonnet-4.6 is the default model for all Cherry claude-code agents.",
     tags: ["model", "config"]
   })
```
