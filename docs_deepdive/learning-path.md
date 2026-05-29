# Learning path through the source

You've read the concepts. Now read the code. This is the order to do it in.

The path is designed so each file builds on what you've already read. By the end you'll have the entire harness in your head.

Time budget: ~3-4 hours of focused reading. Less if you skim, more if you trace details.

## Step 0 — Get the source

```bash
git clone https://github.com/mistralai/mistral-vibe.git
cd mistral-vibe
```

Optionally:

```bash
uv sync --all-extras
```

So you can also *run* it.

## Step 1 — `AGENTS.md` (5 min)

The project's own conventions. Read this once. It tells you:

- Where things live (`vibe/core` = engine, `vibe/cli` = TUI, `vibe/acp` = IDE bridge).
- The coding style you'll see (Pydantic, `match`/`case`, modern type hints, no relative imports).
- The test posture.
- Commands you'll see referenced.

This is also a *worked example of what an AGENTS.md looks like in practice*. Compare with what we covered in [`concepts/memory-and-state.md`](concepts/memory-and-state.md).

## Step 2 — `vibe/core/prompts/cli.md` (5 min)

The default system prompt. This is what the agent thinks its job is. Notice:

- The three-phase framing (Orient → Plan → Execute & Verify).
- The "Hard Rules" section is the harness's enforced behavior preferences.
- The Response Format rules dictate how the model talks. The user sees this as "agent personality" — the agent reads it as constraints.

Don't read any other prompts yet. Just this one.

## Step 3 — `vibe/core/types.py` (10 min)

The vocabulary. Skim this file once. You don't have to internalize everything but you need to know what `LLMMessage`, `Role`, `BaseEvent`, `MessageList`, `AgentStats`, `ApprovalCallback` look like.

This is the same role the [`overview/key-concepts.md`](overview/key-concepts.md) glossary plays — but with types. Both are useful.

## Step 4 — `vibe/core/tools/base.py` (20 min)

Read in order:

1. `InvokeContext` (dataclass at the top) — what tools can access.
2. `ToolPermission` enum.
3. `BaseToolConfig` and `BaseToolState`.
4. `BaseTool` — focus on:
   - `__init__` signature.
   - The abstract `run` signature.
   - `invoke` (calls validate, then run).
   - `_get_tool_args_results` / `_get_tool_config_class` / `_get_tool_state_class` — how the four type variables get extracted from the subclass.
   - `resolve_permission` and `get_file_snapshot` (extension hooks).

Don't read `_extract_result_type` in detail — just know it pulls the result type out of `AsyncGenerator[ToolStreamEvent | YourResult, None]`.

By the end, you should be able to sketch the signature of a minimal tool class on paper.

## Step 5 — `vibe/core/tools/builtins/todo.py` (10 min)

The minimum-viable tool. Notice:

- `TodoArgs`, `TodoResult`, `TodoConfig`, `TodoState` — the four type variables.
- `Todo(BaseTool[TodoArgs, TodoResult, TodoConfig, TodoState], ToolUIData[...])` — concrete instantiation.
- `run` matches on `args.action` and yields one result.
- State (`self.state.todos`) persists across calls because the tool instance is reused.

You've now seen the simplest end-to-end tool.

## Step 6 — `vibe/core/tools/builtins/read_file.py` and `bash.py` (20 min)

Read `read_file.py` first. It's slightly more complex than `todo` (touches the filesystem, can return long content) but still straightforward.

Then `bash.py`. This is the most complex tool in the codebase. Focus on:

- `_extract_commands` — parses the bash command via tree-sitter to handle pipes, &&, etc.
- `resolve_permission` — the per-call permission logic that consults the allowlist and denylist.
- The `run` method — subprocess management, streaming output, cancellation handling.

After `bash.py`, you'll understand the full breadth of what a tool can do.

## Step 7 — `vibe/core/tools/permissions.py` (15 min)

Short file. Read end-to-end:

- `PermissionScope` enum.
- `RequiredPermission` and `PermissionContext` (a tool's per-call result).
- `ApprovedRule` (what a "always allow" creates).
- `PermissionStore` (the in-session memory of approvals).
- `wildcard_match` (the pattern-matching used to decide if a rule covers a request).

You can now trace a tool call's permission decision from top to bottom.

## Step 8 — `vibe/core/tools/manager.py` (15 min)

The tool registry. Focus on:

- `_compute_search_paths` — where tools are found.
- `_iter_tool_classes` and `_load_tools_from_file` — dynamic module import.
- `available_tools` property — applies enable/disable filters.
- `_apply_per_source_filtering` — per-MCP-server, per-connector filtering.
- `integrate_mcp` (sync wrapper) — MCP discovery.

You don't need to read `_compute_module_name` in detail. Just know it dodges Pydantic class-identity collisions.

## Step 9 — `vibe/core/middleware.py` (20 min)

Read the whole file. It's small and self-contained. You'll see:

- `ConversationContext` and `MiddlewareResult` types.
- Each built-in middleware (TurnLimit, PriceLimit, TokenLimit, AutoCompact, ContextWarning, ReadOnlyAgent).
- `make_plan_agent_reminder` — generated text injected when entering plan mode.
- `CHAT_AGENT_REMINDER` and `CHAT_AGENT_EXIT` constants.
- `MiddlewarePipeline.run_before_turn` — the pipeline runner.

You can now picture every check that runs before each turn.

## Step 10 — `vibe/core/agents/models.py` (15 min)

Read the whole file:

- `AgentSafety`, `AgentType`, `BuiltinAgentName` enums.
- `AgentProfile` and `apply_to_config`.
- Each built-in profile (`DEFAULT`, `PLAN`, `CHAT`, `ACCEPT_EDITS`, `AUTO_APPROVE`, `EXPLORE`, `LEAN`).
- `_plan_overrides` — see how the plan agent restricts write_file to the plan dir.

Then skim `agents/manager.py` — it's mostly discovery + filtering, similar shape to the tool manager.

## Step 11 — `vibe/core/system_prompt.py` (20 min)

Read end-to-end. You'll see:

- `ProjectContextProvider` — gathers git status in parallel.
- `_get_default_shell`, `_get_os_system_prompt`, `_get_windows_system_prompt`.
- `_add_commit_signature`, `_get_available_skills_section`, `_get_available_subagents_section`, `_get_scratchpad_section`.
- `_resolve_system_prompt` — handles experiments and overrides.
- `get_universal_system_prompt` — the assembly function.

Trace through `get_universal_system_prompt` once carefully. You'll see how the 11 sections from [`concepts/system-prompt.md`](concepts/system-prompt.md) get concatenated.

## Step 12 — `vibe/core/compaction.py` (5 min)

Trivial file. One function: `collect_prior_user_messages`. Walks newest-first, drops injected and prior summaries, middle-truncates the overflow message. Read it once.

## Step 13 — `vibe/core/hooks/` (15 min)

Read in order:

1. `models.py` — `HookType`, `HookConfig`, `HookInvocation`, `HookExecutionResult`, `HookUserMessage`.
2. `manager.py` — `HooksManager.run` is the loop you read.
3. `executor.py` — the subprocess plumbing. Skim.

Now you've seen the entire hook subsystem.

## Step 14 — `vibe/core/skills/` (15 min)

Read:

1. `models.py` — `SkillMetadata`, `SkillInfo`, `ParsedSkillCommand`.
2. `parser.py` — frontmatter + body splitter.
3. `manager.py` — `SkillManager.parse_skill_command` and `build_skill_prompt` show the slash-command flow.

## Step 15 — `vibe/core/scratchpad.py` (5 min)

Two functions: `init_scratchpad`, `is_scratchpad_path`. Read once.

## Step 16 — `vibe/core/agent_loop.py` (45 min)

The big one. Read in this order:

1. `__init__` (line 234) — what gets initialized.
2. `_setup_middleware` (line 738) — where the pipeline is built.
3. `act` (line 646) — entry point.
4. `_conversation_loop` (line 874) — the main loop. The single most important method in the codebase. Read carefully.
5. `_perform_llm_turn` (line 988) — model call + tool resolution.
6. `_handle_tool_calls` and `_run_tools_concurrently` (line 1208+) — concurrent execution.
7. `_execute_tool_call` (line 1101) — the per-call lifecycle.
8. `_should_execute_tool` and `_ask_approval` (line 1469+) — permission decision.
9. `_chat` (line 1325) — the actual API call. Read.
10. `compact` (line 1686) — the compaction implementation.
11. `switch_agent` (line 1764) and `reload_with_initial_messages` — agent switching.
12. `fork` (line 1607) — message-id-anchored forking.

The other methods are scaffolding — telemetry, experiments, deferred init. You can skim or skip.

By the end, the entire agent loop should be in your head as a single trace.

## Step 17 — `vibe/core/tools/builtins/task.py` (15 min)

Read end-to-end. You've seen everything it touches by now. Trace how it spawns a fresh `AgentLoop(is_subagent=True)`, hooks up callbacks, propagates the scratchpad, and converts subagent events to parent tool stream events.

## Step 18 — `vibe/core/session/session_logger.py` (15 min)

The persistence layer. Skim it. Notice:

- `save_interaction` is idempotent and atomic.
- Each session gets a directory.
- Subagent sessions nest under the parent's dir.

You don't have to internalize this — just know it's where messages are persisted.

## Step 19 (optional) — `vibe/core/llm/format.py` (20 min)

The translation layer between the harness's `LLMMessage` model and provider-specific tool-call formats. Read if you want to understand cross-provider compatibility. Skip if you're focused on agent design.

## Step 20 (optional) — `vibe/acp/` (60 min)

The Agent Client Protocol implementation. This is the harness exposed as a server protocol for IDE integrations (Zed, etc.). Read if you want to see how the harness gets driven by a non-CLI consumer.

## After the path: ways to test understanding

1. **Sketch the agent loop from memory** on paper. You should be able to draw the four phases of a turn without looking.
2. **Implement a custom tool** and drop it in `~/.vibe/tools/`. Confirm it shows up in your next session.
3. **Write a custom agent profile** that combines `enabled_tools = ["grep", "read_file"]` with a custom `system_prompt_id`. Run `vibe --agent <yours>`.
4. **Trace one of your real Vibe sessions** by reading `~/.vibe/sessions/<id>/messages.jsonl` and walking the loop in your head turn by turn.
5. **Write a hook** that prints stdout on exit 2 and watch the agent retry.
6. **Compact mid-conversation** and read `messages` before and after.

If you can do those six, you understand the harness deeply enough to build your own.

---

## What you didn't need to read

- The TUI (`vibe/cli/textual_ui/`). It's beautifully crafted Textual code but it's *one consumer* of the harness, not part of the harness.
- The Audio/voice subsystems (`audio_recorder`, `audio_player`, `transcribe`, `tts`). Optional features.
- The Setup wizard (`vibe/setup/`). First-run UX.
- The experiments framework (`vibe/core/experiments/`). A/B testing for prompts.
- The Nuage / Teleport sub-services. Hosted features specific to Mistral's offering.

You can read any of these later if curious, but they're not on the critical path for "understand the harness."

## Where to go next

You've now read or skimmed every important file in the harness. Try building something:

- A custom agent profile.
- A custom tool.
- A custom skill.
- A hook.
- Fork the loop and add a new middleware kind (e.g., per-tool budget).

Or read another harness for comparison. The two most worth studying:

- Claude Code (closed source, but rich docs).
- [Aider](https://github.com/Aider-AI/aider) — Python, similar shape, different design choices.

Comparing two well-thought-out designs is the fastest way to internalize "the design space" of agent harnesses, beyond any single implementation.
