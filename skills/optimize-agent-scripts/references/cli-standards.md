# CLI Design Standards for AI Agents

Detailed patterns for building CLIs that work reliably with LLM agents. Reference this when building or auditing CLI tools.

## Checklist: Agent-Friendly CLI

When reviewing an existing CLI, check every item:

- [ ] Non-interactive path for every command (no required arrow keys, menus, timed prompts)
- [ ] `--yes` / `--force` bypass for all confirmations
- [ ] Auto-detect non-TTY (`isatty()`) and disable interactive features
- [ ] `--output json` on every command (flat JSON to stdout)
- [ ] `--quiet` / `--no-color` + honors `NO_COLOR` env var
- [ ] `--dry-run` on every destructive command (structured JSON output)
- [ ] Layered `--help` (subcommand owns its own docs, not dumped upfront)
- [ ] Examples in every `--help` with real invocations
- [ ] Exit codes documented in `--help` (semantic, not all-1)
- [ ] Environment variables documented (`flags > env vars > config > defaults`)
- [ ] Consistent noun-verb command structure across all subcommands
- [ ] Idempotent operations (duplicate create → exit 5 not exit 1)
- [ ] Missing required flags → exit immediately with correct example (never hang)
- [ ] Structured success output (IDs, URLs, durations — not decorative prose)
- [ ] Stdin/pipeline support where relevant (`--stdin` flag or auto-detect)
- [ ] Schema introspection command (`cli schema <method>`)

---

## Flag Design Reference

### Universal flags (every CLI should have these)

```
--output [json|text]   Machine-readable output (default: text)
--quiet                Minimal output, no decoration
--no-color             Disable ANSI (also honor NO_COLOR env var)
--no-interactive       Disable all prompts globally
--yes / --force        Skip confirmations
--dry-run              Preview without committing (JSON output)
--help                 Per-subcommand help with examples
```

### JSON flag pattern

Human CLI:
```bash
mycli sheet create --title "Budget" --frozen-rows 1 --tab-color red
```

Agent CLI (zero translation loss):
```bash
mycli sheet create --json '{"title": "Budget", "gridProperties": {"frozenRowCount": 1}, "tabColor": {"red": 1.0}}'
```

Both should work. The `--json` flag passes the payload directly to the underlying API.

### Schema introspection

```bash
mycli schema sheet.create
# Returns:
{
  "method": "sheet.create",
  "required": ["title"],
  "optional": ["gridProperties", "tabColor"],
  "schema": { ... full JSON schema ... }
}
```

---

## --help Template

```
Usage: mycli <command> [options]

Description of what this command does and what it returns.

Options:
  --env STR     Target environment: staging | production [required]
  --tag STR     Image tag (default: latest)
  --id STR      Resource identifier (no ? or # characters)
  --dry-run     Preview changes without committing
  --yes         Skip confirmations
  --output FMT  Output format: json | text (default: text)
  --quiet       Minimal output
  --no-color    Disable ANSI color

Exit codes:
  0  success
  1  general failure (retry-worthy)
  2  usage / argument error
  3  resource not found
  4  permission denied
  5  conflict — resource already exists

Environment:
  MYTOOL_ENV       Overrides --env
  MYTOOL_TOKEN     API authentication token
  NO_COLOR         Disable color output
  MYTOOL_NO_INTERACTIVE  Equivalent to --no-interactive

Examples:
  mycli deploy --env staging
  mycli deploy --env production --tag v1.2.3
  mycli deploy --env staging --dry-run --output json
  mycli deploy --env staging --yes
```

---

## Structured Success Output

```json
{
  "status": "deployed",
  "version": "v1.2.3",
  "env": "staging",
  "url": "https://staging.myapp.com",
  "deploy_id": "dep_abc123",
  "duration_ms": 34000
}
```

For operations that produce lists, use NDJSON (one JSON object per line) for streaming:

```
{"id": "abc", "name": "item-1", "status": "active"}
{"id": "def", "name": "item-2", "status": "inactive"}
```

Include a truncation signal when results are capped:

```json
{"items": [...], "total": 1042, "returned": 100, "truncated": true, "message": "Use --filter to narrow results"}
```

---

## Error Output Template

```json
{
  "error": "resource_not_found",
  "message": "Deploy target 'production-eu' not found",
  "input": {"env": "production-eu"},
  "hint": "Use --output json mycli env list to see available environments",
  "example": "mycli deploy --env production"
}
```

---

## Idempotency Patterns

```bash
# First run: creates resource, exits 0
mycli service create --name my-svc --json '{"port": 8080}'
# Output: {"status": "created", "id": "svc_abc"}

# Second run (same command): detects conflict, exits 5 — agent skips, not aborts
mycli service create --name my-svc --json '{"port": 8080}'
# Stderr: {"error": "conflict", "message": "Service 'my-svc' already exists", "id": "svc_abc"}
```

---

## Dry-Run Output Pattern

`--dry-run` must output structured JSON showing exactly what would happen:

```json
{
  "dry_run": true,
  "action": "deploy",
  "would_create": [
    {"type": "container", "image": "myapp:v1.2.3", "env": "staging"}
  ],
  "would_update": [],
  "would_delete": [],
  "estimated_duration_ms": 45000
}
```

---

## Pipeline and Stdin Patterns

```bash
# Output as input to next command
mycli build --output json | jq '.tag' | mycli deploy --env staging --tag -

# Explicit stdin
cat config.json | mycli config import --stdin

# Composable: use subcommand output in another flag
mycli deploy --env staging --tag $(mycli build --output tag-only)
```

---

## Multi-Surface Architecture

```
myapp/
├── cmd/           CLI entrypoint (human + agent)
├── mcp/           MCP stdio server (delegates to core)
├── core/          Business logic (pure functions, no I/O assumptions)
└── skills/        SKILL.md files for agent-specific guidance
```

---

## Anti-Patterns to Reject

| Anti-Pattern | Why It Fails for Agents |
|---|---|
| Default interactive pager (less/more) | Hangs agent indefinitely |
| ANSI color in stdout | Agent parses escape codes as data |
| `print()` mixed into JSON stdout | Corrupts structured parsing |
| Silent truncation of results | Agent thinks it got all data |
| Generic exit 1 for all errors | Agent can't distinguish retry vs. abort vs. skip |
| Blocking confirmation without `--yes` | Agent hangs indefinitely |
| `--help` with no examples | Agent can't infer usage patterns |
| Mutable output schema in minor versions | Breaks downstream agent parsing silently |
| `*args/**kwargs` in tool descriptions | Agent hallucinates parameter names |
| Unbounded file reads / grep without limits | Context window overflow |
