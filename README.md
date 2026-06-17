# simple_skills

A collection of useful, simple [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills) I've discovered and customized.

## Skills

### customized-code-review

Reviews a GitHub PR against its linked tracker task (Jira, ClickUp, Linear, GitHub Issues, etc.).

Given a PR URL, it:

1. Reads the PR diff and commits.
2. Pulls the linked task's description + comments (read-only) to understand intent.
3. Reviews for **correctness bugs**, **simplification/reuse**, and **whether the task goal is met**. Convention and security checks are secondary.
4. Reports findings in chat, then optionally posts them as inline PR comments (always asks first — never auto-posts).

**Customize it per team** by editing the `## Priorities` block at the top of the skill (top priority, which tracker, high-scrutiny paths, conventions). Everything else stays as-is.

See [`.claude/skills/customized-code-review/SKILL.md`](.claude/skills/customized-code-review/SKILL.md).

## Usage

These are project-level skills under `.claude/skills/`. Claude Code picks them up automatically when working in this repo. Ask Claude to review a PR with its URL, or copy a skill into your own project's `.claude/skills/`.

## Setup (optional, one-time)

Reduce permission prompts: add to `.claude/settings.local.json` under `permissions.allow`. Use the space-prefix form (`Bash(cmd *)`); the `:*` colon form won't match multi-token `gh pr diff`.

```json
"Bash(gh pr view *)",
"Bash(gh pr diff *)"
```

Do NOT allowlist `gh api` — it's used for reads and writes, so keep it prompted to preserve the confirm-before-post gate. Add your tracker's read tools here too.
