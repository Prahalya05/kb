# Interview Questions: AI Developer Tools & Skills

> Model answers are written the way you would say them: direct, with the reasoning and one concrete detail. Product facts are **as of Nov 2026**; in an interview, say "as of the last time I checked the docs" for anything version-specific.

---

## Fundamentals

### 1. What actually happens when an LLM generates a response?

**Model answer.** Text is tokenized into integer IDs with a model-specific tokenizer (usually a BPE variant). The network reads the whole context and outputs logits over the vocabulary for the next token; softmax turns them into probabilities; a decoding strategy picks one token; it is appended and the process repeats until a stop token, a stop sequence or `max_tokens`. The API is stateless: each call carries the full conversation, which is why long chats cost more per turn and why tools invent memory files.

### 2. Why might "temperature 0" not give deterministic output, and what do current APIs say about it?

**Model answer.** Greedy decoding is deterministic only if logits are bit-identical, but serving systems batch requests differently and floating-point addition is not associative, so near-ties can flip and the continuation diverges. Anthropic's migration guide for Claude Opus 5.5 says temperature 0 "never guaranteed identical outputs on prior models", and as of Nov 2026 Opus 5.5 requests should omit `temperature`, `top_p` and `top_k` entirely. For reliability I use structured outputs, validation and multi-trial evals rather than a sampling knob.

### 3. What is context rot and how do you design around it?

**Model answer.** Recall and accuracy degrade as the context grows; Anthropic's docs name it explicitly, and "Lost in the Middle" (Liu et al., 2023) showed information in the middle of long contexts is used worst. I design for the smallest high-signal context: put long documents first and the question last, retrieve just-in-time rather than preloading, delegate exploration to subagents that return summaries, use compaction for long sessions, and keep always-loaded instruction files short.

## Prompting

### 4. How do you improve a prompt systematically?

**Model answer.** First define success criteria and a way to test them, which Anthropic's guide puts before any prompt engineering. Then iterate on clarity (say exactly what done looks like), context with reasons, structure with XML tags, data first and question last, 3–5 diverse examples, and a positive output-format spec. Each change is measured on an eval set with several trials, because a single run of a stochastic system is anecdote.

### 5. When would you use prompt chaining instead of one prompt?

**Model answer.** When I need to inspect, test or branch on intermediate outputs, or enforce a pipeline: extract → solve → check. Models reason well internally now, so the value of chaining is engineering control: each step is logged, gated and individually testable, and failures are localised.

### 6. Prefilling used to force JSON. What do you do now?

**Model answer.** As of Nov 2026, prefilling the last assistant turn returns a 400 on Claude 4.6 and later. I use structured outputs (`output_config.format` with a JSON schema, or `messages.parse` with a Pydantic model in the Python SDK), which use constrained decoding, and I still validate on my side because some constraints are stripped from the sent schema and refusals or truncation can produce non-conforming output.

## Tool calling and structured output

### 7. Walk me through the tool-use loop.

**Model answer.** Send messages plus tool definitions (name, detailed description, JSON Schema). If `stop_reason` is `tool_use`, append the assistant's full content, execute each `tool_use` block after validating its input, and send one user message whose content starts with a `tool_result` per call, matched by `tool_use_id`, with `is_error` and an instructive message on failure. Loop with a turn cap until `end_turn`, and handle `max_tokens`, `refusal`, `pause_turn` and `model_context_window_exceeded` explicitly.

### 8. What makes a good tool for an agent?

**Model answer.** A clear, task-shaped purpose (search, not "list everything"), a 3–4+ sentence description covering when to use and not use it, unambiguous parameter names, token-efficient results with pagination or truncation, human-readable fields instead of opaque IDs, and errors that tell the model how to fix the call. Anthropic's SWE-bench anecdote is my favourite example: they changed a tool to require absolute paths and the model stopped making path mistakes. Design the interface so mistakes are hard.

### 9. How does constrained decoding guarantee valid JSON, and what can still go wrong?

**Model answer.** At each step the decoder masks every token that would make the partial output an invalid prefix under the schema's grammar, so only valid continuations can be sampled. It guarantees syntax, not correctness: the content can be wrong, enum casing may not be guaranteed, constraints like min/max may be stripped, and masking can distort the distribution (for example, when one option has more tokenization paths). In my toy experiment a label with two token spellings was chosen about twice as often as the others.

## MCP

### 10. What problem does MCP solve and how?

**Model answer.** The M × N integration problem between AI applications and tools. MCP standardises the interface: servers expose tools, resources and prompts over JSON-RPC 2.0; hosts implement a client once. It was open-sourced by Anthropic in November 2024 and donated to the Agentic AI Foundation under the Linux Foundation in December 2025. It is explicitly inspired by the Language Server Protocol.

### 11. What changed in the 2026-07-28 MCP revision?

**Model answer.** MCP became stateless. The `initialize` handshake and protocol-level sessions were removed; every request carries its protocol version and client capabilities in `_meta`; servers must implement `server/discover`; results carry a required `resultType`; server-initiated requests were replaced by the multi round-trip pattern where the server returns `input_required` and the client retries; roots, sampling and logging were deprecated; tasks moved to an extension. Servers that need state use explicit handles passed as tool arguments.

### 12. How do you report errors from an MCP tool?

**Model answer.** Two channels: protocol errors (JSON-RPC `error`) for malformed requests and server faults, and tool execution errors (`isError: true` in the result) for things the model can fix, with an actionable message. In the Python SDK v2, raising `ToolError` sends my message to the model; any other exception is treated as a crash and the model only sees "Error executing tool X", which protects internals but makes recovery impossible, so anticipated failures must use `ToolError`.

### 13. What are the security risks of MCP servers?

**Model answer.** Tool descriptions are prompts written by a third party: tool poisoning hides instructions in descriptions, rug pulls change descriptions after approval, and shadowing manipulates how another server's tool is used (Invariant Labs, 2025). The spec says annotations should be treated as untrusted unless from a trusted server and that hosts must get consent before invoking tools. I install servers like dependencies (trusted, pinned, reviewed), prefer read-only servers, keep approvals on side effects with full arguments shown, and isolate servers touching secrets from those touching the internet.

## Coding agents

### 14. How do you configure Claude Code for a team repository?

**Model answer.** A short project CLAUDE.md (under 200 lines) with commands, invariants and pointers; path-scoped rules in `.claude/rules/`; procedures as skills (with `disable-model-invocation` for side effects); read-only reviewer subagents; hooks for deterministic formatting and test gates; permissions with deny rules for secrets and network, evaluated deny → ask → allow; shared MCP servers in `.mcp.json`. For hard guarantees, the sandbox, because regex hooks and command-pattern rules are bypassable.

### 15. A hook blocks `rm -rf`. Is that a security control?

**Model answer.** It is a guardrail, not a boundary. I tested my own hook: `rm -fr`, `rm -r -f` and `find . -delete` all bypassed it. Denylists over a shell are incomplete by construction. The docs point to the OS-level sandbox for hard enforcement; I keep hooks for automation and early feedback and rely on sandboxes, containers and least-privilege credentials for security.

### 16. How do Claude Code, Cursor and Codex differ in instruction loading?

**Model answer.** Claude Code concatenates every CLAUDE.md and CLAUDE.local.md from the filesystem root down to the working directory. Cursor uses `.mdc` rule files with `description`, `globs` and `alwaysApply`, plus nested AGENTS.md where the more specific file takes precedence. Codex takes at most one file per directory (`AGENTS.override.md` beats `AGENTS.md`), concatenates root to cwd with later files overriding, and stops at a 32 KiB default budget. For portability I keep AGENTS.md as the source of truth and have CLAUDE.md import it.

## RAG and agents

### 17. Your RAG app gives wrong answers. How do you debug it?

**Model answer.** Separate retrieval from generation. Build a labelled query set and measure recall@k and MRR. If the right chunk is not retrieved, fix retrieval: chunking (structure-aware, overlap, contextual prefixes), hybrid BM25 + dense with RRF, reranking, consistent embedding models. If it is retrieved but the answer is wrong, fix the prompt (context first, answer-only-from-context, quote first) or the model. Anthropic's contextual retrieval cut top-20 retrieval failures by 49% with contextual embeddings plus contextual BM25, and 67% with reranking.

### 18. Why use LangGraph instead of a while loop?

**Model answer.** For durable, inspectable control flow: explicit nodes and conditional edges, state with reducers, checkpointing per thread ID so runs survive restarts, and `interrupt()` for human approval that can wait hours. The rules matter: a checkpointer and the same thread ID are required to resume, the interrupted node re-runs from the top so side effects belong in a later node, and you must not swallow the interrupt in a bare except. For simple, short-lived agents a loop is clearer.

### 19. When would you use a local model via Ollama, and what breaks first?

**Model answer.** When data cannot leave the machine, for offline work, high-volume cheap tasks like embeddings and classification, or pinned reproducibility. What breaks first: streaming is on by default in `/api/chat`; the default context is small (the FAQ says 4096 tokens) so long prompts get truncated silently; the default temperature is 0.8; and small models are weaker at tool calling. I set `num_ctx`, low temperature for structured tasks, use the `format` schema, validate outputs, and bake settings into a Modelfile.

## Engineering discipline and evals

### 20. How do you evaluate an agent?

**Model answer.** Tasks drawn from real failures (20–50 is a fine start), multiple trials per task, outcome-based graders (code where possible, calibrated LLM judges where needed), and metrics chosen for the product: pass@k when a human picks among attempts, pass^k when the agent must succeed every time. Report confidence intervals, track cost and latency, read transcripts to catch grader bugs, and gate CI on regression evals.

### 21. How do you review AI-generated code?

**Model answer.** Scale effort with risk. For anything non-trivial: check intent, verify every new dependency exists (package hallucination rates of at least 5.2% for commercial and 21.7% for open-source models were measured by Spracklen et al.), test boundaries and aliasing, review security (Pearce et al. found about 40% of Copilot programs in security scenarios vulnerable), and use independent oracles, property-based and differential tests, not just tests the same model wrote. Keep diffs small and leave a regression test for every bug found.

### 22. Do AI coding tools make developers faster?

**Model answer.** It depends, and perception is unreliable. METR's 2025 randomised trial with 16 experienced open-source developers on 246 tasks found AI-allowed tasks took 19% longer, while developers believed they were 20% faster. That is one setting (early-2025 tools, experts on their own large repos), not a universal law, but it is a strong argument for measuring on your own tasks.

### 23. Explain the lethal trifecta and how you'd mitigate it.

**Model answer.** Willison's term: an agent with access to private data, exposure to untrusted content, and the ability to communicate externally can be prompt-injected into exfiltrating data. Detection is probabilistic, so I design it out: split agents so none has all three legs (a quarantined reader without tools, as in the dual-LLM pattern), enforce capability and data-flow policies in code (CaMeL reports 77% task success with provable security on AgentDojo versus 84% undefended), require approvals for side effects, and add egress allowlists.

## Sources

- Each answer draws on a lesson in this track; see those lessons' Sources sections. Key primary sources: Anthropic docs (https://platform.claude.com/docs/), Claude Code docs (https://code.claude.com/docs/), MCP 2026-07-28 changelog (https://modelcontextprotocol.io/specification/2026-07-28/changelog), Codex AGENTS.md docs (https://learn.chatgpt.com/codex/agent-configuration/agents-md), Cursor rules (https://cursor.com/docs/context/rules), LangGraph interrupts (https://docs.langchain.com/oss/python/langgraph/interrupts), Ollama FAQ (https://docs.ollama.com/faq), METR RCT (https://arxiv.org/abs/2507.09089), Spracklen et al. (https://arxiv.org/abs/2406.10279), Pearce et al. (https://arxiv.org/abs/2108.09293), CaMeL (https://arxiv.org/abs/2503.18813), Anthropic contextual retrieval (https://www.anthropic.com/news/contextual-retrieval).

## Verify yourself

1. Before an interview, re-check every number you plan to quote (19%, 5.2%/21.7%, ~40%, 49%/67%, 77%/84%) against the linked abstract or post.
2. Re-check the version-sensitive answers (2, 6, 11, 16) against current docs; say the date you checked.
3. Practise answers 7, 11 and 18 aloud with the lesson code open, pointing at the exact lines.
