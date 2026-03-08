# Cherry API & Local File I/O Reference

Memory operations use local file I/O (jq). Agent management uses Cherry API at `http://127.0.0.1:23333`.
Always use `--noproxy '*'` for curl commands (system has HTTP proxy).

---

## Memory File Operations (local)

Memory file: `~/.cherrystudio/memory-hub/memory-hub.json`

### Write a Fact (semantic memory)

```bash
ID=$(uuidgen | tr '[:upper:]' '[:lower:]')
NOW=$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")

jq --arg id "$ID" --arg now "$NOW" \
  --arg title "Short fact title" --arg text "Detailed fact content" \
  '.entries += [{
    "id": $id, "layer": "semantic",
    "namespace": "cherry/agent/agent_1772907543062_a6jc4lkaj",
    "title": $title, "text": $text,
    "tags": ["from-claude-code"], "importance": 0.8,
    "pinned": false, "archived": false,
    "createdAt": $now, "updatedAt": $now, "lastAccessedAt": $now
  }]' ~/.cherrystudio/memory-hub/memory-hub.json > /tmp/memory-hub-tmp.json && \
mv /tmp/memory-hub-tmp.json ~/.cherrystudio/memory-hub/memory-hub.json
```

### Write a Resource (operational reference)

Same as fact, but with `"layer": "resource"` and namespace `cherry/agent/<id>/resource`.

### Write an Event (episodic memory)

Same as fact, but with `"layer": "episodic"` and namespace `cherry/agent/<id>/session/<session_id>`.

### Read All Memories (resume context)

```bash
jq '{
  latest: [.entries[] | select(.layer == "workspace_snapshot" and .namespace == "cherry/agent/agent_1772907543062_a6jc4lkaj/latest")] | last,
  facts: [.entries[] | select(.layer == "semantic" and (.namespace | startswith("cherry/agent/agent_1772907543062_a6jc4lkaj")))],
  resources: [.entries[] | select(.layer == "resource" and (.namespace | startswith("cherry/agent/agent_1772907543062_a6jc4lkaj")))],
  events: [.entries[] | select(.layer == "episodic" and (.namespace | startswith("cherry/agent/agent_1772907543062_a6jc4lkaj")))]
}' ~/.cherrystudio/memory-hub/memory-hub.json
```

### Search Memories

```bash
jq --arg q "search text" \
  '[.entries[] | select((.title + " " + .text) | test($q; "i"))] | .[] | {title, layer, importance, text: .text[:100]}' \
  ~/.cherrystudio/memory-hub/memory-hub.json
```

### Save Handoff Snapshot

```bash
ID=$(uuidgen | tr '[:upper:]' '[:lower:]')
NOW=$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")
AGENT="agent_1772907543062_a6jc4lkaj"
SESSION="session_1772907543080_kglcx20le"

SNAPSHOT='{"goal":"<GOAL>","status":"<STATUS>","decisions":["<D>"],"next_steps":["<NS>"],"updated_at":"'$NOW'"}'

jq --arg id "$ID" --arg now "$NOW" --arg text "$SNAPSHOT" \
  --arg sns "cherry/agent/$AGENT/session/$SESSION" \
  --arg lns "cherry/agent/$AGENT/latest" \
  '
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
  ]' ~/.cherrystudio/memory-hub/memory-hub.json > /tmp/memory-hub-tmp.json && \
mv /tmp/memory-hub-tmp.json ~/.cherrystudio/memory-hub/memory-hub.json
```

### Delete Entry

```bash
jq --arg id "<ENTRY_ID>" '.entries = [.entries[] | select(.id != $id)]' \
  ~/.cherrystudio/memory-hub/memory-hub.json > /tmp/memory-hub-tmp.json && \
mv /tmp/memory-hub-tmp.json ~/.cherrystudio/memory-hub/memory-hub.json
```

### Initialize Memory File

```bash
mkdir -p ~/.cherrystudio/memory-hub
echo '{"version":1,"entries":[]}' > ~/.cherrystudio/memory-hub/memory-hub.json
```

---

## Cherry Studio API Endpoints (port 23333)

These require the Cherry Studio API server to be enabled (Settings > API Server).

### List Agents

```bash
curl -s --noproxy '*' http://127.0.0.1:23333/v1/agents \
  -H "Authorization: Bearer cs-sk-f995f423-d32a-455e-b49f-66f288f60b12"
```

### Create Agent

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:23333/v1/agents \
  -H "Authorization: Bearer cs-sk-f995f423-d32a-455e-b49f-66f288f60b12" \
  -H "Content-Type: application/json" \
  -d '{"name": "Agent Name", "model": "cherryin:anthropic/claude-sonnet-4.6"}'
```

### List Models

```bash
curl -s --noproxy '*' http://127.0.0.1:23333/v1/models \
  -H "Authorization: Bearer cs-sk-f995f423-d32a-455e-b49f-66f288f60b12"
```

---

## Notes

- Memory operations are pure local file I/O — no server process needed
- Agent management requires Cherry API at :23333
- Use `--noproxy '*'` for all curl commands (system has `http_proxy=127.0.0.1:7897`)
- Use `/tmp/memory-hub-tmp.json` as temp file for atomic writes
- Use `uuidgen | tr '[:upper:]' '[:lower:]'` for entry IDs
- Use `date -u +"%Y-%m-%dT%H:%M:%S.000Z"` for timestamps
