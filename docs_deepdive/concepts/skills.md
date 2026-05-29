# Skills

A skill is a reusable, user-invocable prompt with optional metadata. It's the harness's answer to "I keep typing the same long instructions" and "I want to package an expert workflow."

## The pattern

A skill is just a Markdown file with frontmatter:

```markdown
---
name: code-review
description: Reviews a diff against five quality axes...
license: MIT
allowed-tools: read_file grep bash
user-invocable: true
metadata:
  team: platform
---

You are reviewing code. For each file in the diff, do:
...
```

The file lives in a skills directory. The harness:

1. Discovers it at startup.
2. Lists it (name + description) in the system prompt's "Available Skills" section.
3. Makes the body available either:
   - To the user as a slash command (`/code-review`).
   - To the model as something it can load via the `skill` tool.

This pattern shows up in Claude Code, in Vercel's AI SDK, in many recent agent frameworks. Vibe's implementation is among the simplest.

## Vibe's implementation

### Discovery

`SkillManager` (`vibe/core/skills/manager.py`) walks search paths:

1. `config.skill_paths` (anything the user added).
2. Project skill dirs: `<project>/.vibe/skills/`.
3. User skill dirs: `~/.vibe/skills/`.

For each, it finds subdirectories containing `SKILL.md`. Each `SKILL.md` is parsed:

```python
def _parse_skill_file(self, skill_path: Path) -> SkillInfo:
    content = read_safe(skill_path).text
    frontmatter, body = parse_skill_markdown(content)
    metadata = SkillMetadata.model_validate(frontmatter)
    return SkillInfo.from_metadata(metadata, skill_path, prompt=body.strip())
```

Plus the built-in skills in `vibe/core/skills/builtins/` are added (the `vibe` skill in particular documents the CLI itself).

The result is `SkillManager.available_skills: Mapping[name, SkillInfo]`.

### Frontmatter schema

`SkillMetadata` (`vibe/core/skills/models.py`):

```python
class SkillMetadata(BaseModel):
    name: str = Field(..., pattern=r"^[a-z0-9]+(-[a-z0-9]+)*$", max_length=64)
    description: str = Field(..., max_length=1024)
    license: str | None = None
    compatibility: str | None = None
    metadata: dict[str, str] = {}
    allowed_tools: list[str] = []        # validation_alias: "allowed-tools"
    user_invocable: bool = True           # validation_alias: "user-invocable"
```

The name must match `^[a-z0-9]+(-[a-z0-9]+)*$` — same rule as npm package names. Forces consistency.

`user_invocable` controls whether the skill appears in the slash-command menu. If `false`, the skill is only accessible via the `skill` tool (model-driven).

### Surface to the model

In the system prompt's skills section (`system_prompt.py:_get_available_skills_section`):

```xml
<available_skills>
  <skill>
    <name>code-review</name>
    <description>Reviews a diff against...</description>
    <path>/Users/me/.vibe/skills/code-review/SKILL.md</path>
  </skill>
  ...
</available_skills>
```

The model can decide to use a skill in two ways:

1. **Read the SKILL.md directly** — `read_file(path)`. The full body becomes a tool result.
2. **Call the `skill` tool** — `skill(name="code-review")`. The tool loads the body and yields it as the tool result.

Both paths end with the skill body in the conversation as a tool result. The model then acts on it.

### Slash command flow

When the user types `/code-review`:

1. The TUI parses the input.
2. `SkillManager.parse_skill_command(text)` strips the `/`, looks up the skill, and returns a `ParsedSkillCommand`:

   ```python
   class ParsedSkillCommand(BaseModel):
       name: str
       content: str             # the skill's prompt body
       extra_instructions: str | None = None
   ```

3. `SkillManager.build_skill_prompt(text, parsed)`:
   - If `extra_instructions` is None (user typed just `/code-review`), use the skill's content as-is.
   - Otherwise (user typed `/code-review focus on tests`), append: `{text}\n\n{content}`.
4. The resulting string is sent through `act(...)` as a regular user message.

So the skill body **becomes the user's prompt** for that turn. The model sees:

```
<the skill body>
```

or, with extra instructions:

```
/code-review focus on tests

<the skill body>
```

No magic. The skill is a prompt the user has aliased to a short trigger.

## Built-in skills

`vibe/core/skills/builtins/` ships a few skills, the most notable being `vibe` itself — which documents Vibe's CLI flags, config options, built-in agents, file discovery, and commands. The model can `/vibe` to get a detailed reference of the harness it's running on, useful when the user asks "how do I do X in Vibe".

This is a clever bootstrap: the agent's own documentation is a skill.

## The skill tool

`vibe/core/tools/builtins/skill.py` (paraphrased):

```python
class SkillArgs(BaseModel):
    name: str

class Skill(BaseTool[SkillArgs, SkillResult, ...]):
    description = "Load a skill's instructions into the conversation."

    async def run(self, args, ctx):
        info = ctx.skill_manager.get_skill(args.name)
        if info is None:
            raise ToolError(f"No skill named '{args.name}'")
        yield SkillResult(content=info.prompt)
```

That's it. The tool's job is to fetch the body. The body becomes a tool result. The model has the skill loaded.

This is why "skills" don't need a new code path — they reuse the existing tool-result message pipe.

## Compared to MCP tools

| | Skills | MCP tools |
|---|---|---|
| Container | Markdown file | External server |
| What it provides | A prompt | A callable function |
| Trigger | Slash command, model decision | Model decision |
| State | Stateless | Server can be stateful |
| Use case | Workflow recipe, prompt pattern | New capability |

Skills are *prompts*; MCP tools are *capabilities*. A skill says "here's how to do X using the tools you already have"; an MCP tool says "here's a new thing you can do."

The two compose: a skill body might tell the model to use a specific MCP tool in a specific sequence.

## When to write a skill vs. a tool

Write a skill if the action is achievable with existing tools but requires a specific sequence or persona. "Code review", "write release notes", "audit for security issues" — all skills.

Write a tool if you need a new capability the model doesn't have. "Query our internal Slack", "look up a JIRA ticket", "render a Mermaid diagram" — all tools (or MCP tools).

If a workflow uses 5 specific tool calls in a fixed order, the skill version is shorter, more maintainable, and the model can adapt the sequence based on context. The tool version is rigid but predictable.

## Try it: write a skill

Create `~/.vibe/skills/diff-summary/SKILL.md`:

```markdown
---
name: diff-summary
description: Summarize the current git diff in bullet points
---

Run `git diff` and produce a 3-5 bullet summary of the changes. Group by file.
End with the total number of files changed and lines added/removed.

Do not edit any files. Do not commit. Output only the summary.
```

Restart Vibe. Type `/diff-summary`. The skill body becomes your prompt; the model runs `git diff` and reports back. You've taught the agent a recipe in 5 lines.

## Skill `allowed-tools` (experimental)

The frontmatter `allowed-tools: read_file grep bash` field exists as an experimental signal that the skill expects only these tools to be available. Vibe doesn't currently enforce it strictly, but it's there for future scoping: a skill could run with a restricted toolset, similar to subagents but lighter weight.

## Skill discovery filters

Same as tools: `config.enabled_skills` / `config.disabled_skills` are glob lists. Useful for environments where the user wants to disable noisy or sensitive skills without removing the files.

---

Next: [`hooks.md`](hooks.md) — the post-turn extension point that runs external programs.
