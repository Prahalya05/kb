# 02 · Cursor and Codex Workflows (and a Portable Setup Across All Three Tools)

> Level: Advanced · Time: 3 hours · Prereqs: [01 · Subagents, hooks, permissions](01-claude-code-subagents-hooks-permissions.md), [02-intermediate/04](../02-intermediate/04-build-an-mcp-server-in-python.md)
> Facts from cursor.com/docs and OpenAI's Codex docs, stamped **(as of Nov 2026)**. These products change fast; verify before relying on any flag.

---

## 1. Why this lesson exists

Most teams do not standardise on one AI tool. You might use Cursor in the editor, Codex in CI, and Claude Code in the terminal, sometimes on the same repo on the same day. Each tool has its own instruction files, rule semantics, sandbox model and config format. If you learn only one, you will misconfigure the others in subtle ways (an override file that silently hides a rule; a rule file with the wrong extension that is silently ignored). This lesson teaches Cursor's and Codex's models precisely, compares them with Claude Code, and ends with a **portable repository setup** that works for all three.

## 2. Intuition

The three tools are like three brands of car with the same controls in different places. All have a steering wheel (the agent loop), mirrors (instruction files), brakes (permissions/sandbox) and a sat-nav port (MCP). Swapping cars is easy once you know where each control is, and dangerous if you assume the brake pedal is where your old car had it.

## 3. Precise definitions (as of Nov 2026)

### Cursor

- **Agent modes**: **Agent** (edits, runs commands), **Ask** (read-only), **Plan** ("Plan Mode creates detailed implementation plans before writing any code"; edits only after you approve), **Debug** ("Tricky bugs that need runtime evidence"). Switch with the mode picker or `Shift+Tab`.
- **Checkpoints**: "Restore Checkpoint" on a previous message rolls back file changes made after that point (it does not remove messages).
- **Rules**: Project rules in `.cursor/rules`, "version-controlled and scoped to your codebase"; User rules (global); Team rules (dashboard, Team/Enterprise plans); and `AGENTS.md` ("Simple alternative to `.cursor/rules`"). Project rules "must use the `.mdc` extension"; a plain `.md` in `.cursor/rules` "is ignored because it lacks the frontmatter". Frontmatter: `description`, `globs`, `alwaysApply`. Application types: **Always Apply**, **Apply Intelligently** (by description), **Apply to Specific Files** (globs), **Apply Manually** (`@rule-name`). Nested `AGENTS.md` files apply to their subtree, and "more specific instructions take precedence". Guidance: "Keep rules under 500 lines".
- **MCP**: `.cursor/mcp.json` or `~/.cursor/mcp.json`; stdio, SSE and Streamable HTTP; supports tools, prompts, resources, roots, elicitation; asks before using MCP tools by default.
- **Bugbot**: PR-level review that "can catch bugs and security issues automatically". **Cloud Agents** run agents from the browser at cursor.com/agents. **CLI**: `agent` (print mode `agent -p "..."`).

### Codex

- **Surfaces**: Codex CLI, IDE extension, Codex Cloud, plus ChatGPT apps; SDK and GitHub Action for building with it.
- **Install** (CLI): `npm install -g @openai/codex` or `brew install --cask codex`; sign in with ChatGPT or an API key.
- **Config**: `~/.codex/config.toml`, keys such as `model`, `approval_policy`, `sandbox_mode`, `model_reasoning_effort`; profiles via `--profile name` (`~/.codex/name.config.toml`).
- **Sandbox modes**: `read-only` ("can inspect files, but it can't edit files or run commands without approval"), `workspace-write` ("read files, edit within the workspace, and run routine local commands inside that boundary"; the default low-friction local mode), `danger-full-access` (no sandbox).
- **Approval policies**: `on-request` (works in the sandbox, "asks when it needs to go beyond that boundary"), `never`. The docs state `untrusted` is no longer a selectable policy.
- **OS mechanisms**: macOS Seatbelt; Linux/WSL2 bubblewrap (`bwrap` on `PATH`, else a bundled helper); native Windows sandbox in PowerShell.
- **AGENTS.md discovery**: global `~/.codex/AGENTS.override.md` or `AGENTS.md`; then from the project root down to the current directory, **at most one file per directory** in the order `AGENTS.override.md`, `AGENTS.md`, then configured fallbacks; concatenated root-down; "Files closer to your current directory override earlier guidance because they appear later in the combined prompt"; stops at `project_doc_max_bytes` (default 32 KiB).
- **Skills**: Codex supports Agent Skills, discovered in `.agents/skills` (repo, walking up to the root), `$HOME/.agents/skills`, `/etc/codex/skills`, plus bundled system skills; invoked explicitly (`/skills`, `$` mention) or implicitly when the task matches the `description`.
- **Non-interactive**: `codex exec "<task>"`; "By default, `codex exec` runs in a read-only sandbox"; `--sandbox workspace-write` to allow edits; `--json` makes stdout a JSONL event stream; `--output-schema` requests a final response conforming to a JSON Schema; `codex exec resume --last` continues. "`--full-auto` is no longer supported."
- **MCP**: `[mcp_servers.<name>]` tables in `config.toml` (stdio: `command`, `args`, `env`, `env_vars`, `cwd`; HTTP: `url`, `bearer_token_env_var`, `http_headers`, …; plus `enabled_tools`, `disabled_tools`, timeouts).

## 4. Step by step

### 4.1 Cursor: rules that actually load

**Dumbest:** create `.cursor/rules/style.md` with your style guide. **Wall:** it is ignored, because project rules must be `.mdc` with frontmatter.

**Better:** `.cursor/rules/python-style.mdc`:

```markdown
---
description: Python style for this repo. Apply when writing or editing Python.
globs: src/**/*.py, tests/**/*.py
alwaysApply: false
---
- snake_case everywhere; type hints on public functions (mypy runs in CI).
- Use pathlib, not os.path, because our Windows CI breaks on string paths.
- Tests use pytest fixtures from tests/conftest.py; do not create new DB fixtures.
```

`globs` makes it "Apply to Specific Files"; a good `description` lets "Apply Intelligently" pick it up when relevant; `alwaysApply: true` would load it every session (use sparingly, same economics as CLAUDE.md).

**A Cursor workflow that holds up:**

1. **Ask** mode: "How does request authentication flow through `src/api`?" (learn before editing).
2. **Plan** mode for anything multi-file; read and edit the plan; approve.
3. **Agent** mode executes; watch terminal commands it proposes.
4. If it goes sideways, **Restore Checkpoint** rather than asking it to undo by hand.
5. Review the diff, run tests, commit; let **Bugbot** review the PR as a second reader, not as the only one.

### 4.2 Codex: sandbox-first by design

Codex's centre of gravity is the OS sandbox: rather than parsing commands (the weakness we exploited in the previous lesson's hook), it constrains what any process can touch.

A sensible `~/.codex/config.toml` for daily local work:

```toml
model_reasoning_effort = "medium"
approval_policy = "on-request"
sandbox_mode = "workspace-write"

project_doc_max_bytes = 65536
project_doc_fallback_filenames = ["CLAUDE.md"]   # let Codex also read a repo's CLAUDE.md

[mcp_servers.study-notes]
command = "python3"
args = ["/abs/path/notes_server.py"]
```

`project_doc_fallback_filenames` is a documented key; listing `CLAUDE.md` there is my suggestion for repos that only have a CLAUDE.md. (I left `model` unset deliberately; model names change frequently, so pick one from the current docs.)

### 4.3 Predicting what Codex reads: simulation

Codex's override semantics differ from Claude Code's pure concatenation: **one file per directory**, and an `AGENTS.override.md` *hides* the sibling `AGENTS.md`. Here is the documented algorithm as code, using the docs' own payments example:

```python
# codex_agents_md.py - simulate Codex's AGENTS.md discovery (per the Codex docs, Nov 2026):
# global ~/.codex (override wins), then project root DOWN to cwd, at most ONE file per directory
# (AGENTS.override.md > AGENTS.md > fallbacks), concatenated, stopping at project_doc_max_bytes.
from pathlib import Path      # paths
import tempfile               # scratch tree

MAX_BYTES = 32 * 1024         # documented default for project_doc_max_bytes (32 KiB)


def pick(d: Path, names: list[str]) -> Path | None:
    for n in names:                                     # first existing, non-empty name wins
        f = d / n
        if f.exists() and f.read_text().strip():
            return f
    return None


def discover(codex_home: Path, project_root: Path, cwd: Path, fallbacks=()) -> list[Path]:
    names = ["AGENTS.override.md", "AGENTS.md", *fallbacks]
    out = [g] if (g := pick(codex_home, names[:2])) else []
    rel = cwd.relative_to(project_root).parts           # path pieces from root to cwd
    dirs = [project_root] + [project_root.joinpath(*rel[:i + 1]) for i in range(len(rel))]
    total = sum(len(f.read_bytes()) for f in out)
    for d in dirs:                                      # root first, cwd last (later = stronger)
        f = pick(d, names)
        if f is None:
            continue
        size = len(f.read_bytes())
        if total + size > MAX_BYTES:                    # budget reached: stop adding
            break
        out.append(f); total += size
    return out


t = Path(tempfile.mkdtemp())
home, repo = t / ".codex", t / "repo"
pay = repo / "services" / "payments"
for d in (home, pay):
    d.mkdir(parents=True)
(home / "AGENTS.md").write_text("Prefer pnpm.")
(repo / "AGENTS.md").write_text("Run `make check` before finishing.")
(pay / "AGENTS.md").write_text("Ignored, because an override exists here.")
(pay / "AGENTS.override.md").write_text("Freeze: no schema migrations this week.")
for f in discover(home, repo, pay):
    print(f.relative_to(t), "->", f.read_text())
```

Output:

```text
.codex/AGENTS.md -> Prefer pnpm.
repo/AGENTS.md -> Run `make check` before finishing.
repo/services/payments/AGENTS.override.md -> Freeze: no schema migrations this week.
```

**Line-by-line walkthrough (syntax and logic):**

- `MAX_BYTES = 32 * 1024`: the documented default for `project_doc_max_bytes`.
- `def pick(d, names)`: returns the first existing, non-empty file among `names` in directory `d`. `f.read_text().strip()` is empty for whitespace-only files, and an empty string is falsy, so empty files are skipped ("Codex skips empty files").
- `out = [g] if (g := pick(codex_home, names[:2])) else []`: the `:=` **walrus operator** assigns and tests in one expression: compute `g`, and if it is truthy put it in a one-element list. `names[:2]` restricts the global level to the override and `AGENTS.md` (fallbacks are a per-directory feature in the docs' description).
- `rel = cwd.relative_to(project_root).parts`: e.g. `('services', 'payments')`.
- `dirs = [project_root] + [project_root.joinpath(*rel[:i + 1]) for i in range(len(rel))]`: a list comprehension building `repo`, `repo/services`, `repo/services/payments`. `*rel[:i+1]` unpacks a tuple slice into separate arguments.
- The loop adds one file per directory, root first, and `break`s when the byte budget would be exceeded ("stops adding files once the combined size reaches the limit").

**What the output shows:** three files; `repo/services/payments/AGENTS.md` was **not** loaded because `AGENTS.override.md` exists there. In Claude Code, by contrast, both `CLAUDE.md` and `CLAUDE.local.md` in the same directory would load. Same idea (layered instructions), different rules.

### 4.4 Codex in CI with a schema-checked result

`codex exec` plus `--output-schema` turns an agent run into a typed function call, which is exactly what a pipeline wants:

```bash
# Not run in the sandbox (needs Codex + credentials). Flags per the Codex non-interactive docs, Nov 2026.
cat > review.schema.json <<'EOF'
{"type": "object",
 "properties": {"verdict": {"type": "string", "enum": ["approve", "request_changes"]},
                "issues": {"type": "array", "items": {"type": "string"}}},
 "required": ["verdict", "issues"], "additionalProperties": false}
EOF
codex exec --output-schema review.schema.json --json \
  "Review the diff in this checkout for bugs and missing tests" > events.jsonl
```

It runs read-only by default, which is the right default for review. Only add `--sandbox workspace-write` for jobs that must edit (e.g. "autofix CI failures", a documented example).

### 4.5 The comparison that matters

| Concern | Claude Code | Cursor | Codex |
|---|---|---|---|
| Native instruction file | `CLAUDE.md` (+ `.claude/rules/`) | `.cursor/rules/*.mdc`, `AGENTS.md` | `AGENTS.md` |
| Multiple files per directory | `CLAUDE.md` + `CLAUDE.local.md` both load | Rules are separate files with their own triggers | One per directory; override hides `AGENTS.md` |
| Conflict model | Concatenate root→cwd | More specific nested AGENTS.md takes precedence | Later (closer) overrides; size budget |
| Hard safety | Permission rules + optional sandbox + hooks | Approvals/run modes, checkpoints | OS sandbox modes + approval policy |
| Procedures | Skills in `.claude/skills/` (Agent Skills standard) | Rules (manual/intelligent) | Skills in `.agents/skills/` (Agent Skills standard; `/skills` or `$` to mention) |
| Headless | `claude -p` | `agent -p` | `codex exec` |
| MCP config | `claude mcp add`, `.mcp.json` | `.cursor/mcp.json` | `[mcp_servers.x]` in TOML |

### 4.6 A portable repository setup

Goal: one source of truth, all three tools happy.

```text
repo/
├── AGENTS.md                  # the single source of truth (commands, conventions, gotchas)
├── CLAUDE.md                  # one line: @AGENTS.md  (+ Claude-specific notes if any)
├── .cursor/rules/
│   └── python-style.mdc       # only file-scoped rules that need globs
├── .claude/
│   ├── settings.json          # permissions + hooks (team-shared)
│   └── skills/pr-review/SKILL.md   # Claude Code skill path
├── .agents/skills/pr-review/SKILL.md  # same skill for Codex (copy or symlink)
├── .mcp.json                  # Claude Code project MCP servers
├── .cursor/mcp.json           # same servers, Cursor dialect
└── .codex/config.toml         # same servers, Codex dialect (trusted projects only)
```

Rules of thumb: keep `AGENTS.md` short and tool-neutral; put tool-specific behaviour in the tool's own file; generate the three MCP configs from one source (a tiny script) so they never drift; review all of them in PRs like code.

## 5. Common mistakes

1. **`.md` files in `.cursor/rules`.** Ignored; use `.mdc` with frontmatter.
2. **Forgetting an `AGENTS.override.md`** left over from an incident: it silently replaces that directory's `AGENTS.md` in Codex.
3. **Huge `AGENTS.md` in Codex.** Past `project_doc_max_bytes` (32 KiB default), later files are dropped.
4. **`danger-full-access` because the sandbox "got in the way".** Fix the specific need (writable roots, network) instead.
5. **Copying `--full-auto` from old blog posts.** No longer supported per the docs.
6. **Treating Bugbot/AI review as sufficient review.** It is a second reader.
7. **Drifting MCP configs** across three files.
8. **Assuming identical instruction semantics** across tools.

## 6. Connections to other tracks

- **OS**: Seatbelt, bubblewrap and Windows sandboxing are OS isolation primitives (namespaces, MAC policies).
- **Cloud/DevOps**: `codex exec --json` and `agent -p` slot into CI like any CLI; JSONL event streams are log pipelines.
- **Git**: checkpoints vs commits; worktrees for parallel agents; reviewing config as code.
- **System Design**: single source of truth with generated adapters is the same pattern as generating client SDKs from one API schema.

## 7. Where this shows up in my projects

- **compiler-fuzz** is a strong fit for `codex exec` in CI: a nightly job in `workspace-write` that reproduces a reported GCC/Clang mismatch, runs the ddmin reducer, and emits a schema-validated JSON report (`{seed, compilers, opt_levels, reduced_path}`). The sandbox keeps a runaway reduction from touching anything outside the checkout.
- **Nidhogg** has frontend and backend parts; a nested `AGENTS.md` per part (or Cursor rules with globs) keeps Elasticsearch query conventions out of UI sessions and vice versa.

## 8. Self-check questions

1. Your Cursor rule `.cursor/rules/api.md` never applies. Why?
2. In Codex, `services/payments/` has both `AGENTS.md` and `AGENTS.override.md`. Which loads?
3. What sandbox does `codex exec` use by default, and why is that a good default?
4. Name one way Claude Code's instruction loading differs from Codex's.
5. How do you keep three MCP config dialects consistent?

<details><summary>Answers</summary>

1. Project rules must be `.mdc` files with frontmatter; a plain `.md` in `.cursor/rules` is ignored.
2. Only `AGENTS.override.md` (one file per directory, override first).
3. Read-only; non-interactive runs have no human to approve risky actions, so the least-privilege default prevents unintended edits; opt into `workspace-write` per job.
4. Claude Code concatenates every CLAUDE.md and CLAUDE.local.md from root to cwd; Codex takes at most one file per directory (override wins) and stops at a byte budget.
5. Treat one file (or a small script's data) as the source of truth and generate `.mcp.json`, `.cursor/mcp.json` and `config.toml` from it; review in PRs.

</details>

## 9. Mnemonic (Tamil + English)

**"Cursor-ku MDC, Codex-ku ORE ORU, Claude-ku ELLAME."**
(கர்சருக்கு MDC, கோடெக்ஸுக்கு ஒரே ஒரு, க்ளாடுக்கு எல்லாமே)
- Cursor rules need **.mdc**.
- Codex loads **ore oru** (just one) file per directory.
- Claude Code loads **ellame** (all of them), concatenated.

## Sources

- Cursor, Rules (as of Nov 2026): https://cursor.com/docs/context/rules
- Cursor, Agent help page (modes, checkpoints, Bugbot, Cloud Agents) (as of Nov 2026): https://cursor.com/help/ai-features/agent
- Cursor, Plan mode / modes (as of Nov 2026): https://cursor.com/docs/agent/modes
- Cursor, MCP (as of Nov 2026): https://cursor.com/docs/context/mcp
- Cursor, CLI overview (as of Nov 2026): https://cursor.com/docs/cli/overview
- OpenAI Codex CLI repository (as of Nov 2026): https://github.com/openai/codex
- Codex, AGENTS.md (as of Nov 2026): https://learn.chatgpt.com/codex/agent-configuration/agents-md
- Codex, Sandboxing (as of Nov 2026): https://learn.chatgpt.com/codex/sandboxing
- Codex, Config basics (as of Nov 2026): https://learn.chatgpt.com/codex/config-file/config-basic
- Codex, Non-interactive mode (as of Nov 2026): https://learn.chatgpt.com/docs/non-interactive-mode
- Codex, MCP (as of Nov 2026): https://learn.chatgpt.com/codex/extend/mcp
- Codex, Skills (as of Nov 2026): https://learn.chatgpt.com/codex/skills
- AGENTS.md format site: https://agents.md/

## Verify yourself

1. Run `python3 codex_agents_md.py`; delete the override file in the simulation and predict, then confirm, which payments file loads.
2. With Codex installed, run `codex exec "List active instruction sources"` in a nested directory (the docs use this exact prompt) and compare with the simulation.
3. In Cursor, create both `.cursor/rules/x.md` and `.cursor/rules/x.mdc` with different content; ask the agent which rules it sees.
4. Validate the TOML in this lesson: `python3 -c "import tomllib,sys; tomllib.loads(open(sys.argv[1]).read())" config.toml`.
5. Check the Codex skills page for the current discovery paths (`.agents/skills`, `$HOME/.agents/skills`, `/etc/codex/skills` as of Nov 2026) and copy the `pr-review` skill from 02-intermediate/05 into `.agents/skills/` to share it between Claude Code and Codex.
