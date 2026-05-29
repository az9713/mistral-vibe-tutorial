# The system prompt

The first message of every conversation is the system prompt. Everything the model thinks it "knows" about its job — its role, its rules, its tools, its environment, the project it's working on — comes from this single string.

This doc is about *how that string is assembled*.

## The pattern

A production system prompt isn't a single template — it's a layered concatenation of:

```
Base prompt (the model's role and behavior)
+ Run-mode overrides (e.g. headless)
+ Environment context (OS, shell, model name)
+ Tool prompts (per-tool instructions)
+ Skills index
+ Subagents index
+ Scratchpad info
+ Project context (cwd + git status)
+ User-level AGENTS.md
+ Project-level AGENTS.md
```

Each layer is conditional. Each is a function of state that can change during the session. So the system prompt must be *rebuildable* — and Vibe does rebuild it on agent switch, on skill install, on MCP server hot-reload.

## Vibe's implementation

The single entry point is `get_universal_system_prompt(...)` at `vibe/core/system_prompt.py:get_universal_system_prompt`. Its signature:

```python
def get_universal_system_prompt(
    tool_manager: ToolManager,
    config: VibeConfig,
    skill_manager: SkillManager,
    agent_manager: AgentManager,
    *,
    include_git_status: bool = True,
    scratchpad_dir: Path | None = None,
    headless: bool = False,
    experiment_manager: ExperimentManager | None = None,
) -> str:
```

It returns a `str`. The result becomes the content of the system message in `self.messages[0]`. The harness calls it on:

- Initial `AgentLoop.__init__` (line 311).
- After deferred MCP integration completes (`_complete_init`, line 407).
- After an agent profile switch (`refresh_system_prompt` → called from `switch_agent` / `reload_with_initial_messages`).
- After experiments hydrate.

Every rebuild calls `messages.update_system_prompt(new_prompt)` which mutates `messages[0]` in place.

## What's in each section

### 1. Base prompt

Chosen by `_resolve_system_prompt(config, experiment_manager)`:

- If the user has set `system_prompt_id` to anything other than the default, use that.
- Otherwise, an experiment may pick a variant id.
- Otherwise, the default — `cli.md`.

The default prompt itself (`vibe/core/prompts/cli.md`) tells the model:

- It is Mistral Vibe, a CLI coding agent.
- Today's date (interpolated via `${current_date}`).
- A three-phase workflow: Orient → Plan → Execute & Verify.
- Hard rules: never commit proactively, respect user constraints, don't remove what wasn't asked, don't assert without verifying, break loops after 2 failed attempts.
- Response format rules: minimal, structure-first, no greetings or puffery.

Other prompts exist for specific modes: `lean.md`, `minimal.md`, `explore.md` (the subagent prompt), `tests.md`.

### 2. Headless mode banner

Only added when `headless=True`. Tells the model: no human will respond — don't ask questions, finish the task in one pass. From `_get_headless_section()`.

This is the difference between an interactive shell and `vibe -p "..."`.

### 3. Commit signature reminder

If `config.include_commit_signature`, add a block telling the model the exact heredoc format for `git commit` so commits include `Co-Authored-By: Mistral Vibe`. From `_add_commit_signature()`.

### 4. Model name disclosure

If `config.include_model_info`, add `Your model name is: <name>`. Helps debugging and lets the model adapt (e.g. "I can't read images" if it's a text-only model).

### 5. OS-specific prompt

If `config.include_prompt_detail`, add:

```
The operating system is <platform> with shell `<shell>`
```

On Windows, append a block of compatibility rules: use `dir` not `ls`, backslash paths, use `where` not `which`, no shebangs. From `_get_windows_system_prompt()`.

### 6. Tool prompts

For each tool the model will see in its tool inventory, append its `get_tool_prompt()` text. This is the `*.md` file next to each tool's `.py` source — e.g. `bash.py` → `prompts/bash.md`. Joined with `---` separators.

`get_tool_prompt` is `@functools.cache`d (`tools/base.py:144`), so the prompts are loaded once per process.

### 7. Skills index

If any skills are available, list them in XML-ish format:

```xml
<available_skills>
  <skill>
    <name>code-review</name>
    <description>...</description>
    <path>/path/to/SKILL.md</path>
  </skill>
  ...
</available_skills>
```

The model can then decide to invoke a skill via the `skill` tool. See [`skills.md`](skills.md).

### 8. Subagents index

If any agent profiles have `agent_type = subagent`, list them. The model can delegate to them via the `task` tool. See [`agents-and-subagents.md`](agents-and-subagents.md).

### 9. Scratchpad info

If a scratchpad dir was created, tell the model:

> You have a scratchpad directory at: `/tmp/vibe-scratchpad-...`
> Use this for temporary files: intermediate results, draft scripts, working files…
> Files here are automatically allowed — no permission prompts.

From `_get_scratchpad_section()`. The same dir is passed to subagents.

### 10. Project context

If `config.include_project_context`:

- First check `is_dangerous_directory()` (running from `/`, `$HOME`, etc.). If so, inline the `dangerous_directory.md` warning template instead of the normal context — telling the model to be extra cautious.
- Otherwise, `ProjectContextProvider.get_full_context()`:
  - Absolute path of the project root.
  - `git status` (branch, main branch, status, recent commits) — gathered via parallel `git` subprocesses with a timeout.
- If extra working directories were registered, list them.

### 11. AGENTS.md files

The harness checks `get_harness_files_manager()` for:

- `~/.vibe/AGENTS.md` → user-level instructions.
- `<project>/AGENTS.md` (and additional working dirs) → project-level instructions.

Each gets inlined verbatim into a wrapped block via the `agents_doc.md` utility template:

```
## User instructions
Contents of /home/.../.vibe/AGENTS.md (user-level instructions):
<file content>

## Project instructions (checked into the codebase)
Contents of /project/AGENTS.md:
<file content>
```

These are the project's persistent "system prompt for this repo" — see [`memory-and-state.md`](memory-and-state.md).

## The cli.md prompt — what it actually says

The default base prompt (cli.md, ~80 lines) compresses a lot of agent-design opinion into one document. Read it in full at [`vibe/core/prompts/cli.md`](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/prompts/cli.md). Highlights:

- **Phase 1 — Orient.** Restate the goal. Classify the task as Investigate vs. Change. If unclear, default to Investigate. Don't edit a file you haven't read this session.
- **Phase 2 — Plan (Change tasks only).** State files to change and the change per file before writing code.
- **Phase 3 — Execute & Verify.** Edit one logical unit, then read back or run tests. Never claim completion without verification.
- **Hard Rules:**
  - Never commit unless asked.
  - "No writes" / "just analyze" / "don't touch X" are hard constraints.
  - Don't remove what wasn't asked.
  - Don't assert; verify with a tool.
  - Break loops after 2 failed attempts at the same region.
- **Response format:** minimal (<150 words usually), structure-first (code/diagram/table before prose), no greetings, no puffery, no emoji.

A surprising amount of the agent's *behavior* comes from this file. If you fork Vibe and want a different agent personality, this is where you start.

## Why rebuild instead of patch?

A natural temptation is to *append* state as the session evolves — e.g. "here's a new tool that just got registered, model, FYI." Vibe doesn't. It rebuilds the system message from scratch and replaces it.

Reasons:

1. **Determinism.** The system prompt is a pure function of (config, agent, skills, tools, cwd). Rebuild → trivially correct. Patch → bug-prone.
2. **Cache-friendliness.** Most providers cache prompt prefixes. If the system prompt is structurally stable (only the *content* of, say, the skills section changes), the cache key reflects that. A patched-up running text is harder for the cache to handle.
3. **Hot-reload of MCP servers / skills / agents.** A user adds a new skill mid-session — the next turn's system prompt should mention it. Rebuild does this for free; patch needs an event bus.

The cost is that on rebuild, the *messages* in history don't change — they still reference the old system prompt's framing. In practice the model is robust to this because the rebuild only adds/removes auxiliary sections, never contradicts the base prompt.

## Try it: see the prompt your agent is using

The system prompt is `messages[0].content`. In a running session you can dump it from the debug console (Ctrl+\ in the TUI) or from a programmatic script:

```python
from vibe.core.agent_loop import AgentLoop
from vibe.core.config import VibeConfig
loop = AgentLoop(VibeConfig.load())
print(loop.messages[0].content)
```

You'll see all the sections above concatenated, in order, exactly as the model sees them. This is the single most useful thing you can read to understand what *your* agent is acting on.
