---
name: party
description: "GitHub Projects kanban board agent. On first run, sets up a GitHub Project with Backlog → Ready → In Progress → In Review → Done + Blocked columns and saves your preferences for build skill, merge strategy, and refinement mode. On subsequent runs (via /loop or GitHub Actions), scans for new issues, handles refinement via issue comments, spawns parallel build subagents, tracks PRs, and updates project status automatically. Your kanban board builds itself."
argument-hint: "[--setup] [--issue <number>] [--status]"
---

# Party

You are a continuous development agent. Issues move through a GitHub Projects kanban board as you refine requirements, build features, and merge code. The user watches progress through the board; you do the work on the other side of the screen.

**Args:** {{args}}


## Phase 1: Mode Detection

```bash
cat .party.config.json 2>/dev/null && echo "HAS_CONFIG" || echo "NO_CONFIG"
gh auth status 2>&1 | head -3
git remote get-url origin 2>/dev/null || echo "NO_REMOTE"
echo "GITHUB_ACTIONS=${GITHUB_ACTIONS:-false}"
```

Decide mode:

| Condition | Mode |
|-----------|------|
| `--setup` in args OR no `.party.config.json` | **Setup** → Phase 2 |
| `--issue <N>` in args OR `GITHUB_ACTIONS=true` | **Targeted** → Phase 7 (single issue) |
| `--status` in args | **Status** → print board snapshot, exit |
| Config exists, no special flags | **Tick** → Phase 7 (full scan) |

**Pre-flight:** If `gh auth status` fails: stop and ask user to run `gh auth login`. If no GitHub remote: ask which repo to manage. Record `OWNER` and `REPO` from the remote URL (parse `github.com/OWNER/REPO`).


## Phase 2: Setup Interview

*Runs once. Use AskUserQuestion for each question below — one call per question, in order. Do not present these as markdown text or checkbox lists. Wait for each answer before proceeding to the next question. Once all six answers are collected, continue to Phase 3.*

**Setting up party.md for `<owner/repo>`**

**Question 1 — How should the board agent run?**

Call AskUserQuestion with:
- question: "How should the board agent run?"
- options:
  1. "`/loop 5m /party` — Local terminal, polls every 5 minutes *(Recommended to start)*"
  2. "GitHub Actions — Triggers on issue open/label, runs in the cloud 24/7. Requires `ANTHROPIC_API_KEY` secret."
  3. "Both — Local for active sessions; Actions as fallback when terminal is closed."
- multiSelect: false

**Question 2 — Which skill should build features?**

Call AskUserQuestion with:
- question: "Which skill should build features?"
- options:
  1. "`/ship` — Full pipeline: interview → explore → plan → implement → verify → edge cases → E2E → simplify → security"
  2. "`/ship-fast` — Streamlined: explore → plan → implement → verify *(Recommended)*"
  3. "Basic build — Explore → implement → commit. No quality gates."
- multiSelect: false

**Question 3 — How should built code be merged?**

Call AskUserQuestion with:
- question: "How should built code be merged?"
- options:
  1. "Branch + PR, you review — Agent opens a PR on `party/<issue-number>-<slug>`. You review and merge. *(Recommended)*"
  2. "Branch + PR, auto-merge — Agent opens and immediately merges the PR."
  3. "Direct commit to main — No PR. Fastest, riskiest."
- multiSelect: false

**Question 4 — Refinement phase (clarifying questions before building)**

Call AskUserQuestion with:
- question: "When should the agent ask clarifying questions before building?"
- options:
  1. "Agent decides — Asks if the issue is vague (no acceptance criteria, body <200 chars) *(Recommended)*"
  2. "Always — Every issue gets a clarifying comment"
  3. "Skip — Build from the issue as written"
- multiSelect: false

**Question 5 — Run vibe.md setup first?**

Call AskUserQuestion with:
- question: "Want to run vibe.md setup first?"
- options:
  1. "Yes, run vibe.md setup"
  2. "No, skip for now"
- multiSelect: false

**Question 6 — Use spec.md to plan issues before building?**

Call AskUserQuestion with:
- question: "Want to use spec.md to break big tasks into atomic issues automatically? Run `/spec <task>` and it creates a structured GitHub epic + sub-issues that party.md can pick up."
- options:
  1. "Yes, install spec.md so I can /spec tasks before dropping them in the board *(Recommended for large features)*"
  2. "No, I'll write issues manually"
- multiSelect: false

If "Yes": install spec.md:
```bash
npx --yes skills add amajorai/spec.md -y && echo "SPEC_INSTALLED" || echo "SPEC_INSTALL_FAILED"
```
Detect environment and handle reload the same way as Phase 5 (vibe.md install). Record `SPEC_AVAILABLE=true` in config if installed.

Once all six questions are answered, continue to Phase 3.


## Phase 3: GitHub Project Setup

*Setup mode only.*

### 3a. Detect or create the project

```bash
gh project list --owner $OWNER --format json --limit 50
```

If **"party.md"** exists: use it, record `number` and `id`. Skip to 3c.

If not:
```bash
gh project create --owner $OWNER --title "party.md" --format json
```

Record the `number` and `id`.

### 3b. Configure columns

```bash
gh project field-list $PROJECT_NUM --owner $OWNER --format json
```

Find the `Status` single-select field. We need six columns in order: **Backlog, Ready, In Progress, In Review, Done, Blocked**.

For GraphQL mutations to rename existing columns or add missing ones, see [references/github-project-setup.md](references/github-project-setup.md).

After all mutations, run `gh project field-list` again to collect final option IDs.

### 3c. Ensure labels exist

```bash
gh label create "party" --color "0075ca" --description "Managed by party.md" --force
gh label create "awaiting-clarification" --color "e4e669" --description "Agent asked a question; waiting for reply" --force
gh label create "building" --color "0052cc" --description "Agent is building this issue" --force
gh label create "blocked" --color "d73a4a" --description "Issue is blocked" --force
```

### 3d. Record all IDs

Collect project ID, status field ID, and all six option IDs into a config object (written in Phase 6).


## Phase 4: GitHub Actions Setup

*Only if GitHub Actions mode was chosen in Phase 2.*

Create `.github/workflows/party.yml` — see [references/github-actions-workflow.md](references/github-actions-workflow.md) for the full YAML and setup instructions.


## Phase 5: vibe.md Setup

*Only if the user chose "Yes" in Phase 2.*

```bash
npx --yes skills list 2>/dev/null | grep -qE '^vibe$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

If not installed:

```bash
npx --yes skills add amajorai/vibe.md -y
```

Then detect the running environment:

```bash
echo "CODEX=${CODEX:-false}"
echo "CODEX_SANDBOX=${CODEX_SANDBOX:-}"
```

- **Already installed or Codex mode** (`CODEX=true` or `CODEX_SANDBOX` is set): invoke `Skill({ skill: "vibe" })` immediately.
- **Claude Code mode (newly installed)**: tell the user: **"I've installed the `vibe` skill. Please run `/reload-plugins` in this session so it becomes available, then run `/vibe`."** Do not invoke it yourself. Continue to Phase 6.


## Phase 6: Save Config and Finish Setup

Write `.party.config.json` — see [references/config-template.md](references/config-template.md) for the full schema.

```bash
# Ensure screenshot directory is tracked (not gitignored)
grep -q ".party/screenshots" .gitignore 2>/dev/null && \
  sed -i 's|\.party/screenshots.*||g' .gitignore || true

git add .party.config.json
git add .github/workflows/party.yml 2>/dev/null || true
git commit -m "chore: add party.md config"
git push
```

Print a setup summary:

```
party.md is ready 🌈

  Project:    https://github.com/orgs/<owner>/projects/<number>
  Build:      /<build_skill>
  Merge:      <merge_strategy>
  Refinement: <refinement_mode>
  Run mode:   <run_mode>

To start the board loop:
  /loop 5m /party

Got a big feature to build? Plan it first:
  /spec <what you want to build>
  spec.md breaks it into atomic GitHub issues → party.md ships them automatically.
```


## Phase 7: Tick Mode

*Runs on every `/party` call after setup. Also handles `--issue <N>` (GitHub Actions) and `--status`.*

### 7a. Load config

```bash
cat .party.config.json
```

If missing: stop and tell the user to run `/party --setup`. Extract: `owner`, `projectNumber`, `projectId`, `fields.status.*`, `settings.*`

### 7b. Sync untracked issues

```bash
gh issue list --state open --label "party" --json number,url,title --limit 200
gh project item-list $PROJECT_NUM --owner $OWNER --format json --limit 200
```

For each labeled issue not in the project, add it as Backlog:

```bash
ITEM_ID=$(gh project item-add $PROJECT_NUM --owner $OWNER --url $ISSUE_URL --format json | jq -r '.id')
gh project item-edit \
  --id "$ITEM_ID" \
  --project-id "$PROJECT_ID" \
  --field-id "$STATUS_FIELD_ID" \
  --single-select-option-id "$BACKLOG_OPTION_ID"
```

If `--issue <N>` targeted mode: only process that one issue.

### 7c. Handle Backlog items

*Skip entirely if `--status` was passed — print snapshot in 7g and exit.*

For each Backlog item:

**Skip** if labeled `building` (build already in flight).

**If labeled `awaiting-clarification`:** fetch comments and find the last `**[party.md]**` comment. If a non-agent reply exists after that timestamp: append it to the issue body as "## Clarification", remove the label, move to Ready. If no reply: skip.

**No `awaiting-clarification`:** apply `settings.refinementMode`:
- `skip` → move to Ready immediately
- `always` → post refinement comment, add `awaiting-clarification` label
- `agent-decides` → fetch issue; if body >200 chars AND has acceptance criteria (`- [ ]` or "## Acceptance Criteria") AND mentions a specific file/component: move to Ready. Otherwise: post refinement comment.

**Large/complex issue detection (runs before refinementMode check):** If body >1000 chars OR contains phrases like "redesign", "refactor entire", "new system", "rewrite", "from scratch" AND has no acceptance criteria checklist (`- [ ]` or "## Acceptance Criteria"): post a comment suggesting spec.md and skip the refinementMode branch entirely for this issue:

```
**[party.md]** This looks like a large task. Consider running `/spec` on it first to break it into atomic sub-issues — party.md can then pick each one up automatically.

  npx --yes skills add amajorai/spec.md -y
  /spec <summary of this issue>

Once the sub-issues are created and labeled `party`, I'll pick them up on the next tick.
```

Then add `awaiting-clarification` and skip building until the user replies or creates sub-issues. Do not run the refinementMode branch on this issue.

**Refinement comment template:**
```
**[party.md]** Before I start building, I want to make sure I get this right:

1. What's the expected outcome? What should work or look different after this is done?
2. Are there specific files, components, or behaviours I should focus on?
3. Any edge cases, constraints, or things I should NOT change?

Reply to this comment and I'll update the issue and start building.
```

**Move to Ready:**
```bash
gh project item-edit \
  --id "$ITEM_ID" \
  --project-id "$PROJECT_ID" \
  --field-id "$STATUS_FIELD_ID" \
  --single-select-option-id "$READY_OPTION_ID"
```

### 7d. Build Ready items

*Skip entirely if `--status` was passed.*

Collect all Ready items. For each: move to In Progress, add `building` label, derive a branch slug (lowercase, spaces/special chars → `-`, max 40 chars), add to batch.

Spawn **parallel build subagents** — one per issue — using the Agent tool. For the full prompt template, see [references/build-agent-prompt.md](references/build-agent-prompt.md). Wait for all to complete before continuing.

Each build agent takes **before/after Playwright screenshots** of the running dev server (if one is detected) and posts them as a side-by-side table in the issue completion comment. Screenshots are committed to the feature branch and render inline in GitHub via raw.githubusercontent.com. Non-UI changes (backend, config) will have no dev server and will skip screenshots automatically.

### 7e. Check In Progress items

*Skip entirely if `--status` was passed.*

```bash
gh issue view $NUM --json comments -q '.comments[].body' | grep "\[party\.md\].*Built"
gh pr list --search "closes:#$NUM" --json number,state,mergedAt --limit 5
```

- **PR merged** → move to Done, close issue
- **PR open** → move to In Review
- **No PR, no `building` label, last updated >60 min** → post stale comment, move back to Ready

### 7f. Check In Review items

*Skip entirely if `--status` was passed.*

```bash
gh pr list --search "closes:#$NUM" --json number,state,mergedAt,reviewDecision --limit 5
```

- **PR merged** → move to Done, close issue
- **`CHANGES_REQUESTED`** → comment + move back to In Progress
- **`branch-pr-auto` and PR open** → auto-approve and merge:
  ```bash
  gh pr review $PR_NUM --approve --body "**[party.md]** Auto-approved"
  gh pr merge $PR_NUM --squash --auto
  ```
- **Open, awaiting review** → no action

### 7g. Print board snapshot

```
party.md — <ISO timestamp>
────────────────────────────
Backlog      <N>
Ready        <N>
In Progress  <N>  (building: <N>)
In Review    <N>
Done         <N>  (all time)
Blocked      <N>
────────────────────────────
This tick:
  → <each action taken, e.g.:>
  → Moved #12 "add dark mode" to Ready (clear requirements)
  → Posted refinement comment on #14 "fix the thing"
  → Spawned build for #9 "user auth" (ship-fast)
  → Moved #7 "search" to In Review (PR #23 open)
  → Moved #5 "onboarding" to Done (PR #19 merged)

Next check in ~5 min  (/loop 5m /party)
```

If `--status` was passed: print snapshot and exit without taking any actions.
