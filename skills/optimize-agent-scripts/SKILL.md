---
name: optimize-agent-scripts
description: Applies advanced best practices for optimizing scripts, CLI tools and Python codebases for AI agents. Focuses on context discipline, hallucination defense, and Agent-Computer Interface (ACI) predictability.
metadata:
  author: ai-engineering-team
  version: "3.0"
---

# Optimizing Scripts and Tools for Agentic Work

When writing, reviewing, or refactoring code that agents will execute or interact with, prioritize **predictability, context window discipline, and defense against hallucinations.**

## 1. Tool & Interface Design (ACI)

When writing scripts, CLIs, or MCP servers that serve as agent tools:

- **Raw JSON payloads over ergonomic flags:** Agents prefer `--json '{"title": "A", "gridProperties": {"frozenRowCount": 1}}'` over `--title "A" --frozen-rows 1`. Zero translation loss between intent and API.
- **Dynamic introspection over static docs:** Expose tool schemas via a `schema` command (e.g., `cli schema <method_name>`). Agents query exact signatures at runtime instead of burning tokens on pre-loaded docs.
- **Context discipline:** Never return unbounded output. Force field masks (`?fields=id,name`) and stream long lists as NDJSON. Agents process one object at a time.
- **Tool description quality (highest-ROI single change):** State exactly what each tool _returns_ (not just what it does), use enums for constrained fields, and document result limits in the description itself. Consistent naming, atomic actions, and typed parameters can produce a 10x reduction in tool failures with no architecture changes.

## 2. CLI Output Contract

- **stdout** = data only; **stderr** = errors/progress/color; **exit 0** = stdout trustworthy
- Semantic exit codes (0=success, 1=retry, 2=bad-args, 3=not-found, 4=denied, 5=conflict-skip) — agents branch on these. Full table + `--help` template in `references/cli-standards.md`.
- Every command needs `--output json` (flat JSON) and `--quiet` (~40% fewer tokens).

## 3. --help Optimized for Agents

`--help` is simultaneously the tool description, parameter spec, and usage guide. Required sections: **Examples** with real invocations (agents learn from these faster than prose), **Exit codes**, **Environment** vars with precedence (`flags > env > config > defaults`), required vs. optional labels. Template in `references/cli-standards.md`.

## 4. Non-Interactive Execution

Agents cannot respond to prompts, navigate pagers, or interpret ANSI color codes:

- `--yes` / `--force` / `--no-interactive` bypass all confirmations; auto-detect non-TTY (`isatty()`)
- Honor `NO_COLOR`; never emit ANSI to non-TTY by default
- `--dry-run` must produce structured JSON — agents invoke this before any destructive operation
- Missing required flags → exit 2 immediately with a correct example invocation, never hang

## 5. Idempotency

Agents retry often. Same command twice must be safe: no-op or exit 5 (skip) on duplicate creates — not exit 1 (abort). Non-idempotent destructive ops require `--dry-run` preview then `--yes`. Examples in `references/cli-standards.md`.

## 6. Input Hardening (Hallucination Defense)

Humans typo; agents hallucinate. Harden scripts against specific LLM failure modes:

- **Path traversal:** Sandbox all file ops to CWD; reject `..` in paths
- **Resource ID corruption:** Reject `?` and `#` in IDs (agents embed query params inside them)
- **Double-encoding:** Reject `%` in raw path segments (agents pre-encode strings)
- **Control characters:** Reject anything below ASCII 0x20 in critical variables

Code patterns in `references/python-standards.md`. Echo back failing input in every error response (agents self-correct without guessing).

## 7. Poka-Yoke (Mistake-Proofing) Patterns

One constraint can eliminate an entire failure class. From SWE-agent empirical research (same model, 2x benchmark improvement with ACI-optimized tools vs. raw bash):

| Constraint | Failure class eliminated |
|---|---|
| Require absolute paths for file ops | Directory-change errors |
| Windowed file viewer with overlap + position indicator ("X more lines below") | Context window overflow |
| Lint/syntax validate before persisting edits | Cascading failures from invalid edits |
| Cap search at 100 results, refuse and ask to narrow | Silent truncation confusion |
| Explicit empty-state messages ("no matches found in src/") | Agent interpreting silence as error |

## 8. Command Structure & Success Output

Use consistent **noun-verb** hierarchy everywhere (`mycli service list`, `mycli service create`, `mycli service delete`) — enables deterministic exploration. If one resource group follows the pattern, all must. On success, return machine-useful data (IDs, URLs, durations), not decorative prose. Formats and examples in `references/cli-standards.md`.

## 9. MCP vs CLI — Abstraction Tax Awareness

Every abstraction layer between an agent and the underlying API introduces fidelity loss. Each wrapping layer in a Data → API → MCP chain compounds that loss:

| Approach | Fidelity | Context Cost | Best For |
|---|---|---|---|
| MCP (constrained tools) | Lower | Low upfront | Simple, stable operations |
| MCP (full surface) | High in theory | Context explosion | Not recommended |
| CLI + Skills | High | On-demand via `--help` / `schema` | Complex enterprise APIs |
| Raw API + client lib | Maximum | Agent manages schema | Power users |

For CLIs with many commands: don't load all commands upfront. Use on-demand discovery. Agents pay context costs only when relevant. Test with agents — they reveal different failure modes than human testing.

---

> **Deep dives:**
> - Python code patterns: `references/python-standards.md`
> - CLI design patterns with examples: `references/cli-standards.md`
