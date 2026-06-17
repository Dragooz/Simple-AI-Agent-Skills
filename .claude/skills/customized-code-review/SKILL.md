---
name: customized-code-review
description: Review a GitHub PR against its linked tracker task. Takes a GitHub PR URL, reads the diff and commits, pulls the linked task description + comments (read-only) for intent, then reviews for correctness bugs, simplification/reuse, and whether the task goal is met. Convention and security checks are secondary. Use when the user gives a GitHub PR link and asks for a review.
---

# Code Review

Review a GitHub PR, grounded in the linked tracker task's intent.

## Priorities (EDIT THIS)

> Swap these few lines per team — everything below stays as-is.
>
> - **Top priority:** the one thing a bug here would damage most (e.g. affecting core feature, data integrity, user safety). A bug in it is always MUST-FIX, never advisory.
> - **Tracker:** which task tracker holds the linked task (Jira, ClickUp, Linear, GitHub Issues…), how its links look in a PR body, and its read-only commands.
> - **High-scrutiny paths:** files/areas that get the harshest review. E.g. List down the core file that serve the critical business logic and require most attention
> - **Conventions:** project style rules (optional, never blocking). E.g. variable name preference (short or long, etc.)

## Input

A **GitHub PR URL**, e.g. `https://github.com/<org>/<repo>/pull/123`.

**Always require an explicit PR URL.** Never assume the repo from the working dir. If none given, ask and stop until you get it. Repo is read from the URL — `gh pr view <URL>` works for any repo.

## Command Reference

Single source of truth for commands. Steps refer to these by name. Substitute `<org>`/`<repo>`/`<num>`/`<URL>` from the PR URL; get `<head_sha>` from `pr-head-sha`.

**Universal rules:**

- Run `gh pr`/`gh pr diff` **bare** — never append `2>&1` or any redirect (breaks allowlist prefix-matching, forces a prompt).
- Quote any `gh api` URL with `?ref=` — the shell globs `?`.
- All writes pass `-X POST` explicitly.

### Reads (GitHub)

`pr-view` — title/desc/commits. Scan `body` for the tracker link.

```bash
gh pr view <URL> --json title,body,commits,url
```

`pr-diff` — the net diff to review.

```bash
gh pr diff <URL>
```

`pr-head-sha` — head SHA, for file fetches + inline comments.

```bash
gh pr view <URL> --json commits --jq '.commits[-1].oid'
```

`file-at-head` — read ONE file at PR head. Verifying negative findings only — ask user first.

```bash
gh api "repos/<org>/<repo>/contents/<path>?ref=<head_sha>" --jq '.content' | base64 -d
```

`pending-reviews` — open draft review check before posting inline (avoids HTTP 422).

```bash
gh api repos/<org>/<repo>/pulls/<num>/reviews --jq '.[] | select(.state=="PENDING") | .id'
```

### Reads (tracker via MCP — READ ONLY, never write)

The linked task lives in a project-management tracker exposed as an **MCP server** (ClickUp, Linear, Jira/Atlassian, Notion, Asana, monday.com, etc.). Use its MCP tools — never scrape the tracker's web UI.

**Discover the tracker MCP** (do this once, at step 2):

1. Match the tracker from the PR-body link domain (`linear.app` → Linear, `atlassian.net` → Jira, `clickup.com` → ClickUp, …).
2. Find that tracker's connected MCP tools via `ToolSearch` — e.g. `ToolSearch("linear issue")`, `ToolSearch("clickup task comments")`, `ToolSearch("jira get issue")`. This loads the tool schemas so you can call them.
3. If the MCP needs auth, the read call returns an `authenticate` tool — surface it to the user and stop until connected. Do not proceed by guessing.

**Read only these, never anything that writes** (no create/update/comment tools):

- **Task description** (the _what_) — the get-task / get-issue tool.
- **Full comment thread** (the _contested_) — the get-comments tool.

If no tracker MCP is connected, say so and offer to proceed on the diff alone (see step 2).

### Writes (GitHub — confirm with user FIRST, every time)

`comment-inline` — anchored to a line (RIGHT side of diff, line number at head SHA).

```bash
gh api -X POST repos/<org>/<repo>/pulls/<num>/comments \
  -f body="<finding>" -f commit_id="<head_sha>" \
  -f path="<file>" -F line=<line> -f side=RIGHT --jq '.html_url'
```

`comment-general` — unanchored finding ("endpoint missing", "scope noise").

```bash
gh api -X POST repos/<org>/<repo>/issues/<num>/comments -f body="<finding>" --jq '.html_url'
```

## Steps

### 1. Read the PR

Run `pr-view` and `pr-diff`. Scan `pr-view` body for the tracker link. Review the **net diff**, not each commit; commit messages are context only.

**Review purely from the GitHub PR diff.** Do NOT `git checkout`, switch branches, or read local files — the local branch may differ. Only if extra context is genuinely required (diff calls a function you can't judge from the diff, and it changes a finding): STOP, ask permission, name the file + why, then `file-at-head` read-only. Never pull extra files silently.

### 2. Read the task (READ ONLY)

Get the task id from the PR body link. Connect to the tracker via its **MCP server** — see _Reads (tracker via MCP)_ above for discovery (`ToolSearch` by tracker name) and auth. Then read the **description** (intent) and the **full comment thread** — comments are where the goal-check lives (unresolved tension, contract decisions, deferred scope, prior flags). Skip only true noise (status pings, "done", emoji). **Read-only: never call a create/update/comment MCP tool.** If no link: say so, ask for the id or proceed without. If the tracker MCP is unavailable or unauthenticated: say so, surface the `authenticate` tool, and offer to proceed on the diff alone.

### 3. Review

High-signal only — confident, actionable. No nitpicks.

**Before reviewing — two checks:**

- **Formatting noise.** If the diff is mostly reformatting (linter wrapping, import reorder) unrelated to the task, say so loudly and review only the substantive slice. Reformatting churn on sensitive files is itself worth flagging.
- **Verify negative findings at head SHA.** A finding claiming something is _missing/absent/unhandled_ is dangerous — the diff only shows changed lines. If the thing could live outside the diff, `file-at-head` to confirm before flagging. (Positive findings — a bug on a line in the diff — need no check.)

**What to review, by priority:**

1. **Correctness bugs** (PRIMARY) — logic errors, edge cases, null handling, off-by-one, race, wrong query, broken control flow. Real bugs, not style. High-scrutiny paths → MUST-FIX.
2. **Simplification / reuse** (PRIMARY) — over-engineering, dead code, repeated logic to DRY, simpler equivalent, unused imports/vars introduced.
3. **Goal achieved?** (PRIMARY) — does the diff do what the task asked? Missing scope, misread requirement, edge left unhandled. If task description and code don't align, warn loudly and tell the user to update the task (we never write the tracker); state which side drifted.
4. **Security** (secondary) — injection, missing authz, leaked secrets/tokens, sensitive-data handling. Any hardcoded token → flag for immediate destruction.
5. **Convention** (OPTIONAL — never blocking) — one short section, each item "optional / author's call".

Each finding: `file:line` + what's wrong + concrete fix. Skip if not confident.

**Personal lens** (tier by severity: intolerable → MUST-FIX; else → advisory "Style preferences" block):

- **Simplicity** — minimum code that solves it. Flag over-engineering, speculative abstraction.
- **Goal-oriented** — every changed line traces to the goal. Flag scope creep, drive-by refactors.
- **Readable naming** — self-explaining names. Prefer readable over short; flag length only when it's redundant bloat.
- **Commenting** — flag both missing "why" on non-obvious logic AND noisy obvious over-commenting.
- **Concise** — no bloat in code or in the change.

### 4. Report

Markdown in chat, grouped by category, severity-ordered. Lead with goal-met verdict, then primary findings (incl. MUST-FIX lens violations), then advisory **Style preferences**, then optional **Convention**. Trailing blocks non-blocking. Zero findings → say so plainly. Don't manufacture findings.

### 5. Offer PR comments

Ask: **post findings as GitHub PR comments?** Never auto-post.

- On yes: first run `pending-reviews`. If a draft review exists, STOP and ask the user to submit/discard it (inline comments 422 otherwise); don't auto-submit their draft. Then post — `comment-inline` for line-anchored, `comment-general` otherwise. Get `<head_sha>` via `pr-head-sha`.

## Notes

- Read-only on the tracker, always.
- Never post to GitHub without explicit confirmation.
- Token/secret in diff → flag for immediate destruction.

## Setup (optional, one-time)

Reduce prompts: add to `.claude/settings.local.json` `permissions.allow`. Use the space-prefix form (`Bash(cmd *)`); the `:*` colon form won't match multi-token `gh pr diff`.

```
"Bash(gh pr view *)",
"Bash(gh pr diff *)"
```

Do NOT allowlist `gh api` — used for reads and writes, so keep it prompted to preserve the confirm-before-post gate.

**Tracker MCP:** connect your tracker's MCP server (ClickUp, Linear, Jira/Atlassian, Notion, …) in Claude Code so its read tools are available — the skill discovers them at review time via `ToolSearch`. Allowlist only the tracker's **read** MCP tools; leave write tools prompted (the skill never writes to the tracker anyway).
