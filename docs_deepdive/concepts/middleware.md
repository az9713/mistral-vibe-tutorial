# Middleware

The pipeline that runs *before every turn* in the agent loop. Middleware is how the harness enforces invariants — turn limits, price ceilings, mode-specific behavior, context compaction triggers — without putting any of that logic inside the loop body.

## The pattern

A middleware pipeline is an ordered list of checkers. Each checker inspects the conversation state and returns one of:

- **CONTINUE** — proceed with the turn as usual.
- **STOP** — end the loop immediately.
- **COMPACT** — trigger context compaction, then continue the loop.
- **INJECT_MESSAGE** — insert a synthetic user message before the model call, then continue.

The pipeline runs middlewares in order. The first `STOP` or `COMPACT` short-circuits the rest. `INJECT_MESSAGE` results from multiple middlewares are concatenated into one combined message.

This is the same shape as web-framework middleware (Express, ASGI). It works here for the same reasons: clean separation of concerns, easy to add/remove, each middleware is independently testable.

## Vibe's implementation

Defined in `vibe/core/middleware.py`. The contract:

```python
class ConversationMiddleware(Protocol):
    async def before_turn(self, context: ConversationContext) -> MiddlewareResult: ...
    def reset(self, reset_reason: ResetReason = ResetReason.STOP) -> None: ...
```

`ConversationContext` carries the data middlewares need:

```python
@dataclass
class ConversationContext:
    messages: MessageList
    stats: AgentStats
    config: VibeConfig
```

`MiddlewareResult`:

```python
@dataclass
class MiddlewareResult:
    action: MiddlewareAction = MiddlewareAction.CONTINUE
    message: str | None = None       # for INJECT_MESSAGE
    reason: str | None = None        # for STOP (shown to user)
    metadata: dict[str, Any] = field(default_factory=dict)
```

The pipeline:

```python
class MiddlewarePipeline:
    middlewares: list[ConversationMiddleware]

    async def run_before_turn(self, context):
        messages_to_inject = []
        for mw in self.middlewares:
            result = await mw.before_turn(context)
            if result.action == INJECT_MESSAGE and result.message:
                messages_to_inject.append(result.message)
            elif result.action in {STOP, COMPACT}:
                return result
        if messages_to_inject:
            return INJECT_MESSAGE with combined message
        return CONTINUE
```

Note the short-circuit: as soon as a middleware says STOP or COMPACT, the rest don't run. Injections are buffered and combined.

## Where the pipeline is invoked

The loop calls it at the top of each iteration:

```python
# vibe/core/agent_loop.py:_conversation_loop ~line 906
while not should_break_loop:
    result = await self.middleware_pipeline.run_before_turn(self._get_context())
    async for event in self._handle_middleware_result(result):
        yield event
    if result.action == MiddlewareAction.STOP:
        return
    ...
```

`_handle_middleware_result` translates the action:

- **STOP** → yield `AssistantEvent(content="<vibe-stop>{reason}</vibe-stop>", stopped_by_middleware=True)`. The caller knows to render this differently. The loop returns.
- **INJECT_MESSAGE** → append `LLMMessage(role=user, content=message, injected=True)` to history. Note `injected=True` — these messages are visible to the model but flagged so the harness knows they aren't real user input (used by compaction).
- **COMPACT** → run `compact()`, yield `CompactStartEvent` and `CompactEndEvent` around it. See [`context-management.md`](context-management.md).
- **CONTINUE** → no-op.

## The built-in middlewares

Configured in `AgentLoop._setup_middleware()` (line 738), in order:

### TurnLimitMiddleware

```python
class TurnLimitMiddleware:
    async def before_turn(self, context):
        if context.stats.steps - 1 >= self.max_turns:
            return MiddlewareResult(STOP, reason=f"Turn limit of {self.max_turns} reached")
        return MiddlewareResult()
```

Only added when `--max-turns N` is passed. Used by programmatic / CI runs that want a hard cap.

### PriceLimitMiddleware

Same shape, but the gate is `context.stats.session_cost > self.max_price`. Activated by `--max-price`.

### TokenLimitMiddleware

Same shape, gate is `context.stats.session_total_llm_tokens > self.max_tokens`. Activated by `--max-session-tokens`.

These three are *budget enforcers*. They exist to make the harness predictable for headless / batch users.

### AutoCompactMiddleware

```python
class AutoCompactMiddleware:
    async def before_turn(self, context):
        threshold = context.config.get_active_model().auto_compact_threshold
        if threshold > 0 and context.stats.context_tokens >= threshold:
            return MiddlewareResult(
                action=COMPACT,
                metadata={"old_tokens": context.stats.context_tokens, "threshold": threshold},
            )
        return MiddlewareResult()
```

Runs every turn. The check is "am I about to send a request that exceeds the threshold?" — so compaction triggers *preemptively*, before the over-large request goes out, not after a context-too-long error.

The threshold is per-model (small models have small windows). Compaction is described in [`context-management.md`](context-management.md).

### ContextWarningMiddleware

```python
class ContextWarningMiddleware:
    def __init__(self, threshold_percent: float = 0.5):
        self.threshold_percent = threshold_percent
        self.has_warned = False

    async def before_turn(self, context):
        if self.has_warned:
            return MiddlewareResult()
        max_context = context.config.get_active_model().auto_compact_threshold
        if context.stats.context_tokens >= max_context * 0.5:
            self.has_warned = True
            warning_msg = f"<vibe_warning>You have used 50% of your total context...</vibe_warning>"
            return MiddlewareResult(action=INJECT_MESSAGE, message=warning_msg)
        return MiddlewareResult()
```

One-shot: fires once when the user crosses 50% of the model's context window. Injects a warning the model can see; the model often responds by being more concise. The `has_warned` flag prevents re-firing.

This is a *behavioral nudge* via middleware — the user / UI doesn't see the warning (it's in the conversation, not on screen), but the model adapts.

### ReadOnlyAgentMiddleware

The most interesting one. Used to enforce `plan` and `chat` mode.

```python
class ReadOnlyAgentMiddleware:
    def __init__(self, profile_getter, agent_name, reminder, exit_message):
        self._profile_getter = profile_getter
        self._agent_name = agent_name
        self._reminder = reminder        # injected when entering this mode
        self.exit_message = exit_message # injected when leaving this mode
        self._was_active = False

    async def before_turn(self, context):
        is_active = self._is_active()    # current agent name == self._agent_name?
        was_active = self._was_active

        if was_active and not is_active:           # just left → exit message
            self._was_active = False
            return MiddlewareResult(INJECT_MESSAGE, message=self.exit_message)
        if is_active and not was_active:           # just entered → reminder
            self._was_active = True
            return MiddlewareResult(INJECT_MESSAGE, message=self.reminder)
        self._was_active = is_active
        return MiddlewareResult()
```

The pattern: a middleware that owns a piece of conversational state ("the agent just switched into plan mode") and translates state transitions into injected messages.

For plan mode, the reminder is `make_plan_agent_reminder(...)` which generates a paragraph like:

> Plan mode is active. You MUST NOT make any edits (except to the plan file below, or in your scratchpad), run any non-readonly tools, or otherwise make any changes to the system. This supersedes any other instructions you have received.
>
> ## Plan File Info
> Create or edit your plan at `~/.vibe/plans/<id>.md` using the write_file and search_replace tools.
> ...

When the user switches back out, the exit message:

> Plan mode has ended. If you have a plan ready, you can now start executing it. If not, you can now use editing tools and make changes to the system.

This means *mode switching is a conversation event*, not a config change the model has to detect. The middleware turns silent state mutations into explicit messages.

## A subtle point: order matters

The order of middlewares in the pipeline matters because of the short-circuit. Vibe's order:

1. Turn / Price / Token limits — *budget enforcers*, fail fast.
2. AutoCompact — triggers before the LLM call, so compaction happens before the next round.
3. ContextWarning — informational, depends on threshold.
4. ReadOnlyAgent (plan) — mode reminder.
5. ReadOnlyAgent (chat) — mode reminder.

If `AutoCompact` ran *after* `ContextWarning`, the warning would inject just before compaction wiped the message that contained it. By running compaction first, the warning never collides with compaction.

## Reset semantics

`MiddlewarePipeline.reset(reset_reason)` calls `mw.reset(reset_reason)` on each middleware. Used after:

- A session reset (full clear): `reset_reason=STOP`.
- Compaction completed: `reset_reason=COMPACT`.

Most middlewares ignore reset (they're stateless). `ContextWarningMiddleware` uses reset to clear `has_warned` so the warning can fire again in the fresh post-compaction window. `ReadOnlyAgentMiddleware` uses it to clear `_was_active` so the reminder re-injects after a reset.

Distinguishing `STOP` vs. `COMPACT` lets each middleware decide if it should treat a reset as a continuation or a fresh start.

## When to add your own middleware

A middleware is the right tool when you have a check that should run *every turn* and that can yield a verdict on whether to continue, stop, compact, or inject. Examples of things that fit:

- A rate limiter ("at most N tool calls per minute").
- A safety filter ("if last tool result mentions sensitive content, inject a redaction warning").
- A custom cost tracker ("STOP if predicted cost of next call exceeds X").
- A time-of-day gate ("inject a reminder during work hours").

Things that *don't* fit middleware:

- Logic that runs *after* a tool call (use the tool's result handling).
- Behavior that depends on which specific tool was called (use `resolve_permission`).
- Persistent rules across sessions (config or AGENTS.md is the right place).

## Try it: add a middleware

```python
from vibe.core.middleware import (
    ConversationContext, MiddlewareAction, MiddlewareResult, ResetReason,
)

class HelloMiddleware:
    async def before_turn(self, context: ConversationContext) -> MiddlewareResult:
        if context.stats.steps == 1:  # first turn
            return MiddlewareResult(
                action=MiddlewareAction.INJECT_MESSAGE,
                message="<system>Remember to be extra careful today.</system>",
            )
        return MiddlewareResult()

    def reset(self, reset_reason: ResetReason = ResetReason.STOP) -> None:
        pass
```

In your code, after constructing an `AgentLoop`:

```python
loop.middleware_pipeline.add(HelloMiddleware())
```

Now every fresh interaction's first turn gets your injection. That's the entire surface area.

---

Next: [`context-management.md`](context-management.md) — what COMPACT actually does.
