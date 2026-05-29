# What is an agent harness?

An **agent harness** is the runtime that sits between a human (or other caller), a tool inventory, and a language model — and turns a single user request into a sequence of model calls, tool executions, and side effects, with all the bookkeeping that makes the loop safe, recoverable, and useful.

If the model is the engine, the harness is the chassis, transmission, dashboard, and crash structure around it.

## The 60-second mental model

```
            ┌─────────────────────────────────────────────────────┐
            │                                                     │
            │   USER ──prompt──►  HARNESS  ──messages──►  MODEL   │
            │                       ▲                        │    │
            │                       │                        │    │
            │                  tool results              tool calls
            │                       │                        │    │
            │                       └──── TOOLS  ◄───────────┘    │
            │                                                     │
            └─────────────────────────────────────────────────────┘
```

Concretely the harness owns:

1. **A conversation history** that grows turn-by-turn — system message, user messages, assistant messages, tool calls, tool results.
2. **A turn loop** — keep calling the model until it produces a message with no tool calls.
3. **A tool registry** — which tools exist, what arguments they take, what they're allowed to do.
4. **A permission system** — for each tool call, decide allow / ask / deny.
5. **A context window manager** — when the history grows too big, compact it.
6. **An I/O surface** — events streamed out to a UI, callbacks back in from the user.

That's it. Every coding agent (Claude Code, Cursor's agent, Aider, Codex, Vibe) is some variant of those six responsibilities. The interesting design space is *how* each one is implemented.

## Why study Vibe in particular

Vibe is small enough to read end-to-end (~30 files in `vibe/core/`), professional enough to use in production, and explicit about its choices. The harness is also a near-perfect superset of what most agent harnesses do — if you understand Vibe's, you can read any of the others.

Specifically, Vibe demonstrates:

| Pattern | Where to look |
|---------|---------------|
| Streaming async-generator event bus | `vibe/core/agent_loop.py` (every public method yields `AsyncGenerator[BaseEvent]`) |
| Middleware pipeline gating each turn | `vibe/core/middleware.py` |
| Pluggable tools via Pydantic-typed base class | `vibe/core/tools/base.py` |
| Three-level permission with per-call overrides | `vibe/core/tools/permissions.py` |
| Context compaction (summarize-and-restart) | `vibe/core/compaction.py` + `agent_loop.compact()` |
| Sub-agent delegation as a tool | `vibe/core/tools/builtins/task.py` |
| Agent profiles as TOML overrides on a base config | `vibe/core/agents/models.py` |
| Skills as discoverable prompt packs | `vibe/core/skills/manager.py` |
| Lifecycle hooks executed as subprocesses | `vibe/core/hooks/manager.py` |
| Per-session scratchpad for working files | `vibe/core/scratchpad.py` |
| File snapshots for rewind | `vibe/core/rewind/manager.py` |

## What an agent harness is *not*

An agent harness is not the model. The model is a sealed function: messages in, message out. The harness is everything around that function.

An agent harness is not a chat UI. Vibe ships with a Textual TUI (`vibe/cli/textual_ui/`) but that's a separate layer that consumes events from the harness. The harness is also driven by:

- `vibe -p "..."` (programmatic / one-shot mode) — `vibe/core/programmatic.py`
- ACP (Agent Client Protocol) for IDE integrations — `vibe/acp/`
- Tests — directly instantiating `AgentLoop`

The harness is the part that's the same in all those entry points.

## The contract Vibe's harness exposes

The harness's primary API is `AgentLoop.act(msg)` — defined at `vibe/core/agent_loop.py:646`:

```python
async def act(
    self,
    msg: str,
    client_message_id: str | None = None,
    *,
    auto_title: str | None = None,
) -> AsyncGenerator[BaseEvent, None]:
    ...
```

You give it a string. You get back an async stream of events:

- `UserMessageEvent` — your message was recorded
- `AssistantEvent` — assistant text (often arriving in chunks if streaming)
- `ReasoningEvent` — reasoning content (for thinking models)
- `ToolCallEvent` — the assistant decided to call a tool
- `ToolStreamEvent` — partial output from a streaming tool
- `ToolResultEvent` — a tool finished
- `CompactStartEvent` / `CompactEndEvent` — context compaction happened
- `SessionTitleUpdatedEvent`, `AgentProfileChangedEvent`, `PlanReviewRequestedEvent`, …

Everything else in this guide is *how that one method does its job*.

## The single hardest idea to internalize

**The model has no memory.** Every API call sends the *entire* conversation history. The harness owns that history. The harness's most important job is deciding what stays in the window and what gets dropped or summarized.

Everything called "memory", "context", "compaction", "summary", "skills", "AGENTS.md", "scratchpad" — they're all just answers to: *what do we put in the next request to the model?*

Hold that frame and the rest of this guide will fall into place.

---

Next: [Key concepts](key-concepts.md).
