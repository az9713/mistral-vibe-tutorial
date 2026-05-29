# The agent loop

The single most important thing to understand. Everything else in the harness is something the loop calls or something that calls into the loop.

## The pattern

A modern coding-agent loop is:

```
1. Receive a user message.
2. Append it to the conversation history.
3. Send the entire history (plus tool schemas) to the model.
4. The model returns either:
     (a) Plain text  → done; surface to user.
     (b) Tool calls  → execute them, append results to history, GOTO 3.
```

That's it. The "agent" is just steps 3–4 in a loop. The interesting engineering is in *how the harness gates that loop, handles failures, and avoids unbounded growth*.

## Vibe's implementation

The loop lives in `AgentLoop._conversation_loop()` at `vibe/core/agent_loop.py:874`. The relevant skeleton:

```python
async def _conversation_loop(self, user_msg: str, ...) -> AsyncGenerator[BaseEvent]:
    user_message = LLMMessage(role=Role.user, content=user_msg, ...)
    self.messages.append(user_message)
    self.stats.steps += 1
    yield UserMessageEvent(...)

    should_break_loop = False
    first_llm_turn = True
    while not should_break_loop:
        # 1. Middleware pipeline gate
        result = await self.middleware_pipeline.run_before_turn(self._get_context())
        async for event in self._handle_middleware_result(result):
            yield event
        if result.action == MiddlewareAction.STOP:
            return

        # 2. Model call (and possible tool execution)
        self.stats.steps += 1
        async for event in self._perform_llm_turn():
            yield event
            await self._save_messages()

        # 3. Loop termination check
        last_message = self.messages[-1]
        should_break_loop = last_message.role != Role.tool

        if self._drain_pending_injections():
            should_break_loop = False

        # 4. Post-agent-turn hooks (only when about to break)
        if should_break_loop and self._hooks_manager:
            async for hook_event in self._hooks_manager.run(HookType.POST_AGENT_TURN, ...):
                if isinstance(hook_event, HookUserMessage):
                    self.messages.append(LLMMessage(role=Role.user, content=hook_event.content, injected=True))
                    should_break_loop = False
                else:
                    yield hook_event
```

Four phases per turn, repeating until the model emits an assistant message with no tool calls (or middleware stops it).

## Why an async generator?

Notice the loop is an `async def ... -> AsyncGenerator[BaseEvent, None]`. The harness *streams* events to its caller — it never returns "the final answer." This shape buys three things:

1. **The UI can render mid-flight.** Assistant text shows up as it's generated, tool calls render the moment they appear, tool results render as they finish.
2. **Cancellation is structured.** If the caller stops consuming the generator, `GeneratorExit` propagates, and the `finally` blocks cancel in-flight tool tasks.
3. **The loop has no return shape to design.** The "result" is the cumulative effect of the events. Different callers (TUI, ACP server, headless programmatic mode) consume the same events differently.

This is the choice that lets one `AgentLoop` class serve interactive CLI, programmatic one-shot, and IDE integration — see `vibe/core/programmatic.py` and `vibe/acp/`.

## What controls the loop's exit?

There are exactly three ways the loop ends:

1. **Natural end** — the model returns an assistant message with no tool calls. `last_message.role == "assistant"` → `should_break_loop = True`.
2. **Middleware STOP** — a middleware returns `MiddlewareAction.STOP` (e.g. `TurnLimitMiddleware` hit). The harness yields a synthetic `<vibe-stop>` event and `return`s.
3. **User cancellation** — the user interrupts (Ctrl-C). The cancellation propagates through `asyncio.CancelledError`; tools see their `invoke` cancelled, the loop yields a cancellation event and returns.

There is no fourth case. There is no "max iterations" hardcoded. There is no "if the model says 'I'm done' in text". The contract is: **tool call → keep going. No tool call → stop.**

## What happens around the loop

The decorator `@requires_init` (line 206) on `act()` ensures deferred initialization (MCP discovery, experiments hydration) completes before the loop starts.

`act()` itself wraps `_conversation_loop` in:

1. `_clean_message_history()` — patches missing tool responses. If a prior turn died mid-flight leaving an `assistant(tool_calls=[X])` message with no corresponding `tool(tool_call_id=X)` response, this fills in a synthetic cancellation response so the next API call doesn't 400.
2. `rewind_manager.create_checkpoint()` — captures message-list state for `/rewind`.
3. `agent_span(...)` — OpenTelemetry trace span around the whole turn.

## The four kinds of turn

The loop body looks uniform, but in practice turns come in four flavors:

| Flavor | Model emits | Loop response |
|--------|-------------|---------------|
| **Reasoning** | Reasoning content (for thinking models) + tool calls + text | Yield ReasoningEvent, then proceed as below |
| **Tool-calling** | One or more `tool_calls` | Execute → append results → loop again |
| **Mid-task talk** | Assistant text + tool calls | Yield AssistantEvent; execute tools; loop again |
| **Final** | Assistant text, no tool calls | Yield AssistantEvent; break |

The model decides which. The harness does not.

## Concurrent tool execution

When the model emits multiple `tool_calls` in a single message, Vibe runs them in parallel. From `_run_tools_concurrently()` (line 1236):

```python
queue: asyncio.Queue[ToolCallEvent | ToolResultEvent | ToolStreamEvent | None] = asyncio.Queue()
tasks = [asyncio.create_task(self._execute_tool_to_queue(tc, queue)) for tc in tool_calls]
async def _signal_when_all_done():
    try:
        await asyncio.gather(*tasks, return_exceptions=True)
    finally:
        await queue.put(None)
monitor = asyncio.create_task(_signal_when_all_done())
...
```

The queue is a fan-in pattern: each task pushes its `ToolCallEvent` / `ToolStreamEvent` / `ToolResultEvent`s into a shared queue; the loop yields from the queue. The model sees the tool results in *call order* (because results are appended to `messages` in call order via `_handle_tool_response`), but the UI sees the events in *completion order*. Both views are useful.

If the consumer cancels (closes the generator), every in-flight task is cancelled.

## Tool result formatting

When a tool finishes, the harness has a Pydantic result model. It has to turn that into a string the model can read. From `_execute_tool_call()` (line ~1171):

```python
result_dict = result_model.model_dump()
text = "\n".join(f"{k}: {v}" for k, v in result_dict.items())
extra = tool_instance.get_result_extra(result_model)
if extra:
    text += "\n\n" + extra
```

So `{"path": "pyproject.toml", "content": "..."}` becomes:

```
path: pyproject.toml
content: ...
```

The `get_result_extra` hook lets a tool inject contextual info — for instance, `read_file` uses it to add a note when an AGENTS.md was discovered alongside the file being read. This is how Vibe surfaces dynamic context without changing the tool's typed result.

## Error handling: failed tool calls

`format_handler.resolve_tool_calls()` (called in `_perform_llm_turn`) returns both `ResolvedToolCall`s (validated) and `FailedToolCall`s (couldn't parse args, unknown tool, etc.).

Failed calls are emitted *before* successful ones, via `_emit_failed_tool_events`:

```python
error_msg = f"<{TOOL_ERROR_TAG}>{failed.tool_name}: {failed.error}</{TOOL_ERROR_TAG}>"
yield ToolResultEvent(error=error_msg, ...)
self.messages.append(self.format_handler.create_failed_tool_response_message(failed, error_msg))
```

The model sees a tool-role message describing what went wrong. It can self-correct on the next turn (retry with valid args, choose a different tool, give up and respond to the user).

This is the harness paying the "fail-forward" tax: rather than aborting, surface failures as data and let the model adapt.

## Where to look next

- For *what* the model sees in the system prompt: [`system-prompt.md`](system-prompt.md)
- For *what* a tool is and how it gets registered: [`tools.md`](tools.md)
- For the middleware that gates every turn: [`middleware.md`](middleware.md)
- For what happens when the history grows too big: [`context-management.md`](context-management.md)
- For the end-to-end trace of one turn: [`../architecture/data-flow.md`](../architecture/data-flow.md)

## Try it: read the loop in 5 minutes

To convince yourself you understand it, open `vibe/core/agent_loop.py` and:

1. Find `_conversation_loop` (search for `async def _conversation_loop`).
2. Find `_perform_llm_turn`.
3. Find `_run_tools_concurrently`.

Trace one user message through those three functions, in order, without reading anything else. You'll have the loop in your head in under five minutes.
