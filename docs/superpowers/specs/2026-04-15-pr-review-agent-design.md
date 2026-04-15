# PR Review Agent — Design Spec

**Date:** 2026-04-15
**Status:** Approved

---

## Overview

An automated Claude Code subagent that reviews all open pull requests in the `ai-os-skills` repository every hour. It checks each PR's diff against a skill structure conformance checklist and a code quality checklist, then writes a markdown report to `~/ai-os-skills/pr-reviews/` and prints it to the terminal.

---

## Architecture

### Components

**1. Agent file** — `~/.claude/agents/pr-reviewer.md`

A Claude Code subagent definition. On each invocation it:
1. Runs `gh pr list --state open` from `~/ai-os-skills` to get all open PRs (repo auto-detected from git remote).
2. For each PR, fetches the diff via `gh pr diff <number>` from the same directory.
3. Reviews the diff against both checklists (see Review Checklists).
4. Writes a report to `~/ai-os-skills/pr-reviews/PR-<number>.md`.
5. Prints the report to the terminal.

**2. Scheduled trigger** — created via the `schedule` skill

Runs the `pr-reviewer` agent hourly using Claude Code's remote trigger system. Each run is stateless. Reports are overwritten on each run so they always reflect the latest diff.

### Data Flow

```
[Hourly cron trigger]
    → pr-reviewer agent
        → gh pr list (from ~/ai-os-skills, auto-detected)
        → for each open PR:
            → gh pr diff <number>
            → LLM review (conformance + code quality)
            → write ~/ai-os-skills/pr-reviews/PR-<number>.md
            → print report to terminal
```

---

## Review Checklists

### Skill Structure Conformance

Rules derived from `CLAUDE.md`:

- `SKILL.md` exists at the skill root with valid YAML frontmatter containing `name` and `description` fields.
- `name` value matches the skill directory name (kebab-case).
- `description` is a single sentence ending with a period.
- No unexpected files at the skill root — only `SKILL.md`, `scripts/`, `references/`, and `package.json` are permitted.
- If `scripts/` contains runnable scripts, they support a `--dry-run` or `--preview` flag.
- If a `package.json` is present, dependencies are scoped to that skill (no reliance on a shared `node_modules`).

### Code Quality

Applies to files in `scripts/`:

- No hardcoded secrets, API keys, or credentials.
- Shared lib paths are resolved relative to `../../..` (the Clawdbot workspace root pattern).
- No broken `require`/`import` paths for shared libs (e.g., `lib/persona.js`).
- No obvious logic errors, unhandled promise rejections, or missing error handling at system boundaries (user input, external API calls).
- Scripts do not assume an interactive terminal — they must work headless.

---

## Report Format

Reports are written to `~/ai-os-skills/pr-reviews/PR-<number>.md`.

```markdown
# PR Review: #<N> — <title>
Date: YYYY-MM-DD
Status: PASS | NEEDS ATTENTION

## Conformance Issues
- <issue or "None">

## Code Quality Issues
- <issue or "None">

## Summary
<1–2 sentence summary of overall assessment>
```

`Status` is `PASS` when both sections report no issues; otherwise `NEEDS ATTENTION`.

---

## Error Handling & Edge Cases

| Scenario | Behavior |
|---|---|
| No open PRs | Print "No open PRs found" and exit cleanly. No report written. |
| PR touches no skill files (e.g., only `README.md`) | Skip conformance checks. Run code quality review on changed files only. Note the skip in the report. |
| `gh` not authenticated | Surface the auth error message and exit. Do not retry silently. |
| `pr-reviews/` directory does not exist | Create it on first run before writing the first report. |
| Report already exists for a PR number | Overwrite with the latest review. |

**Hard constraint:** The agent is read-only. It never posts to GitHub, never merges, and never closes PRs.

---

## Scheduling

- Frequency: every hour
- Mechanism: Claude Code remote trigger via the `schedule` skill
- Each run reviews all currently open PRs (not just new ones)

---

## File Locations

| Artifact | Path |
|---|---|
| Agent definition | `~/.claude/agents/pr-reviewer.md` |
| Review reports | `~/ai-os-skills/pr-reviews/PR-<number>.md` |
| This spec | `~/ai-os-skills/docs/superpowers/specs/2026-04-15-pr-review-agent-design.md` |
