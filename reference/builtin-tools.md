# Built-in Agent Tools Reference

These 13 tools are provided by Cherry's `claude-code` agent runtime. They are
the same tools available in Claude Code CLI. The Cherry agent uses them to
perform coding tasks within the session.

---

## File Operations

### 1. Read

Read the contents of a file.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `file_path` | string | **yes** | Absolute path to the file |
| `offset` | number | no | Starting line number |
| `limit` | number | no | Max lines to read |

**When to use:** Before editing any file. Always read first, then edit.
Supports images (PNG, JPG), PDFs (with `pages` param), and Jupyter notebooks.

### 2. Write

Create or overwrite a file entirely.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `file_path` | string | **yes** | Absolute path |
| `content` | string | **yes** | Full file content |

**When to use:** Creating new files only. For modifying existing files, prefer `Edit` or `MultiEdit`.

### 3. Edit

Make targeted string replacements in a file. Requires reading the file first.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `file_path` | string | **yes** | Absolute path |
| `old_string` | string | **yes** | Exact text to find (must be unique in file) |
| `new_string` | string | **yes** | Replacement text |
| `replace_all` | boolean | no | Replace all occurrences (default: false) |

**When to use:** Most common editing tool. Use for surgical changes. The `old_string`
must match exactly including whitespace and indentation.

### 4. MultiEdit

Perform multiple edits on a single file atomically.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `file_path` | string | **yes** | Absolute path |
| `edits` | array | **yes** | Array of `{old_string, new_string}` pairs |

**When to use:** When you need to make several related changes to one file in a
single operation. Safer than multiple sequential `Edit` calls.

---

## Search & Navigation

### 5. Glob

Find files by name pattern using glob syntax.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `pattern` | string | **yes** | Glob pattern (e.g., `"**/*.ts"`, `"src/**/*.test.ts"`) |
| `path` | string | no | Directory to search in (default: cwd) |

**When to use:** Finding files by name or extension. Faster than `Bash` with `find`.
Results sorted by modification time.

### 6. Grep

Search file contents using regex patterns (powered by ripgrep).

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `pattern` | string | **yes** | Regex pattern to search for |
| `path` | string | no | File or directory to search |
| `glob` | string | no | File filter (e.g., `"*.ts"`) |
| `output_mode` | enum | no | `"content"`, `"files_with_matches"`, `"count"` |
| `-i` | boolean | no | Case insensitive |
| `-C` | number | no | Context lines around matches |
| `head_limit` | number | no | Limit output lines |

**When to use:** Searching for code patterns, function definitions, import statements,
or any text across the codebase. More powerful than `Bash` with `grep`.

---

## Shell Execution

### 7. Bash

Execute shell commands.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `command` | string | **yes** | The shell command to run |
| `description` | string | no | Human-readable description |
| `timeout` | number | no | Timeout in ms (max: 600000) |
| `run_in_background` | boolean | no | Run asynchronously |

**When to use:** System commands, git operations, package management, running tests,
starting servers, and any operation that requires shell execution. Prefer dedicated
tools (Read, Edit, Glob, Grep) over Bash equivalents when possible.

**Common uses in Cherry Bridge context:**
- `pnpm dev` / `pnpm test` / `pnpm lint` — development commands
- `git status` / `git diff` / `git log` — version control
- `curl --noproxy '*' http://127.0.0.1:43123/...` — memory-hub HTTP API calls
- `launchctl start/stop com.cherrystudio.memory-hub` — service management
- `lsof -iTCP:43123` — check if memory-hub is running

---

## Notebook Operations

### 8. NotebookRead

Read and display Jupyter notebook contents including code, text, and outputs.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `file_path` | string | **yes** | Path to `.ipynb` file |

### 9. NotebookEdit

Modify Jupyter notebook cells.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `file_path` | string | **yes** | Path to `.ipynb` file |
| `cell_index` | number | **yes** | Cell to edit (0-indexed) |
| `new_source` | string | **yes** | New cell content |

**When to use:** Only when working with Jupyter notebooks. Rarely needed in typical
Cherry Studio development.

---

## Task Management

### 10. Task

Run a sub-agent to handle complex, multi-step tasks.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | **yes** | What the sub-agent should do |
| `prompt` | string | **yes** | Detailed instructions for the sub-agent |

**When to use:** Delegate independent sub-tasks (research, exploration, parallel
implementation) to a sub-agent. The sub-agent gets its own context window.

### 11. TodoWrite

Create and manage structured task lists.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `todos` | array | **yes** | Array of `{content, status, activeForm}` objects |

Status values: `"pending"`, `"in_progress"`, `"completed"`

**When to use:** Track multi-step tasks. Update status as you work. Only one task
should be `in_progress` at a time.

---

## Web Access

### 12. WebFetch

Fetch content from a URL.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | **yes** | URL to fetch |
| `headers` | object | no | Custom request headers |

**When to use:** Fetching API responses, documentation pages, or remote resources.
For Cherry Bridge HTTP calls, prefer `Bash` with `curl --noproxy '*'` to avoid
proxy issues.

### 13. WebSearch

Perform web searches with domain filtering.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `query` | string | **yes** | Search query |
| `domains` | string[] | no | Limit to specific domains |

**When to use:** Researching APIs, finding documentation, looking up error messages.

---

## Tool Selection Guide for Cherry Bridge

| Task | Preferred Tool | Why |
|------|---------------|-----|
| Check memory-hub health | `Bash` with curl | Direct HTTP, bypass proxy |
| Read source files | `Read` | Dedicated, shows line numbers |
| Edit source files | `Edit` / `MultiEdit` | Precise, reviewable |
| Find files | `Glob` | Faster than `find` |
| Search code | `Grep` | Regex support, filtering |
| Run tests | `Bash` | Shell execution needed |
| Create files | `Write` | Dedicated file creation |
| Manage memory-hub service | `Bash` with launchctl | System service control |
| Track implementation progress | `TodoWrite` | Structured task tracking |
| Delegate research | `Task` | Parallel sub-agent |
