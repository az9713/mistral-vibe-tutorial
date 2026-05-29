# The permission system

The single feature that separates a useful coding agent from a destructive one. Every tool call goes through a permission check before the harness runs it.

## The pattern

A defensible permission model needs three things:

1. **A default verdict per tool** — most tools should be safe-by-default; some should always require approval.
2. **Per-call refinement** — `bash echo hi` is different from `bash rm -rf /`. The same tool can need different scrutiny based on its arguments.
3. **Persistent rules** — once the user approves a class of action ("always allow `git status`"), don't ask again this session.

Vibe implements all three with three primitives: `ToolPermission`, `RequiredPermission`, and `ApprovedRule`.

## The verdicts: ToolPermission

Defined in `vibe/core/tools/base.py:75`:

```python
class ToolPermission(StrEnum):
    ALWAYS = auto()   # auto-execute
    ASK    = auto()   # prompt the user
    NEVER  = auto()   # refuse silently with feedback
```

Every tool has one as its default, set in its config. From the built-ins:

| Tool | Default |
|------|---------|
| `read_file`, `grep`, `todo`, `skill`, `ask_user_question`, `exit_plan_mode` | `ALWAYS` |
| `bash`, `write_file`, `search_replace`, `webfetch`, `websearch`, `task` | `ASK` |
| (none by default) | `NEVER` |

`NEVER` is used by agent profiles to disable tools that shouldn't be available in a mode — e.g. the `plan` agent sets `write_file.permission = "never"` with an allowlist for the plan file itself.

## Per-call override: resolve_permission

The hook lives in `BaseTool`:

```python
def resolve_permission(self, args: ToolArgs) -> PermissionContext | None:
    return None
```

The default implementation returns `None`, meaning "use the config-level permission". Tools that want to refine override it.

`PermissionContext`:

```python
class PermissionContext(BaseModel):
    permission: ToolPermission
    required_permissions: list[RequiredPermission] = []
    reason: str | None = None
```

So a tool can return:

- `PermissionContext(permission=ALWAYS)` — execute now, don't ask.
- `PermissionContext(permission=NEVER, reason="...")` — refuse with a feedback string the model sees.
- `PermissionContext(permission=ASK, required_permissions=[...])` — ask the user, with this list of *required permissions* the user is granting if they say yes.

### Example: bash

`Bash.resolve_permission(args)` (in `bash.py`):

1. Parse the command string with tree-sitter to extract the individual commands (handles `&&`, `|`, etc.).
2. For each command, build a "session pattern" — what an "always allow this" rule would look like.
3. If every command matches the bash config's `allowlist` (e.g. `git status *`, `ls *`, `cat *`), return `ALWAYS`.
4. If any matches the `denylist` (e.g. `rm -rf /`, `vim`), return `NEVER`.
5. Otherwise return `ASK` with `required_permissions = [RequiredPermission(scope=COMMAND_PATTERN, invocation_pattern=..., session_pattern=...)]`.

So the model sees a single uniform `bash` tool, but the harness applies a per-command judgment.

### Example: write_file

`WriteFile.resolve_permission(args)`:

1. Resolve the target path.
2. If the path is inside the scratchpad, return `ALWAYS`.
3. If the path is outside the current working directory and `OUTSIDE_DIRECTORY` hasn't been approved this session, require it.
4. Build a `FILE_PATTERN` required permission for the path.
5. Return `ASK` with those required permissions.

### Example: task

`Task.resolve_permission(args)`:

1. Look at `args.agent` (the subagent name).
2. If it matches the config denylist, return `NEVER`.
3. If it matches the allowlist (default contains `explore`), return `ALWAYS`.
4. Else fall through.

These three patterns — *command parsing*, *path scoping*, *target whitelisting* — are the common shapes per-call permission takes. The harness gives you the hook; what you do with it is per-tool.

## Required permissions: the granular scope

`RequiredPermission` (in `vibe/core/tools/permissions.py`):

```python
class RequiredPermission(BaseModel):
    scope: PermissionScope          # COMMAND_PATTERN | OUTSIDE_DIRECTORY | FILE_PATTERN | URL_PATTERN
    invocation_pattern: str          # the exact thing being requested
    session_pattern: str             # what an "always" rule would cover
    label: str                       # for the UI
```

The distinction between `invocation_pattern` and `session_pattern` matters:

- The user is being asked to approve *this specific call* (the invocation pattern).
- If they say "always allow", the rule covers a *family* (the session pattern).

For `bash`, `invocation_pattern = "git diff src/foo.py"` and `session_pattern = "git diff *"`. "Always allow" means "any future `git diff <anything>`".

## Approved rules and the PermissionStore

`ApprovedRule`:

```python
class ApprovedRule(BaseModel):
    tool_name: str
    scope: PermissionScope
    session_pattern: str
```

`PermissionStore` (same file):

- Holds a `list[ApprovedRule]` and a `dict[tool_name, ToolPermission]` for "always allow this whole tool" decisions.
- `covers(tool_name, required_permission)` — does any stored rule already cover this request? Pattern match via `wildcard_match`.
- `add_rule(rule)` — record an approval.
- `set_tool_permission(name, perm)` — set tool-level permission.

A `PermissionStore` is owned by the `AgentLoop`. Subagents share the parent's store (passed into `Task` via `InvokeContext.permission_store`). This is the right call: a subagent shouldn't get to ask for permissions independently — its parent has already vouched for it.

## The decision flow

Put together, `_should_execute_tool` (`agent_loop.py:1469`) does:

```
if bypass_tool_permissions:           # e.g. auto-approve agent
    return EXECUTE

ctx = tool.resolve_permission(args)   # per-call hook
if ctx is None:
    ctx = PermissionContext(permission=config.tool_permission)

match ctx.permission:
    case ALWAYS:                      # tool says: just run
        return EXECUTE
    case NEVER:                       # tool says: refuse
        return SKIP with feedback
    case ASK:
        if ctx.required_permissions:
            uncovered = [rp for rp in required if not store.covers(tool_name, rp)]
            if all are covered:
                return EXECUTE
            ask the user about the uncovered ones
        else:
            ask the user
        translate answer to EXECUTE / SKIP
```

The `ask the user` step is `_ask_approval` (line 1514). It calls `self.approval_callback(tool_name, args, tool_call_id, required_permissions)`. The UI is responsible for showing the user the call details, the required permissions, and offering:

- **Yes** — execute once.
- **No** — skip with the user's feedback text becoming the tool result the model sees.
- **Always (this session)** — add an `ApprovedRule`s for each required permission, then execute.
- **Always (saved)** — same plus persist to disk via `config.add_tool_allowlist_patterns`.

If `approval_callback` is `None` (headless mode), `_ask_approval` returns `SKIP` with `"Tool execution not permitted."` — fail closed.

## Agent profiles modulate the system

Different agent profiles ship different permission defaults:

- **default** — `ASK` everywhere except read-only.
- **plan** — `write_file` and `search_replace` become `NEVER` except for paths in the plans dir. This is enforced by the agent profile's overrides, not by the tool — see `agents/models.py:_plan_overrides()`.
- **accept-edits** — `write_file` and `search_replace` become `ALWAYS`. Bash still asks.
- **auto-approve** — sets `bypass_tool_permissions: True`. Every check returns `EXECUTE` immediately.
- **chat** — restricts to `[grep, read_file, ask_user_question, task]` via `enabled_tools` and bypasses permissions (everything else is removed).

These are *configuration*, not new code paths. The same `_should_execute_tool` logic handles them.

## Why "ask" beats "deny"

A natural design choice is "deny dangerous things by default and let the user opt in". Vibe's choice is "ask and learn": prompt the user, but remember the answer.

The reasons:

- **Users approve patterns, not individual calls.** Allowing `git status` once doesn't teach the harness; allowing `git status *` does. Required permissions encode the pattern explicitly.
- **Approvals are valuable signal for telemetry.** What gets approved tells you which guards are mis-tuned. (Vibe records this in `telemetry_client.send_tool_call_finished`.)
- **The model can adapt to refusals.** A skipped tool yields a `feedback` string that becomes a tool result. The model can read it and try a different approach. Refusing fail-closed without feedback would leave the model in the dark.

## Try it: trace a bash approval

Open `vibe/core/tools/builtins/bash.py`. Find `resolve_permission`. Walk through it with `args.command = "rm -rf node_modules"`:

1. Does it parse cleanly? (yes — one command, name `rm`)
2. Does it match any allowlist pattern? (no — `rm` isn't there by default)
3. Does it match denylist? (no — `rm` isn't blanket-denied)
4. Build `RequiredPermission(scope=COMMAND_PATTERN, invocation_pattern="rm -rf node_modules", session_pattern="rm *")`.
5. Return `ASK` with that requirement.

Now you can predict what the UI will show the user, and you can edit the bash config in `~/.vibe/config.toml` to add `"rm node_modules*"` to the allowlist to make this auto-approve next time. That round-trip is the entire mental model of the permission system.

---

Next: [`middleware.md`](middleware.md) — the pipeline that gates each turn before any of this fires.
