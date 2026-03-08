# Handoff Lifecycle Workflow

When and how to load context (resume) and save snapshots (handoff).

---

## Core Principle

> The memory file is the durable handoff bus, not a transcript store.

Resume by reading structured handoff state from the JSON file, not by replaying transcripts.

---

## Loading Context (Resume)

### When to load

- **Start of every conversation** involving an existing Cherry agent
- **After context compaction** (long conversation was summarized)
- **When user mentions prior work** ("remember what we did?")

### How to load

```bash
jq '{
  latest: [.entries[] | select(.layer == "workspace_snapshot" and .namespace == "cherry/agent/agent_1772907543062_a6jc4lkaj/latest")] | last,
  facts: [.entries[] | select(.layer == "semantic" and (.namespace | startswith("cherry/agent/agent_1772907543062_a6jc4lkaj")))],
  resources: [.entries[] | select(.layer == "resource" and (.namespace | startswith("cherry/agent/agent_1772907543062_a6jc4lkaj")))],
  events: [.entries[] | select(.layer == "episodic" and (.namespace | startswith("cherry/agent/agent_1772907543062_a6jc4lkaj")))]
}' ~/.cherrystudio/memory-hub/memory-hub.json
```

### How to use loaded context

- **From latest snapshot**: Resume goal, continue from status, respect decisions
- **From semantic**: Apply preferences, respect constraints
- **From resource**: Use correct paths, commands, endpoints
- **From episodic**: Avoid known pitfalls, apply workarounds

### No context found?

If latest is null and entries are empty — this is a fresh agent. Proceed normally.

---

## Saving Snapshots (Handoff)

### When to save

| Trigger | Priority |
|---------|----------|
| Task complete | **Required** |
| Meaningful milestone | **Required** |
| Before task switch | **Required** |
| User requests handoff | **Required** |
| Session ending | **Required** |
| After significant discovery | **Recommended** |
| 15+ minutes since last snapshot | **Nice to have** |

### When NOT to save

- Mid-thought with no conclusion
- Simple Q&A that doesn't change state
- Reading files without changes
- Exploring without reaching a decision

### How to save

See SKILL.md Phase 4 for the complete jq command. The snapshot writes to both:
- `cherry/agent/<id>/session/<sid>` (history)
- `cherry/agent/<id>/latest` (fast resume, replaces previous)

### Quality criteria

| Field | Good | Bad |
|-------|------|-----|
| `goal` | "Migrate to pure skill architecture" | "Working on stuff" |
| `status` | "All files rewritten, server stopped, tested OK" | "Made progress" |
| `decisions` | "Use local file I/O, no server process" | "Decided things" |
| `next_steps` | "Push to remote repos, update Cherry agent prompt" | "Do more" |

---

## Handoff Patterns

### Claude-to-Cherry

```
Claude saves snapshot → User opens Cherry → Cherry reads memory file → Cherry continues
```

### Cherry-to-Claude

```
Cherry writes memories → User returns to Claude → Claude reads memory file → Claude continues
```

### Periodic checkpoints

```
Milestone 1 → save snapshot → Milestone 2 → save snapshot → ...
Each overwrites "latest", adds to session history
```
