# PR Review Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code subagent that reviews all open PRs in `ai-os-skills` every hour against structure conformance and code quality checklists, writing reports to `~/ai-os-skills/pr-reviews/` and printing them to terminal.

**Architecture:** A single agent file (`~/.claude/agents/pr-reviewer.md`) contains all review logic as LLM instructions. It uses `gh` CLI to list PRs and fetch diffs, evaluates each diff against two checklists, and writes + prints a markdown report. An hourly cron trigger (via the `schedule` skill) invokes the agent automatically.

**Tech Stack:** Claude Code subagents, `gh` CLI (GitHub), `schedule` skill (cron triggers), markdown reports.

---

### Task 1: Create the pr-reviews directory placeholder

**Files:**
- Create: `~/ai-os-skills/pr-reviews/.gitkeep`

- [ ] **Step 1: Create the directory and placeholder file**

```bash
mkdir -p ~/ai-os-skills/pr-reviews
touch ~/ai-os-skills/pr-reviews/.gitkeep
```

- [ ] **Step 2: Verify it exists**

```bash
ls ~/ai-os-skills/pr-reviews/
```

Expected output:
```
.gitkeep
```

- [ ] **Step 3: Commit**

```bash
cd ~/ai-os-skills
git add pr-reviews/.gitkeep
git commit -m "chore: add pr-reviews directory for automated review reports"
```

---

### Task 2: Create the pr-reviewer agent file

**Files:**
- Create: `~/.claude/agents/pr-reviewer.md`

- [ ] **Step 1: Write the agent file**

Create `~/.claude/agents/pr-reviewer.md` with this exact content:

```markdown
---
name: "pr-reviewer"
description: "Use this agent to review all open pull requests in the ai-os-skills repository. It checks each PR's diff for skill structure conformance and code quality issues, writes a markdown report to ~/ai-os-skills/pr-reviews/, and prints the report to terminal."
model: sonnet
---

You are an automated PR reviewer for the `ai-os-skills` repository. Your job is to review all currently open pull requests and produce structured review reports.

## Setup

Before running reviews, ensure the reports directory exists:

```bash
mkdir -p ~/ai-os-skills/pr-reviews
```

## Step 1: List open PRs

Run this from `~/ai-os-skills`:

```bash
cd ~/ai-os-skills && gh pr list --state open --json number,title,headRefName
```

If `gh` returns an auth error, stop immediately and print:
> "Error: gh CLI is not authenticated. Run `gh auth login` and retry."

If no PRs are open, print:
> "No open PRs found. Nothing to review."
Then stop.

## Step 2: For each open PR, fetch the diff

```bash
cd ~/ai-os-skills && gh pr diff <number>
```

## Step 3: Review the diff

Evaluate the diff against these two checklists. Only check items that are relevant to the changed files.

### Conformance Checklist (for any changed skill directory)

- [ ] `SKILL.md` exists at the skill root with valid YAML frontmatter
- [ ] Frontmatter contains both `name` and `description` fields
- [ ] `name` value matches the skill directory name (kebab-case, exact match)
- [ ] `description` is a single sentence ending with a period
- [ ] No unexpected files at the skill root — only `SKILL.md`, `scripts/`, `references/`, `package.json` are allowed
- [ ] If `scripts/` contains runnable scripts, at least one of `--dry-run` or `--preview` is supported
- [ ] If `package.json` is present, it is scoped to that skill only (no reference to a shared `node_modules`)

### Code Quality Checklist (for files in `scripts/`)

- [ ] No hardcoded secrets, API keys, or credentials (no string literals that look like keys/tokens)
- [ ] Shared lib paths use `require('../../../lib/...')` pattern (relative to workspace root, three levels up)
- [ ] No broken `require`/`import` paths for shared libs (e.g., `lib/persona.js`, `lib/...`)
- [ ] No obvious logic errors (e.g., using a variable before it is defined, wrong loop condition)
- [ ] No unhandled promise rejections (async functions should have try/catch or `.catch()` at top level)
- [ ] No missing error handling at system boundaries: all external API calls and file reads are wrapped
- [ ] Script does not use `readline`, `process.stdin`, or any interactive terminal assumption

### Edge cases

- If the PR only modifies `README.md`, `CLAUDE.md`, or other non-skill files: skip conformance checks, run code quality only on any changed script files, note the skip in the report.
- If the PR adds a brand new skill directory: apply full conformance checklist.
- If the PR deletes files: note the deletions but do not flag them as issues unless they break the conformance structure.

## Step 4: Write and print the report

**Report filename:** `~/ai-os-skills/pr-reviews/PR-<number>.md`
(Overwrite if it already exists — always reflect the latest diff.)

**Report format:**

```
# PR Review: #<number> — <title>
Date: <YYYY-MM-DD>
Branch: <headRefName>
Status: PASS | NEEDS ATTENTION

## Conformance Issues
- <list each issue found, or "None">

## Code Quality Issues
- <list each issue found, or "None">

## Summary
<1–2 sentences summarizing the overall assessment and what the reviewer should do next, if anything>
```

Set `Status` to `PASS` when both sections have no issues. Set to `NEEDS ATTENTION` otherwise.

After writing the file, print its full contents to the terminal.

## Step 5: Repeat for all open PRs

Process every PR returned in Step 1. After all reports are written, print:
> "Review complete. <N> PR(s) reviewed. Reports saved to ~/ai-os-skills/pr-reviews/"
```

- [ ] **Step 2: Verify the file was created and frontmatter is valid**

```bash
head -10 ~/.claude/agents/pr-reviewer.md
```

Expected output starts with:
```
---
name: "pr-reviewer"
description: "Use this agent to review all open pull requests...
```

---

### Task 3: Manual smoke test

**Files:**
- No file changes — this is a verification step only

- [ ] **Step 1: Check if there are any open PRs to test against**

```bash
cd ~/ai-os-skills && gh pr list --state open
```

If there are open PRs, continue to Step 2. If there are none, skip to Step 3.

- [ ] **Step 2: Run the agent manually on a real PR**

In Claude Code, run:

```
/agents
```

Select `pr-reviewer` and invoke it. Confirm it:
1. Lists the open PRs
2. Fetches a diff
3. Writes a report to `~/ai-os-skills/pr-reviews/PR-<number>.md`
4. Prints the report to terminal

Check the report was written:

```bash
ls ~/ai-os-skills/pr-reviews/
cat ~/ai-os-skills/pr-reviews/PR-*.md
```

- [ ] **Step 3: If no open PRs — create a test PR to verify end-to-end**

```bash
cd ~/ai-os-skills
git checkout -b test/pr-reviewer-smoke-test
# Make a trivial change to trigger a reviewable diff
echo "" >> README.md
git add README.md
git commit -m "test: smoke test for pr-reviewer agent"
gh pr create --title "test: pr-reviewer smoke test" --body "Temporary PR to verify pr-reviewer agent works end-to-end. Close after testing."
```

Then run the agent (see Step 2). After verifying, close the PR and delete the branch:

```bash
gh pr close <number>
git checkout main
git branch -d test/pr-reviewer-smoke-test
```

---

### Task 4: Set up the hourly schedule trigger

**Files:**
- No files — the schedule is created via the `schedule` skill

- [ ] **Step 1: Create the hourly trigger**

In Claude Code, invoke the `schedule` skill:

```
/schedule
```

When prompted, configure the trigger with these values:
- **Prompt / task:** `Review all open PRs in ai-os-skills using the pr-reviewer agent`
- **Agent:** `pr-reviewer`
- **Schedule:** Every hour (`0 * * * *`)
- **Working directory:** `~/ai-os-skills`

- [ ] **Step 2: Verify the trigger was created**

In Claude Code, run:

```
/schedule list
```

Confirm a trigger appears with an hourly schedule pointing to the `pr-reviewer` agent.

- [ ] **Step 3: Commit the pr-reviews/.gitkeep if any test reports were generated**

```bash
cd ~/ai-os-skills
# Remove any test reports before committing
rm -f pr-reviews/PR-*.md
git status
```

If `pr-reviews/.gitkeep` is already committed from Task 1, no further commit is needed.

---

## Verification Checklist

After completing all tasks, confirm:

- [ ] `~/.claude/agents/pr-reviewer.md` exists and has valid frontmatter
- [ ] `~/ai-os-skills/pr-reviews/` directory exists
- [ ] Agent ran successfully and produced at least one report (or "No open PRs found" output)
- [ ] Hourly schedule trigger is active (visible in `/schedule list`)
- [ ] Reports are written to `~/ai-os-skills/pr-reviews/PR-<number>.md` with correct format
