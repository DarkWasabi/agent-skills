# Agent Skills

Personal collection of [Claude Code](https://claude.com/claude-code) skills — packaged instructions that Claude invokes automatically (or via `/skill-name`) for recurring workflows.

## What's a skill?

A skill is a directory containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and instructions Claude follows when the description matches the current task. Claude Code discovers skills placed under its skills search paths and loads the matched skill's content into context on demand.

## Skills in this repo

| Skill | Use when |
|---|---|
| [`clean-code-refactor`](clean-code-refactor/SKILL.md) | Restructuring existing code for readability/testability without changing behavior — "clean up", "refactor", "modernize". Not for new features or bug fixes. |
| [`building-reactables`](building-reactables/SKILL.md) | Building or reviewing state management with the [Reactables](https://github.com/reactables) library (`@reactables/core`, `@reactables/react`, `@reactables/forms`) — reactables, async effects, debounce/cancellation, composing state. |
| [`spec-review`](spec-review/SKILL.md) | Critical multi-persona review of a design spec (business analyst, system architect, senior QA + CTO synthesis) before planning or implementation. |

## Installing

Copy or symlink a skill directory into your Claude Code skills path (e.g. `~/.claude/skills/`), or clone this whole repo there:

```bash
git clone <this-repo> ~/.claude/skills/agent-skills
```

Claude Code picks up any `SKILL.md` under its configured skill directories automatically.

## Adding a new skill

1. Create a new directory named after the skill.
2. Add a `SKILL.md` with frontmatter:
   ```yaml
   ---
   name: my-skill
   description: Use when ... (be specific — this is how Claude decides to invoke it)
   ---
   ```
3. Write clear, imperative instructions in the body. Keep it stack-agnostic unless the skill is inherently tied to a specific library/framework.
4. Supporting files (checklists, reviewer personas, reference docs) go in subdirectories alongside `SKILL.md`.
