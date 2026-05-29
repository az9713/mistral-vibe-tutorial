# Hooks

The escape hatch. When the harness finishes a turn and is about to stop, it can run external programs. Those programs can return information to the agent, force a retry, or simply log what happened.

## The pattern

A lifecycle hook lets external systems plug into specific moments of the agent's runtime without forking the harness. The contract is small:

- The harness invokes the hook with structured input.
- The hook produces an exit code and optional stdout.
- The harness interprets the result:
  - "OK" → continue.
  - "Retry" → restart the loop with the hook's output as a synthetic user message.
  - "Warning" → log, continue.

This is the same shape as Git hooks (`pre-commit`, `post-receive`) and CI hooks. Vibe applies the pattern to a coding agent.

## Vibe's implementation

Defined in `vibe/core/hooks/`. The current hook types are limited to one:

```python
class HookType(StrEnum):
    POST_AGENT_TURN = auto()
```

`POST_AGENT_TURN` fires when the agent is *about to break the loop* — i.e., the model emitted a final assistant message with no tool calls.

### Hook configuration

Hooks are TOML files configured through the `HookConfigResult` (loaded by the CLI from a known location). The schema:

```python
class HookConfig(BaseModel):
    name: str
    type: HookType
    command: str
    timeout: float = 30.0
    description: str | None = None
```

Example config (conceptual):

```toml
[[hooks]]
name = "lint-check"
type = "post_agent_turn"
command = "uv run ruff check ."
timeout = 60.0
description = "Re-lint after the agent finishes and ask for fixes if anything broke"
```

### Hook invocation

`HooksManager.run(hook_type, session_id, session_logger)` (`vibe/core/hooks/manager.py`):

For each configured hook of the requested type:

1. Build a `HookInvocation`:

   ```python
   HookInvocation(
       session_id=...,
       transcript_path=...,    # path to messages.jsonl
       cwd=str(Path.cwd().resolve()),
       hook_event_name="post_agent_turn",
   )
   ```

2. `HookExecutor.run(hook, invocation)` spawns the subprocess. The invocation is fed as stdin (JSON) or via env vars (depends on executor implementation). The hook's command runs with a timeout.

3. The executor returns `HookExecutionResult`:

   ```python
   class HookExecutionResult(BaseModel):
       hook_name: str
       exit_code: int | None
       stdout: str
       stderr: str
       timed_out: bool
   ```

4. The manager interprets:

   | Exit code | Meaning | Effect |
   |-----------|---------|--------|
   | `0` (SUCCESS) | Hook passed | `HookEndEvent(status=OK)` |
   | `2` (RETRY) with stdout | Hook wants the agent to retry | `HookEndEvent(status=ERROR, content="Failed, retrying...")`, then `HookUserMessage(content=stdout)` |
   | other | Hook misbehaved or reported a warning | `HookEndEvent(status=WARNING, content=stdout or stderr)` |
   | timeout | Hook took too long | `HookEndEvent(status=WARNING, content="Timed out after Xs")` |

5. When a hook returns exit code 2, the manager:
   - Calls `_retry_state.should_retry(hook.name)` (max 3 retries per hook per turn).
   - If retries remain, yields a `HookUserMessage(content=stdout)`.
   - The agent loop catches that, appends it as a `user` role message with `injected=True`, sets `should_break_loop = False`, and runs another turn.

### How the loop consumes hook events

In `_conversation_loop` (around line 936):

```python
if should_break_loop and self._hooks_manager:
    hook_retry: HookUserMessage | None = None
    async for hook_event in self._hooks_manager.run(
        HookType.POST_AGENT_TURN, self.session_id, self.session_logger
    ):
        if isinstance(hook_event, HookUserMessage):
            hook_retry = hook_event
        else:
            yield hook_event   # forwarded to UI
    if hook_retry is not None:
        self.messages.append(
            LLMMessage(
                role=Role.user,
                content=hook_retry.content,
                injected=True,
            )
        )
        should_break_loop = False
```

Note: the loop only yields *non-HookUserMessage* events to the UI. The retry message is kept private and injected into history. So the UI sees:

- A list of hooks running.
- Each hook's status (OK / WARNING / ERROR + content).
- Then the loop continues if there's a retry.

The model sees:

- A new injected user message in its next API call, containing what the hook wrote to stdout.

This separation matters. The hook is conceptually "speaking to the model on the user's behalf"; the user shouldn't have to see the literal hook output as if it were their own message.

## Why exit code 2 means retry

Bash convention: `0` = success, anything else = failure. But the harness needs a richer signal — "failed in a way the agent should respond to" vs. "failed in a way the user should know about."

Picking `2` for retry leaves `1` available for "real failure" (which currently maps to WARNING but could be specialized further). It's also far enough from common shell-error codes that hook authors are unlikely to return it accidentally.

## Retry budget

`HookRetryState`:

```python
_MAX_RETRIES = 3

class HookRetryState:
    def remaining_retries(self, hook_name): ...
    def track_retry(self, hook_name): ...
    def track_success(self, hook_name): ...
    def should_retry(self, hook_name): ...
```

State is per-hook, reset at the start of each `act(...)` call (see `_hooks_manager.reset_retry_count()` at line 899). So a single user message can cause at most 3 retries per hook.

The retry budget exists because a misbehaving hook (always exits 2) would otherwise cause an infinite loop. Three is a practical cap that lets a real "lint, fix, re-lint, fix, re-lint, done" sequence complete but stops runaway.

## Practical hook patterns

### Lint after every turn

```toml
[[hooks]]
name = "ruff"
type = "post_agent_turn"
command = """
set -e
if ! uv run ruff check . > /dev/null 2>&1; then
    echo "Ruff found issues. Please fix:"
    uv run ruff check .
    exit 2
fi
"""
timeout = 30.0
```

Exit 0 if clean, exit 2 with the ruff output if dirty. The agent gets a retry with the linter's complaint as the prompt.

### Type-check after every turn

Same pattern with `pyright` / `tsc` / `mypy`. Combine with the lint hook for "code must be clean to land".

### Append a context note

```toml
[[hooks]]
name = "remind-tests"
type = "post_agent_turn"
command = """
echo "Reminder: did you update the tests for your changes?"
exit 2
"""
```

This is mildly cursed — it always retries. The cap of 3 saves you. Use sparingly.

### Notify when done

```toml
[[hooks]]
name = "notify"
type = "post_agent_turn"
command = "osascript -e 'display notification \"Agent finished\" with title \"Vibe\"'"
timeout = 5.0
```

Exit 0, no stdout → no agent action. Just a side effect for the user.

## Why hooks instead of middleware?

Middleware runs *every turn, before the LLM call*, and lives inside the harness. Hooks run *once per user message, after the loop is about to break*, and execute external programs.

Use middleware if:
- The logic is Python and pure (no subprocesses).
- It needs to fire every turn.
- It needs to inspect the live message list / stats.

Use hooks if:
- The logic is a shell command or external script.
- It should only fire when the agent thinks it's done.
- It encodes project-level policy (linters, type checkers, formatters).

Hooks are also language-agnostic — your team's hook can be a Bash, Python, Go, or Rust program. No Vibe-specific API.

## Things hooks can't do

- They cannot read or modify the conversation history except by writing to stdout when exiting 2.
- They cannot call tools.
- They cannot prompt the user.
- They don't see the messages directly — they get `transcript_path`. If they want to read the conversation, they read the JSONL file.

This is intentional. Hooks are an *out-of-band* extension; giving them in-band powers would erase the separation that makes them safe.

## Hook config issues

If a TOML file is malformed, the `HookConfigResult` records the issue in `issues: list[HookConfigIssue]`:

```python
class HookConfigIssue(BaseModel):
    file: Path
    message: str
```

These surface in the UI (or logs) at session start, so the user knows their hook didn't load. The harness doesn't refuse to start; it just runs without that hook.

## Try it: a one-shot reminder hook

Save as `/tmp/vibe-hook-reminder.sh`:

```bash
#!/bin/sh
echo "Don't forget to run tests before declaring done."
exit 2
```

`chmod +x /tmp/vibe-hook-reminder.sh`.

Configure (in whatever TOML file Vibe reads hooks from for your install):

```toml
[[hooks]]
name = "reminder"
type = "post_agent_turn"
command = "/tmp/vibe-hook-reminder.sh"
timeout = 5.0
```

Restart Vibe. Make a small request. When the agent thinks it's done, the hook fires, the reminder gets injected, the loop runs another turn. After 3 retries, the harness gives up and surfaces "Failed, retries exhausted (3/3)".

That's the entire hook lifecycle in 5 lines of shell.

---

Next: [`human-in-the-loop.md`](human-in-the-loop.md) — how the agent talks back to the user mid-task.
