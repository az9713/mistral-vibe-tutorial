# Key concepts

Every term used in the rest of this guide. Read once, refer back as needed.

## Core runtime objects

**Agent loop** — The runtime that drives one user request to completion through multiple model calls and tool executions. Implemented by `AgentLoop` in `vibe/core/agent_loop.py`. There is exactly one `AgentLoop` per interactive session and one per spawned subagent.

**Turn** — One pass through the loop: middleware checks → model call → optional tool execution. A single user message produces multiple turns until the model emits a final assistant message with no tool calls. Counted in `AgentStats.steps`.

**Conversation history (`messages`)** — The ordered list of `LLMMessage` objects sent to the model on every API call. The first message is always the system prompt. Messages carry `role` (`system` / `user` / `assistant` / `tool`), optional `content`, optional `tool_calls`, and metadata like `injected` and `message_id`.

**Event** — A `BaseEvent` subclass yielded from the harness's async generators to the caller. The UI subscribes to these. Examples: `UserMessageEvent`, `AssistantEvent`, `ToolCallEvent`, `ToolResultEvent`, `CompactStartEvent`. Defined in `vibe/core/types.py`.

**Backend** — The thin LLM provider adapter. Implements `BackendLike` (see `vibe/core/llm/types.py`). Vibe ships adapters for Mistral, OpenAI, Anthropic, and others. The agent loop calls `backend.complete(...)` / `backend.complete_streaming(...)`.

## Tools

**Tool** — A `BaseTool` subclass with a Pydantic args model, Pydantic result model, config, and optional state. Implements `async def run(args, ctx)` as an async generator. Defined by `vibe/core/tools/base.py`.

**Built-in tools** — Tools shipped with Vibe in `vibe/core/tools/builtins/`: `bash`, `read_file`, `write_file`, `search_replace`, `grep`, `todo`, `task`, `ask_user_question`, `exit_plan_mode`, `webfetch`, `websearch`, `skill`.

**MCP tool** — A tool whose implementation lives in an external [Model Context Protocol](https://modelcontextprotocol.io/) server. Discovered through `MCPRegistry`. Wrapped as a `MCPTool` subclass at runtime.

**Connector tool** — A first-party MCP-style server hosted by Mistral. Discovered via `ConnectorRegistry`.

**Tool manager** — `ToolManager` (`vibe/core/tools/manager.py`). Walks search paths, imports `BaseTool` subclasses, applies enable/disable filters, and exposes `available_tools`.

**Tool permission** — `ALWAYS` (auto-execute), `ASK` (prompt the user), `NEVER` (refuse). Set per-tool in config or overridden per-invocation via `tool.resolve_permission(args)`.

**Required permission** — A finer-grained authorization a tool may request before running. Scopes: `COMMAND_PATTERN` (a bash command pattern), `OUTSIDE_DIRECTORY` (path leaves the workdir), `FILE_PATTERN`, `URL_PATTERN`. Defined in `vibe/core/tools/permissions.py`.

**Approved rule** — A previously-granted permission for a `(tool_name, scope, session_pattern)` triple, held in the `PermissionStore`. "Always allow" remembers a rule.

## Agents

**Agent profile** — A named configuration overlay on top of the user's base `VibeConfig`. Built-ins: `default`, `chat`, `plan`, `accept-edits`, `auto-approve`, `explore`, `lean`. Defined in `vibe/core/agents/models.py`. Custom profiles are TOML files in `~/.vibe/agents/` or project `.vibe/agents/`.

**Agent type** — Either `agent` (a profile usable as the primary agent via `--agent`) or `subagent` (a profile usable only via the `task` tool).

**Agent safety** — Label on a profile: `SAFE` / `NEUTRAL` / `DESTRUCTIVE` / `YOLO`. Surfaces in the UI to flag what a profile is willing to do without asking.

**Subagent** — An agent profile with `agent_type = "subagent"`. The `task` tool spawns a fresh `AgentLoop(is_subagent=True)` running the subagent. Subagents share the parent's `PermissionStore` and scratchpad but get their own messages, stats, and session log.

## Middleware

**Middleware** — A class implementing `before_turn(context) -> MiddlewareResult`. Runs at the start of each turn. Built-ins: `TurnLimitMiddleware`, `PriceLimitMiddleware`, `TokenLimitMiddleware`, `AutoCompactMiddleware`, `ContextWarningMiddleware`, `ReadOnlyAgentMiddleware`. Defined in `vibe/core/middleware.py`.

**Middleware action** — The verdict a middleware returns: `CONTINUE` (proceed normally), `STOP` (end the loop), `COMPACT` (trigger context compaction), `INJECT_MESSAGE` (insert a synthetic user message before the model call).

**Middleware pipeline** — `MiddlewarePipeline` runs middlewares in order. First `STOP`/`COMPACT` short-circuits; `INJECT_MESSAGE`s are combined into one injected message.

## Context management

**Compaction** — Summarizing the conversation history into a single short message so the loop can keep going. Implemented by `AgentLoop.compact()` in `agent_loop.py:1686`. Triggered automatically by `AutoCompactMiddleware`, manually by `/compact`, or programmatically.

**Compaction model** — A (usually smaller, cheaper) model used to produce the summary. Configured by `compaction_model` in `VibeConfig`. Defaults to the active model.

**Compaction summary** — The output of compaction, wrapped with `COMPACT_SUMMARY_PREFIX` and inserted as a synthetic user message. Defined in `vibe/core/prompts/compact_summary_prefix.md`.

**Context tokens** — The token count of the *current* request (prompt + completion). Distinct from session-cumulative totals. `AutoCompactMiddleware` compares `context_tokens` against `model.auto_compact_threshold`.

**Auto-compact threshold** — Per-model token count at which compaction fires. Set in `ModelConfig`.

## Memory & persistence

**Session** — One conversation, identified by `session_id`. Persisted on disk by `SessionLogger`. Replayable via `vibe --continue`.

**Parent session ID** — When a session is reset (compaction, `/clear`) or forked, the previous session ID is recorded as `parent_session_id` so the lineage can be reconstructed.

**Scratchpad** — A per-session temp directory created by `init_scratchpad()`. Files inside are exempt from the permission system. Shared with subagents. See `vibe/core/scratchpad.py`.

**Rewind** — Restoring files to the state they were in before a tool ran. Backed by `FileSnapshot`s captured by `tool.get_file_snapshot(args)` and stored in `RewindManager`.

**Todo list** — In-memory state held by the `todo` built-in tool. Persists across turns within a session because the tool instance is reused.

**AGENTS.md** — A Markdown file that the harness loads and inlines into the system prompt. Two scopes:
- User-level: `~/.vibe/AGENTS.md`
- Project-level: `AGENTS.md` at the project root (and any additional working dirs)

**Harness files manager** — `get_harness_files_manager()` in `vibe/core/config/harness_files.py`. Discovers AGENTS.md files, user/project tools/agents/skills directories.

## Skills

**Skill** — A directory containing `SKILL.md` (frontmatter + body). The body becomes a prompt that gets injected when the user types `/skillname` or when the model decides to use it via the `skill` tool. Defined by `vibe/core/skills/models.py`.

**Built-in skill** — Skills shipped with Vibe in `vibe/core/skills/builtins/`. The `vibe` skill documents the CLI itself.

**Skill discovery** — `SkillManager` walks `~/.vibe/skills/` and project `.vibe/skills/` and parses every `SKILL.md`. The frontmatter is validated against `SkillMetadata`.

## Hooks

**Hook** — An external command run at a lifecycle moment. Currently the only type is `POST_AGENT_TURN`. Configured in TOML. Defined by `vibe/core/hooks/models.py`.

**Hook exit codes** — `0` = success, `2` = retry (stdout becomes an injected user message, agent runs another turn). Anything else = warning logged but loop continues.

**Hook retry** — Each hook can fire up to 3 retries per turn. Tracked by `HookRetryState`.

## Human-in-the-loop

**Approval callback** — `ApprovalCallback`, the function the harness calls to ask the user whether a tool may execute. Set by `set_approval_callback`. The CLI's TUI implements this as the [y/n/always/edit] prompt.

**User input callback** — `UserInputCallback`, used by the `ask_user_question` tool to surface multi-choice questions to the UI.

**Plan session** — `PlanSession`. Tracks the path of `plan.md` while the agent is in `plan` mode. Detects manual user edits to inject the updated plan back into the conversation.

## Configuration

**VibeConfig** — The Pydantic root config. Loaded from `~/.vibe/config.toml`. Holds tools, providers, models, MCP servers, connectors, skills/agents paths, limits.

**Provider** — An LLM service (Mistral, OpenAI, …) with API base, key env var, headers.

**Model** — A model name bound to a provider, with pricing, alias, temperature, `auto_compact_threshold`, etc.

**Active model** — The model currently selected for the main turn loop. Resolved via `config.get_active_model()`.

---

Next: [Data flow of one turn](../architecture/data-flow.md), or jump into the `concepts/` directory.
