# Agent Bootstrap Workflow

Creating a new Cherry agent and preparing it for memory handoff collaboration.

---

## Prerequisites

1. **Memory file exists**: `ls ~/.cherrystudio/memory-hub/memory-hub.json`
2. **Cherry API is accessible**: `curl -s --noproxy '*' http://127.0.0.1:23333/v1/models -H "Authorization: Bearer cs-sk-f995f423-d32a-455e-b49f-66f288f60b12"`

If memory file doesn't exist:
```bash
mkdir -p ~/.cherrystudio/memory-hub
echo '{"version":1,"entries":[]}' > ~/.cherrystudio/memory-hub/memory-hub.json
```

---

## Step-by-Step: Create a New Agent

### Step 1: Determine agent parameters

| Parameter | Source | Default |
|-----------|--------|---------|
| `name` | User provides or infer from project | Required |
| `model` | Fixed | `"cherryin:anthropic/claude-sonnet-4.6"` |

### Step 2: Create agent via Cherry API

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:23333/v1/agents \
  -H "Authorization: Bearer cs-sk-f995f423-d32a-455e-b49f-66f288f60b12" \
  -H "Content-Type: application/json" \
  -d '{"name": "Cherry Studio Dev Agent", "model": "cherryin:anthropic/claude-sonnet-4.6"}'
```

### Step 3: Record the IDs in memory file

```bash
ID=$(uuidgen | tr '[:upper:]' '[:lower:]')
NOW=$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")

jq --arg id "$ID" --arg now "$NOW" \
  --arg title "Agent and session IDs" \
  --arg text "agent_id: <AGENT_ID>\nsession_id: <SESSION_ID>\nmodel: cherryin:anthropic/claude-sonnet-4.6" \
  '.entries += [{
    "id": $id, "layer": "resource",
    "namespace": "cherry/agent/<AGENT_ID>/resource",
    "title": $title, "text": $text,
    "tags": ["agent-config", "from-claude-code"], "importance": 0.8,
    "pinned": false, "archived": false,
    "createdAt": $now, "updatedAt": $now, "lastAccessedAt": $now
  }]' ~/.cherrystudio/memory-hub/memory-hub.json > /tmp/memory-hub-tmp.json && \
mv /tmp/memory-hub-tmp.json ~/.cherrystudio/memory-hub/memory-hub.json
```

### Step 4: Verify in Cherry UI

The agent should appear in Cherry's Agent page.

### Step 5: Save initial handoff snapshot

Save a snapshot entry to the memory file (see SKILL.md Phase 4).

---

## Reuse an Existing Agent

Search memory file for agent config:
```bash
jq '[.entries[] | select(.tags | index("agent-config"))] | .[0]' \
  ~/.cherrystudio/memory-hub/memory-hub.json
```

Or list agents from Cherry API:
```bash
curl -s --noproxy '*' http://127.0.0.1:23333/v1/agents \
  -H "Authorization: Bearer cs-sk-f995f423-d32a-455e-b49f-66f288f60b12" | jq '.data[] | {id, name}'
```
