# Agents and subagents

The same harness can act with very different personalities — read-only explorer, full-access coder, plan-only consultant. The mechanism is **agent profiles**: named overlays on the base config. And one of those profiles can be a **subagent**: a sandboxed loop spawned via the `task` tool to do focused work.

This doc covers both, because they share the same underlying machinery.

## The pattern

A general way to handle "different modes" is one of:

1. **A flag per mode.** Add `if mode == "plan": disable_writes()` throughout the codebase. Painful.
2. **One class per mode.** `PlanAgent(BaseAgent)`, `ChatAgent(BaseAgent)`. Lots of duplication.
3. **One agent, multiple configurations.** Same `AgentLoop`, different `VibeConfig`. Switch by swapping the config.

Vibe picks #3. An **agent profile** is a struct with a name, display label, safety classification, and a dict of *overrides* that get deep-merged onto the base config. The active profile's `apply_to_config(base)` returns the effective config that the rest of the harness reads.

## Agent profile

Defined in `vibe/core/agents/models.py`:

```python
@dataclass(frozen=True)
class AgentProfile:
    name: str
    display_name: str
    description: str
    safety: AgentSafety               # SAFE | NEUTRAL | DESTRUCTIVE | YOLO
    agent_type: AgentType = AgentType.AGENT    # AGENT | SUBAGENT
    overrides: dict[str, Any] = field(default_factory=dict)
    install_required: bool = False
```

`apply_to_config(base)` deep-merges `overrides` onto the base config. Special handling:

- `base_disabled` is appended to `disabled_tools` (additive).
- If the *base* config has `disabled_tools`, those win over the profile's `enabled_tools` (so environment-level disables, like ACP mode disabling `bash`, can't be undone by a profile).

`AgentManager` (`vibe/core/agents/manager.py`):

- Discovers profiles: built-ins from `BUILTIN_AGENTS` + TOML files in `~/.vibe/agents/` and `<project>/.vibe/agents/`.
- Holds `active_profile`.
- `config` property returns `active_profile.apply_to_config(base)` (cached).
- `switch_profile(name)` swaps and invalidates the cache.

## Built-in profiles

| Profile | Type | Safety | What it does |
|---------|------|--------|--------------|
| `default` | agent | NEUTRAL | Standard. Tools default to ASK. |
| `chat` | agent | SAFE | Read-only, conversational. Tools: `grep`, `read_file`, `ask_user_question`, `task`. Bypass permissions on those. |
| `plan` | agent | SAFE | Read-only + can edit only the plan file. `write_file` / `search_replace` set to `NEVER` except for `~/.vibe/plans/*`. |
| `accept-edits` | agent | DESTRUCTIVE | `write_file` and `search_replace` set to `ALWAYS`. Bash still asks. |
| `auto-approve` | agent | YOLO | `bypass_tool_permissions = True`. No prompts at all. |
| `explore` | subagent | SAFE | Read-only subagent. Tools: `grep`, `read_file`. Uses `explore.md` system prompt. |
| `lean` | agent | NEUTRAL | Specialized for Lean 4 work. Different model, different system prompt, longer bash timeout. `install_required: True`. |

Two are worth studying in detail.

### `plan` — restricted editing

```python
def _plan_overrides() -> dict[str, Any]:
    plans_pattern = str(PLANS_DIR.path / "*")
    return {
        "tools": {
            "write_file": {"permission": "never", "allowlist": [plans_pattern]},
            "search_replace": {"permission": "never", "allowlist": [plans_pattern]},
        }
    }
```

The trick is `permission=never, allowlist=[<plans_dir>/*]`. The default verdict is NEVER, but matching the allowlist promotes to ALWAYS. Net effect: the agent can write/edit plan files freely, but trying to edit anything else in the project is denied.

Combined with `ReadOnlyAgentMiddleware(plan)` (see [`middleware.md`](middleware.md)) which injects the plan-mode reminder into the conversation, the model knows *and* the harness enforces.

### `lean` — model + prompt + config

```python
LEAN = AgentProfile(
    name=BuiltinAgentName.LEAN,
    agent_type=AgentType.AGENT,
    install_required=True,
    overrides={
        "system_prompt_id": "lean",
        "active_model": "leanstral",
        "providers": [...],
        "models": [{...}],
        "compaction_model": {...},
        "tools": {"bash": {"default_timeout": 1200}},
        "base_disabled": ["exit_plan_mode"],
    },
)
```

Switching to `lean` swaps the system prompt (`lean.md`), the active model, even the compaction model, and bumps the bash timeout. `install_required: True` means it only shows up if the user has explicitly opted in (`installed_agents = ["lean"]` in config). This pattern — *agents are config overlays* — scales beautifully.

## Custom profiles

Drop a TOML file in `~/.vibe/agents/myagent.toml`:

```toml
display_name = "My Agent"
description = "..."
safety = "neutral"

# anything below this gets merged onto the base config:
system_prompt_id = "minimal"
enabled_tools = ["grep", "read_file", "bash"]

[tools.bash]
permission = "ask"
allowlist = ["pytest *", "git status"]
```

`AgentProfile.from_toml(path)` reads it. The file name (`myagent`) is the profile name. `apply_to_config` does the rest.

So building an agent for "review this PR" or "build dashboards" is a TOML, not a code change.

## Switching profiles at runtime

`AgentLoop.switch_agent(name)` (line 1764):

```python
async def switch_agent(self, agent_name: str) -> None:
    if agent_name == self.agent_profile.name:
        return
    self.agent_manager.switch_profile(agent_name)
    await self.reload_with_initial_messages(reset_middleware=False)
```

`reload_with_initial_messages` rebuilds the tool manager, skill manager, system prompt, and stats — because the new profile's overrides may change all of those. `reset_middleware=False` because the existing middleware pipeline is fine; only the `ReadOnlyAgentMiddleware`s care about the active profile, and they re-detect on the next turn.

In the TUI, Shift+Tab cycles through profiles. Each switch re-renders the entire system prompt and changes which tools the model is offered. The conversation history is unchanged.

## Subagents

A subagent is an agent profile with `agent_type = SUBAGENT`. The only one built-in is `explore`:

```python
EXPLORE = AgentProfile(
    name=BuiltinAgentName.EXPLORE,
    display_name="Explore",
    description="Read-only subagent for codebase exploration",
    safety=AgentSafety.SAFE,
    agent_type=AgentType.SUBAGENT,
    overrides={"enabled_tools": ["grep", "read_file"], "system_prompt_id": "explore"},
)
```

The `explore` subagent uses a different system prompt (`explore.md`) that emphasizes terse "code/diagram-first" output, has only `grep` and `read_file` available, and is meant to do focused exploration without polluting the parent's context.

## How `task` spawns a subagent

The `task` tool (`vibe/core/tools/builtins/task.py`):

```python
async def run(self, args: TaskArgs, ctx: InvokeContext | None = None):
    agent_profile = ctx.agent_manager.get_agent(args.agent)
    if agent_profile.agent_type != AgentType.SUBAGENT:
        raise ToolError(
            f"Only subagents can be used with the task tool. "
            f"This is a security constraint to prevent recursive spawning."
        )

    session_logging = SessionLoggingConfig(
        save_dir=str(ctx.session_dir / "agents") if ctx.session_dir else "",
        session_prefix=args.agent,
        ...
    )
    base_config = VibeConfig.load(session_logging=session_logging)
    subagent_loop = AgentLoop(
        config=base_config,
        agent_name=args.agent,
        entrypoint_metadata=ctx.entrypoint_metadata,
        is_subagent=True,
        defer_heavy_init=True,
        permission_store=ctx.permission_store,
    )

    if ctx.approval_callback:
        subagent_loop.set_approval_callback(ctx.approval_callback)

    task_text = args.task
    if ctx.scratchpad_dir:
        task_text = (
            f"Scratchpad directory: {ctx.scratchpad_dir}\n"
            "You can read and write files here without permission prompts.\n\n"
            f"{args.task}"
        )

    accumulated_response: list[str] = []
    async with aclosing(subagent_loop.act(task_text)) as events:
        async for event in events:
            if isinstance(event, AssistantEvent) and event.content:
                accumulated_response.append(event.content)
            elif isinstance(event, ToolResultEvent):
                # surface tool progress to the parent's UI
                yield ToolStreamEvent(...)

    turns_used = sum(msg.role == Role.assistant for msg in subagent_loop.messages)
    yield TaskResult(
        response="".join(accumulated_response),
        turns_used=turns_used,
        completed=completed,
    )
```

What's happening:

1. **Check it's actually a subagent.** Hard error if the user tries to spawn a primary agent. This prevents recursive `task → task → task` chains.
2. **Create a new session log dir under the parent's** (`session_dir / "agents"`). Subagent activity is preserved under the parent's session id.
3. **Spawn `AgentLoop(is_subagent=True)`**:
   - `is_subagent` flag suppresses scratchpad creation (uses parent's).
   - `defer_heavy_init=True` for fast startup — MCP discovery happens lazily.
   - `permission_store` is the parent's, not a new one. Subagent doesn't independently re-prompt for whitelisted commands.
4. **Forward the approval callback.** Subagent can still ask the user — but through the parent's UI.
5. **Prepend scratchpad info to the task.** Subagent sees "Scratchpad directory: ... You can read and write files here without permission prompts." in its first user message.
6. **Run the subagent's loop** by iterating `subagent_loop.act(task_text)`.
7. **Collect assistant text** as the response. Tool results are surfaced as `ToolStreamEvent`s to the parent's UI so the user can see progress.
8. **Return** `TaskResult(response, turns_used, completed)`.

The parent's loop sees a normal tool result. The parent's history gains one `role=tool` message containing the subagent's full response.

## Why subagents are valuable

Two main wins:

1. **Context isolation.** The parent's working context stays focused. The subagent burns its own tokens exploring or reading 50 files; the parent gets back a 200-token answer.
2. **Specialized profiles.** A read-only `explore` subagent is fundamentally safer than the same exploration done by the (possibly write-capable) parent.

Both are concrete forms of the more general pattern: *expensive subprocedures should not pollute the parent's reasoning context.*

## Why not arbitrary nesting?

The hard check `agent_type != AgentType.SUBAGENT → ToolError` blocks the parent from spawning another full agent. Sub-subagents are technically possible (a subagent could call `task` if its enabled_tools includes it), but the explore subagent doesn't have `task`. Vibe deliberately keeps the spawn tree at depth ≤ 2.

This is a *design constraint*, not a technical one. Arbitrary nesting makes accounting (tokens, time, recoverable rewind) hard to reason about. Capping at one level keeps the model.

## Switching profiles mid-task

A tool can call `ctx.switch_agent_callback(new_profile_name)` to change the active agent. The only built-in tool that does this is `exit_plan_mode` — when the user approves the plan, the agent flips from `plan` to `default` mode automatically.

This is the harness saying "modes are first-class state". A tool can request a mode change; the harness handles the rebuild.

## Try it: write a custom subagent

Create `~/.vibe/agents/testchecker.toml`:

```toml
display_name = "Test Checker"
description = "Read-only subagent that summarizes test files"
safety = "safe"
agent_type = "subagent"
enabled_tools = ["grep", "read_file"]
system_prompt_id = "explore"   # or "minimal"
```

Now in an interactive session you can ask:

> use the task tool with agent="testchecker" to summarize the tests in `tests/`

The parent will spawn a fresh AgentLoop running your subagent, which will grep+read its way through `tests/`, and return a summary. You've built a specialist with one TOML file.
