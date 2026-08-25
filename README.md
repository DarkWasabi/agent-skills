# Agent Skills

Personal collection of agent skills — packaged instructions for recurring workflows. Each skill is a directory containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and can be installed independently with the [`skills`](https://skills.sh) CLI.

## What's a skill?

A skill is a directory containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and instructions an agent follows when the description matches the current task. Compatible agents discover installed skills from their configured skill directories and load matched skill content on demand.

## Skills in this repo

| Skill | Use when |
|---|---|
| [`clean-code-refactor`](clean-code-refactor/SKILL.md) | Restructuring existing code for readability/testability without changing behavior — "clean up", "refactor", "modernize". Not for new features or bug fixes. |
| [`building-reactables`](building-reactables/SKILL.md) | Building or reviewing state management with the [Reactables](https://github.com/reactables) library (`@reactables/core`, `@reactables/react`, `@reactables/forms`) — reactables, async effects, debounce/cancellation, composing state. |
| [`spec-review`](spec-review/SKILL.md) | Independent codebase-grounded review of a completed design spec for inconsistencies, hidden risks, and unnecessary complexity before implementation planning. |

## Installing with npx

No package from this repo needs to be published to npm. The `skills` CLI discovers each `SKILL.md` directly from GitHub.

List available skills:

```bash
npx skills add DarkWasabi/agent-skills --list
```

Install one skill:

```bash
npx skills add DarkWasabi/agent-skills --skill clean-code-refactor
npx skills add DarkWasabi/agent-skills --skill building-reactables
npx skills add DarkWasabi/agent-skills --skill spec-review
```

Install all skills:

```bash
npx skills add DarkWasabi/agent-skills
```

For a non-interactive global Claude Code install, add `-g -a claude-code -y`:

```bash
npx skills add DarkWasabi/agent-skills --skill spec-review -g -a claude-code -y
```

## Manual install

Copy or symlink a skill directory into your agent's skills path. For Claude Code, that is typically `~/.claude/skills/`.

To clone the full collection for Claude Code:

```bash
git clone https://github.com/DarkWasabi/agent-skills.git ~/.claude/skills/agent-skills
```

## Adding a new skill

1. Create a new directory named after the skill.
2. Add a `SKILL.md` with frontmatter:
   ```yaml
   ---
   name: my-skill
   description: Use when ... (be specific — this is how the agent decides to invoke it)
   ---
   ```
3. Write clear, imperative instructions in the body. Keep it stack-agnostic unless the skill is inherently tied to a specific library/framework.
4. Supporting files (checklists, reviewer personas, reference docs) go in subdirectories alongside `SKILL.md`.
