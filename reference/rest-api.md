# Memory Hub REST API Reference

All endpoints are at `http://127.0.0.1:43123`. Always use `--noproxy '*'` on systems with HTTP proxy configured.

---

## Cherry Bridge Endpoints

### Write a Fact (semantic memory)

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/cherry/memories/fact \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "agent_1772907543062_a6jc4lkaj",
    "title": "Short fact title",
    "text": "Detailed fact content",
    "tags": ["tag1", "tag2"],
    "importance": 0.8
  }'
```

### Write a Resource (operational reference)

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/cherry/memories/resource \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "agent_1772907543062_a6jc4lkaj",
    "title": "Resource title",
    "text": "File path, command, or endpoint",
    "tags": ["paths"],
    "importance": 0.7
  }'
```

### Write an Event (episodic memory)

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/cherry/memories/event \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "agent_1772907543062_a6jc4lkaj",
    "sessionId": "session_1772907543080_kglcx20le",
    "title": "Event title",
    "text": "What happened and why it matters",
    "tags": ["debugging"],
    "importance": 0.6
  }'
```

### Load Handoff Context (resume)

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/cherry/handoff/context \
  -H "Content-Type: application/json" \
  -d '{"agentId": "agent_1772907543062_a6jc4lkaj"}'
```

Response includes: `latest` (snapshot), `semantic`, `resource`, `episodic` (categorized memories).

### Save Handoff Snapshot

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/cherry/handoff/snapshot \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "agent_1772907543062_a6jc4lkaj",
    "sessionId": "session_1772907543080_kglcx20le",
    "source": "claude-external",
    "goal": "Current user goal",
    "status": "Progress summary",
    "answerSummary": "Key result or answer",
    "decisions": ["Confirmed decision 1"],
    "facts": ["Durable fact discovered"],
    "resources": ["/important/file/path"],
    "openQuestions": ["Unresolved question"],
    "nextSteps": ["What should happen next"]
  }'
```

### Ensure Agent (bootstrap)

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/cherry/agents/ensure \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Agent Name",
    "model": "cherryin:anthropic/claude-sonnet-4.6",
    "workspacePath": "/path/to/project",
    "description": "Optional description"
  }'
```

### Check Cherry Bridge Status

```bash
curl -s --noproxy '*' http://127.0.0.1:43123/v1/cherry/status
```

---

## Base Memory Endpoints

### Search Memories

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "search text",
    "limit": 10,
    "layer": ["semantic", "resource"],
    "tags": ["optional-tag"]
  }'
```

### Write Memory (low-level)

```bash
curl -s --noproxy '*' -X POST http://127.0.0.1:43123/v1/memories \
  -H "Content-Type: application/json" \
  -d '{
    "layer": "semantic",
    "namespace": "cherry/agent/agent_xxx",
    "title": "Title",
    "text": "Content",
    "tags": ["tag1"],
    "importance": 0.7
  }'
```

### Get Entry by ID

```bash
curl -s --noproxy '*' http://127.0.0.1:43123/v1/resources/entries/<entry-id>
```

### Health Check

```bash
curl -s --noproxy '*' http://127.0.0.1:43123/health
```

### Runtime Status

```bash
curl -s --noproxy '*' http://127.0.0.1:43123/v1/status
```

---

## Notes

- All POST endpoints return `{"success": true, "data": ...}` on success
- The `--noproxy '*'` flag is required because the system has `http_proxy=127.0.0.1:7897`
- Use `-s` flag to suppress curl progress output
- Pipe through `| jq .` for pretty-printed output (optional)
