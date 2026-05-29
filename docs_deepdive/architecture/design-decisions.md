# Design decisions

The harness's shape is a series of *choices*. Each one was a fork in the road; the right one isn't always obvious. This doc walks the most consequential picks, what alternatives existed, and what Vibe gains and gives up by its choice.

If you want to understand the harness at the level of "why is it like this", read this doc.

## 1. Async generators as the primary abstraction

**The choice:** `AgentLoop.act()` and most public methods return `AsyncGenerator[BaseEvent, None]`. The harness *yields* events as work happens; the caller iterates.

**Alternatives:**

- Callbacks. Caller registers `on_assistant`, `on_tool_call`, etc. The harness invokes them.
- Event bus. Caller subscribes to a global pubsub.
- Return a final result and an "events" log at the end.

**Why this wins:**

- Native streaming. Assistant text shows up as it's generated without extra plumbing.
- Cancellation falls out for free. Caller stops iterating → generator gets `GeneratorExit` → in-flight tool tasks get cancelled.
- Backpressure. If the UI can't render fast enough, the harness blocks on `yield`. No event queue overflow.
- One contract for many callers. TUI, ACP server, programmatic mode, tests all iterate the same generator.

**Cost:**

- Mental overhead. `async def ... -> AsyncGenerator[...]` with `yield` inside is harder to reason about than a synchronous function with callbacks. Pyright stripped is unforgiving.
- Composition is awkward. Combining multiple generators (e.g., the loop yields from `_perform_llm_turn` which yields from `_handle_tool_calls`) requires `async for ... yield ...` boilerplate.

This pattern shows up across the codebase — every tool's `run` is also an async generator. The team committed.

## 2. Tools as Pydantic-typed classes, discovered from disk

**The choice:** `BaseTool` is generic over four type variables `(ToolArgs, ToolResult, ToolConfig, ToolState)`. Tools are subclasses found by walking `*.py` files under known search paths.

**Alternatives:**

- A simpler decorator-based API (`@tool def my_tool(...)`).
- A registry the user has to populate explicitly.
- External-only tools (MCP).

**Why this wins:**

- The same machinery serves built-ins and user tools. No special path for "extending the agent."
- JSON schema for the model is generated from the Pydantic args model. No hand-written schema, no schema drift.
- Validation is uniform. Invalid args from the model produce a structured error the model sees, not a Python exception that crashes the loop.
- State per tool (`ToolState`) is type-safe and Pydantic-validated.

**Cost:**

- The generic-over-four-types declaration is intimidating for new tool authors. `BaseTool[MyArgs, MyResult, MyConfig, BaseToolState]` is a lot to ask.
- Dynamic discovery requires careful module-name handling — see `_compute_module_name` in `tools/manager.py` which dodges Pydantic class-identity collisions by hashing paths.

The team accepted the verbosity to keep the system explicit. AGENTS.md even cautions against shortcuts: "no relative imports, no `# type: ignore`".

## 3. Middleware as the per-turn gate

**The choice:** A pipeline of `before_turn(context) -> MiddlewareResult` runs before every model call. The harness handles `CONTINUE`, `STOP`, `COMPACT`, `INJECT_MESSAGE`.

**Alternatives:**

- Inline the checks in `_conversation_loop`. Just `if context_tokens >= threshold: compact()`. Less abstraction.
- Decorators on the loop body.
- An event-listener pattern.

**Why this wins:**

- Independently testable. Each middleware is a tiny class with one method.
- Composable. The CLI adds limit middlewares only when their flags are passed.
- New gates don't touch loop code. Add `RateLimitMiddleware`? Just write a class and `pipeline.add(...)`.

**Cost:**

- A list of middlewares is opaque about order-dependence. Vibe documents the order in `_setup_middleware` but a future contributor could break things by reordering.
- Verbose for one-line checks. `class TurnLimitMiddleware` is a lot for `if steps >= max: stop`.

The escape hatch: when a check is genuinely one-shot, the loop body still does it directly (e.g., `_clean_message_history` runs before each `act`).

## 4. Permissions: three levels + per-call refinement

**The choice:** Tools have a default `ALWAYS / ASK / NEVER`. A `resolve_permission(args)` method on each tool can override per-call. Approved rules are stored in a `PermissionStore` for "always" memory.

**Alternatives:**

- One permission per tool. Easy but misses bash-style "some commands are fine, others aren't."
- Capability-based — tools request specific capabilities, the system grants them or doesn't. More expressive but more code.
- Whitelist patterns only — no asking.

**Why this wins:**

- The granularity matches how humans think: "I want to allow `git status` always but ask about `rm`." Patterns approach matches.
- The model gets feedback when denied (the `reason` field on `PermissionContext`). It can self-correct.
- The same store is shared by subagents, which is the right blast-radius rule.

**Cost:**

- The interaction of `tool.resolve_permission(args)` returning a `PermissionContext` with `required_permissions` is non-obvious. New tool authors miss it and write tools that don't integrate with "always allow" properly.
- Approval UX is the slowest path. Every approval is a network round-trip with the user.

The design's main insight: approvals are *learnable*. Recording rules turns each approval into permission to skip 100 future approvals.

## 5. Compaction over truncation

**The choice:** When context grows large, run a separate model call to summarize history, then replace history with `[system, recent_users..., summary]`. Reset session id, preserve `parent_session_id`.

**Alternatives:**

- Truncate from start (drop oldest).
- Truncate from middle.
- Sliding window of N most recent messages.
- Periodic compression with embeddings + retrieval.

**Why this wins:**

- Recent literal user messages are preserved verbatim (within a 20k token budget). The model can still answer "what file did you tell me to look at?"
- The summary is *purpose-built* via the compaction prompt — it's a handoff, not a recap. Different from naive `summarize_messages(messages)`.
- The summary is wrapped with a recognizable prefix so future compactions don't recursively summarize summaries.

**Cost:**

- Costs a model call. Vibe mitigates by routing it to a smaller `compaction_model`.
- Lossy. Tool result details (read_file contents, grep output) are mostly gone. The model has to re-read if it needs specifics.
- Adds a code path (`compact()`, `_reset_session`) that's complex and has its own failure modes.

The team's bet: human-quality summarization of a 100k-token session is achievable by a small LLM, and the model on the other side can pick up from a good summary. Empirically this works.

## 6. Sub-agents as a tool

**The choice:** The `task` tool spawns a fresh `AgentLoop(is_subagent=True)` and runs it to completion, returning the assistant text as a tool result.

**Alternatives:**

- A separate harness API for subagents (`loop.spawn_subagent`).
- No subagents — let the parent do all work.
- Process-isolated subagents (subprocess).

**Why this wins:**

- The model already understands tools. "Delegate this work to `explore`" is just a tool call.
- Context isolation is automatic. Subagent has fresh messages, doesn't see parent's history.
- Permission store is shared but conversation isn't — the right cut.

**Cost:**

- The recursion guard (`agent_type != SUBAGENT → error`) is required. Without it the agent could spawn copies of itself indefinitely.
- The parent's UI has to render the subagent's progress events somehow. Vibe surfaces them as `ToolStreamEvent`s, which is workable but adds visual noise.

This pattern is starting to converge across agent frameworks (Claude Code, Cursor's "spawn agent", etc.). Vibe's implementation is among the cleanest because the recursion limit is enforced at the type level.

## 7. Skills as Markdown files, not code

**The choice:** A skill is a directory with `SKILL.md` containing frontmatter + body. Discovered from disk. Triggered as slash commands or via the `skill` tool.

**Alternatives:**

- Skills as Python classes.
- Skills as MCP servers.
- Skills as templated prompts in code.

**Why this wins:**

- The barrier to writing a skill is zero. Make a directory, write Markdown.
- No code review needed for adding a workflow — it's a doc.
- Discovery is trivial.

**Cost:**

- No execution semantics. A skill is just text — it can't enforce its own constraints.
- `allowed-tools` exists in frontmatter but isn't strictly enforced (experimental).
- Names must follow strict regex (lowercase-with-hyphens) so they're unambiguous as slash commands.

The choice is essentially "the lowest-friction extension point possible." It works because LLMs are good at following prompted instructions.

## 8. Hooks as subprocesses, not in-process callbacks

**The choice:** A `POST_AGENT_TURN` hook is a shell command. The harness runs it as a subprocess with a timeout. Exit code 2 with stdout means "retry, here's the injected user message."

**Alternatives:**

- In-process callbacks (Python functions registered with the harness).
- Webhooks.
- Embedded scripting language (Lua, etc.).

**Why this wins:**

- Language-agnostic. Hook authors write Bash, Python, Go — whatever.
- Process isolation. A misbehaving hook can't crash the harness.
- The contract (exit code + stdout) is universal.

**Cost:**

- Subprocess startup is slow (30-100ms). On every turn this adds up.
- Hooks can't easily read the conversation — they get `transcript_path` and have to parse JSONL.
- Cross-platform subtleties (shell quoting, env vars).

This is the right call for the use case (linting, type-checking, notifying). For tighter coupling, middleware is the answer.

## 9. AGENTS.md, not config-only customization

**The choice:** Project-level and user-level conventions live in Markdown files (`AGENTS.md`) that are inlined into the system prompt at startup.

**Alternatives:**

- TOML config keys for everything.
- A custom system prompt entirely overriding the default.
- A "rules" engine.

**Why this wins:**

- Markdown is the model's native input. The model reads AGENTS.md the same way it reads any other text.
- Project conventions belong in the repo. `AGENTS.md` is git-trackable.
- User conventions belong in `~/.vibe/`. Same.

**Cost:**

- AGENTS.md can grow large. There's no warning if your AGENTS.md takes 10% of the context window.
- No structure enforcement. A poorly written AGENTS.md is just bad prompt engineering.

## 10. Pydantic everywhere

**The choice:** All public-ish data shapes are Pydantic models. Args, results, config, events, hook config, skill metadata. Even tool state.

**Alternatives:**

- Dataclasses + manual validation.
- TypedDicts.
- Plain dicts.

**Why this wins:**

- Validation is uniform. Same `model_validate` everywhere.
- JSON schema generation for tools is free.
- Pyright catches misuse at the boundary.
- Frozen models for events make equality and hashing free.

**Cost:**

- Pydantic v2 is fast but not free. Per-call validation adds latency.
- Generic Pydantic models can confuse Pyright. AGENTS.md documents workarounds.
- A Pydantic schema change is sometimes a breaking change to serialized state on disk.

The team's bet is that Pydantic's correctness + type-safety win beats the runtime cost. For an agent harness with strict input parsing, it's hard to argue otherwise.

## 11. Session reset over context clear

**The choice:** Compaction and `/clear` *reset the session* — new session id, old one stored as `parent_session_id`. Don't just truncate the message list in place.

**Alternatives:**

- Just clear `messages` and keep going on the same session.
- New session, parent forgotten.

**Why this wins:**

- Lineage is reconstructable. You can always find the original session by walking parent ids.
- Telemetry is correct. Each session has a coherent start/end with consistent stats.
- The next turn's `prompt_tokens` matches the fresh history exactly — no carryover ambiguity.

**Cost:**

- Slightly more disk usage (multiple session dirs per logical "conversation").
- The UI has to handle "you're now in session X, but the conversation visually continues from session X-1."

This is the right move for any agent that runs long. Session as an immutable unit + lineage links is a clean abstraction.

## 12. Backend abstraction, not vendored providers

**The choice:** `BackendLike` is a protocol. Backends are factory-instantiated from `provider.backend` (mistral, openai, anthropic, etc.). The agent loop doesn't know which provider it's using.

**Alternatives:**

- Hard-code Mistral.
- Wrap a single SDK like LiteLLM.

**Why this wins:**

- Multi-provider out of the box. Users can mix-and-match: Mistral for main, GPT-4o for compaction.
- Easy to add new providers.
- The protocol is small (`complete`, `complete_streaming`, `count_tokens`).

**Cost:**

- Provider-specific features (system prompts vs. message-based, image inputs, etc.) get normalized to a lowest-common-denominator API.
- Some tool-call schemas vary across providers. `APIToolFormatHandler` papers over this but it's still a friction point.

## Patterns the harness deliberately *avoids*

- **Vector retrieval / RAG over conversation history.** The compaction approach replaces retrieval. The team's bet is "summarize is good enough" beats "retrieve relevant chunks" for coding agents.
- **Function-calling with hand-written JSON schemas.** Pydantic generates schemas; humans don't touch them.
- **Async-everywhere-or-bust.** Some code is sync (e.g., `_compute_search_paths`) because there's no benefit to being async. The team picks the right paradigm per case.
- **Single global config.** Agent profiles let any session run with a derived config. The base config is just the starting point.
- **Reactive UI bound to harness state.** The UI consumes events; it doesn't have a magic two-way binding.

## What you should take away

The harness is a series of explicit, justifiable design choices, not a "natural" emergent structure. Most of the choices follow a pattern:

> **Make the harness simple by making the contract narrow, and let users / models / tools extend along well-defined extension points.**

Tools, skills, agents, hooks, middleware, MCP servers — all five are extension points. None of them require modifying the harness's core. That's the real lesson for anyone building their own agent runtime: design the *contract*, not the implementations.

---

Next: [`../learning-path.md`](../learning-path.md) — the recommended order to read the source.
