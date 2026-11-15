# 03 · MCP: Why It Exists and What the Spec Actually Says

> Level: Intermediate · Time: 3 hours · Prereqs: [01 · Tool calling](01-tool-calling-and-structured-output.md), JSON-RPC basics are taught here
> Spec facts are from the **2026-07-28** revision of the Model Context Protocol, checked **as of Nov 2026**. Wire examples were captured from a server built with the official Python SDK **v2.3.0**.

---

## 1. Why this lesson exists

In the previous lesson you defined a tool *inside your application*: a Python function plus a JSON schema, wired to one model API. Now imagine you want that same "search my notes" tool in Claude Code, in Cursor, in Codex, and in your own LangGraph agent. Without a standard, you write four integrations, each with its own config format and calling convention. Multiply by every tool (GitHub, Jira, Postgres, Slack) and every AI application, and you get the classic **M × N integration problem**.

Anthropic open-sourced the **Model Context Protocol (MCP)** on November 25, 2024, describing the problem as: "Every new data source requires its own custom implementation, making truly connected systems difficult to scale." MCP turns M × N into M + N: each tool is wrapped once as an **MCP server**; each AI application implements an **MCP client** once.

On December 9, 2025, MCP was donated to the **Agentic AI Foundation**, "a directed fund under the Linux Foundation", alongside goose (Block) and AGENTS.md (OpenAI). The MCP blog then reported "over 97 million monthly SDK downloads, 10,000 active servers".

## 2. Intuition: USB-C for AI applications, and LSP before it

The spec itself names its inspiration: "MCP takes some inspiration from the Language Server Protocol, which standardizes how to add support for programming languages across a whole ecosystem of development tools." Before LSP, every editor implemented Python support, Go support, Rust support separately (editors × languages). With LSP, a language ships one language server and every LSP-speaking editor gets autocomplete and go-to-definition. MCP does the same for "context and tools" across AI apps.

```text
Without MCP (M x N glue)                With MCP (M + N)
 Claude Code ──┬── GitHub                Claude Code ─┐            ┌─ GitHub server
 Cursor  ──────┼── Postgres              Cursor ──────┼── MCP ─────┼─ Postgres server
 Codex  ───────┼── Notes                 Codex ───────┤ (JSON-RPC) ├─ Notes server
 My agent ─────┴── Slack                 My agent ────┘            └─ Slack server
 (every pair hand-wired)                 (each side implemented once)
```

## 3. Precise definitions (spec 2026-07-28)

**Roles.** "The protocol uses JSON-RPC 2.0 messages to establish communication between":
- **Host**: the LLM application that initiates connections (Claude Code, Cursor, your app).
- **Client**: a connector inside the host; one client talks to one server.
- **Server**: a service that provides context and capabilities.

**Server features (primitives):**

| Primitive | Who controls it | Spec wording | Example |
|---|---|---|---|
| **Tools** | The model ("model-controlled") | "Functions for the AI model to execute" | `search_notes(query)` |
| **Resources** | The application/user | "Context and data, for the user or the AI model to use" | `notes://tcp-handshake` |
| **Prompts** | The user | "Templated messages and workflows for users" | a `/quiz_me` command |

**Client features:** **Elicitation**, "server-initiated requests for additional information from users". (Sampling and Roots still exist but are **deprecated** in 2026-07-28; see 4.5.)

**Message types** (JSON-RPC 2.0):
- **Request**: has `jsonrpc: "2.0"`, an `id` (string or integer; "MUST NOT be null"), a `method`, optional `params`.
- **Result response**: same `id`, a `result` object that "MUST include a `resultType` field" (`"complete"` or `"input_required"`).
- **Error response**: same `id`, an `error` with integer `code` and `message`.
- **Notification**: a one-way message with no `id`; "the receiver MUST NOT send a response".

**Transports:** **stdio** (the client launches the server as a subprocess and exchanges newline-delimited JSON over stdin/stdout) and **Streamable HTTP** (the server is a URL; requests are HTTP POSTs). The older HTTP+SSE transport is deprecated.

## 4. Step by step: how a protocol like this gets designed

### 4.1 Dumbest approach: every app defines its own plugin API

Each AI app publishes "how to write a plugin for us". Walls: tool authors write N plugins; config formats diverge; security models diverge; a tool cannot advertise its capabilities in a standard way; nothing composes.

### 4.2 Idea 1: a shared message format, JSON-RPC 2.0

JSON-RPC is a tiny, well-known spec: requests with IDs, responses matched by ID, notifications without IDs, standard error codes (`-32700` parse error, `-32600` invalid request, `-32601` method not found, `-32602` invalid params, `-32603` internal error). MCP adds its own reserved range: in 2026-07-28, codes `-32020` to `-32099` are reserved for the spec, e.g. `-32022` `UnsupportedProtocolVersion`.

### 4.3 Idea 2: discovery instead of hard-coding

A client should not need to know a server's tools in advance. So: `tools/list` returns names, descriptions and JSON Schemas (exactly the tool definitions from the previous lesson), `tools/call` invokes one. Same pattern for `resources/list`/`resources/read` and `prompts/list`/`prompts/get`.

### 4.4 Idea 3 (and how it changed): negotiating versions and capabilities

Earlier revisions (2024-11-05 through 2025-11-25) began every connection with an **`initialize` handshake**: the client sent its protocol version and capabilities, the server replied with its own, then the client sent `notifications/initialized`. The connection was a stateful session.

**Wall:** stateful sessions are hard to scale behind load balancers (sticky sessions), hard to resume after a dropped connection, and they tempt servers to hide state in the connection.

**The 2026-07-28 revision made MCP stateless.** From its changelog:

- "Remove the `initialize`/`notifications/initialized` handshake. Every request now carries its protocol version and client capabilities in `_meta` (`io.modelcontextprotocol/protocolVersion`, `io.modelcontextprotocol/clientCapabilities`)."
- "Remove protocol-level sessions and the `Mcp-Session-Id` header from the Streamable HTTP transport." Servers needing cross-call state "use explicit, server-minted handles passed as ordinary tool arguments".
- "Add `server/discover`: servers MUST implement this RPC to advertise their supported protocol versions, capabilities, and identity."
- "All results now carry a required `resultType` field."
- "Remove `ping`, `logging/setLevel`, and `notifications/roots/list_changed`."
- The **Multi Round-Trip Requests (MRTR)** pattern replaces server-initiated requests: instead of the server calling back into the client mid-request, it returns `resultType: "input_required"` with `inputRequests`, and the client **retries the original request** with `inputResponses`.

The base-protocol page states the principle bluntly: "Servers MUST NOT rely on prior requests over the same connection to establish context", and "an open connection, such as a STDIO process, is not a conversation or session".

```mermaid
sequenceDiagram
    participant C as MCP client (in host)
    participant S as MCP server
    Note over C,S: 2026-07-28: no handshake; every request carries _meta
    C->>S: server/discover (optional, up-front version check)
    S-->>C: supportedVersions, capabilities, serverInfo
    C->>S: tools/list  {_meta: protocolVersion, clientCapabilities}
    S-->>C: {resultType: "complete", tools: [...], ttlMs, cacheScope}
    C->>S: tools/call {name, arguments, _meta}
    alt needs user input (MRTR)
        S-->>C: {resultType: "input_required", inputRequests, requestState}
        C->>S: tools/call again (new id) + inputResponses + requestState
    end
    S-->>C: {resultType: "complete", content: [...], isError: false}
```

**Why you must know both models:** many servers and clients in the wild (as of Nov 2026) still speak the handshake-era revisions. The Python SDK v2.3.0 client defaults to `mode="auto"`, which, per its source, tries `server/discover` and falls back to the `initialize` handshake.

### 4.5 Deprecations you will see in older tutorials

From the 2026-07-28 changelog, "Deprecate the Roots, Sampling, and Logging features." Suggested migrations: pass directories "via tool parameters, resource URIs, or server configuration instead of Roots"; "integrate directly with LLM provider APIs instead of Sampling"; "log to `stderr` (stdio) or use OpenTelemetry instead of Logging". Deprecated features have "a minimum twelve-month deprecation window" under the new lifecycle policy. Tasks (long-running operations) moved out of core into an **extension**, `io.modelcontextprotocol/tasks`.

### 4.6 See the wire yourself

Reading a spec is not the same as seeing bytes. This script spawns the notes server from the next lesson and speaks raw JSON-RPC to it over stdio, without any SDK client.

```python
# raw_wire.py - talk to an MCP stdio server with hand-written JSON-RPC, no SDK client.
# Purpose: SEE the protocol. Spec revision 2026-07-28; server built with MCP Python SDK v2.3.0.
import json                       # encode/decode JSON-RPC messages
import subprocess                 # spawn the server as a child process
import sys                        # sys.executable = this python interpreter

# Every 2026-07-28 request carries version + client capabilities in params._meta (no handshake).
META = {
    "io.modelcontextprotocol/protocolVersion": "2026-07-28",
    "io.modelcontextprotocol/clientCapabilities": {},
    "io.modelcontextprotocol/clientInfo": {"name": "raw-wire-demo", "version": "0.1"},
}

# Launch the server; text=True gives str pipes; stderr is left alone so server logs show up.
proc = subprocess.Popen([sys.executable, "notes_server.py"],
                        stdin=subprocess.PIPE, stdout=subprocess.PIPE, text=True)


def rpc(msg_id: int, method: str, params: dict | None = None) -> dict:
    """Send one request line and read one response line (stdio = newline-delimited JSON)."""
    req = {"jsonrpc": "2.0", "id": msg_id, "method": method,
           "params": {**(params or {}), "_meta": META}}
    proc.stdin.write(json.dumps(req) + "\n")      # one message per line, no embedded newlines
    proc.stdin.flush()                            # push it through the pipe now
    return json.loads(proc.stdout.readline())     # block until the matching response line


r = rpc(1, "server/discover")                     # new in 2026-07-28: versions + capabilities
print("discover ->", r["result"]["supportedVersions"], list(r["result"]["capabilities"]))

r = rpc(2, "tools/list")
print("tools/list ->", [t["name"] for t in r["result"]["tools"]], "| resultType:", r["result"]["resultType"])

r = rpc(3, "tools/call", {"name": "no_such_tool", "arguments": {}})
print("unknown tool ->", json.dumps(r)[:160])

proc.stdin.close()                                # closing stdin tells a stdio server to exit
proc.wait(timeout=10)
```

Output (server log lines on stderr omitted):

```text
discover -> ['2026-07-28'] ['prompts', 'resources', 'tools']
tools/list -> ['add_note', 'search_notes'] | resultType: complete
unknown tool -> {"jsonrpc": "2.0", "id": 3, "result": {"content": [{"text": "Unknown tool: no_such_tool", "type": "text"}], "isError": true, "resultType": "complete", "_meta": 
```

**Line-by-line walkthrough (syntax and logic):**

- `META = {...}`: the `_meta` envelope that **every** 2026-07-28 request must carry. The two required keys are the protocol version and the client capabilities (empty dict = "I offer no optional client features"). `clientInfo` is recommended ("Clients SHOULD include").
- `subprocess.Popen([sys.executable, "notes_server.py"], stdin=PIPE, stdout=PIPE, text=True)`: starts the server as a child process with pipes connected to its stdin and stdout. `sys.executable` is the path of the running Python interpreter, so the child uses the same environment. `text=True` means we read and write `str` instead of `bytes`.
- `def rpc(msg_id, method, params=None) -> dict:` builds a request. `{**(params or {}), "_meta": META}` uses dict unpacking: copy the caller's params (or an empty dict if `None`) and add `_meta`.
- `proc.stdin.write(json.dumps(req) + "\n")` then `flush()`: stdio transport is **one JSON message per line**, so a message must not contain raw newlines (`json.dumps` escapes them). Without `flush()` the bytes may sit in a buffer and both sides wait forever (a classic deadlock).
- `json.loads(proc.stdout.readline())`: read exactly one line, one response. (A production client must also handle notifications arriving between responses and match responses to requests by `id`.)
- The three calls: `server/discover`, `tools/list`, and a call to a tool that does not exist.
- `proc.stdin.close(); proc.wait(timeout=10)`: closing stdin signals EOF; a stdio server exits when its input closes.

**What the output teaches:**

1. `server/discover` advertised `['2026-07-28']` and capabilities `prompts`, `resources`, `tools`.
2. `tools/list` returned `resultType: complete`.
3. Calling an unknown tool came back as a **result with `isError: true`**, not as a JSON-RPC error. The spec's tools page lists "Unknown tool" under **protocol errors** (JSON-RPC `error`), while SDK v2.3.0 returned it as a tool execution error. Robust clients must handle both forms. This is the kind of detail you only learn by running things.

I also sent `tools/list` **without** `_meta`; SDK v2.3.0 replied with error `-32602`: "params._meta must be an object carrying the required 'io.modelcontextprotocol/protocolVersion' and 'io.modelcontextprotocol/clientCapabilities' envelope keys", which matches the spec's rule that such a request "is malformed; the server MUST reject it with JSON-RPC error code `-32602`".

### 4.7 Tools in detail (the primitive you will use most)

A tool definition has `name`, optional `title`, `description`, `inputSchema` (JSON Schema, default dialect 2020-12), optional `outputSchema`, optional `annotations` and `icons`. Tool names "SHOULD be between 1 and 128 characters", using letters, digits, `_`, `-` and `.`.

Results contain **unstructured** `content` (text, image, audio, resource links, embedded resources) and optionally **`structuredContent`** that conforms to `outputSchema`. For backwards compatibility, a tool returning structured content "SHOULD also return the serialized JSON in a TextContent block". Note the spec's own clarification: `structuredContent` "is unrelated to LLM 'structured outputs' (schema-constrained model generation)".

Two error channels:
1. **Protocol errors** (JSON-RPC `error`): malformed requests, server errors; "models are less likely to be able to fix" these.
2. **Tool execution errors** (`isError: true` in the result): "actionable feedback that language models can use to self-correct". Clients "SHOULD provide tool execution errors to language models".

**Statelessness and tools:** because there is no session, a server that needs state across calls (a shopping cart, a DB transaction) should return an explicit **handle** from a creation tool and accept it as an argument later. The spec's guidance: handles should be opaque, have stated lifetimes, be authorised on every call, and expired handles should produce an `isError` result so the model can recover.

### 4.8 Security principles (normative intent)

The spec's "Key Principles": **user consent and control** ("Users must explicitly consent to and understand all data access and operations"), **data privacy**, and **tool safety**: "Tools represent arbitrary code execution and must be treated with appropriate caution", and "descriptions of tool behavior such as annotations should be considered untrusted, unless obtained from a trusted server". It also says "there SHOULD always be a human in the loop with the ability to deny tool invocations". MCP "cannot enforce these security principles at the protocol level"; hosts must. Real attacks on exactly this boundary (tool poisoning, rug pulls) are covered in 04-pro/03.

## 5. Common mistakes

1. **Printing to stdout in a stdio server.** stdout *is* the protocol channel; a stray `print()` corrupts it. Log to stderr.
2. **Following a 2025 tutorial verbatim.** `initialize` handshakes, `Mcp-Session-Id`, sampling and roots are gone or deprecated in 2026-07-28. Check the revision your client and server speak.
3. **Hiding state in the connection.** Forbidden by the stateless model; use explicit handles.
4. **Treating tool descriptions as trusted.** A malicious server's description is a prompt you did not write.
5. **Confusing `structuredContent` with structured outputs.** One is server data; the other is constrained model generation.
6. **Returning protocol errors for user-fixable problems.** If the model could fix it (bad date format), return `isError: true` with instructions.
7. **Forgetting `flush()`** when hand-writing stdio clients, then debugging a "hang" for an hour.

## 6. Connections to other tracks

- **Networks**: JSON-RPC over stdio or HTTP; Streamable HTTP is plain HTTP POST; statelessness mirrors REST design. Sticky sessions vs stateless services is a load-balancing topic.
- **OS**: stdio transport is pipes between parent and child processes (`fork`/`exec`/`pipe` in the OS track); EOF on stdin as a shutdown signal is classic Unix.
- **System Design**: M × N → M + N is the adapter/hub pattern; `ttlMs`/`cacheScope` on list results are HTTP-style caching hints.
- **Compilers**: LSP is the direct ancestor; if you have built a language server, MCP will feel familiar.
- **Security**: consent, least privilege, untrusted metadata, OAuth for HTTP transports.

## 7. Where this shows up in my projects

- **Nidhogg** (Elasticsearch fleet ops console with an agentic Assistant): an MCP server exposing a few read-only, well-described fleet tools (cluster health, index stats, a Logly search) would make that functionality available to Claude Code or Cursor directly, without reimplementing integrations per host. Mark mutating operations clearly and keep them out of the default tool set.
- **QiskitSage**'s review agents could consume repository context through MCP resources (diff, file contents) instead of bespoke loaders, making the reviewer host-agnostic.

## 8. Self-check questions

1. Explain M × N vs M + N in one sentence each.
2. Which primitive is model-controlled, and which is user-controlled?
3. What replaced the `initialize` handshake in 2026-07-28, and what must every request carry?
4. A server needs a multi-step "draft then send" workflow. How should it carry state under the stateless model?
5. Why did our raw client need `flush()` after each write?
6. When should a tool failure be `isError: true` rather than a JSON-RPC error?

<details><summary>Answers</summary>

1. M × N: every one of M apps integrates separately with every one of N tools. M + N: each app implements one MCP client and each tool one MCP server.
2. Tools are model-controlled; prompts are user-controlled (resources are application/user-driven context).
3. Per-request `_meta` with `io.modelcontextprotocol/protocolVersion` and `io.modelcontextprotocol/clientCapabilities` (and optionally `clientInfo`); `server/discover` exists for up-front version/capability discovery.
4. Return an opaque, authorised, time-limited handle (e.g. `draft_id`) from the first tool and require it as an argument to later tools; never rely on the connection.
5. Writes to a pipe are buffered; without flushing, the request may never reach the server, and both processes block waiting.
6. When the model could plausibly fix it (bad argument, business-rule violation, API failure with retry advice). Protocol errors are for malformed requests and server faults.

</details>

## 9. Mnemonic (Tamil + English)

**"Thayir, Rasam, Payasam: Tools-Resources-Prompts."** (தயிர், ரசம், பாயசம்)
A South Indian meal's three closing items map onto MCP's three server primitives, **T**-**R**-**P**. And who serves them? The **model** grabs the Thayir (tools) itself, the **app** pours the Rasam (resources), the **user** asks for Payasam (prompts).

**"Meta illama request-ey illa."** (மெட்டா இல்லாம ரிக்வெஸ்ட்-ஏ இல்ல: no `_meta`, no request.) In 2026-07-28 every request carries `_meta`.

## Sources

- MCP specification, latest (2026-07-28) overview (as of Nov 2026): https://modelcontextprotocol.io/specification/latest
- MCP 2026-07-28 changelog: https://modelcontextprotocol.io/specification/2026-07-28/changelog
- MCP 2026-07-28 base protocol: https://modelcontextprotocol.io/specification/2026-07-28/basic
- MCP 2026-07-28 tools: https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- MCP TypeScript schema (source of truth): https://github.com/modelcontextprotocol/specification/blob/main/schema/2026-07-28/schema.ts
- JSON-RPC 2.0 specification: https://www.jsonrpc.org/specification
- Anthropic, "Introducing the Model Context Protocol" (Nov 25, 2024): https://www.anthropic.com/news/model-context-protocol
- MCP blog, "MCP joins the Agentic AI Foundation" (Dec 9, 2025): https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation/
- Language Server Protocol: https://microsoft.github.io/language-server-protocol/
- MCP Python SDK v2 migration guide: https://py.sdk.modelcontextprotocol.io/v2/migration/

## Verify yourself

1. `pip install "mcp==2.3.0"`, save `notes_server.py` (next lesson) and `raw_wire.py` together, run `python3 raw_wire.py`. Then remove `"_meta": META` from `rpc()` and confirm the `-32602` error.
2. Add a request for a method that does not exist (`"foo/bar"`) and record which JSON-RPC error code comes back.
3. Open the changelog and list which features were *removed* vs *deprecated*. Why does the distinction matter for a client you ship today?
4. Compare the tools page's "Unknown tool" example (a protocol error) with what SDK v2.3.0 returned in our run; check whether newer SDK versions changed this.
