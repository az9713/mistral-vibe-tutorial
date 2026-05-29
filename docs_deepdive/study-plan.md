# Study plan

The full lifecycle plan to go from *"never seen an agent harness"* to *"I can build and defend my own"*. Six phases, ~30–40 hours total. Doable as three weekends or one focused week.

Every phase has a **deliverable** and a **done-when** check. Don't move on without it.

> This document is the *meta-plan*. [learning-path.md](learning-path.md) is the file-by-file reading guide used inside Phase 2. See [§ How this relates to learning-path.md](#how-this-relates-to-learning-pathmd) at the bottom for when to use which.

---

## Time budget

| Phase | Hours | Cumulative |
|-------|------:|-----------:|
| 0. Setup          |  1 |  1 |
| 1. Mental model   |  3 |  4 |
| 2. Read Vibe core | 10 | 14 |
| 3. Compare        |  4 | 18 |
| 4. Build minimal  | 12 | 30 |
| 5. Extend         |  8 | 38 |
| 6. Synthesize     |  3 | 41 |

---

## Phase 0 — Setup (1 hour)

**Goal:** make sure you can actually run things.

- `git clone https://github.com/mistralai/mistral-vibe.git && cd mistral-vibe && uv sync --all-extras` (already done in this workspace — you're inside it).
- Set `MISTRAL_API_KEY` in `~/.vibe/.env`.
- Run `vibe` interactively. Get to a working session. Try `/help`, then ask it to read a file.
- `git clone https://github.com/huggingface/smolagents.git` (for the contrast reading in Phase 3).
- Open [`docs_deepdive/`](.) in a markdown viewer — it's your guide.

**Done when:** you have a working `vibe` session and you know where Vibe stores sessions (`~/.vibe/sessions/`).

---

## Phase 1 — Mental model (3 hours)

**Goal:** understand what an agent harness *is* before reading any code.

Read in this order:
1. [overview/what-is-an-agent-harness.md](overview/what-is-an-agent-harness.md) — 10 min
2. [overview/key-concepts.md](overview/key-concepts.md) — 15 min
3. [architecture/data-flow.md](architecture/data-flow.md) end-to-end — 20 min. **This is the trace that anchors everything.**
4. `vibe/core/prompts/cli.md` directly — the actual system prompt the model sees. 10 min.
5. Run a Vibe session and observe one task. Watch the events stream past. Try to name each one as it appears.

**Output:** a one-page sketch of the loop (paper or doc) — boxes for User → Harness → Model → Tools, arrows labelled. **Draw without looking at anything.**

**Done when:** you can explain in 90 seconds, with no notes: *"what is the loop, what are the four phases of a turn, why does it stop?"*

---

## Phase 2 — Read Vibe's core (10 hours)

**Goal:** internalize one production harness.

Follow [learning-path.md](learning-path.md) step by step (steps 0–20). It's ordered so each file builds on what came before. Time budget per step is in that document.

After each step, write **one sentence** on what surprised you. Keep these — they're your "I get it now" moments and they're worth more than notes.

**Milestones inside this phase:**
- After step 4 (`vibe/core/tools/base.py`) — you can write a `BaseTool` subclass signature on paper from memory.
- After step 9 (`vibe/core/middleware.py`) — you can name all 6 built-in middlewares and what each does.
- After step 16 (`vibe/core/agent_loop.py`) — you can draw the entire `_conversation_loop` state machine on a whiteboard.

**Output:** pick one tool (`bash`, `task`, or `todo`) and write a 1-page explanation of how a single call to it flows through the harness, citing `file:line` for every step. Compare with [concepts/tools.md](concepts/tools.md) — your explanation should add details the doc doesn't.

**Done when:** you can answer all six "Try it" prompts at the end of [learning-path.md](learning-path.md#after-the-path-ways-to-test-understanding) without looking anything up.

---

## Phase 3 — Compare (4 hours)

**Goal:** see the design space, not just one implementation.

- **smolagents (~2h)** — read `src/smolagents/agents.py`. Notice that smolagents uses *code-action* style (the model writes Python that calls tools) instead of Vibe's *tool-call* style (structured tool calls). **Why?**
- **Aider (~1.5h)** — skim `aider/coders/`, focus on `base_coder.py` and one edit-format coder (e.g. `editblock_coder.py`). Notice Aider has no MCP, no skills, no subagents — but it has *repo maps* and *edit-format negotiation* that Vibe doesn't.
- **Claude Code docs (~30 min)** — read the sub-agents and agent-teams pages. Note what Claude Code does that Vibe doesn't.

**Output:** a comparison table — rows are subsystems (loop, tools, permissions, memory, multi-agent), columns are Vibe / smolagents / Aider / Claude Code. One phrase per cell.

**Done when:** you can argue, *with evidence*, why Vibe chose its shape — not just describe it. E.g., *"Vibe uses tool-call style instead of code-action because…"*

---

## Phase 4 — Build minimal (12 hours)

**Goal:** internalize by writing one yourself.

Build a tiny CLI coding agent from scratch, in any language. Target: **under 500 lines of core code**. Hard cap — if you blow past it you've over-designed.

**Must have:**
- Turn loop with conversation history (list of role-tagged messages).
- Three tools minimum: `read_file`, `write_file`, `bash`.
- Per-tool permission with `ALWAYS` / `ASK` / `NEVER`.
- A system prompt assembled from a base + tool descriptions + cwd context.
- Events streamed to the terminal as work happens.

**Must NOT have** (resist): middleware, hooks, MCP, skills, subagents, compaction. You're proving the loop works.

**Suggested approach:** start by copying the data-flow trace from [architecture/data-flow.md](architecture/data-flow.md) and writing comments for each step in an empty file. Then fill in the comments with code. This is how Vibe's design becomes yours.

**Output:** a working `myagent` you can run that successfully reads, edits, and runs commands on a small test project (try: *"add a function to this file that does X, then run the tests"*).

**Done when:** the agent completes a 2-step task (read a file, edit it) with permission prompts working. Code committed somewhere.

---

## Phase 5 — Extend (8 hours)

**Goal:** add one of the "interesting" features and feel where the design pressure is.

Pick **one**. Don't try more than one — depth, not breadth.

| Feature | Why it's the right next step |
|---|---|
| **Compaction** | The single most consequential pattern beyond the basic loop. Forces you to think about message preservation and session lineage. |
| **Subagent via a `task` tool** | Teaches scoping, isolation, and the *"tool can spawn another loop"* pattern. |
| **Hooks** | Easy to add. Teaches the *"exit code as protocol"* pattern; connects you to OS-level concerns. |
| **An MCP client** | Teaches the external-tool integration pattern. Biggest payoff for a real harness. |

Before coding, re-read the corresponding [`concepts/<name>.md`](concepts/). Then implement.

**Output:** the feature works end-to-end. If you picked compaction, your agent can have a long conversation, hit a threshold, summarize, and continue successfully.

**Done when:** you can articulate, in writing, what was harder than you expected — and what Vibe design decision you now understand *because you hit the same wall*.

---

## Phase 6 — Synthesize (3 hours)

**Goal:** make sure what you learned sticks.

Write **one** of:
- A blog post comparing your tiny harness to Vibe — what you kept, what you simplified, what you don't yet understand why Vibe does.
- A design doc for *"the agent harness I'd build for $YOUR_PROJECT"* — concrete decisions, justified.
- A 30-minute talk you could give to your team explaining how agent harnesses work, using your code as the worked example.

The output matters less than the act of producing one. Writing forces you to find the gaps.

**Done when:** another engineer reads it and asks a question you can answer.

---

## Common ways this plan fails

- **Skipping Phase 1** because *"I already know what an agent is."* You don't, until you can draw the loop on paper. 30 min of drawing saves 5 hours of confused reading.
- **Skipping Phase 4** because *"I'll just keep reading."* Reading is half the value. The other half only comes from building.
- **Going wide in Phase 5.** Pick one feature. Depth > breadth.
- **Not writing Phase 6.** Writing is the integration step. Skip it and the knowledge degrades within weeks.

## One concrete tip

After Phase 2, set up jump-to-symbol on Vibe's source in your editor (`uv sync --all-extras` from this directory, then open in your editor of choice). When you're building in Phase 4, you'll constantly want to peek at how Vibe handled something. Make peeking instant.

---

## How this relates to learning-path.md

This repo ships **two** study documents. They are not alternatives — they are different scales of the same journey.

| | **study-plan.md** (this file) | **learning-path.md** |
|---|---|---|
| **Scope** | Full lifecycle: setup → build → synthesize | Source-code reading only |
| **Length** | 6 phases, ~30–40h total | 20 ordered steps through `vibe/core/`, ~3–4h focused reading |
| **Includes building?** | Yes (Phase 4 & 5 are ~20h of coding) | No |
| **Includes comparing other harnesses?** | Yes (Phase 3) | No |
| **Includes writing/synthesis?** | Yes (Phase 6) | No |
| **Where it fits** | The meta-plan | Phase 2 of the meta-plan |
| **Output** | A working harness you wrote + a synthesis doc | A loaded mental model of Vibe's internals |

### When to use which

- **Use [study-plan.md](study-plan.md)** when you are starting out and want the *full* journey from zero to "I can defend my own design." Pick this if you want to *build* a harness, not just understand one.
- **Use [learning-path.md](learning-path.md)** when you already have an agent-harness mental model (from another codebase, prior reading, or experience) and you just need to load Vibe's specific internals into your head. Pick this if your goal is *"learn this codebase fast"* rather than *"learn the discipline."*
- **Use both, in order**, when following the full plan: study-plan.md is the outer frame, learning-path.md is what you execute during Phase 2.

If unsure: start with study-plan.md. Phase 2 will hand you off to learning-path.md at the right moment.
