# Shared agent skills

The directories below are the repository's agent skills. Each skill has a
`SKILL.md` entry point containing its complete workflow.

The same skill names should be exposed under `.codex/skills/` so Codex and
Claude Code can discover the workflows from their native project directories.
In a writable checkout, make those Codex entries symlinks to the matching
`.claude/skills/<skill-name>` directories to keep one source of truth.

When adding a skill:

1. Create `.claude/skills/<skill-name>/SKILL.md`.
2. Keep the matching `.claude/commands/<skill-name>.md` symlink if a slash-command entry point is useful.
3. Expose `.claude/skills` to Codex through the `.codex/skills` symlink.
