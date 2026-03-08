# Memory Classification Strategy

What to store, where to store it, and when to store it.
The goal is to build a useful, compact memory base — not an exhaustive log.

---

## Three Memory Layers

| Layer | jq layer value | Namespace | Purpose | Lifespan |
|-------|---------------|-----------|---------|----------|
| **Semantic** | `"semantic"` | `cherry/agent/<id>` | Stable knowledge | Permanent |
| **Resource** | `"resource"` | `cherry/agent/<id>/resource` | Operational references | Permanent |
| **Episodic** | `"episodic"` | `cherry/agent/<id>/session/<sid>` | Notable events | Session-linked |

Plus **Workspace Snapshot** (`"workspace_snapshot"`) for structured state summaries.

---

## Semantic Layer: Facts

Stable information true across sessions:

- **User preferences**: "User prefers Biome over Prettier"
- **Technical decisions**: "Using local file I/O for memory, no server"
- **Constraints**: "Redux data model changes blocked until v2.0.0"
- **Recurring goals**: "Primary project: Cherry Studio desktop AI assistant"

**Importance**: 0.5–1.0 (core constraints: 0.9+, preferences: 0.7–0.8)

---

## Resource Layer: Operational References

Reusable references for navigation and operation:

- **File paths**: "Memory file: ~/.cherrystudio/memory-hub/memory-hub.json"
- **Commands**: "Run tests: pnpm test"
- **Endpoints**: "Cherry API: http://127.0.0.1:23333"
- **IDs and config**: "Default model: cherryin:anthropic/claude-sonnet-4.6"

**Importance**: 0.4–0.8 (critical refs: 0.8, common paths: 0.6–0.7)

---

## Episodic Layer: Notable Events

Events worth remembering:

- **Failures and root causes**: "curl to localhost returned 502 because http_proxy was set"
- **Workarounds**: "Use --noproxy '*' for all localhost requests"
- **Environment anomalies**: "System has HTTP proxy configured globally"
- **Debugging insights**: "Cherry POST /v1/agents does not persist mcps field"

**Importance**: 0.3–0.9 (critical workarounds: 0.7–0.9, minor observations: 0.3–0.4)

---

## Before Writing: Deduplication

Before creating any new entry:

1. **Search first**: `jq --arg q "<topic>" '[.entries[] | select((.title + " " + .text) | test($q; "i"))]' ~/.cherrystudio/memory-hub/memory-hub.json`
2. **If similar exists**: Update existing entry instead of creating duplicate
3. **If no match**: Create new entry

---

## Decision Flowchart

```
Something worth remembering?
  |
  YES → Is it stable across sessions? → YES → semantic (importance 0.8)
  |       NO → Is it a reusable reference? → YES → resource (importance 0.7)
  |               NO → Is it a notable event? → YES → episodic (importance 0.6)
  |                       NO → Is it a full state summary? → YES → workspace_snapshot
  |                               NO → Don't store it.
```

---

## Anti-Patterns

| Anti-pattern | Do instead |
|-------------|------------|
| Storing raw transcripts | Extract key decisions and facts |
| Storing chain-of-thought | Store the conclusion only |
| Storing every file read | Store only notable paths |
| Duplicate entries | Search before writing |
| Vague descriptions | Be specific and operational |
| Storing temporary state as fact | Use snapshot for current state |
