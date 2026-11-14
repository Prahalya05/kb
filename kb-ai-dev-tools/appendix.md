# Appendix: Cheat Sheets, References, Tools, Books, Courses, Papers

> Product facts are **as of Nov 2026**. Re-check before use; links are to primary sources.

---

## A. Cheat sheets

### A.1 Anthropic Messages API (Python SDK 2.8.0, Nov 2026)

```python
# Not run here (needs ANTHROPIC_API_KEY). Parameter names checked against anthropic 2.8.0.
import anthropic
client = anthropic.Anthropic()                        # reads ANTHROPIC_API_KEY
schema = {"type": "object", "properties": {"label": {"type": "string"}},
          "required": ["label"], "additionalProperties": False}
tool = {"name": "disk_usage",
        "description": "Report free bytes for the filesystem containing `path`. Read-only. "
                       "Use for disk-space questions. Returns a short string.",
        "input_schema": {"type": "object", "properties": {"path": {"type": "string"}},
                         "required": ["path"], "additionalProperties": False},
        "strict": True}
msgs = [{"role": "user", "content": "Classify this ticket: 'charged twice'"}]
resp = client.messages.create(
    model="claude-opus-5-5", max_tokens=1024,
    system="You are a support triage assistant.",
    messages=msgs,
    tools=[tool],
    tool_choice={"type": "auto"},                     # 'any'/'tool' rejected on several 5.x models
    output_config={"effort": "medium",                # thinking depth on current models
                   "format": {"type": "json_schema", "schema": schema}},
)
print(resp.stop_reason)   # end_turn | tool_use | max_tokens | stop_sequence | pause_turn | refusal | model_context_window_exceeded
print(resp.usage)         # input_tokens, output_tokens, cache_* fields
print(client.messages.count_tokens(model="claude-opus-5-5", messages=msgs))   # free estimate, per-model tokenizer
```

Do not send (current models, Nov 2026): a trailing assistant prefill (400 on 4.6+), `temperature`/`top_p`/`top_k` (omit on Opus 5.5), `thinking: {"type": "enabled", "budget_tokens": N}` (400 on 4.7+).

### A.2 Tool loop invariants

1. Cap turns. 2. Echo the full assistant content. 3. One `tool_result` per `tool_use`, same id. 4. `tool_result` blocks first in the user message. 5. Errors as `is_error: true` with fix-it text. 6. Validate inputs before executing; treat outputs as untrusted.

### A.3 MCP 2026-07-28 at a glance

| Item | Value |
|---|---|
| Wire | JSON-RPC 2.0; `resultType` required on results |
| Per-request `_meta` | `io.modelcontextprotocol/protocolVersion`, `.../clientCapabilities` (required), `.../clientInfo` (should) |
| Discovery | `server/discover` (servers MUST implement) |
| Server primitives | tools (`tools/list`, `tools/call`), resources (`resources/list`, `resources/read`), prompts (`prompts/list`, `prompts/get`) |
| Client feature | elicitation (via MRTR `input_required`) |
| Deprecated | roots, sampling, logging, HTTP+SSE transport |
| Removed | `initialize` handshake, sessions/`Mcp-Session-Id`, `ping`, `logging/setLevel` |
| Transports | stdio (newline-delimited JSON; logs to stderr), Streamable HTTP |
| Spec error codes | `-32020` HeaderMismatch, `-32021` MissingRequiredClientCapability, `-32022` UnsupportedProtocolVersion; resource not found → `-32602` |

### A.4 MCP Python SDK v2 (2.3.0)

```python
# mini_server.py - smallest complete MCPServer (SDK v2.3.0) plus an in-process client check.
import asyncio
from mcp import Client
from mcp.server.mcpserver import MCPServer                 # was FastMCP in v1
from mcp.server.mcpserver.exceptions import ToolError      # anticipated failure -> model sees message

mcp = MCPServer("mini")


@mcp.tool()                                                # docstring -> description, hints -> inputSchema
def add(a: int, b: int) -> int:
    """Add two integers. Use for arithmetic the user asks about. Rejects results above 10**9."""
    if a + b > 10**9:
        raise ToolError("result too large; keep a + b at or below 10**9")
    return a + b


@mcp.resource("greeting://{name}")                         # resource template, read by URI
def greeting(name: str) -> str:
    return f"Hello, {name}!"


@mcp.prompt()                                              # user-invocable prompt template
def explain(topic: str) -> str:
    return f"Explain {topic} in three sentences."


async def check() -> None:
    async with Client(mcp) as c:                           # in-process connection, no subprocess
        r = await c.call_tool("add", {"a": 2, "b": 3})
        print(r.structured_content, r.is_error)            # snake_case attributes in v2


if __name__ == "__main__":
    asyncio.run(check())                                   # for real use: mcp.run(transport="stdio")
```

Output: `{'result': 5} False`. Swap `asyncio.run(check())` for `mcp.run(transport="stdio")` (or `transport="streamable-http", host="127.0.0.1", port=9000`) to serve it.

### A.5 Registering an MCP server

| Host | How |
|---|---|
| Claude Code | `claude mcp add [--scope local\|project\|user] [--env K=V] --transport stdio NAME -- CMD ARGS`; `--transport http NAME URL`; project scope → `.mcp.json` |
| Cursor | `.cursor/mcp.json` or `~/.cursor/mcp.json`: `{"mcpServers": {"name": {"command", "args", "env"} or {"url"}}}` |
| Codex | `~/.codex/config.toml`: `[mcp_servers.name]` with `command`/`args`/`env` or `url`/`bearer_token_env_var` |

### A.6 Instruction files

| Tool | Files | Combination rule |
|---|---|---|
| Claude Code | managed, `~/.claude/CLAUDE.md`, `./CLAUDE.md` or `./.claude/CLAUDE.md`, `CLAUDE.local.md`, `.claude/rules/*.md` (`paths:`) | Concatenate root→cwd; `@import` up to 4 hops; <200 lines |
| Cursor | `.cursor/rules/*.mdc` (`description`, `globs`, `alwaysApply`), `AGENTS.md` (nested) | Per-rule triggers; more specific nested AGENTS.md wins; <500 lines per rule |
| Codex | `~/.codex/AGENTS(.override).md`, `AGENTS.override.md` > `AGENTS.md` > fallbacks per dir | One file per dir, root→cwd, later overrides; 32 KiB budget |

### A.7 Claude Code control surface

| Need | Mechanism |
|---|---|
| Always-on facts | CLAUDE.md |
| Scoped conventions | `.claude/rules/` with `paths:` |
| Procedures | Skills (`.claude/skills/<name>/SKILL.md`; `disable-model-invocation`, `context: fork`, `allowed-tools`) |
| Isolated work | Subagents (`.claude/agents/*.md`: `name`, `description`, `tools`, `model`) |
| Deterministic enforcement/automation | Hooks (`PreToolUse` etc.; exit 2 blocks) |
| Access policy | `permissions.allow/ask/deny` (deny→ask→allow), modes, sandbox |
| Headless | `claude -p "summarise the failing tests"` |

### A.8 Codex quick reference

`npm install -g @openai/codex` · `codex` (interactive) · `codex exec "task"` (read-only by default) · `--sandbox workspace-write|danger-full-access` · `--json` (JSONL events) · `--output-schema file.json` · `codex exec resume --last` · config `~/.codex/config.toml` (`approval_policy = "on-request"|"never"`, `sandbox_mode`) · skills in `.agents/skills/`.

### A.9 Ollama quick reference

`ollama pull M` · `ollama run M` · `ollama ps` · `ollama create NAME -f Modelfile` · API `http://localhost:11434/api/chat` (`stream` defaults true; `format`; `options.num_ctx`, `options.temperature`; `tools`) · `/api/embed` (unit vectors) · OpenAI-compatible `/v1` · Anthropic-compatible base `http://localhost:11434`.

### A.10 Eval formulas

- pass@k = 1 − C(n−c, k) / C(n, k)
- pass^k = C(c, k) / C(n, k)
- RRF(d) = Σ_i 1 / (60 + rank_i(d))
- cosine(a, b) = a·b / (‖a‖‖b‖)

## B. Tools

| Tool | What it is | Link |
|---|---|---|
| Claude Code | Anthropic's agentic coding tool (CLI, IDE, desktop, web) | https://code.claude.com/docs/ |
| Cursor | AI code editor with Agent, rules, MCP, CLI | https://cursor.com/docs |
| Codex CLI | OpenAI's local coding agent (Apache-2.0) | https://github.com/openai/codex |
| MCP Inspector | Interactive MCP server debugger (`npx @modelcontextprotocol/inspector`) | https://github.com/modelcontextprotocol/inspector |
| MCP Python SDK | Official SDK (`pip install mcp`) | https://github.com/modelcontextprotocol/python-sdk |
| skills-ref | Agent Skills validator | https://github.com/agentskills/agentskills/tree/main/skills-ref |
| LangGraph | Stateful agent orchestration | https://github.com/langchain-ai/langgraph |
| Ollama | Local model runtime | https://github.com/ollama/ollama |
| Hypothesis | Property-based testing for Python | https://hypothesis.readthedocs.io/ |
| Pydantic | Validation and JSON Schema generation | https://docs.pydantic.dev/ |

## C. Official documentation (primary sources)

- Anthropic platform docs: https://platform.claude.com/docs/ (prompt engineering, tool use, structured outputs, context windows, token counting, stop reasons, evals)
- Claude Code docs: https://code.claude.com/docs/ (overview, memory, skills, sub-agents, hooks, permissions, MCP)
- MCP specification: https://modelcontextprotocol.io/specification/latest and changelog https://modelcontextprotocol.io/specification/2026-07-28/changelog
- Agent Skills specification: https://agentskills.io/specification
- AGENTS.md: https://agents.md/
- Cursor docs: https://cursor.com/docs/context/rules, https://cursor.com/docs/context/mcp, https://cursor.com/docs/cli/overview
- Codex docs: https://learn.chatgpt.com/codex/agent-configuration/agents-md, https://learn.chatgpt.com/codex/sandboxing, https://learn.chatgpt.com/codex/extend/mcp, https://learn.chatgpt.com/codex/skills
- LangGraph docs: https://docs.langchain.com/oss/python/langgraph/overview
- Ollama docs: https://docs.ollama.com/

## D. Engineering essays worth rereading

- Anthropic, "Building effective agents" (Dec 19, 2024): https://www.anthropic.com/engineering/building-effective-agents
- Anthropic, "Writing effective tools for agents" (Sept 11, 2025): https://www.anthropic.com/engineering/writing-tools-for-agents
- Anthropic, "Effective context engineering for AI agents" (Sept 29, 2025): https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic, "Demystifying evals for AI agents" (Jan 9, 2026): https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- Anthropic, "Introducing Contextual Retrieval" (Sept 19, 2024): https://www.anthropic.com/news/contextual-retrieval
- Simon Willison, "The lethal trifecta for AI agents" (June 16, 2025): https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
- Simon Willison, "The Dual LLM pattern" (Apr 25, 2023): https://simonwillison.net/2023/Apr/25/dual-llm-pattern/
- Invariant Labs, "MCP Security Notification: Tool Poisoning Attacks" (Apr 1, 2025): https://invariantlabs.ai/blog/mcp-security-notification

## E. Books

- Chip Huyen, *AI Engineering: Building Applications with Foundation Models* (O'Reilly, Dec 2024): https://www.oreilly.com/library/view/ai-engineering/9781098166298/
- Manning, Raghavan, Schütze, *Introduction to Information Retrieval* (Cambridge, 2008; free online): https://nlp.stanford.edu/IR-book/
- Jurafsky & Martin, *Speech and Language Processing* (3rd ed. draft, free online): https://web.stanford.edu/~jurafsky/slp3/

## F. Courses and tutorials

- Stanford CS146S, The Modern Software Developer: https://themodernsoftware.dev/ (bulletin: https://bulletin.stanford.edu/courses/2274401)
- Anthropic, interactive prompt engineering tutorial: https://github.com/anthropics/prompt-eng-interactive-tutorial
- Claude Academy (linked from the Claude Code docs; includes "Claude Code 101"): https://academy.claude.com/

## G. Papers (chronological)

| Year | Paper | Why read it | Link |
|---|---|---|---|
| 2016 | Sennrich et al., Subword units (BPE) | Tokenization | https://arxiv.org/abs/1508.07909 |
| 2016 | Malkov & Yashunin, HNSW | Vector search at scale | https://arxiv.org/abs/1603.09320 |
| 2017 | Vaswani et al., Attention Is All You Need | The architecture | https://arxiv.org/abs/1706.03762 |
| 2019 | Holtzman et al., Neural Text Degeneration | Why nucleus sampling | https://arxiv.org/abs/1904.09751 |
| 2019 | Reimers & Gurevych, Sentence-BERT | Sentence embeddings | https://arxiv.org/abs/1908.10084 |
| 2020 | Brown et al., GPT-3 | Few-shot in-context learning | https://arxiv.org/abs/2005.14165 |
| 2020 | Lewis et al., RAG | Retrieval-augmented generation | https://arxiv.org/abs/2005.11401 |
| 2021 | Chen et al., Codex/HumanEval | Unbiased pass@k | https://arxiv.org/abs/2107.03374 |
| 2021 | Pearce et al., Asleep at the Keyboard | Security of AI code | https://arxiv.org/abs/2108.09293 |
| 2022 | Wei et al., Chain-of-thought | Reasoning prompts | https://arxiv.org/abs/2201.11903 |
| 2022 | Wang et al., Self-consistency | Voting | https://arxiv.org/abs/2203.11171 |
| 2022 | Kojima et al., Zero-shot reasoners | "Let's think step by step" | https://arxiv.org/abs/2205.11916 |
| 2022 | Yao et al., ReAct | Reason + act loops | https://arxiv.org/abs/2210.03629 |
| 2023 | Schick et al., Toolformer | Learning to use tools | https://arxiv.org/abs/2302.04761 |
| 2023 | Greshake et al., Indirect prompt injection | Agent security | https://arxiv.org/abs/2302.12173 |
| 2023 | Zheng et al., LLM-as-a-Judge | Judge biases | https://arxiv.org/abs/2306.05685 |
| 2023 | Liu et al., Lost in the Middle | Long-context use | https://arxiv.org/abs/2307.03172 |
| 2023 | Willard & Louf, Guided generation | Constrained decoding | https://arxiv.org/abs/2307.09702 |
| 2023 | Jimenez et al., SWE-bench | Coding-agent benchmark | https://arxiv.org/abs/2310.06770 |
| 2024 | Yang et al., SWE-agent | Agent-computer interfaces | https://arxiv.org/abs/2405.15793 |
| 2024/25 | Spracklen et al., Package hallucinations | Slopsquatting risk | https://arxiv.org/abs/2406.10279 |
| 2025 | Debenedetti et al., CaMeL | Injection defence by design | https://arxiv.org/abs/2503.18813 |
| 2025 | Becker et al. (METR), Developer productivity RCT | Measure, don't assume | https://arxiv.org/abs/2507.09089 |

Classic non-arXiv references: Robertson & Zaragoza, BM25 (2009), https://dl.acm.org/doi/10.1561/1500000019 · Cormack et al., RRF (SIGIR 2009), https://dl.acm.org/doi/10.1145/1571941.1572114 · Claessen & Hughes, QuickCheck (ICFP 2000), https://dl.acm.org/doi/10.1145/351240.351266 · Zeller & Hildebrandt, Delta debugging (TSE 2002), https://doi.org/10.1109/32.988498.

## Sources

All links above are the sources; each was consulted while writing this track (Nov 2026). Vendor pages change; the "as of Nov 2026" stamps mark what they said then.

## Verify yourself

1. Validate the JSON/TOML snippets from the cheat sheets you copy: `python3 -c "import json,sys; json.load(open(sys.argv[1]))" .mcp.json` and `python3 -c "import tomllib,sys; tomllib.load(open(sys.argv[1],'rb'))" ~/.codex/config.toml`.
2. For the SDK cheat sheets, run `python3 -c "import mcp, anthropic; print(mcp.__name__, anthropic.__version__)"` and compare versions; if yours differ, re-check the attribute names.
3. Spot-check three arXiv IDs from the paper table by opening them.
