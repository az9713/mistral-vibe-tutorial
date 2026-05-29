# Tools

A tool is the harness's primary mechanism for *doing something in the world*. The model decides which tools to call and with what arguments. The harness validates, gates, executes, and returns a result the model can read.

This doc explains Vibe's tool contract — base class, discovery, execution lifecycle — and walks through the built-in tools.

## The pattern

A general tool contract has four pieces:

1. **Schema** — a typed description of the arguments the tool accepts. The model needs this to know how to call the tool.
2. **Implementation** — a function or method that executes the tool, given validated arguments.
3. **Result** — a typed result the harness can serialize and feed back to the model.
4. **Metadata** — name, description, permission rules, optional prompt text.

The unique problem is making this *extensible*: users should be able to add their own tools without forking the harness. Vibe solves this with filesystem-discovered Python modules and a `BaseTool` generic class.

## The base class

`BaseTool` is defined at `vibe/core/tools/base.py:114`. It's a generic over four types:

```python
class BaseTool[
    ToolArgs: BaseModel,
    ToolResult: BaseModel,
    ToolConfig: BaseToolConfig,
    ToolState: BaseToolState,
](ABC):
    description: ClassVar[str] = "..."
    prompt_path: ClassVar[Path] | None = None

    def __init__(self, config_getter: Callable[[], ToolConfig], state: ToolState) -> None: ...

    @abstractmethod
    async def run(
        self, args: ToolArgs, ctx: InvokeContext | None = None
    ) -> AsyncGenerator[ToolStreamEvent | ToolResult, None]:
        raise NotImplementedError
        yield  # makes it a generator
```

Concrete subclasses parameterise the four type variables:

```python
class ReadFile(
    BaseTool[ReadFileArgs, ReadFileResult, ReadFileConfig, BaseToolState],
    ToolUIData[ReadFileArgs, ReadFileResult],
):
    description: ClassVar[str] = "Read a file from the filesystem."

    async def run(self, args: ReadFileArgs, ctx: InvokeContext | None = None):
        ...
        yield ReadFileResult(path=..., content=...)
```

The four type variables are extracted at class definition time by `_get_tool_args_results()` / `_get_tool_config_class()` / `_get_tool_state_class()` so the harness can:

- Generate JSON schema from `ToolArgs` for the model (`get_parameters()`).
- Validate incoming arguments against `ToolArgs` (`invoke` calls `args_model.model_validate(raw)`).
- Type the result for downstream consumers.

### `run` returns an async generator

A tool yields zero or more `ToolStreamEvent`s, then exactly one `ToolResult`. Stream events flow to the UI as the tool works; the final result is the value the harness records in history.

This shape is why long-running tools (running tests, calling a remote API, exploring a codebase) can show progress to the user without buffering.

## The InvokeContext

Tools that need harness state get it through `InvokeContext` (passed as `ctx`):

```python
@dataclass
class InvokeContext:
    tool_call_id: str
    approval_callback: ApprovalCallback | None = None
    agent_manager: AgentManager | None = None
    user_input_callback: UserInputCallback | None = None
    sampling_callback: MCPSamplingHandler | None = None
    session_dir: Path | None = None
    entrypoint_metadata: EntrypointMetadata | None = None
    plan_file_path: Path | None = None
    switch_agent_callback: SwitchAgentCallback | None = None
    skill_manager: SkillManager | None = None
    scratchpad_dir: Path | None = None
    permission_store: PermissionStore | None = None
```

What's in there tells you exactly what tools are allowed to do. Notice no direct access to `messages` or `config` — tools cannot mutate history or change config. They can:

- Call `approval_callback` to ask the user for permission.
- Call `user_input_callback` to ask a clarifying question (`ask_user_question`).
- Use `sampling_callback` for MCP-style nested LLM calls.
- Spawn a subagent via `agent_manager` (`task` tool).
- Read/write to `scratchpad_dir` without permission prompts.
- Switch the active agent profile via `switch_agent_callback`.

The InvokeContext is the entire blast radius a tool has into the harness.

## Tool lifecycle (per call)

When the model calls a tool, the harness goes through this sequence (`agent_loop._execute_tool_call`, line 1101):

```
┌─ get tool instance from ToolManager
│
├─ _should_execute_tool(tool, args, call_id)
│    ├─ if bypass_tool_permissions → EXECUTE
│    ├─ tool.resolve_permission(args)  ── per-call override
│    │     ├─ ALWAYS → EXECUTE
│    │     ├─ NEVER  → SKIP with feedback
│    │     └─ ASK    → check approved rules → ask user → EXECUTE/SKIP
│    └─ fall through to config-level permission
│
├─ tool.get_file_snapshot(args)         ── for rewind, if file-mutating
│    └─ rewind_manager.add_snapshot
│
├─ tool.invoke(ctx, **args_dict)
│    ├─ args_model.model_validate(raw)
│    └─ tool.run(args, ctx)
│         ├─ yield ToolStreamEvent       ── streamed to UI
│         └─ yield ToolResult            ── final
│
├─ format result_dict as "key: value" text
├─ append tool.get_result_extra(result) if any
├─ append role=tool message to history
└─ yield ToolResultEvent
```

If `tool.run` raises:

- `ToolError` → caught, formatted as `<vibe_tool_error>...</vibe_tool_error>`, appended as tool message, model sees it next turn.
- `ToolPermissionError` → recorded as rejected; the user is informed; loop continues.
- `asyncio.CancelledError` → user-cancelled; recorded; re-raised.
- Anything else → formatted as tool error, appended, model sees it.

This is the "fail-forward" property: tool failures are *data* for the model, not exceptions for the user.

## Tool discovery

`ToolManager._compute_search_paths()` (`vibe/core/tools/manager.py:96`) walks, in order:

1. `vibe/core/tools/builtins/` — built-in tools.
2. `config.tool_paths` — anything the user added.
3. Project tools dirs (e.g. `.vibe/tools/`).
4. User tools dirs (`~/.vibe/tools/`).

For each path, `_iter_tool_classes()` `rglob`s `*.py`, imports each module dynamically, and finds every concrete `BaseTool` subclass. The resulting `{name: cls}` map is `available_tools`.

Filtering on top:

- `config.enabled_tools` — if set, only matching tools (glob match).
- `config.disabled_tools` — if set, drop matching tools.
- Per-MCP-server / per-connector enable/disable lists.
- `tool.is_available(config)` — runtime check (some tools self-disable if missing dependencies).

So custom tools "just work": drop a `.py` file in `~/.vibe/tools/`, define a `BaseTool` subclass, and it's available next session.

## Permission system in brief

Each tool has a `BaseToolConfig`:

```python
class BaseToolConfig(BaseModel):
    permission: ToolPermission = ToolPermission.ASK
    allowlist: list[str] = []
    denylist: list[str] = []
    sensitive_patterns: list[str] = []
```

`permission` is the default decision: `ALWAYS` / `ASK` / `NEVER`.

But a tool can override per-call by implementing `resolve_permission(self, args)`. Example, from `bash.py`: the bash tool returns `ALWAYS` if the command matches the allowlist (`git status`, `ls`, `cat`, etc.), `NEVER` if it matches the denylist (`rm -rf /`, `vim`, `nano`), and otherwise falls through to ASK.

Full treatment in [`permissions.md`](permissions.md).

## What's in the box: built-in tools

| Tool | What it does | Default permission |
|------|--------------|--------------------|
| `bash` | Execute a shell command with timeout, env, stdin/stdout streaming | ASK (with per-command allow/denylist) |
| `read_file` | Read a file, optional offset/limit | ALWAYS |
| `write_file` | Write a new file | ASK (per-path) |
| `search_replace` | Replace exact strings inside a file | ASK (per-path) |
| `grep` | Search file contents (ripgrep) | ALWAYS |
| `todo` | Read/write the in-session todo list | ALWAYS |
| `task` | Spawn a subagent | ASK (allowlist includes `explore` by default) |
| `ask_user_question` | Multi-choice questions to the user | ALWAYS |
| `exit_plan_mode` | End plan mode, request user approval of plan | ALWAYS |
| `webfetch` | HTTP GET a URL, return text | ASK |
| `websearch` | Search the web via configured provider | ASK |
| `skill` | Load a skill's prompt into the conversation | ALWAYS |

Each lives in `vibe/core/tools/builtins/<name>.py` and has a sibling `prompts/<name>.md` with usage hints that get inlined into the system prompt (see [`system-prompt.md`](system-prompt.md) §6).

## Examples worth reading

To internalize the contract, read these three tools in order:

1. **`todo.py`** — the simplest interesting tool. State persists across calls within a session (because `BaseToolState` is held on the tool instance). Shows the args/result pattern at minimum complexity.
2. **`bash.py`** — the most complex. Demonstrates per-invocation permission resolution (parsing the command with tree-sitter), `ToolStreamEvent`s during execution, denylists, and how the harness handles cancellation.
3. **`task.py`** — meta. Spawns a fresh `AgentLoop` inside a tool. Shows how subagent isolation is implemented and how scratchpad / permission state propagates.

## MCP tools

The Model Context Protocol is an external-server tool standard. MCP servers expose tools that get adopted into Vibe's tool inventory at startup (or hot-reload).

`MCPRegistry` (in `vibe/core/tools/mcp/`) connects to each configured server, lists its tools, and dynamically creates `MCPTool` subclasses (one per remote tool). Those subclasses participate in the same lifecycle as Python tools — they have args/result Pydantic models (derived from the server's JSON schema) and a `run` that proxies to the MCP server.

To the agent loop, an MCP tool is indistinguishable from a built-in. The harness doesn't care.

## What tools cannot do

By design, tools cannot:

- Mutate the conversation history directly. Only the harness appends `tool` role messages.
- Change the active model or agent profile silently. They can request via `switch_agent_callback`, but that goes through `AgentLoop.switch_agent` which rebuilds the system prompt.
- Call the LLM directly (except through MCP sampling, which is gated).
- Bypass the permission system. Even `ALWAYS` tools go through `_should_execute_tool`.

These boundaries are what make the harness reasonable about. The tool surface is the only place "anything can happen", and even there the surface is narrow.

---

Next: [`permissions.md`](permissions.md) for how the harness decides whether to run a tool.
