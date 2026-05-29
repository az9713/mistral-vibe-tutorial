# Mistral Vibe — Agent Harness Learning Workspace

This directory is **both** the upstream [mistralai/mistral-vibe](https://github.com/mistralai/mistral-vibe) source tree **and** the user's personal learning workspace. The goal is to understand how a production CLI coding-agent harness is built, using Vibe as the worked example, and to eventually build on top of it.

After working in a parent workspace, the user consolidated everything into this directory. The Vibe source is *here*, not in a nested subdirectory.

## Directory layout

```
mistral-vibe-main/   ← working directory (you are here); also the Vibe repo root
├── CLAUDE.md                ← this briefing — read first every session
├── AGENTS.md                ← Vibe's own contributor conventions (upstream)
├── README.md, CHANGELOG.md, CONTRIBUTING.md, LICENSE   ← upstream meta
├── pyproject.toml, uv.lock, flake.nix, action.yml      ← upstream build
│
├── vibe/                    ← the Vibe source itself
│   ├── core/                ← the harness — what we're studying
│   │   ├── agent_loop.py        ← main loop
│   │   ├── middleware.py
│   │   ├── compaction.py
│   │   ├── system_prompt.py
│   │   ├── tools/               ← BaseTool + builtins/
│   │   ├── agents/              ← profiles + manager
│   │   ├── skills/, hooks/, session/, prompts/, llm/, ...
│   ├── cli/                 ← Textual TUI (a consumer of the harness)
│   ├── acp/                 ← Agent Client Protocol bridge
│   └── setup/               ← first-run wizard
│
├── docs/                    ← Vibe's *upstream* docs (acp-setup.md, proxy-setup.md, README.md) — short, not the learning material
│
├── docs_deepdive/            ← the LEARNING material — 17 files, generated in a prior session
│   ├── index.md                 ← entry point
│   ├── learning-path.md         ← ordered reading plan for the source
│   ├── overview/                ← what-is-an-agent-harness, key-concepts
│   ├── concepts/                ← agent-loop, system-prompt, tools, permissions,
│   │                                middleware, context-management, memory-and-state,
│   │                                agents-and-subagents, skills, hooks, human-in-the-loop
│   └── architecture/            ← data-flow, design-decisions
│
├── tests/                   ← upstream test suite
├── scripts/, distribution/, pyinstaller/  ← upstream packaging
└── .github/, .vscode/       ← upstream meta
```

The `docs_deepdive/` directory is the canonical learning material — refer to it before re-deriving anything about Vibe's internals. All `file:line` references inside those docs are relative to *this* directory, so they resolve as-is (e.g. `vibe/core/agent_loop.py:874` is `./vibe/core/agent_loop.py` line 874).

## What Vibe is (in one paragraph)

Mistral Vibe is an open-source Python 3.12+ CLI coding agent (MIT licensed) installed via `uv tool install mistral-vibe` (binary: `vibe`). Its harness lives in `vibe/core/` (~30 files) and covers: a streaming async-generator agent loop, a Pydantic-typed tool system with filesystem-discovered subclasses, three-level permissions with per-call overrides, a middleware pipeline that gates each turn, summarize-and-restart context compaction, subagents spawned as a tool, slash-command skills, post-turn hooks, session persistence, multi-provider LLM backend, and MCP integration. Default models are Mistral's; the harness is multi-provider via `BackendLike`.

## The user's goal

Learn agent harness design well enough to build their own and/or build on top of Vibe. This is a self-directed study project; the user follows the study plan below at their own pace.

## Study plan (6 phases, ~30-40 hours)

The full plan with deliverables and "done when" checks is the canonical reference. Execute phases in order; each phase has a verification step — don't skip them.

### Phase 0 — Setup (1h)
Install Vibe (`uv tool install mistral-vibe`), set `MISTRAL_API_KEY` in `~/.vibe/.env`, clone `huggingface/smolagents` for later contrast. **Done when:** `vibe` runs interactively and the user can find `~/.vibe/sessions/`.

### Phase 1 — Mental model (3h)
Read `docs_deepdive/overview/what-is-an-agent-harness.md`, `docs_deepdive/overview/key-concepts.md`, `docs_deepdive/architecture/data-flow.md`, then `vibe/core/prompts/cli.md` directly. Run a Vibe session and observe each event. **Output:** hand-drawn one-page loop sketch (no notes). **Done when:** can explain in 90 seconds with no notes — what is the loop, what are the four phases of a turn, why does it stop.

### Phase 2 — Read Vibe's core (10h)
Follow `docs_deepdive/learning-path.md` step by step (steps 0–20). After each step, write one sentence on what surprised you. Milestones: after step 4 (`vibe/core/tools/base.py`) can sketch `BaseTool` signature from memory; after step 9 (`vibe/core/middleware.py`) can name all 6 built-in middlewares; after step 16 (`vibe/core/agent_loop.py`) can draw `_conversation_loop` state machine. **Output:** 1-page explanation of one tool's full lifecycle with `file:line` refs. **Done when:** can answer all six "Try it" prompts at the end of `learning-path.md` without looking anything up.

### Phase 3 — Compare (4h)
- smolagents (~2h): read `src/smolagents/agents.py`. Notice code-action vs. tool-call style — why?
- Aider (~1.5h): skim `aider/coders/base_coder.py` + one edit-format coder. Notice no MCP, no skills, no subagents, but repo maps + edit-format negotiation.
- Claude Code docs (~0.5h): read sub-agents and agent-teams pages.

**Output:** 4-column comparison table (Vibe / smolagents / Aider / Claude Code) × subsystems. **Done when:** can argue with evidence *why* Vibe chose its shape.

### Phase 4 — Build minimal (12h)
Write a tiny CLI coding agent from scratch, any language, **under 500 LOC core**. Must have: turn loop with history, three tools (`read_file`, `write_file`, `bash`), per-tool permission (ALWAYS/ASK/NEVER), assembled system prompt, streamed events. Must NOT have (resist): middleware, hooks, MCP, skills, subagents, compaction. Suggested start: copy the trace from `docs_deepdive/architecture/data-flow.md` as comments in an empty file, then fill in code. **Done when:** agent completes a 2-step task with permission prompts working.

### Phase 5 — Extend (8h)
Pick **one** advanced feature and add it to the harness from Phase 4:
- **Compaction** (most consequential) — most insight per hour.
- **Subagent via `task` tool** — teaches scoping/isolation.
- **Hooks** — teaches exit-code-as-protocol.
- **An MCP client** — biggest payoff for a real harness.

Don't try more than one. Re-read the corresponding `docs_deepdive/concepts/<name>.md` before coding. **Done when:** the feature works end-to-end; user can articulate in writing what was harder than expected and which Vibe design choice now makes sense because of it.

### Phase 6 — Synthesize (3h)
Write one of: blog post comparing your harness to Vibe; design doc for "the harness I'd build for my project"; talk-deck for a 30-min internal explainer. **Done when:** another engineer reads it and asks a question you can answer.

### Common failure modes
- Skipping Phase 1 because "I already know what an agent is." Don't.
- Skipping Phase 4 because "I'll keep reading." Half the value is in building.
- Going wide in Phase 5. Pick one feature.
- Not writing Phase 6. Writing is the integration step.

### Concrete tip
Set up editable install + jump-to-symbol in your editor from this directory (`uv sync --all-extras` then open in your editor). During Phase 4 you'll constantly want to peek at the Vibe source — make peeking instant.

## Building on top of Vibe (three modes)

### Mode 1 — Extend in-place via plugins
**Verdict: yes, excellent. Default choice.** No fork. Drop files in `~/.vibe/` (personal) or `.vibe/` (project):

| Want to add | Drop in | Format |
|---|---|---|
| A new tool | `~/.vibe/tools/mytool.py` | `BaseTool` subclass |
| A new agent profile | `~/.vibe/agents/mymode.toml` | TOML overrides |
| A new skill | `~/.vibe/skills/myskill/SKILL.md` | Markdown + frontmatter |
| A hook | TOML config | Any shell command |
| External tools | `mcp_servers` in `config.toml` | MCP server spec |
| Custom system prompt | `system_prompt_id` config + a prompt file | Markdown |

Filesystem-discovered, no core changes, no merge conflicts on upstream updates.

### Mode 2 — Fork and modify the core
**Verdict: workable but expensive.**

Pros: MIT, type-strict, well-tested, multi-provider backend abstraction (`vibe/core/llm/backend/`) is real.

Cons: Mistral-specific code (`vibe/core/nuage/`, `vibe/core/teleport/`, `vibe/core/experiments/`, telemetry) is woven through — gating/ripping is non-trivial. Mistral defaults bake in (models, `vibe/core/prompts/cli.md` personality, commit signature). Research-quality codebase = no API stability = drift cost on upstream merges. Python + `uv` is mandatory.

**Rule: fork late.** Spend Phase 5 adding the feature as a plugin first. Only fork when you hit something a plugin genuinely can't do (new middleware type, new conversation shape, compliance-driven removal of Mistral paths).

If you do fork: this directory is already a clone of `main`. Add your remote, create a branch off `main`, and start there.

### Mode 3 — Use as a library / SDK in another product
**Verdict: poor. Use a different tool.**

Vibe is a CLI application, not a library. No `pip install vibe-core` separation. `AgentLoop` (in `vibe/core/agent_loop.py`) is importable but drags CLI/Textual/telemetry deps. No documented public API. For library use: Anthropic Agent SDK, OpenAI Agents SDK, or smolagents.

### Recommendation
**Live in Mode 1 for at least 3 months.** Most "I need to fork" instincts dissolve after 2-3 custom tools and an agent profile. Move to Mode 2 only on a hard plugin-limit. If you find yourself wanting Mode 3, switch tools.

## Key source pointers (all paths relative to this directory)

| Subsystem | File |
|---|---|
| Main loop | `vibe/core/agent_loop.py` (`_conversation_loop` ~line 874) |
| Tool base class | `vibe/core/tools/base.py` |
| Built-in tools | `vibe/core/tools/builtins/` |
| Middleware pipeline | `vibe/core/middleware.py` |
| Compaction | `vibe/core/compaction.py` + `agent_loop.compact()` |
| System prompt assembly | `vibe/core/system_prompt.py` |
| Agent profiles | `vibe/core/agents/models.py` |
| Default system prompt text | `vibe/core/prompts/cli.md` |
| Subagent spawn | `vibe/core/tools/builtins/task.py` |
| Skills | `vibe/core/skills/manager.py` |
| Hooks | `vibe/core/hooks/manager.py` |
| Session persistence | `vibe/core/session/session_logger.py` |

## Workflow ideas on the shelf

Six Claude Code dynamic-workflow designs for this codebase are persisted in memory at `project-workflow-ideas.md`. They are designed but not yet run. If the user asks about workflow ideas or wants to kick one off, read that memory entry first — it has the full shape, fan-out, and "best for" pick per idea. The titles, for orientation:

1. **Comprehension audit + doc-drift sweep** — surfaces stale claims in `docs_deepdive/` and surprising lines in `vibe/core/`.
2. **Type-escape sweep** — finds and adversarially verifies every `cast`/`# type: ignore`/`Any`/`getattr(x, "f", default)` in the codebase. Best showcase of Opus 4.8's improved honesty.
3. **Fork-readiness map** — quantifies cost-to-fork by mapping Mistral-specific code (`nuage/`, `teleport/`, `experiments/`, telemetry).
4. **Plugin gallery generator** — writes 10 working Vibe plugins in parallel with worktree isolation.
5. **Cross-implementation pattern catalog** — Vibe + smolagents + Aider read in parallel per subsystem axis. Compresses Phase 3 of the study plan into one workflow run.
6. **Vibe-on-Opus-4.8 regression eval** — checks whether `cli.md` still produces intended behavior on the new model.

To invoke any of these: include the word `workflow` in your prompt and describe the task. Claude writes the script; approve to run. After a successful run, press `s` in `/workflows` to save as `/<name>`.

## Two AGENTS.md files in scope

This is the Vibe repo root, so when you run `vibe` here, the harness inlines `./AGENTS.md` into its system prompt — that's *Vibe's own* contributor conventions (`uv` usage, no relative imports, Pyright strict, etc.). Don't confuse it with this `CLAUDE.md`, which only Claude Code reads.

If you want Vibe to know about your *learning* goals, add a personal `~/.vibe/AGENTS.md` (user-level, loaded in every Vibe session in every project).

## Working conventions for this project

- Source of truth for Vibe internals is `docs_deepdive/` first, then the source itself. Don't re-derive; reference.
- When referring to Vibe code, use `file:line` form so the user can navigate. Paths are relative to this directory.
- The user wants concise, structure-first responses — lead with code/table/diagram, prose second. No greetings, no puffery, no emoji.
- The Vibe source is *here*, not in a subdirectory. Read files directly; don't re-fetch from GitHub unless checking for upstream changes.
- Don't edit upstream files (`vibe/`, `tests/`, `pyproject.toml`, etc.) unless the user explicitly asks for a code modification — by default this is read-only learning territory.

## What this is NOT

This directory is **not** related to Claude Code's dynamic workflows, ultracode, or `/deep-research` — those are Anthropic / Claude Code features that came up briefly in a prior session and were removed by user request. If the user mentions "workflow", treat it as a generic word unless they specifically reference Claude Code.
