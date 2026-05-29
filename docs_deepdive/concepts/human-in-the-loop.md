# Human in the loop

The harness has three distinct mechanisms for asking the user something during a turn. They look superficially similar but serve different purposes. Knowing which is which is the difference between "agent feels collaborative" and "agent feels needy."

## The three mechanisms

| Mechanism | Used for | Initiated by | UI shape |
|-----------|----------|--------------|----------|
| **Approval callback** | "Can I run this tool?" | Harness (every `ASK` tool call) | y/n/always/edit prompt |
| **`ask_user_question` tool** | "Which option do you want?" | Model (decides to ask) | Multi-choice picker |
| **`exit_plan_mode` tool** | "Is this plan good?" | Model in plan mode | Plan review dialog |

All three route through harness-owned callbacks. The UI implements them. The model can request them in different ways.

## Approval callback

The harness's primary HITL mechanism. Already covered in [`permissions.md`](permissions.md); this section focuses on the contract from the UI's perspective.

`AgentLoop.set_approval_callback(callback)` sets the function the harness will call. Signature:

```python
ApprovalCallback = Callable[
    [tool_name: str, args: BaseModel, tool_call_id: str, required_permissions: list[RequiredPermission]],
    Awaitable[tuple[ApprovalResponse, str | None]]
]

class ApprovalResponse(StrEnum):
    YES = auto()
    NO = auto()
```

When invoked, the UI should:

1. Pause input handling for the rest of the agent's output.
2. Render the tool name, the validated args (as JSON or formatted), and the list of `RequiredPermission`s the user is being asked to grant.
3. Offer four typical choices:
   - **Yes** → return `(YES, None)`.
   - **No** → return `(NO, None)` or `(NO, "<feedback to the model>")`.
   - **Always (session)** → handle via `loop.approve_always(tool_name, required_permissions)` then return `(YES, None)`.
   - **Always (saved to config)** → handle via `loop.approve_always(..., save_permanently=True)` then return `(YES, None)`.

The feedback string when saying "No" is important: it becomes the tool result the model sees. So a "No" with feedback `"This would overwrite a generated file, edit src/foo.ts instead"` is *productive* — the model gets a hint, not just a refusal.

In the TUI, edit-the-args is also offered for some tools. The user can amend, say, the bash command before approving. The amended args go through the same validation; if they pass, the tool runs with them.

## The ask_user_question tool

The model decides it wants to ask. It calls:

```python
ask_user_question(
    questions=[
        Question(
            question="Which database should we use?",
            header="Database",
            options=[
                Choice(label="PostgreSQL", description="Relational, strong consistency"),
                Choice(label="MongoDB", description="Document, flexible schema"),
                Choice(label="SQLite", description="Embedded, no server"),
            ],
            multi_select=False,
            hide_other=False,
        ),
        # ... up to 4 questions
    ],
    footer_note="You can pick a different option via 'Other'",
)
```

The args model (`vibe/core/tools/builtins/ask_user_question.py`):

```python
class Question(BaseModel):
    question: str
    header: str = ""        # 1-2 word chip label
    options: list[Choice] = Field(min_length=2, max_length=4)
    multi_select: bool = False
    hide_other: bool = False

class AskUserQuestionArgs(BaseModel):
    questions: list[Question] = Field(min_length=1, max_length=4)
    footer_note: str | None = None
```

Constraints:

- 2-4 options per question (forces the model to give the user a real choice, not a binary).
- 1-4 questions per call (the UI shows them as tabs if multiple).
- An "Other" free-text option is automatically appended unless `hide_other=True`.

The tool's `run` calls `ctx.user_input_callback(args)` which is implemented by the UI:

- Display the questions.
- Wait for the user's selections.
- Return `AskUserQuestionResult(answers=[Answer(question, answer, is_other)], cancelled=False)`.

If the user cancels (Esc), `cancelled=True` and the model sees a tool result indicating the cancellation. The model should treat this as "user doesn't want to answer right now" and continue with its best judgment.

### Why bake "Other" into the protocol?

A common failure mode of multiple-choice prompts is the model picking three options it can think of and missing the one the user actually wants. The automatic "Other" option means the user never feels boxed in — they can always type free text.

The trade-off: free text is harder for the model to act on. But that's still better than the user feeling forced.

### Why limit to 4 options / 4 questions?

`min_length=2` forces a real choice. `max_length=4` keeps the UI usable. Anything more than 4 options should probably be a different interaction (open prompt, search, etc.).

### When should the model use this vs. just asking?

The default Vibe system prompt biases toward *deciding rather than asking*: "When ambiguous, default to investigate" (cli.md). But certain decisions are genuinely user-only:

- Authentication method (OAuth vs. API key vs. service account).
- Storage choice (Postgres vs. SQLite).
- Code style (functional vs. OO).
- Naming conventions where multiple are reasonable.

In those cases the model should call `ask_user_question` rather than guess. The system prompt nudges this: see the "Interaction Design" section of `cli.md`.

## The exit_plan_mode tool

A specific HITL mechanism for the `plan` agent mode. When the model has finished writing a plan, it calls `exit_plan_mode()`. This triggers:

1. The harness yields a `PlanReviewRequestedEvent` with the path to the plan file.
2. The UI opens the plan file for the user (in $EDITOR, in a side pane, however).
3. The user reviews. They can manually edit the plan file. They can approve or reject.
4. On approval, the agent switches profile from `plan` to `default` (via `ctx.switch_agent_callback("default")`).
5. The next user message kicks off implementation.

The clever bit is `PlanSession` (`vibe/core/plan_session.py`): it snapshots a hash of the plan file when `exit_plan_mode` is called, and detects if the user manually edited the file afterwards. If they did, on the next turn the harness injects:

> The user has manually updated the plan file. Here is the updated version — use this as the source of truth for implementation: `<file content>`

So the user's edits are *first-class*: they become part of the conversation. Without this, the user might edit the plan, the agent would still operate on its earlier in-memory version, and the two would diverge.

This is the right shape for plan-then-execute: the artifact is the plan file, the file is editable by both parties, the agent re-reads on every transition.

## Common pattern: tool feedback as conversation

A theme across all three HITL mechanisms: **the user's response becomes a normal message in the conversation**.

- Approval-callback "No with feedback" → the feedback becomes the `role=tool` result the model sees on the next turn.
- `ask_user_question` answers → become a `role=tool` result with a structured `AskUserQuestionResult`.
- `exit_plan_mode` + user edits → injected as a `role=user` message before the next turn.

The model treats them all the same way: as new information in history that it should react to. This uniformity is what lets the model handle them gracefully without special-casing each one.

## Headless mode

When `headless=True` (e.g. `vibe -p "..."`), the harness:

- Adds a "Headless Mode" section to the system prompt: "no human is available... make the best judgment call and proceed."
- Default approval behavior is fail-closed: `_ask_approval` returns SKIP if `approval_callback` is `None`.
- `ask_user_question` will simply error or return a cancellation result — the tool effectively becomes useless.

The combination means a headless run will:
- Use only tools whose default permission is `ALWAYS` or which the agent profile sets to `ALWAYS`.
- Skip tools that need approval (with feedback to the model).
- Not block on user questions.

The pragmatic choice for CI / scripts is `vibe -p --agent auto-approve`: bypass permissions entirely, do the task in one pass.

## Cancellation propagation

The user's Ctrl+C produces a CancelledError that propagates up:

1. Whatever tool is currently running gets `CancelledError` raised inside `asyncio.iter_chunks` or whatever it's awaiting.
2. The tool can catch and clean up (e.g., bash kills its subprocess), then re-raise.
3. `_execute_tool_call` catches CancelledError, emits a `<vibe_cancellation>` tool result, increments `stats.tool_calls_failed`, and re-raises.
4. `_conversation_loop` catches at the outer level, yields the cancellation event, returns.

Cancellation is *not* the same as the user saying "stop, do something different." It's specifically "abort the current call." If the user wants to redirect, they send a new message.

This is a subtle but important distinction. A new message after a cancelled call is appended as a *new* user message; the cancelled tool result is also in history. The model can see both and respond accordingly.

## Try it: read the approval callback

If you're using the TUI, the approval callback is implemented in `vibe/cli/textual_ui/`. Find the `approval_modal` or similar. Notice:

- It's a standard Textual modal screen.
- The "Always" button calls `loop.approve_always(tool_name, required_permissions)` before resolving.
- The "No" button optionally prompts for feedback.

That implementation, plus `AskUserQuestion`'s widget, are about 200 lines of UI code. Everything else is harness logic that the UI consumes.

---

That completes the concepts. Read [`../architecture/design-decisions.md`](../architecture/design-decisions.md) for the *why* behind the harness's overall shape.
