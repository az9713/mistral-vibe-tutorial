# Data flow: one user message, end to end

This is the trace that anchors the entire guide. Once you've read this once, the concept docs become annotations on the steps below.

We follow what happens when the user types `read pyproject.toml and tell me the Python version` in an interactive Vibe session. Line numbers reference [`vibe/core/agent_loop.py`](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/agent_loop.py).

## 0. Session is already alive

Before the user types anything, the harness is sitting in this state:

- `AgentLoop` instance constructed (`__init__`, line 234).
- `messages: MessageList` contains a single `LLMMessage` with `role=system` and content equal to the assembled system prompt — built by `get_universal_system_prompt(...)` (line 311). See [`system-prompt.md`](../concepts/system-prompt.md) for what goes into it.
- `tool_manager` has walked the tool search paths and registered every `BaseTool` subclass it found, plus discovered MCP servers and connectors.
- `skill_manager` has parsed every `SKILL.md` it could find.
- `agent_manager` has loaded built-in and user agent profiles; the active profile is whatever was passed to `--agent` (default: `default`).
- `middleware_pipeline` has been set up by `_setup_middleware()` (line 738) with TurnLimit / PriceLimit / TokenLimit (if limits passed), AutoCompact, ContextWarning, and a ReadOnlyAgent middleware for each of `plan` and `chat` mode.
- `permission_store` is empty (no rules approved yet).
- `scratchpad_dir` is a temp directory like `/tmp/vibe-scratchpad-abc123-XYZ/`.
- `session_logger` is writing to `~/.vibe/sessions/<id>/`.

## 1. User message lands

`act("read pyproject.toml and tell me the Python version")` is called from the UI layer. The `@requires_init` decorator first awaits `wait_until_ready()` so any deferred MCP/experiment initialization is complete.

`act()` (line 646):

1. Calls `_clean_message_history()` — patches missing tool-call responses so the history is well-formed.
2. Calls `rewind_manager.create_checkpoint()` — snapshots message-list state so the user can rewind this turn later.
3. Opens an `agent_span(...)` for OpenTelemetry tracing.
4. Yields control to `_conversation_loop(...)` (line 874).

## 2. Conversation loop opens

`_conversation_loop` (line 874):

1. Wraps the raw string in `LLMMessage(role=user, content=...)`, appends to `self.messages`, increments `stats.steps`.
2. Yields a `UserMessageEvent` to the UI.
3. Enters the per-turn `while not should_break_loop:` loop.

## 3. Before-turn middleware runs (turn 1)

`middleware_pipeline.run_before_turn(context)` runs each middleware in order. With default settings:

- `TurnLimitMiddleware` — usually inactive (no `--max-turns`).
- `AutoCompactMiddleware` — compares `context_tokens` to threshold. Context is small; returns `CONTINUE`.
- `ContextWarningMiddleware` — well below 50 %; returns `CONTINUE`.
- `ReadOnlyAgentMiddleware(plan)` — not in plan mode; `CONTINUE`.
- `ReadOnlyAgentMiddleware(chat)` — not in chat mode; `CONTINUE`.

Combined result: `CONTINUE`. No event yielded.

## 4. Model call (turn 1)

`_perform_llm_turn()` (line 988) is invoked. With streaming disabled:

1. `_chat()` (line 1325) is called.
2. It resolves the active model + provider, calls `telemetry_client.send_request_sent(...)`, then:
   ```python
   result = await self.backend.complete(
       model=active_model,
       messages=self.messages,
       tools=available_tools,
       tool_choice=tool_choice,
       ...
   )
   ```
3. The backend POSTs to the provider's chat completions endpoint. The request body contains the full message history (system prompt + the user message) and the JSON schema of every available tool.
4. The response message — an assistant message with one or more `tool_calls` — is appended to `self.messages`.
5. `_update_stats(...)` updates token counts.

## 5. Tool call resolution

Back in `_perform_llm_turn` (line 988):

1. `format_handler.parse_message(last_message)` extracts `tool_calls` from the assistant message.
2. `format_handler.resolve_tool_calls(parsed, tool_manager)` validates each call against the tool's Pydantic args schema, producing `ResolvedToolCall`s and `FailedToolCall`s.

In our example the model returned one call to `read_file` with args `{"target_file": "pyproject.toml"}`. It validates cleanly.

## 6. Tool execution (concurrent)

`_handle_tool_calls(resolved)` (line 1208):

1. Yields a `ToolCallEvent` per tool call so the UI can render "→ read_file(pyproject.toml)".
2. Calls `_run_tools_concurrently(tool_calls)` (line 1236) which `asyncio.create_task`s each call into a single `asyncio.Queue` and yields events as they arrive.

Each tool call is processed by `_process_one_tool_call` → `_execute_tool_call` (line 1101):

1. **Lookup tool instance** — `tool_manager.get("read_file")` returns the cached `ReadFile()` instance.
2. **Decide whether to execute** — `_should_execute_tool(...)` (line 1469):
   - `bypass_tool_permissions` is False → check the permission store.
   - `tool.resolve_permission(args)` returns `None` (no per-call override).
   - The config-level permission for `read_file` is `ALWAYS` → verdict = `EXECUTE`, no approval callback fires.
3. **Snapshot** — `tool.get_file_snapshot(args)` returns `None` for read-only tools. (For `write_file` / `search_replace` it would return a `FileSnapshot` and the rewind manager would store it.)
4. **Invoke** — `tool_instance.invoke(ctx=InvokeContext(...), **args_dict)`. `invoke` (in `BaseTool`) validates the args, then iterates `tool.run(args, ctx)`. The tool yields zero or more `ToolStreamEvent`s as it works, then yields exactly one Pydantic result model.
5. **Stringify the result** — fields of the result model are joined as `key: value` lines, `tool.get_result_extra(result)` is appended, and a `tool` role message is added to `self.messages` via `_handle_tool_response`.
6. **Emit** — yield a `ToolResultEvent` to the UI with the structured result.

If multiple tool calls were made, they run *in parallel* via `asyncio.gather`. Their results are appended in completion order to the queue (so UI sees them as they finish) but each appends its own `tool` message to history.

## 7. Loop check

Back in `_conversation_loop`:

```python
last_message = self.messages[-1]
should_break_loop = last_message.role != Role.tool
```

The last message *is* a tool result (`role=tool`), so `should_break_loop = False`. Pending injected messages drain into history. The loop iterates.

## 8. Before-turn middleware runs (turn 2)

Same checks as step 3, on a slightly longer history. All `CONTINUE`.

## 9. Model call (turn 2)

`_chat()` again, with the full history this time:

```
[system, user, assistant(tool_calls=[read_file]), tool(...)]
```

The model now has the file content. It generates a final assistant message with no tool calls — just text:

> The Python version constraint in pyproject.toml is `>=3.12`.

This message is appended to `self.messages`. `_perform_llm_turn` yields an `AssistantEvent` to the UI.

## 10. Loop terminates

```python
last_message = self.messages[-1]
should_break_loop = last_message.role != Role.tool  # True — it's 'assistant'
```

The `while` loop exits.

## 11. Post-agent-turn hooks

If `hooks_manager` is configured with `POST_AGENT_TURN` hooks, they fire now (line 936). Each hook runs as a subprocess with `HookInvocation` as input (session_id, transcript_path, cwd, hook_event_name).

- Exit 0 → log success, continue.
- Exit 2 with stdout → log error, inject stdout as a user message, set `should_break_loop = False`, the loop re-enters for another turn.
- Other → log a warning, continue.

In our example there are no hooks. The function reaches its `finally` block:

```python
finally:
    await self._save_messages()
```

`session_logger.save_interaction(...)` writes the updated history to disk.

`_conversation_loop` returns. `act()` finishes its generator. The UI sees end-of-stream.

## Visual summary

```
USER
 │
 │ act("...")
 ▼
┌────────────────────────────────────────────────────────────────────────┐
│ AgentLoop._conversation_loop                                           │
│                                                                        │
│   append user message ─► yield UserMessageEvent                        │
│                                                                        │
│   ┌────── while not should_break_loop ────────────────────────────┐    │
│   │                                                               │    │
│   │   middleware_pipeline.run_before_turn(context)                │    │
│   │      │                                                        │    │
│   │      ├─ STOP    ─► yield <vibe-stop> event ► return           │    │
│   │      ├─ COMPACT ─► run compact() ► continue                   │    │
│   │      ├─ INJECT  ─► append injected user message ► continue    │    │
│   │      └─ CONTINUE                                              │    │
│   │                                                               │    │
│   │   _perform_llm_turn                                           │    │
│   │      │                                                        │    │
│   │      ├─► backend.complete(...) ─► appends assistant msg       │    │
│   │      ├─► format_handler.parse + resolve_tool_calls            │    │
│   │      └─► if any tool calls:                                   │    │
│   │             yield ToolCallEvent per call                      │    │
│   │             ┌── asyncio.gather over each tool ──┐             │    │
│   │             │  _should_execute_tool (permission)│             │    │
│   │             │  get_file_snapshot                │             │    │
│   │             │  tool.invoke → tool.run           │             │    │
│   │             │  append role=tool message         │             │    │
│   │             │  yield ToolResultEvent            │             │    │
│   │             └────────────────────────────────────┘            │    │
│   │                                                               │    │
│   │   should_break_loop = (last_message.role != "tool")           │    │
│   │   if hooks and break: run POST_AGENT_TURN hooks               │    │
│   │                                                               │    │
│   └───────────────────────────────────────────────────────────────┘    │
│                                                                        │
│   _save_messages() ─► disk                                             │
└────────────────────────────────────────────────────────────────────────┘
 │
 │ events stream out
 ▼
UI
```

## Things to notice

1. **The loop continues until the model decides to stop.** There is no hardcoded "after N tool calls, finalize" — the only stop signal is the model emitting a message with no `tool_calls`.

2. **Tools run concurrently if the model batched them.** A single assistant message can contain multiple tool calls; they all dispatch in parallel via `asyncio.create_task` + a shared queue.

3. **Every tool result becomes a `role=tool` message.** This is how the model "sees" what happened. The harness is responsible for stringifying Pydantic result models into something the model can read — `key: value` lines plus an optional `get_result_extra(...)` blob.

4. **Permissions are checked per-call.** The same tool can require approval on one call and run silently on the next, depending on `tool.resolve_permission(args)` and the session's approved rules.

5. **Hooks can re-enter the loop.** A `POST_AGENT_TURN` hook returning exit code 2 effectively says "the work isn't done; here's a synthetic user message, keep going."

6. **Compaction can fire mid-loop.** If the request *before* turn N would exceed the threshold, the middleware swaps `COMPACT` for `CONTINUE` and the harness:
   - Runs a smaller model with the [compaction prompt](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/prompts/compact.md).
   - Replaces the message list with `[system, *recent_user_messages, summary]`.
   - Issues a new session id (with the old one stored as `parent_session_id`).
   - Returns to the loop, which now runs against a much smaller history.

That last point in particular is the key innovation of modern long-running agents — see [`context-management.md`](../concepts/context-management.md).

---

Read next: [`agent-loop.md`](../concepts/agent-loop.md) for a closer look at the loop's state machine.
