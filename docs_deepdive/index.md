# Mistral Vibe Agent Harness — Learner's Guide

A deep tour of the **agent harness** inside [`mistralai/mistral-vibe`](https://github.com/mistralai/mistral-vibe). The goal of this guide is not to teach you Mistral Vibe as a product — it is to teach you **how a production CLI coding agent is built** using Vibe as a worked example.

If you have ever wondered *what actually sits between the user typing a message and the model deciding to call `bash`*, this guide takes that runtime apart, names each piece, and shows you the exact file and class that implements it.

## How to use this guide

| Order | Read | Why |
|------|------|-----|
| 1 | [What is an agent harness?](overview/what-is-an-agent-harness.md) | Builds the mental model. Worth reading even if you know the term. |
| 2 | [Key concepts](overview/key-concepts.md) | Glossary every later doc assumes. |
| 3 | [Data flow of one turn](architecture/data-flow.md) | End-to-end trace of a single user message. Anchor for everything else. |
| 4 | The `concepts/` files | Deep dives on each subsystem, in any order. |
| 5 | [Design decisions](architecture/design-decisions.md) | The *why* — non-obvious shape choices and their tradeoffs. |
| 6 | [Learning path through the source](learning-path.md) | The exact order to read the `vibe/core/` directory. |

## Two study documents

This guide ships two study documents at different scales:

| Document | Scope | Use when |
|---|---|---|
| [study-plan.md](study-plan.md) | **Full lifecycle, 6 phases, ~30–40h.** Setup → mental model → read core → compare other harnesses → build minimal → extend → synthesize. | You want the full journey from zero to *"I can build and defend my own harness."* |
| [learning-path.md](learning-path.md) | **Source-code reading only, 20 steps, ~3–4h.** A file-by-file tour of `vibe/core/`. | You already have an agent-harness mental model and just need to load Vibe's specifics. Also: this is Phase 2 of `study-plan.md`. |

If unsure, start with [study-plan.md](study-plan.md) — Phase 2 hands you off to [learning-path.md](learning-path.md) at the right moment.

## Concepts

| Subsystem | File | What you learn |
|-----------|------|----------------|
| Agent loop | [concepts/agent-loop.md](concepts/agent-loop.md) | The turn machine — how a single user prompt expands into N model calls and tool executions |
| System prompt | [concepts/system-prompt.md](concepts/system-prompt.md) | How the static instructions for the model are assembled and rebuilt |
| Tools | [concepts/tools.md](concepts/tools.md) | Tool contract, discovery, execution, streaming, error handling |
| Permissions | [concepts/permissions.md](concepts/permissions.md) | Three-level permission model and per-invocation overrides |
| Middleware | [concepts/middleware.md](concepts/middleware.md) | The before-turn pipeline that gates the loop |
| Context management | [concepts/context-management.md](concepts/context-management.md) | Compaction, warnings, limits — the bag of tricks for a finite window |
| Memory and state | [concepts/memory-and-state.md](concepts/memory-and-state.md) | What persists and where: messages, todos, scratchpad, sessions, AGENTS.md |
| Agents and subagents | [concepts/agents-and-subagents.md](concepts/agents-and-subagents.md) | Agent profiles, mode switching, task delegation |
| Skills | [concepts/skills.md](concepts/skills.md) | User-invocable prompt packs and slash commands |
| Hooks | [concepts/hooks.md](concepts/hooks.md) | The post-turn extension point for external programs |
| Human in the loop | [concepts/human-in-the-loop.md](concepts/human-in-the-loop.md) | Approvals, clarifying questions, plan review |

## Architecture

- [Design decisions](architecture/design-decisions.md) — the *why* behind key choices: async generators, Pydantic everywhere, sub-agent isolation, compaction over truncation
- [Data flow](architecture/data-flow.md) — annotated trace of one user message from keystroke to final assistant text

## Pointers into the source

| You're looking for | Open this |
|--------------------|-----------|
| The main loop | [`vibe/core/agent_loop.py`](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/agent_loop.py) |
| Tool base class | [`vibe/core/tools/base.py`](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/tools/base.py) |
| All built-in tools | [`vibe/core/tools/builtins/`](https://github.com/mistralai/mistral-vibe/tree/main/vibe/core/tools/builtins) |
| Middleware pipeline | [`vibe/core/middleware.py`](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/middleware.py) |
| Compaction | [`vibe/core/compaction.py`](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/compaction.py) + `agent_loop.py:compact()` |
| System prompt assembly | [`vibe/core/system_prompt.py`](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/system_prompt.py) |
| Built-in agent profiles | [`vibe/core/agents/models.py`](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/agents/models.py) |
| The actual system prompt text | [`vibe/core/prompts/cli.md`](https://github.com/mistralai/mistral-vibe/blob/main/vibe/core/prompts/cli.md) |

> Every file reference in this guide uses Vibe's own conventions: `path:line` (e.g. `vibe/core/agent_loop.py:874`). Paths are relative to the repo root.
