# Context management

A model's context window is finite. A long coding session will exceed it. The harness's job is to keep the loop running through that.

This is the design space where modern coding agents differ most. Vibe's approach: *preemptive compaction with prefix preservation*.

## The pattern

When the conversation grows too large, you have a small menu of options:

1. **Truncate from the start.** Drop oldest messages. Cheap. Loses the system prompt unless you pin it.
2. **Truncate from the middle.** Drop the messy middle. Loses recent context less, but the dropped region was probably the actual work.
3. **Summarize and replace.** Run a separate model call to summarize history, then replace history with the summary plus the most recent turns. Costs a model call, but preserves *meaning*.
4. **Sliding window of N most recent.** Last K tokens only. Simple but kills long-running tasks.

Vibe uses approach 3 — *summarize and replace*, plus pinning of recent user messages and the system message. It calls this **compaction**.

## When compaction fires

Three triggers:

1. **Auto** — `AutoCompactMiddleware` (see [`middleware.md`](middleware.md)) checks `context_tokens >= threshold` *before* each turn. If true, the loop runs `compact()` instead of calling the model. The compacted history is what the next turn sees.
2. **User** — typing `/compact` in the TUI calls `AgentLoop.compact()` directly. Useful when the user knows the history has gotten messy.
3. **Programmatic** — anything holding the `AgentLoop` can call `await loop.compact()`.

The threshold is set per model. From `ModelConfig`:

```toml
[[models]]
name = "mistral-medium-2502"
auto_compact_threshold = 100000   # tokens
```

So smaller models compact sooner.

## What compact() does

`AgentLoop.compact()` lives at `vibe/core/agent_loop.py:1686`. Walking through it:

### Step 1 — Save current state

```python
self._clean_message_history()                  # fix orphaned tool calls
await self.session_logger.save_interaction(...) # disk
```

The pre-compaction history is saved with the session id. Even after compaction, the original messages still exist on disk under the old session id. The lineage is preserved.

### Step 2 — Pick the recent user messages to preserve

```python
summary_prefix = UtilityPrompt.COMPACT_SUMMARY_PREFIX.read()
prior_user_messages = collect_prior_user_messages(list(self.messages), summary_prefix)
```

`collect_prior_user_messages` (in `vibe/core/compaction.py`) walks the history *newest-first* and picks user messages — non-injected, not previous compaction summaries — up to a 20,000-token budget. The newest message that doesn't fit is middle-truncated. The result is reversed so it's in chronological order.

The intuition: a summary captures the *narrative*, but specific user messages contain *literal asks* (file paths, identifiers, constraints) that summaries can lose. Keeping them verbatim is cheap insurance.

### Step 3 — Run the compaction model

```python
summary_request = self.config.compaction_prompt
if extra_instructions:
    summary_request += f"\n\n## Additional Instructions\n{extra_instructions}"

with self.messages.silent():
    self.messages.append(LLMMessage(role=Role.user, content=summary_request))
    summary_result = await self._chat(
        model_override=self.config.get_compaction_model()
    )
```

The compaction prompt (`vibe/core/prompts/compact.md`) is short and direct:

> You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary for another LLM that will resume this task.
>
> Include:
> - The user's current goal and any explicit constraints or preferences
> - Key decisions made and their rationale
> - Files touched and the current state of in-progress work (paths + one-line status)
> - What remains to be done — the concrete next step
> - Any data, identifiers, or references the next LLM needs to continue
>
> Be concise and structured. One line per modified file unless a snippet is load-bearing. Do not repeat information already captured. Do not include a "Final Answer" section — the entire response IS the handoff.

Two important choices:

- The summary is framed as a **handoff to another LLM** — not a description of what happened. This produces structured, actionable summaries instead of narrative recaps.
- A *different model* runs the summarization. Set via `config.compaction_model` — typically a smaller, cheaper model. Vibe doesn't waste the expensive model on summarization.

`messages.silent()` is a context manager that suppresses observer events while the compaction call runs — the UI doesn't see the summary-request user message or the summary-response assistant message as part of the conversation.

### Step 4 — Replace history

```python
system_message = self.messages[0]
wrapped_summary = f"{summary_prefix}\n{summary_content}"
summary_message = LLMMessage(role=Role.user, content=wrapped_summary)
self.messages.reset([system_message, *prior_user_messages, summary_message])
```

The new history is exactly:

```
[system_prompt, prior_user_msg_1, prior_user_msg_2, ..., summary_user_msg]
```

Notice the summary is a **user role** message, not a system or assistant message. This is intentional:

- Putting it as user means the next assistant turn can directly act on it ("continue the task").
- Wrapping it with `COMPACT_SUMMARY_PREFIX` (a recognizable marker) lets *future* `collect_prior_user_messages` filter previous summaries out — preventing nested summary buildup.

### Step 5 — Reset session, recount tokens

```python
active_model = self.config.get_active_model()
await self._reset_session()                       # new session_id, parent_session_id = old

actual_context_tokens = await self.backend.count_tokens(
    model=active_model,
    messages=self.messages,
    tools=self.format_handler.get_available_tools(self.tool_manager),
    ...
)
self.stats.context_tokens = actual_context_tokens
await self.session_logger.save_interaction(...)
self.middleware_pipeline.reset(reset_reason=ResetReason.COMPACT)
```

The compacted state gets a *new* session id. The previous session id is stored as `parent_session_id` so the lineage is reconstructable. The token count is recalculated using the provider's count endpoint — a real number, not an estimate. Middlewares are reset (`ContextWarningMiddleware` will re-warn at 50% of the new history).

### Step 6 — Return to the loop

`compact()` returns the summary text. The middleware that triggered it yields `CompactEndEvent`. The agent loop checks `should_break_loop` and continues the turn.

The next API call goes out with a tiny history.

## Two reasons this works better than truncation

### Continuity of instruction

The system prompt is preserved. The user's most recent requests are preserved verbatim. The summary tells the model "here's the state, here's what's next." Truncation breaks all three.

### Token accounting hygiene

After compaction, the model's `prompt_tokens` cost on the next call is the small new prefix, not the original-history cost. Vibe's stats reset accordingly. A user can compact-then-continue indefinitely; the per-turn cost stays bounded.

## What compaction does *not* preserve

- **Tool call results.** Any specifics buried in `read_file` outputs are gone unless the model summarized them well. This is why the compaction prompt asks for "Files touched and the current state of in-progress work (paths + one-line status)" — the next model needs to know which files exist and may need to re-read them.
- **Reasoning chains.** Anything in `ReasoningEvent`s is lost. The summary should contain the *conclusions* of reasoning, not the chain itself.
- **Previous compactions.** By design, compaction summaries don't stack. `collect_prior_user_messages` filters them out by prefix.

So a long-running session is *eventually* operating on the K most recent user messages plus a summary of everything else. The trick of the design is making the summary good enough that the model can act on it like it had the original history.

## Context warnings

`ContextWarningMiddleware` injects a one-time message when context_tokens crosses 50% of the threshold. The model sees:

> You have used 50% of your total context (X/Y tokens)

It's a hint, not a constraint. In practice the model responds by being terser. The warning doesn't appear on the UI — it's a private nudge.

## Token limits and hard stops

`TokenLimitMiddleware` and `PriceLimitMiddleware` are hard stops. They check cumulative session totals and STOP the loop. The model never sees them; the user sees a synthetic stop event.

These are belt-and-braces for headless or CI runs that need a guaranteed exit point even if auto-compact malfunctions.

## When ContextTooLongError fires

Sometimes the provider rejects a request mid-flight with "context too long." Vibe maps that to `ContextTooLongError` (in `vibe/core/types.py`). The TUI catches it and shows a recovery option: compact and retry.

This is the fallback path when auto-compaction's threshold underestimated. Should be rare in practice — `auto_compact_threshold` is typically set well below the model's actual window so there's headroom.

## Pattern: stateless replay vs. stateful continuation

A subtle insight: Vibe's compaction is a stateless replay strategy *for the model* but stateful continuation *for the harness*.

- The model can't tell whether it has the original history or a summary. As far as it's concerned it just got handed a fresh conversation with a long user message describing the state and one final user message saying "continue."
- The harness, however, remembers the lineage via `parent_session_id`. Disk artifacts (todo state, scratchpad files, on-disk `plan.md`, the user's actual code changes) are unchanged.

This separation — *real work is on disk, working memory is in the conversation* — is the cornerstone of long-running agent design.

## Try it: trigger a manual compaction

In an interactive session: type `/compact`.

You'll see:

1. A `CompactStartEvent` (UI shows a spinner / progress indicator).
2. A pause while the compaction model runs.
3. A `CompactEndEvent` with `old_context_tokens` / `new_context_tokens`.

After it, `messages[0]` is unchanged (system prompt), `messages[1..-2]` are your recent user messages, and `messages[-1]` is the summary wrapped with the prefix. The next thing you type continues from there.
