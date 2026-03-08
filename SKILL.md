---
name: cherry-memory-bridge
description: |
  Pure-skill architecture for Cherry Studio agent orchestration and shared memory.
  No server process required — all memory ops use local file I/O (jq),
  agent management uses Cherry API (:23333).
  Covers: bootstrap, work, handoff, resume.
---

# Cherry Memory Bridge — Pure Skill Architecture

All memory operations use **local file I/O** (Read/Write tools + jq).
Agent management uses **curl to Cherry API** (:23333).
No external server, no MCP, no background process.

## Configuration

```yaml
agent_id: agent_1772907543062_a6jc4lkaj
session_id: session_1772907543080_kglcx20le
model: cherryin:anthropic/claude-sonnet-4.6
memory_file: ~/.cherrystudio/memory-hub/memory-hub.json
cherry_api: http://127.0.0.1:23333
cherry_api_key: cs-sk-f995f423-d32a-455e-b49f-66f288f60b12
```

## Architecture

```
AI Tool (Claude / Codex / Gemini)
    ├── Read/Write ──→ ~/.cherrystudio/memory-hub/memory-hub.json
    └── curl :23333 ──→ Cherry Studio API (Agent CRUD only)
```

**Requirements**: Any AI tool with Bash + file read/write capability.

---

## Lifecycle

```
[1. Bootstrap] --> [2. Resume] --> [3. Work] --> [4. Handoff]
     |                 |              |               |
  curl :23333      read file      jq write        jq write
  (agent CRUD)     (jq filter)    (per event)     (snapshot)
```

### Phase 1: Bootstrap (first time only)

Create a Cherry agent via Cherry API:

```bash
# Create agent
curl -s --noproxy '*' -X POST http://127.0.0.1:23333/v1/agents \
  -H "Authorization: Bearer cs-sk-f995f423-d32a-455e-b49f-66f288f60b12" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Cherry Studio Dev Agent",
    "model": "cherryin:anthropic/claude-sonnet-4.6",
    "type": "claude-code"
  }'
```

Store the returned `id` in this skill's configuration. Then ensure the memory file exists:

```bash
mkdir -p ~/.cherrystudio/memory-hub
[ -f ~/.cherrystudio/memory-hub/memory-hub.json ] || echo '{"version":1,"entries":[]}' > ~/.cherrystudio/memory-hub/memory-hub.json
```

### Phase 2: Resume (start of each session)

Read the memory file and extract context for the current agent:

```bash
jq '{
  latest: [.entries[] | select(.layer == "workspace_snapshot" and .namespace == "cherry/agent/agent_1772907543062_a6jc4lkaj/latest")] | last,
  facts: [.entries[] | select(.layer == "semantic" and (.namespace | startswith("cherry/agent/agent_1772907543062_a6jc4lkaj")))],
  resources: [.entries[] | select(.layer == "resource" and (.namespace | startswith("cherry/agent/agent_1772907543062_a6jc4lkaj")))],
  events: [.entries[] | select(.layer == "episodic" and (.namespace | startswith("cherry/agent/agent_1772907543062_a6jc4lkaj")))]
}' ~/.cherrystudio/memory-hub/memory-hub.json
```

Process the result:
1. `latest` — most recent handoff snapshot (goal, status, decisions)
2. `facts` — durable semantic memories (preferences, decisions)
3. `resources` — file paths, commands, endpoints
4. `events` — notable failures, workarounds

Use this context to resume work without asking the user to repeat themselves.

### Phase 3: Work (during execution)

Write memories as they happen using jq — do not batch them:

```bash
# Template for all memory writes
ID=$(uuidgen | tr '[:upper:]' '[:lower:]')
NOW=$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")

jq --arg id "$ID" --arg now "$NOW" \
  --arg title "<TITLE>" --arg text "<TEXT>" \
  --arg layer "<LAYER>" --arg ns "<NAMESPACE>" \
  --argjson imp <IMPORTANCE> \
  '.entries += [{
    "id": $id, "layer": $layer, "namespace": $ns,
    "title": $title, "text": $text,
    "tags": ["from-claude-code"],
    "importance": $imp, "pinned": false, "archived": false,
    "createdAt": $now, "updatedAt": $now, "lastAccessedAt": $now
  }]' ~/.cherrystudio/memory-hub/memory-hub.json > /tmp/memory-hub-tmp.json && \
mv /tmp/memory-hub-tmp.json ~/.cherrystudio/memory-hub/memory-hub.json
```

| What happened | Layer | Namespace suffix | Importance |
|---------------|-------|-----------------|------------|
| Technical decision / preference | `semantic` | (none) | 0.8 |
| File path / command / endpoint | `resource` | `/resource` | 0.7 |
| Failure / workaround / anomaly | `episodic` | `/session/<session_id>` | 0.6-0.7 |

**Namespace base**: `cherry/agent/agent_1772907543062_a6jc4lkaj`

### Phase 4: Handoff (end of session or task boundary)

Save a workspace snapshot capturing session state:

```bash
ID=$(uuidgen | tr '[:upper:]' '[:lower:]')
NOW=$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")
AGENT="agent_1772907543062_a6jc4lkaj"
SESSION="session_1772907543080_kglcx20le"

SNAPSHOT='{"goal":"<GOAL>","status":"<STATUS>","decisions":[<DECISIONS>],"next_steps":[<NEXT_STEPS>],"updated_at":"'$NOW'"}'

# Write jq filter to temp file (avoids != shell escaping issue)
cat > /tmp/handoff-filter.jq << 'JQEOF'
.entries = [.entries[] | select(.namespace != $lns)] |
.entries += [
  {"id": $id, "layer": "workspace_snapshot", "namespace": $sns,
   "title": "handoff snapshot", "text": $text,
   "tags": ["kind:snapshot","source:claude-external"],
   "importance": 0.7, "pinned": false, "archived": false,
   "createdAt": $now, "updatedAt": $now, "lastAccessedAt": $now},
  {"id": ($id + "-latest"), "layer": "workspace_snapshot", "namespace": $lns,
   "title": "handoff snapshot", "text": $text,
   "tags": ["kind:snapshot","source:claude-external"],
   "importance": 0.7, "pinned": false, "archived": false,
   "createdAt": $now, "updatedAt": $now, "lastAccessedAt": $now}
]
JQEOF

jq --from-file /tmp/handoff-filter.jq \
  --arg id "$ID" --arg now "$NOW" --arg text "$SNAPSHOT" \
  --arg sns "cherry/agent/$AGENT/session/$SESSION" \
  --arg lns "cherry/agent/$AGENT/latest" \
  ~/.cherrystudio/memory-hub/memory-hub.json > /tmp/memory-hub-tmp.json && \
mv /tmp/memory-hub-tmp.json ~/.cherrystudio/memory-hub/memory-hub.json
```

**When to handoff:**
- Task is complete or milestone reached
- About to switch context
- User explicitly requests
- Session is ending

---

## Decision Trees

### "What layer should this memory go to?"

```
Stable across sessions? (preferences, decisions, constraints)
  YES → semantic (importance: 0.8)

Reusable reference? (file path, command, port, endpoint)
  YES → resource (importance: 0.7)

Notable event? (failure, workaround, anomaly)
  YES → episodic (importance: 0.6-0.7)

Full session state summary?
  YES → workspace_snapshot via handoff
```

### "Should I save a snapshot now?"

```
Done with a task or subtask?           → YES
About to switch context?               → YES
User asked to hand off?                → YES
15+ minutes since last snapshot?       → YES
Mid-thought with no conclusion?        → NO, wait
```

---

## What NOT to Store

- Raw chain-of-thought
- Full conversation transcripts
- Speculative or unverified conclusions
- Temporary debugging state (unless notable workaround)
- Duplicate information already in an existing entry

---

## Namespace Protocol

| Namespace | Purpose |
|-----------|---------|
| `cherry/agent/<agent_id>` | Long-lived semantic memory |
| `cherry/agent/<agent_id>/resource` | Reusable operational context |
| `cherry/agent/<agent_id>/session/<session_id>` | Per-session rolling snapshot |
| `cherry/agent/<agent_id>/latest` | Most recent handoff snapshot (fast resume) |

---

## Search

**For small datasets (<500 entries)**: Read the entire file and let the AI do semantic matching.
This is more accurate than any text-based search.

**For text filtering**: Use jq:

```bash
jq --arg q "keyword" '[.entries[] | select((.title + " " + .text) | test($q; "i"))]' \
  ~/.cherrystudio/memory-hub/memory-hub.json
```

**For fuzzy matching** (when needed):

```bash
python3 -c "
import json, difflib, sys
q = sys.argv[1].lower()
data = json.load(open('$HOME/.cherrystudio/memory-hub/memory-hub.json'))
results = []
for e in data['entries']:
    combined = (e.get('title','') + ' ' + e.get('text','')).lower()
    score = difflib.SequenceMatcher(None, q, combined).ratio()
    if score > 0.25:
        results.append((score, e['title'], e['text'][:80], e['layer']))
for s, t, x, l in sorted(results, reverse=True):
    print(f'{s:.2f} [{l}] {t}: {x}')
" "search query"
```

---

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| Memory file not found | First run or deleted | Create with `echo '{"version":1,"entries":[]}' > ~/.cherrystudio/memory-hub/memory-hub.json` |
| jq error | Malformed JSON | Check file integrity; restore from backup if needed |
| Cherry API :23333 refused | API server not enabled | Enable in Cherry Settings > API Server |
| curl 502 | HTTP proxy intercepting | Add `--noproxy '*'` |

---

## Cross-Tool Compatibility

This skill works with **any AI tool** that supports:
- **Bash** (for jq, curl, uuidgen, date)
- **File read/write** (for reading/writing the JSON file)

Tested with: Claude Code, Cherry Studio agents.
Compatible with: Codex, Gemini, any tool with shell access.
