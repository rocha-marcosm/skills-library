# Python Code Standards for Agentic Work

When writing Python code that agents interact with, optimize for machine-readability, predictability, and context window discipline.

## 1. Type Safety and Schemas

Agents rely on types to understand how to interact with your codebase.

- **Strict Pydantic models:** Use Pydantic to validate all inputs and outputs between agent nodes.
- **Comprehensive type hints:** Every function must have complete type hints: `def process(data: Dict[str, Any]) -> List[str]`.
- **Avoid `*args` and `**kwargs`:** These create black boxes. Use explicit parameter definitions so agents can predict call signatures.

## 2. Output Discipline (stdout/stderr Contract)

The stdout/stderr separation is critical — agents parse stdout as data and stderr as diagnostics:

```python
import sys, json, os

def emit(data: dict, json_mode: bool):
    if json_mode:
        print(json.dumps(data), file=sys.stdout)
    else:
        print(data.get("message", ""), file=sys.stdout)

def fail(code: int, error_type: str, message: str, context: dict | None = None):
    payload = {"error": error_type, "message": message, **(context or {})}
    # Echo back failing input in context so agents can self-correct
    print(json.dumps(payload), file=sys.stderr)
    sys.exit(code)
```

Never mix debug prints into stdout when `--output json` is active. All progress/spinner output goes to stderr.

## 3. Non-Interactive Detection

Auto-detect non-TTY contexts and disable interactive features:

```python
import sys, os

def is_interactive(args) -> bool:
    return (
        sys.stdin.isatty()
        and not getattr(args, "no_interactive", False)
        and not os.getenv("MYTOOL_NO_INTERACTIVE")
    )
```

Always provide `--yes` / `--no-interactive` flags. Missing required inputs → fail fast with exit 2 and a correct example invocation, never hang.

## 4. Explicit State and Error Handling

Agents use exceptions as their feedback loop.

- **Semantic exit codes:** Use the taxonomy from `references/cli-standards.md`. Key: exit 5 = conflict/skip (not exit 1), exit 1 = transient/retry. Never use generic exit 1 for all errors.
- **Semantic exceptions:** Never raise generic `Exception`. Raise explicit classes with detailed messages: `raise MissingKeyError("Column 'user_id' not found in chunk — available: ['id', 'email']")`.
- **Process guards:** If a script starts a server, check for active PIDs and fail explicitly (`"Port 8000 in use by PID 12345"`) rather than hanging.

## 5. Context Discipline (Bounded Output)

Agents have finite context windows. Never read or return unbounded data:

```python
MAX_RESULTS = 100
MAX_FILE_LINES = 200

def read_file_window(path: str, offset: int = 0) -> dict:
    with open(path) as f:
        lines = f.readlines()
    total = len(lines)
    window = lines[offset:offset + MAX_FILE_LINES]
    return {
        "content": "".join(window),
        "offset": offset,
        "total_lines": total,
        "has_more": (offset + MAX_FILE_LINES) < total,
        "next_offset": offset + MAX_FILE_LINES
    }

def list_resources(query: str) -> dict:
    results = db.search(query, limit=MAX_RESULTS + 1)
    truncated = len(results) > MAX_RESULTS
    return {
        "items": results[:MAX_RESULTS],
        "truncated": truncated,
        "message": f"Showing {min(len(results), MAX_RESULTS)} results. Narrow your query." if truncated else None
    }
```

Return pagination tokens instead of dumping all records. Show "X more lines below" in file viewers. Explicit empty-state messages ("no matches found in src/") prevent agents from interpreting silence as an error.

## 6. Hallucination Defense Patterns

### Hard guardrails (framework-level, cannot be bypassed by the LLM)

Validate inputs before any destructive operation:

```python
import re
from pathlib import Path

BLOCKED_PATTERNS = [
    r'\.\.',           # path traversal
    r'[?#]',           # embedded query params in IDs
    r'%[0-9a-fA-F]{2}', # pre-encoded strings (double-encode risk)
]

def validate_resource_id(resource_id: str) -> None:
    for pattern in BLOCKED_PATTERNS:
        if re.search(pattern, resource_id):
            raise ValueError(f"Invalid resource ID '{resource_id}': matched blocked pattern '{pattern}'")

def validate_path(path: str) -> Path:
    resolved = Path(path).resolve()
    cwd = Path.cwd().resolve()
    if not str(resolved).startswith(str(cwd)):
        raise PermissionError(f"Path '{path}' escapes working directory '{cwd}'")
    return resolved
```

### Soft steering (agent self-corrects rather than failing)

Return recovery hints in errors so agents can fix themselves:

```python
def fail_with_hint(code: int, message: str, hint: str, example: str | None = None):
    payload = {"error": message, "hint": hint}
    if example:
        payload["example"] = example
    print(json.dumps(payload), file=sys.stderr)
    sys.exit(code)

# Usage:
fail_with_hint(
    code=2,
    message="No image tag specified",
    hint="Use --tag to specify a version",
    example="mycli deploy --env staging --tag v1.2.3"
)
```

### Grounded retrieval (replace hallucinated data with real queries)

Never let an agent invent values for things that should come from a real source. Tool implementations must query actual data sources and return "no results" rather than letting the LLM fill in values:

```python
def get_available_tags() -> list[str]:
    # Query the actual registry — never return hardcoded/fabricated values
    return registry_client.list_tags()
```

## 7. Explicit ORMs vs. Raw SQL

Agents are heavily trained on raw SQL but frequently hallucinate when using highly abstracted custom ORM logic. Use raw SQL execution tools or standard, uncustomized ORMs (SQLAlchemy, Prisma). Custom query builders with unusual chaining syntax are high-hallucination surfaces.
