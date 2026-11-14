# Glossary: AI Developer Tools & Skills (A–Z)

> Each entry: **plain meaning** · *precise meaning* · lesson where it is taught. Tool-specific entries are **as of Nov 2026**.

---

**Adaptive thinking**: the model decides how much to reason before answering. · *On Claude 4.6+ models, `thinking: {type: "adaptive"}`; always on for Claude Opus 5.5, where depth is controlled only by `effort`.* · [02-intermediate/02](02-intermediate/02-prompt-techniques-and-chaining.md)

**Agent**: an AI that acts in a loop, not just answers. · *An LLM that dynamically directs its own process and tool usage until a task is done ("Building effective agents").* · [01-beginner/03](01-beginner/03-meet-your-ai-coding-tools.md)

**Agent harness / scaffold**: the program around the model. · *"The system that enables a model to act as an agent": context assembly, tools, execution, permissions.* · [01-beginner/03](01-beginner/03-meet-your-ai-coding-tools.md), [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md)

**Agent Skills (standard)**: a portable format for agent procedures. · *A directory with `SKILL.md` (YAML frontmatter `name`, `description` + Markdown body) plus optional `scripts/`, `references/`, `assets/`; spec at agentskills.io; supported by Claude Code and Codex.* · [02-intermediate/05](02-intermediate/05-claude-code-memory-and-skills.md)

**AGENTS.md**: a README for coding agents. · *Plain-Markdown instruction file read natively by Codex, Cursor and others; stewarded by the Agentic AI Foundation; Claude Code imports it via `@AGENTS.md` in CLAUDE.md.* · [03-advanced/02](03-advanced/02-cursor-and-codex-workflows.md)

**Agentic AI Foundation (AAIF)**: the neutral home of MCP. · *"A directed fund under the Linux Foundation" (Dec 9, 2025) with founding projects MCP, goose and AGENTS.md.* · [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md)

**Autoregressive generation**: write one token, append, repeat. · *Sampling t_{n+1} from p(· | t_1…t_n) iteratively.* · [01-beginner/01](01-beginner/01-llms-for-builders.md)

**BM25**: keyword ranking with sensible weighting. · *Lexical scoring combining IDF, saturating term frequency (k1) and length normalisation (b).* · [03-advanced/03](03-advanced/03-rag-and-embeddings.md)

**BPE (byte-pair encoding)**: how text is chopped into tokens. · *Learn merges of the most frequent adjacent symbol pairs; apply merges in learned order to tokenize.* · [01-beginner/01](01-beginner/01-llms-for-builders.md)

**Chain-of-thought (CoT)**: show the working before the answer. · *Prompting for intermediate reasoning steps (Wei et al., 2022); on current frontier APIs largely replaced by built-in thinking.* · [02-intermediate/02](02-intermediate/02-prompt-techniques-and-chaining.md)

**Checkpoint (Cursor)**: undo point for agent edits. · *"Restore Checkpoint" rolls back file changes made after a message.* · [03-advanced/02](03-advanced/02-cursor-and-codex-workflows.md)

**Checkpointer (LangGraph)**: saves graph state after each step. · *Persistence layer keyed by `thread_id`; required for interrupts.* · [03-advanced/04](03-advanced/04-agents-with-langgraph.md)

**CLAUDE.md**: Claude Code's instruction file. · *Memory files at managed, user (`~/.claude/CLAUDE.md`), project (`./CLAUDE.md`) and local (`CLAUDE.local.md`) scopes, concatenated root→cwd; target under 200 lines.* · [02-intermediate/05](02-intermediate/05-claude-code-memory-and-skills.md)

**Compaction**: summarise old conversation to free space. · *Summarising a context near its limit and continuing from the summary; Anthropic offers server-side compaction in beta for Claude 4.6+.* · [04-pro/03](04-pro/03-context-security-and-open-problems.md)

**Constrained decoding**: generation that cannot break the format. · *Masking tokens that would make the output an invalid prefix under a grammar/schema at each step.* · [04-pro/03](04-pro/03-context-security-and-open-problems.md)

**Context engineering**: deciding what the model sees. · *"Strategies for curating and maintaining the optimal set of tokens … during LLM inference" (Anthropic, 2025).* · [04-pro/03](04-pro/03-context-security-and-open-problems.md)

**Context rot**: more context, worse recall. · *"As token count grows, accuracy and recall degrade" (Anthropic docs).* · [01-beginner/01](01-beginner/01-llms-for-builders.md)

**Context window**: what the model can see at once. · *Max tokens of input plus output for one request; 1M tokens on current Claude models.* · [01-beginner/01](01-beginner/01-llms-for-builders.md)

**Cosine similarity**: how aligned two vectors are. · *a·b/(‖a‖‖b‖); equals the dot product for unit vectors.* · [03-advanced/03](03-advanced/03-rag-and-embeddings.md)

**Differential testing**: compare two implementations. · *Same inputs to two programs; disagreements reveal bugs (McKeeman, 1998).* · [04-pro/02](04-pro/02-ai-assisted-engineering-discipline.md)

**Dual LLM pattern**: split trusted and untrusted processing. · *A privileged LLM with tools sees only trusted input; a quarantined LLM without tools handles untrusted content (Willison, 2023).* · [04-pro/03](04-pro/03-context-security-and-open-problems.md)

**Effort**: how hard the model should think. · *Request parameter (`output_config.effort`: low…max) controlling thinking depth on current Claude models.* · [02-intermediate/02](02-intermediate/02-prompt-techniques-and-chaining.md)

**Elicitation (MCP)**: a server asks the user for input. · *Client feature; in 2026-07-28 delivered via the multi round-trip request pattern.* · [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md)

**Embedding**: text as a vector of numbers. · *A learned map from text to fixed-length vectors where nearby means similar.* · [03-advanced/03](03-advanced/03-rag-and-embeddings.md)

**Eval / eval suite**: tests for AI behaviour. · *A set of tasks with graders, run over multiple trials, reporting rates and uncertainty.* · [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md)

**Few-shot prompting**: teach by examples in the prompt. · *Including input→output demonstrations so the task is inferred in-context.* · [02-intermediate/02](02-intermediate/02-prompt-techniques-and-chaining.md)

**Grader**: the thing that scores an output. · *"Logic that scores some aspect of the agent's performance": code-based, model-based or human.* · [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md)

**Hook (Claude Code)**: code that runs at agent lifecycle events. · *Handlers (command/http/mcp_tool/prompt/agent) on events like `PreToolUse`; exit code 2 blocks.* · [03-advanced/01](03-advanced/01-claude-code-subagents-hooks-permissions.md)

**Host / client / server (MCP)**: the three MCP roles. · *Host = AI application; client = connector inside it (one per server); server = provider of tools/resources/prompts.* · [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md)

**Hybrid retrieval**: keyword + semantic search together. · *Combining sparse and dense rankings, typically via RRF.* · [03-advanced/03](03-advanced/03-rag-and-embeddings.md)

**Indirect prompt injection**: hostile instructions hidden in data. · *Adversarial text in web pages/emails/tool results that the model follows (Greshake et al., 2023).* · [04-pro/03](04-pro/03-context-security-and-open-problems.md)

**Interrupt (LangGraph)**: pause for a human. · *`interrupt(value)` pauses a node, persists state; `Command(resume=...)` supplies its return value; the node re-runs from the top.* · [03-advanced/04](03-advanced/04-agents-with-langgraph.md)

**JSON-RPC 2.0**: MCP's message format. · *Requests with ids, responses matched by id, notifications without ids, standard error codes.* · [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md)

**Lethal trifecta**: the recipe for data exfiltration. · *Private data + untrusted content + external communication in one agent (Willison, 2025).* · [04-pro/03](04-pro/03-context-security-and-open-problems.md)

**LLM-as-judge**: a model grading outputs. · *Model-based grader with a rubric; must be calibrated; has position/verbosity/self-enhancement biases.* · [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md)

**Logits**: raw next-token scores. · *Unnormalised vector over the vocabulary; softmax turns it into probabilities.* · [01-beginner/01](01-beginner/01-llms-for-builders.md)

**MCP (Model Context Protocol)**: a standard plug for AI tools. · *Open JSON-RPC protocol connecting AI applications to tools, resources and prompts; latest revision 2026-07-28.* · [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md)

**MCPServer**: the Python SDK v2 server class. · *Formerly `FastMCP`; decorators `@mcp.tool()`, `@mcp.resource()`, `@mcp.prompt()`.* · [02-intermediate/04](02-intermediate/04-build-an-mcp-server-in-python.md)

**Modelfile (Ollama)**: a recipe for a local model variant. · *Instructions `FROM`, `PARAMETER`, `SYSTEM`, `TEMPLATE`, …; built with `ollama create`.* · [03-advanced/05](03-advanced/05-local-models-with-ollama.md)

**MRTR (multi round-trip requests)**: how MCP servers ask for more input. · *Server returns `resultType: "input_required"` with `inputRequests`; client retries the original request with `inputResponses`.* · [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md)

**Nucleus (top-p) sampling**: sample from the likely core. · *Keep the smallest token set whose probability sum ≥ p, renormalise (Holtzman et al., 2019).* · [01-beginner/01](01-beginner/01-llms-for-builders.md)

**Oracle**: whatever decides if an answer is right. · *Expected value, reference implementation, or property used to judge outputs.* · [04-pro/02](04-pro/02-ai-assisted-engineering-discipline.md)

**pass@k**: did it work at least once in k tries? · *Unbiased estimator 1 − C(n−c,k)/C(n,k) (Chen et al., 2021).* · [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md)

**pass^k**: did it work every time in k tries? · *Probability all k trials succeed; estimator C(c,k)/C(n,k).* · [04-pro/01](04-pro/01-evals-for-prompts-and-agents.md)

**Permission mode (Claude Code)**: how much the agent may do unasked. · *`default` (Manual), `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`; rules evaluated deny → ask → allow.* · [03-advanced/01](03-advanced/01-claude-code-subagents-hooks-permissions.md)

**Prefill**: starting the assistant's reply for it. · *Ending `messages` with a partial assistant turn; returns 400 on Claude 4.6+ models.* · [01-beginner/02](01-beginner/02-prompting-first-principles.md)

**Progressive disclosure**: load detail only when needed. · *Skill metadata (~100 tokens) always, body (<5000 tokens recommended) on activation, resources on demand.* · [02-intermediate/05](02-intermediate/05-claude-code-memory-and-skills.md)

**Prompt chaining**: one job per call, in sequence. · *Decomposing a task into sequential calls with inspectable intermediate outputs.* · [02-intermediate/02](02-intermediate/02-prompt-techniques-and-chaining.md)

**Property-based testing**: test rules, not examples. · *Assert properties over generated inputs; failures are shrunk to minimal counterexamples (QuickCheck, Hypothesis).* · [04-pro/02](04-pro/02-ai-assisted-engineering-discipline.md)

**Quantisation**: storing weights in fewer bits. · *Lower-precision weights to fit memory and bandwidth, at some quality cost.* · [03-advanced/05](03-advanced/05-local-models-with-ollama.md)

**RAG (retrieval-augmented generation)**: look it up, then answer. · *Retrieve relevant chunks and condition generation on them (Lewis et al., 2020).* · [03-advanced/03](03-advanced/03-rag-and-embeddings.md)

**recall@k**: did the right document make the top k? · *Fraction of queries with a relevant item in the top k results.* · [03-advanced/03](03-advanced/03-rag-and-embeddings.md)

**Reducer (LangGraph)**: how state updates merge. · *Function combining old and new values per key, e.g. `operator.add` to append lists.* · [03-advanced/04](03-advanced/04-agents-with-langgraph.md)

**Resource (MCP)**: data a server exposes by URI. · *"Context and data, for the user or the AI model to use"; read with `resources/read`.* · [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md)

**RRF (reciprocal rank fusion)**: merge rankings by rank. · *score(d) = Σ 1/(k + rank_i(d)), k=60 in Cormack et al. (2009).* · [03-advanced/03](03-advanced/03-rag-and-embeddings.md)

**Sandbox (Codex / Claude Code)**: OS-enforced limits. · *Filesystem/network isolation for agent processes (Codex: Seatbelt, bubblewrap, Windows sandbox; modes read-only/workspace-write/danger-full-access).* · [03-advanced/02](03-advanced/02-cursor-and-codex-workflows.md)

**Self-consistency**: ask several times, take the majority. · *Sample N reasoning paths, vote on final answers (Wang et al., 2022).* · [02-intermediate/02](02-intermediate/02-prompt-techniques-and-chaining.md)

**Slopsquatting**: malware under hallucinated package names. · *Registering names LLMs invent so suggested installs pull attacker code (Spracklen et al., 2025 measured hallucination rates).* · [04-pro/02](04-pro/02-ai-assisted-engineering-discipline.md)

**Statelessness**: nothing is remembered between calls. · *Each request carries all needed context; true of LLM APIs and, since 2026-07-28, of MCP.* · [01-beginner/01](01-beginner/01-llms-for-builders.md), [02-intermediate/03](02-intermediate/03-mcp-why-and-the-spec.md)

**Stop reason**: why the model stopped. · *`end_turn`, `tool_use`, `max_tokens`, `stop_sequence`, `pause_turn`, `refusal`, `model_context_window_exceeded`.* · [02-intermediate/01](02-intermediate/01-tool-calling-and-structured-output.md)

**Strict tool use**: tool inputs guaranteed to match schema. · *`"strict": true` on a tool definition.* · [02-intermediate/01](02-intermediate/01-tool-calling-and-structured-output.md)

**Structured outputs**: answers guaranteed to match a schema. · *`output_config.format` with `json_schema`, via constrained decoding.* · [02-intermediate/01](02-intermediate/01-tool-calling-and-structured-output.md)

**structuredContent (MCP)**: machine-readable tool result data. · *JSON value in a tool result conforming to `outputSchema`; unrelated to LLM structured outputs.* · [02-intermediate/04](02-intermediate/04-build-an-mcp-server-in-python.md)

**Subagent**: a helper agent with its own context. · *Separate context window, system prompt, tools and permissions; returns a summary.* · [03-advanced/01](03-advanced/01-claude-code-subagents-hooks-permissions.md)

**Temperature**: randomness knob. · *Divides logits before softmax; removed from requests on Claude Opus 5.5, still available in Ollama (default 0.8).* · [01-beginner/01](01-beginner/01-llms-for-builders.md)

**Thread ID (LangGraph)**: the name of one saved conversation. · *Key under which a checkpointer stores state; "your persistent cursor".* · [03-advanced/04](03-advanced/04-agents-with-langgraph.md)

**Token**: the unit models read and bill. · *An integer ID from a model-specific vocabulary.* · [01-beginner/01](01-beginner/01-llms-for-builders.md)

**Tool poisoning**: malicious instructions in tool descriptions. · *Hidden text in MCP tool descriptions that the model follows (Invariant Labs, 2025).* · [04-pro/03](04-pro/03-context-security-and-open-problems.md)

**tool_result / tool_use**: the request and reply slips of tool calling. · *`tool_use` block (id, name, input) from the model; `tool_result` block (tool_use_id, content, is_error) from you, first in the next user message.* · [02-intermediate/01](02-intermediate/01-tool-calling-and-structured-output.md)

**ToolError (MCP SDK v2)**: an anticipated tool failure. · *Exception whose message reaches the model with `isError: true`; other exceptions show only a generic message.* · [02-intermediate/04](02-intermediate/04-build-an-mcp-server-in-python.md)

**Top-k sampling**: only consider the k likeliest tokens. · *Truncate to the k highest-probability tokens and renormalise.* · [01-beginner/01](01-beginner/01-llms-for-builders.md)

## Sources

- Each definition's source is cited in the linked lesson. Key primary sources: Anthropic platform docs (https://platform.claude.com/docs/), Claude Code docs (https://code.claude.com/docs/), MCP spec 2026-07-28 (https://modelcontextprotocol.io/specification/2026-07-28/changelog), Agent Skills spec (https://agentskills.io/specification), LangGraph docs (https://docs.langchain.com/oss/python/langgraph/overview), Ollama docs (https://docs.ollama.com/), Codex docs (https://learn.chatgpt.com/codex/sandboxing), Cursor docs (https://cursor.com/docs/context/rules).

## Verify yourself

1. Pick five entries marked with Nov 2026 product facts (prefill, temperature, effort, MCP statelessness, Codex sandbox modes) and confirm them against the linked sources today.
2. For each formula entry (pass@k, pass^k, RRF, cosine), recompute one example by hand and compare with the code output in the linked lesson.
