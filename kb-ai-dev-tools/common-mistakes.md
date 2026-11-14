# Common Mistakes: AI Developer Tools & Skills

> The mistakes that cost the most time, grouped by theme, each with why it happens and the fix. Version-specific items are **as of Nov 2026**.

---

## A. Mental-model mistakes

| # | Mistake | Why it happens | Fix | Lesson |
|---|---|---|---|---|
| A1 | "The model remembers our last session." | Chat UIs resend history, hiding statelessness | Assume amnesia; use instruction files, memory, explicit context | [01-beginner/01](01-beginner/01-llms-for-builders.md) |
| A2 | Estimating cost/size in words or characters | Tokens are invisible | Count tokens with the provider's counter, per model (tokenizers differ, ~30% more on Claude 4.7+) | [01-beginner/01](01-beginner/01-llms-for-builders.md) |
| A3 | Trusting confident output | Fluency comes from prediction, not verification | Validate, cite, test; ask for quotes from sources | [01-beginner/02](01-beginner/02-prompting-first-principles.md) |
| A4 | "Bigger context fixes everything" | 1M-token windows look limitless | Context rot is real; retrieve just-in-time, keep context small | [04-pro/03](04-pro/03-context-security-and-open-problems.md) |
| A5 | Assuming AI makes you faster | It feels faster | Measure; METR found 19% slower vs 20% perceived faster | [01-beginner/03](01-beginner/03-meet-your-ai-coding-tools.md) |

## B. Prompting mistakes

| # | Mistake | Fix | Lesson |
|---|---|---|---|
| B1 | No success criteria before prompting | Write checkable criteria first | [01-beginner/02](01-beginner/02-prompting-first-principles.md) |
| B2 | Rules without reasons; shouting (NEVER/CRITICAL) | Explain why; calm language (newer models overtrigger) | [01-beginner/02](01-beginner/02-prompting-first-principles.md) |
| B3 | Question first, long data after | Data first, question last | [01-beginner/02](01-beginner/02-prompting-first-principles.md) |
| B4 | Homogeneous few-shot examples | 3–5 diverse examples incl. edge cases | [02-intermediate/02](02-intermediate/02-prompt-techniques-and-chaining.md) |
| B5 | Copying 2024 tricks (prefill, temperature 0, `budget_tokens`, forced tool_choice) | Use structured outputs, `effort`, `auto` + strict tools; check migration guides | [02-intermediate/02](02-intermediate/02-prompt-techniques-and-chaining.md) |
| B6 | `textwrap.dedent` on f-strings that interpolate multi-line data | Build prompts from a list of sections and join | [01-beginner/02](01-beginner/02-prompting-first-principles.md) |

## C. Tool calling and structured output

| # | Mistake | Fix | Lesson |
|---|---|---|---|
| C1 | Unbounded agent loop | Turn cap; handle every `stop_reason` | [02-intermediate/01](02-intermediate/01-tool-calling-and-structured-output.md) |
| C2 | Text before `tool_result` blocks, or dropping the assistant tool_use turn | Echo full assistant content; results first | [02-intermediate/01](02-intermediate/01-tool-calling-and-structured-output.md) |
| C3 | Exceptions crash the loop | Return `is_error: true` with fix-it text | [02-intermediate/01](02-intermediate/01-tool-calling-and-structured-output.md) |
| C4 | Executing model-requested actions unchecked | Allowlists, path resolution, authorisation in code | [exercises I1](exercises.md) |
| C5 | Believing structured outputs = correct outputs | Still validate; schema can bias answers | [04-pro/03](04-pro/03-context-security-and-open-problems.md) |
| C6 | One-line tool descriptions | 3–4+ sentences: what, when, when not, args, returns | [02-intermediate/01](02-intermediate/01-tool-calling-and-structured-output.md) |

## D. MCP

| # | Mistake | Fix | Lesson |
|---|---|---|---|
| D1 | `print()` to stdout in a stdio server | Log to stderr | [02-intermediate/04](02-intermediate/04-build-an-mcp-server-in-python.md) |
| D2 | Following pre-2026 tutorials (`initialize`, sessions, sampling, roots, `FastMCP`) | Check the revision (2026-07-28) and SDK major version (v2 `MCPServer`) | [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md) |
| D3 | camelCase attributes in SDK v2 Python (`isError`) | snake_case (`is_error`) | [02-intermediate/04](02-intermediate/04-build-an-mcp-server-in-python.md) |
| D4 | Raising generic exceptions for user-fixable errors | `ToolError` with an instructive message | [02-intermediate/04](02-intermediate/04-build-an-mcp-server-in-python.md) |
| D5 | Hidden per-connection state | Explicit, opaque, expiring handles | [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md) |
| D6 | Installing unknown servers and auto-approving tools | Pin, review descriptions, keep approvals, isolate | [04-pro/03](04-pro/03-context-security-and-open-problems.md) |
| D7 | Relative paths and secrets in host configs | Absolute paths; env vars; local scope for secrets | [02-intermediate/04](02-intermediate/04-build-an-mcp-server-in-python.md) |

## E. Coding-agent configuration

| # | Mistake | Fix | Lesson |
|---|---|---|---|
| E1 | Giant CLAUDE.md / AGENTS.md | <200 lines; skills and scoped rules for the rest | [02-intermediate/05](02-intermediate/05-claude-code-memory-and-skills.md) |
| E2 | Typos in frontmatter keys (silently ignored) | Validate skills/subagents | [02-intermediate/05](02-intermediate/05-claude-code-memory-and-skills.md) |
| E3 | Prose for hard rules | Permissions, hooks, sandbox | [03-advanced/01](03-advanced/01-claude-code-subagents-hooks-permissions.md) |
| E4 | Hook exits 1 to block | Exit 2 or JSON deny | [03-advanced/01](03-advanced/01-claude-code-subagents-hooks-permissions.md) |
| E5 | Regex denylist as security boundary | OS sandbox; least privilege | [03-advanced/01](03-advanced/01-claude-code-subagents-hooks-permissions.md) |
| E6 | `.md` files in `.cursor/rules` | `.mdc` with frontmatter | [03-advanced/02](03-advanced/02-cursor-and-codex-workflows.md) |
| E7 | Forgotten `AGENTS.override.md` in Codex | Audit overrides; one file per directory | [03-advanced/02](03-advanced/02-cursor-and-codex-workflows.md) |
| E8 | `bypassPermissions` / `danger-full-access` on a laptop | Only in disposable containers/VMs | [03-advanced/01](03-advanced/01-claude-code-subagents-hooks-permissions.md) |
| E9 | Starting agents outside a git repo or in `~` | Clean git tree, project directory | [01-beginner/03](01-beginner/03-meet-your-ai-coding-tools.md) |

## F. RAG, agents, local models

| # | Mistake | Fix | Lesson |
|---|---|---|---|
| F1 | Only evaluating final answers | Measure recall@k/MRR separately | [03-advanced/03](03-advanced/03-rag-and-embeddings.md) |
| F2 | Non-deterministic indexing (salted `hash()`) | Stable hashes, fixed seeds | [03-advanced/03](03-advanced/03-rag-and-embeddings.md) |
| F3 | Chunks that lose their subject | Structure-aware or contextual chunks | [03-advanced/03](03-advanced/03-rag-and-embeddings.md) |
| F4 | Side effects before `interrupt()` | Separate node after approval | [03-advanced/04](03-advanced/04-agents-with-langgraph.md) |
| F5 | No checkpointer / wrong thread ID | Durable checkpointer, stable thread IDs | [03-advanced/04](03-advanced/04-agents-with-langgraph.md) |
| F6 | Ollama defaults (streaming, small `num_ctx`, temperature 0.8) | `stream: false` or NDJSON handling, set `num_ctx`, lower temperature | [03-advanced/05](03-advanced/05-local-models-with-ollama.md) |

## G. Evaluation and review

| # | Mistake | Fix | Lesson |
|---|---|---|---|
| G1 | One run per task | Multiple trials, rates, CIs | [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md) |
| G2 | pass@k for autonomous jobs | pass^k | [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md) |
| G3 | Unvalidated LLM judges | Calibrate vs humans; swap order | [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md) |
| G4 | Never reading transcripts | Read many; grader bugs hide there | [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md) |
| G5 | AI-written tests as the only oracle | Properties, references, spec values | [04-pro/02](04-pro/02-ai-assisted-engineering-discipline.md) |
| G6 | Installing suggested packages unchecked | Verify existence, publisher, lockfile | [04-pro/02](04-pro/02-ai-assisted-engineering-discipline.md) |
| G7 | Letting agents edit failing tests | Forbid in instructions and check in review | [04-pro/02](04-pro/02-ai-assisted-engineering-discipline.md) |

## H. Security

| # | Mistake | Fix | Lesson |
|---|---|---|---|
| H1 | "The prompt tells it to ignore injected instructions" | Code-level policy, not prompt-level | [04-pro/03](04-pro/03-context-security-and-open-problems.md) |
| H2 | One agent with all three trifecta legs | Split capabilities; approvals; egress control | [04-pro/03](04-pro/03-context-security-and-open-problems.md) |
| H3 | Trusting tool results or retrieved docs | Treat as untrusted data | [02-intermediate/01](02-intermediate/01-tool-calling-and-structured-output.md) |
| H4 | Exposing Ollama on all interfaces | Bind localhost or authenticate | [03-advanced/05](03-advanced/05-local-models-with-ollama.md) |

## Mnemonic (Tamil + English)

**"Ninaivu illai, Nambikkai illai, Alavu edu."** (நினைவு இல்லை, நம்பிக்கை இல்லை, அளவு எடு: no memory, no blind trust, take measurements.) Nearly every mistake above is one of: assuming the model remembers, trusting output or inputs without checks, or not measuring.

## Sources

- Each row links to the lesson where the mistake and its fix are explained with primary sources (Anthropic docs, Claude Code docs, MCP spec 2026-07-28, Cursor and Codex docs, LangGraph and Ollama docs, and the papers cited there).

## Verify yourself

1. For each section, reproduce one mistake on purpose with the lesson's code (e.g. `ValueError` instead of `ToolError`, `hash()` instead of `crc32`, exit 1 in a hook) and observe the symptom.
2. Re-check the version-specific rows (B5, D2, D3, E6, E7) against current docs.
