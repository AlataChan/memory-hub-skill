# Memory Hub Base Tools Reference

These 8 tools are the core memory-hub MCP tools. They operate directly on the local
memory store without any Cherry API dependency.

---

## 1. memory_write

Write or update a memory entry in the local memory hub.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | no | Existing entry ID to overwrite. Omit to create new. |
| `layer` | enum | **yes** | `"semantic"`, `"episodic"`, `"resource"`, or `"workspace_snapshot"` |
| `namespace` | string | **yes** | Namespace path (e.g., `"cherry/agent/xxx"`) |
| `title` | string | **yes** | Short title for the entry |
| `text` | string | **yes** | Full content body |
| `tags` | string[] | no | Tags for filtering |
| `importance` | number | no | 0.0 to 1.0 (default: 0.5) |
| `pinned` | boolean | no | Pin to prevent auto-archival |
| `archived` | boolean | no | Mark as archived |

### Example

```json
{
  "layer": "semantic",
  "namespace": "cherry/agent/agent_xxx",
  "title": "User prefers TypeScript strict mode",
  "text": "The user always enables TypeScript strict mode and prefers explicit types over inference.",
  "tags": ["preference", "typescript"],
  "importance": 0.8
}
```

### When to use

Use `memory_write` for direct low-level writes when you need full control over layer,
namespace, and metadata. For Cherry Bridge workflows, prefer `remember_fact`,
`remember_resource`, or `remember_event` which auto-construct namespaces.

---

## 2. memory_search

Search memory-hub entries by text and filters.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `query` | string | **yes** | Search text (empty string returns all) |
| `limit` | integer | no | Max results, 1-100 (default: 20) |
| `layer` | string[] | no | Filter by layers (e.g., `["semantic", "resource"]`) |
| `namespace` | string[] | no | Filter by namespaces |
| `tags` | string[] | no | Filter by tags (AND logic) |
| `includeArchived` | boolean | no | Include archived entries (default: false) |

### Example

```json
{
  "query": "TypeScript configuration",
  "limit": 10,
  "layer": ["semantic", "resource"],
  "tags": ["typescript"]
}
```

### When to use

Use to find existing memories before writing duplicates. Also use at session start
to load relevant context. For Cherry Bridge workflows, prefer `load_handoff_context`
which combines multiple searches automatically.

---

## 3. memory_update

Update an existing memory entry by ID.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **yes** | Entry ID to update |
| `title` | string | no | New title |
| `text` | string | no | New content body |
| `tags` | string[] | no | Replace tags |
| `importance` | number | no | New importance (0.0-1.0) |
| `pinned` | boolean | no | Pin/unpin |
| `archived` | boolean | no | Archive/unarchive |

### Example

```json
{
  "id": "entry_abc123",
  "text": "Updated: user now prefers Biome over ESLint for formatting.",
  "importance": 0.9
}
```

### When to use

Use when correcting or enriching an existing entry. First search to find the entry ID,
then update it. Prefer update over creating duplicates.

---

## 4. memory_delete

Delete a memory entry by ID. Irreversible.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **yes** | Entry ID to delete |

### Example

```json
{
  "id": "entry_abc123"
}
```

### When to use

Use when a memory is confirmed wrong or obsolete. Prefer archiving (`memory_update`
with `archived: true`) over deletion when the information might be useful later.

---

## 5. memory_pin

Pin or unpin a memory entry. Pinned entries are never auto-archived.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **yes** | Entry ID |
| `pinned` | boolean | **yes** | `true` to pin, `false` to unpin |

### Example

```json
{
  "id": "entry_abc123",
  "pinned": true
}
```

### When to use

Pin entries that are critical and must always be available:
- Core user preferences
- Project-wide constraints
- Security-sensitive decisions

---

## 6. workspace_snapshot_save

Save a workspace snapshot for a namespace. Overwrites any previous snapshot
in the same namespace.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `namespace` | string | **yes** | Target namespace (e.g., `"cherry/agent/xxx/latest"`) |
| `title` | string | **yes** | Snapshot title |
| `text` | string | **yes** | Full snapshot content (JSON or YAML recommended) |
| `tags` | string[] | no | Tags for filtering |
| `importance` | number | no | 0.0-1.0 |

### Example

```json
{
  "namespace": "cherry/agent/agent_xxx/latest",
  "title": "Handoff snapshot rev 3",
  "text": "{\"goal\":\"Add dark mode\",\"status\":\"CSS variables defined, toggle component pending\",\"decisions\":[\"Use CSS custom properties\"],\"next_steps\":[\"Build ThemeToggle component\"]}",
  "tags": ["kind:snapshot"],
  "importance": 0.9
}
```

### When to use

Low-level snapshot save. For Cherry Bridge workflows, prefer `save_handoff_snapshot`
which writes to both `latest` and `session` namespaces and normalizes the schema.

---

## 7. workspace_snapshot_get

Get the latest workspace snapshot for a namespace.

### Parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `namespace` | string | **yes** | Target namespace |
| `id` | string | no | Specific snapshot ID (default: latest) |

### Example

```json
{
  "namespace": "cherry/agent/agent_xxx/latest"
}
```

### When to use

Low-level snapshot read. For Cherry Bridge workflows, prefer `load_handoff_context`
which reads the snapshot and all related memory in one call.

---

## 8. memory_status

Inspect the current memory-hub runtime status and aggregate stats. No parameters.

### Response includes

- `status.state`: `"idle"` or `"active"`
- `status.dataDir`: storage directory path
- `status.idleTtlMs`: idle timeout in milliseconds
- `stats.totalEntries`: total entry count
- `stats.byLayer`: entry counts per layer
- `stats.byNamespace`: entry counts per namespace

### When to use

Use to verify memory-hub is running and to get an overview of stored data.
Useful for debugging and health checks.
