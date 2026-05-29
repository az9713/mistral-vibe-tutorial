# Memory and state

The word "memory" is overloaded. Different parts of a coding agent's "memory" live in different places, persist for different durations, and answer different questions. This doc inventories every kind of state Vibe carries and explains what each is good for.

## The single most important distinction

> **The model has no memory.** Every API call sends the full message history.

So when we say "the agent remembers", we actually mean one of three things:

1. **The history is in the next API request.** This is *in-context memory*. It's bounded by the model's window.
2. **The harness has it in Python objects and will put it in the next request.** This is *runtime memory* — todo list, scratchpad, permission rules, agent state.
3. **It's on disk and will be re-read.** This is *persistent memory* — session logs, AGENTS.md files, config, plan files, the user's actual code.

Every "memory" feature in Vibe lives in one of those three categories. Understanding which is the most useful disambiguation a learner can make.

## The full inventory

| Kind | Lives in | Survives | Visibility to model |
|------|----------|----------|---------------------|
| System prompt | `messages[0]` | The session | Full text every turn |
| Conversation history | `messages[1:]` | Until compaction | Full text every turn |
| Stats (steps, cost, tokens) | `AgentStats` | The session | Not directly; affects middleware decisions |
| Todo list | `Todo` tool state | The session, lost on compaction | Only when the model calls `todo(action="read")` |
| Scratchpad files | `/tmp/vibe-scratchpad-<sid>-XXX/` | The session, removed by OS | When the model reads them |
| Approved permission rules | `PermissionStore` | The session | Not visible; affects whether tools execute silently |
| File snapshots | `RewindManager` | The session | Not visible; restored on `/rewind` |
| Plan file | `~/.vibe/plans/<id>.md` | Across sessions on disk | Read by the model when it `read_file`s it |
| Session log | `~/.vibe/sessions/<id>/` | Forever | Loaded on `vibe --continue` |
| User AGENTS.md | `~/.vibe/AGENTS.md` | Forever | Inlined into system prompt at startup and on rebuild |
| Project AGENTS.md | `<repo>/AGENTS.md` | In git | Same |
| Compaction summary | `messages[-1]` after compaction | Until next compaction | Yes — replaces prior history |
| Parent session id | `AgentLoop.parent_session_id` | The session | Not visible; metadata on saved log |
| Config (`config.toml`) | `~/.vibe/config.toml` | Forever | Indirect — shapes available tools, agents, models |

That's the whole list. The rest of this doc walks the interesting ones.

## In-context memory: `messages`

`AgentLoop.messages` is a `MessageList` (in `vibe/core/types.py`). It's a list of `LLMMessage` with observer hooks. Every modification fires the observer so the UI / session logger see the change.

What it holds:

- `messages[0]` — the system message. Mutable via `messages.update_system_prompt(s)`.
- `messages[1:]` — the conversation. User, assistant, and tool messages in order.

Sublist semantics matter:

- `messages.append(m)` — normal addition. Fires observer.
- `messages.silent()` — context manager that suspends observers. Used by `compact()` to insert and remove the summary-request user message without UI noise.
- `messages.reset(new_messages)` — wholesale replacement. Used by compaction and `clear_history`.
- `messages.insert(i, m)` — used by `_fill_missing_tool_responses` to backfill orphaned tool calls.

## Runtime tool state: the todo list

The `todo` tool maintains `state: TodoState` on its instance:

```python
class TodoState(BaseToolState):
    todos: list[TodoItem] = Field(default_factory=list)
```

Because the tool is *cached as an instance* in the `ToolManager`, state persists across calls. The model can call `todo(action="write", todos=[...])` once to set it, then `todo(action="read")` any number of times to inspect.

Critically, the todos are *not* automatically visible to the model — only when it reads them. So the todo list is a model-managed working memory: the model decides when to update and when to consult.

What survives compaction? The `Todo` tool *instance* survives (it's on the loop, not in `messages`), so the in-memory todos persist across compaction. But the *conversation references* to them in prior turns are gone. In practice the model re-reads the todo on its next turn after compaction.

## Runtime files: the scratchpad

`init_scratchpad(session_id)` creates a temp directory like `/tmp/vibe-scratchpad-<short_session_id>-XXX/`. The path is stored in a process-wide dict and exposed via `get_scratchpad_dir(session_id)`.

The system prompt tells the model:

> You have a scratchpad directory at: `<path>`
> Use this for temporary files: intermediate results, draft scripts, working files, outputs that don't belong in the project.
> Files here are automatically allowed — no permission prompts.
> Session-scoped. Shared with subagents.

Two crucial properties:

1. **Permission bypass.** `WriteFile.resolve_permission(args)` returns `ALWAYS` if the target is inside the scratchpad. The model can write there freely.
2. **Subagent sharing.** When the `task` tool spawns a subagent, the parent's scratchpad path is included in the subagent's task text. Subagents can read what the parent wrote and vice versa.

The scratchpad is the answer to "where does the agent put its drafts." It's not for the user's project — that's what `write_file` for project paths is for.

## Persistent state: AGENTS.md

This is the harness's "you've taught me something" file. Two scopes:

- **User-level**: `~/.vibe/AGENTS.md` (or `$VIBE_HOME/AGENTS.md`). Loaded on every session, in every project.
- **Project-level**: `AGENTS.md` at the project root. Loaded when the session's cwd is in that project tree.

Both are inlined into the system prompt by `get_universal_system_prompt`'s "AGENTS.md files" section (see [`system-prompt.md`](system-prompt.md)). The model sees them as static text.

What goes in here:

- Conventions ("always use `uv` for Python", "match existing style", "ban relative imports").
- Project layout ("`vibe/core` is the engine, `vibe/cli` is the TUI").
- Commands the agent should know ("`uv run pytest` runs the suite", "`uv run pyright` for types").
- Gotchas ("don't `--amend` commits", "always rebase, never force-push").

Vibe's own [AGENTS.md](https://github.com/mistralai/mistral-vibe/blob/main/AGENTS.md) is a great example to study.

AGENTS.md is the answer to "how do I teach my agent something durable about this codebase?" It survives across sessions, models, machines (when checked into the repo), and even harness upgrades.

## Persistent state: session logs

`SessionLogger` (`vibe/core/session/session_logger.py`) writes the full state of an interaction to disk:

```
~/.vibe/sessions/<session_id>/
├── messages.jsonl           # full conversation history
├── metadata.json            # title, parent_session_id, scheduled loops, ...
└── agents/                  # subagent session logs
```

After every assistant or tool message, `save_interaction(...)` is called. So a session log is the closest thing to "ground truth" — even if the process crashes, the log is up to date.

`vibe --continue` resumes the most recent session. `vibe --resume <id>` resumes a specific one. The harness loads the messages, recomputes stats, restores middleware, restarts the loop.

## Persistent state: rewind snapshots

`RewindManager` (`vibe/core/rewind/manager.py`) captures `FileSnapshot`s before file-mutating tools run. `tool.get_file_snapshot(args)` returns `FileSnapshot(path, content)` where `content` is the file bytes before the write (or `None` if it didn't exist).

A `/rewind` command unwinds:

1. Restore each snapshot to disk.
2. Truncate the message history back to a checkpoint.
3. Reset stats.

Rewind is a *user-level undo* for an agent session. Knowing it exists changes how brave the user can be about letting the agent run.

## Persistent state: the plan file

When the agent is in `plan` mode, it writes to `~/.vibe/plans/<id>.md`. This is just a file — write_file edits it, search_replace can patch it — but it's special in two ways:

1. The `plan` agent's overrides whitelist *only* this path for write_file / search_replace (see `agents/models.py:_plan_overrides`).
2. `PlanSession` (`vibe/core/plan_session.py`) watches the file. If the user manually edits it (in another editor), the next turn injects the updated plan into the conversation:

   > The user has manually updated the plan file. Here is the updated version — use this as the source of truth for implementation: ...

So the plan file is a *shared artifact* between the user and the agent. The harness uses filesystem watching to keep it in sync.

## Compaction summary as memory

After `compact()`, the conversation is `[system, prior_users..., summary]`. The summary is the model's own writing about what happened. Calling it "memory" is accurate but understated — it's *curated* memory, written by an LLM with the explicit job of distilling the session.

For long-running sessions, compaction summaries become the dominant form of conversational memory. The user's literal messages are still there (per `collect_prior_user_messages` budget), but the working context is the summary.

## Parent session id: lineage

When a session resets (compaction or `/clear`), the new session gets a new id and the old one is stored as `parent_session_id`. The session log records this. The result: even after a compaction, you can trace back to the original session, replay the actual messages, and audit what was summarized.

This is *the* feature that makes compaction defensible. The summary may be lossy, but the original is recoverable.

## Subagent state: shared and not shared

When the `task` tool spawns a subagent:

- **Shared** with the parent: `PermissionStore` (so subagent doesn't independently re-prompt for everything), `scratchpad_dir` (so files are accessible), `approval_callback` (so subagent can still ask the user), `entrypoint_metadata` (telemetry consistency).
- **Not shared**: `messages` (subagent gets fresh history), `stats` (own token/cost budget), `session_logger` (writes to `<parent>/agents/<sub_session>/`), `agent_profile` (the subagent's profile, e.g. `explore`).

This is the right cut. Working memory (filesystem, permissions) flows down; conversational state stays separate so the subagent's noise doesn't pollute the parent.

## Try it: count your "memory" right now

Open an interactive Vibe session. Have a 5-turn conversation that involves reading a file and writing a small todo. Then:

1. Open `~/.vibe/sessions/` and find your session dir. Read `messages.jsonl` — that's your full conversation history.
2. Look at `/tmp/vibe-scratchpad-*` — see if anything is there.
3. Open `~/.vibe/AGENTS.md` (it may not exist; that's fine). Anything here is in your system prompt right now.
4. In your session, type a question that depends on something from turn 1. The model still knows it — that's `messages` working.
5. Type `/compact`. After it finishes, ask the same question. The model still knows it — that's the summary working.

You've now seen four distinct kinds of memory cooperate in one session.
