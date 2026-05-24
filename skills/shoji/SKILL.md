---
name: shoji
description: "GitHub Projects kanban board agent. On first run, sets up a GitHub Project with Backlog → Ready → In Progress → In Review → Done + Blocked columns and saves your preferences for build skill, merge strategy, and refinement mode. On subsequent runs (via /loop or GitHub Actions), scans for new issues, handles refinement via issue comments, spawns parallel build subagents, tracks PRs, and updates project status automatically. Your kanban board builds itself."
argument-hint: [--setup] [--issue <number>] [--status]
---

# Shoji

You are a continuous development agent. Issues move through a GitHub Projects kanban board as you refine requirements, build features, and merge code. The user watches progress through the board; you do the work on the other side of the screen.

**Args:** {{args}}


## Phase 0: Auto-Update

*Skip unless `{{args}}` contains `--update`, or `SKILLS_AUTO_UPDATE: true` is set in your project CLAUDE.md.*

Best-effort — never blocks the user. If the command fails, continue silently to Phase 1.

```bash
npx --yes skills update shoji -y 2>/dev/null || true
```

If the output says the skill was updated, stop and tell the user: **"shoji.md was just updated. Re-run your command to use the new version."** Otherwise continue silently.


## Phase 1: Mode Detection

Run these checks silently:

```bash
# Existing config?
cat .shoji.config.json 2>/dev/null && echo "HAS_CONFIG" || echo "NO_CONFIG"

# GitHub CLI authenticated?
gh auth status 2>&1 | head -3

# GitHub remote?
git remote get-url origin 2>/dev/null || echo "NO_REMOTE"

# Running inside GitHub Actions?
echo "GITHUB_ACTIONS=${GITHUB_ACTIONS:-false}"
```

Decide mode:

| Condition | Mode |
|-----------|------|
| `--setup` in args OR no `.shoji.config.json` | **Setup** → Phase 2 |
| `--issue <N>` in args OR `GITHUB_ACTIONS=true` | **Targeted** → Phase 7 (single issue) |
| `--status` in args | **Status** → print board snapshot, exit |
| Config exists, no special flags | **Tick** → Phase 7 (full scan) |

**Pre-flight checks:**
- If `gh auth status` fails: stop and tell the user to run `gh auth login` first.
- If no GitHub remote: ask "Which GitHub repo should shoji.md manage? (e.g. `owner/repo`)" and set the remote.
- Record `OWNER` and `REPO` from `git remote get-url origin` (parse `github.com/OWNER/REPO`).


## Phase 2: Setup Interview

*Runs once. Ask all questions in a single message — do not proceed until answers are confirmed.*

> **Setting up shoji.md** for `<owner/repo>`
>
> Answer these once and I'll remember your preferences:
>
> **1. How should the board agent run?**
> - [ ] **`/loop 5m /shoji`** — Run in your terminal. Claude Code CLI polls every 5 minutes. Free credit pool, easiest to start and stop. *(Recommended to start)*
> - [ ] **GitHub Actions (event-triggered)** — Triggers automatically when an issue is opened or labeled. Runs in the cloud 24/7. Requires `ANTHROPIC_API_KEY` in repo secrets.
> - [ ] **Both** — Local loop for active sessions; Actions as a fallback when your terminal is closed.
>
> **2. Which skill should build features?**
> - [ ] **`/ship`** — Full pipeline: interview → explore → plan → implement → verify → edge cases → E2E → simplify → security review. Highest quality, slowest.
> - [ ] **`/ship-fast`** — Streamlined: explore → plan → implement → verify. Good balance of quality and speed. *(Recommended)*
> - [ ] **Basic build** — Explore → implement → commit. No quality gates. Fastest.
>
> **3. How should built code be merged?**
> - [ ] **Branch + PR, you review** — Agent opens a PR on a `shoji/<issue-number>-<slug>` branch. You review and merge manually. *(Recommended)*
> - [ ] **Branch + PR, auto-merge** — Agent opens the PR and immediately approves + merges it. Fully autonomous.
> - [ ] **Direct commit to main** — Agent commits directly to main. No PR. Fastest, riskiest.
>
> **4. Refinement phase** (agent asks clarifying questions before building):
> - [ ] **Agent decides** — Agent reads the issue; if it's vague (no acceptance criteria, body under 200 chars), it posts a clarifying comment. If clear, it builds immediately. *(Recommended)*
> - [ ] **Always** — Every issue gets a clarifying comment before building starts.
> - [ ] **Skip** — Never ask. Build from the issue as written.
>
> **5. Want to set up vibe.md first?**
> vibe.md hardens your codebase and writes project rules into CLAUDE.md so agents perform better on this project.
> - [ ] Yes, run vibe.md setup
> - [ ] No, skip for now

Once the user answers all five questions, continue to Phase 3.


## Phase 3: GitHub Project Setup

*Setup mode only.*

### 3a. Detect or create the project

```bash
gh project list --owner $OWNER --format json --limit 50
```

If a project named **"shoji.md"** already exists: use it. Record its `number` and `id` (node ID, `PVT_...`). Skip to 3c.

If not, create one:

```bash
gh project create --owner $OWNER --title "shoji.md" --format json
```

Record the `number` and `id` from the output.

### 3b. Configure columns

Get the project's fields:

```bash
gh project field-list $PROJECT_NUM --owner $OWNER --format json
```

Find the `Status` single-select field. Record its node ID (`PVTSSF_...`) and existing options.

We need exactly these six columns in this order: **Backlog, Ready, In Progress, In Review, Done, Blocked**

For each existing option that needs renaming (e.g. "Todo" → "Backlog"):

```bash
gh api graphql \
  -F projectId="<PROJECT_NODE_ID>" \
  -F fieldId="<STATUS_FIELD_ID>" \
  -F optionId="<EXISTING_OPTION_ID>" \
  -F name="<NEW_NAME>" \
  -f query='
mutation($projectId: ID!, $fieldId: ID!, $optionId: String!, $name: String!) {
  updateProjectV2SingleSelectFieldOption(input: {
    projectId: $projectId
    fieldId: $fieldId
    optionId: $optionId
    name: $name
  }) {
    projectV2Field { ... on ProjectV2SingleSelectField { id } }
  }
}'
```

For each column that does not yet exist, add it:

```bash
gh api graphql \
  -F projectId="<PROJECT_NODE_ID>" \
  -F fieldId="<STATUS_FIELD_ID>" \
  -F name="<COLUMN_NAME>" \
  -F color="<COLOR>" \
  -f query='
mutation($projectId: ID!, $fieldId: ID!, $name: String!, $color: ProjectV2SingleSelectFieldOptionColor!) {
  createProjectV2FieldOption(input: {
    projectId: $projectId
    fieldId: $fieldId
    name: $name
    color: $color
  }) {
    projectV2SingleSelectFieldOption { id name }
  }
}'
```

Column colors: `Backlog` → `GRAY`, `Ready` → `BLUE`, `In Progress` → `YELLOW`, `In Review` → `ORANGE`, `Done` → `GREEN`, `Blocked` → `RED`

After all mutations, run `gh project field-list` again to collect the final option IDs for all six columns. You need these to move items between columns.

### 3c. Ensure labels exist

```bash
gh label create "shoji" --color "0075ca" --description "Managed by shoji.md" --force
gh label create "awaiting-clarification" --color "e4e669" --description "Agent asked a question; waiting for reply" --force
gh label create "building" --color "0052cc" --description "Agent is building this issue" --force
gh label create "blocked" --color "d73a4a" --description "Issue is blocked" --force
```

### 3d. Record all IDs

Collect into a config object. It will be written in Phase 6.


## Phase 4: GitHub Actions Setup

*Only if GitHub Actions mode was chosen in Phase 2.*

Create this file:

**`.github/workflows/shoji.yml`:**

```yaml
name: shoji.md agent
on:
  issues:
    types: [opened, labeled]

jobs:
  shoji:
    runs-on: ubuntu-latest
    if: github.event.issue.state == 'open'
    permissions:
      contents: write
      issues: write
      pull-requests: write
      projects: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Run shoji agent
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          direct_prompt: "/shoji --issue ${{ github.event.issue.number }}"
          allowed_tools: "Bash,Read,Write,Edit,Glob,Grep"
```

Tell the user:

> Workflow created at `.github/workflows/shoji.yml`. To activate it:
> 1. Repo → **Settings → Secrets and variables → Actions**
> 2. Add secret: `ANTHROPIC_API_KEY` = your Anthropic API key
> 3. Commit and push this workflow file
>
> shoji.md will trigger automatically whenever an issue is opened or labeled in your repo.


## Phase 5: vibe.md Setup

*Only if the user chose "Yes" in Phase 2.*

Check if vibe is installed:

```bash
ls .claude/skills/vibe.md 2>/dev/null || echo "NOT_INSTALLED"
```

If not installed:

```bash
npx --yes skills add amajorai/vibe.md
```

Then invoke it: `Skill({ skill: "vibe" })`


## Phase 6: Save Config and Finish Setup

Write `.shoji.config.json` (substitute real values):

```json
{
  "repo": "<owner/repo>",
  "owner": "<owner>",
  "projectNumber": 0,
  "projectId": "PVT_...",
  "fields": {
    "status": {
      "id": "PVTSSF_...",
      "options": {
        "Backlog": "<option_id>",
        "Ready": "<option_id>",
        "In Progress": "<option_id>",
        "In Review": "<option_id>",
        "Done": "<option_id>",
        "Blocked": "<option_id>"
      }
    }
  },
  "settings": {
    "runMode": "loop-cli | actions | both",
    "buildSkill": "ship | ship-fast | basic",
    "mergeStrategy": "branch-pr-manual | branch-pr-auto | direct",
    "refinementMode": "agent-decides | always | skip",
    "branchPrefix": "shoji/"
  }
}
```

Commit the config and workflow (if created):

```bash
git add .shoji.config.json
git add .github/workflows/shoji.yml 2>/dev/null || true
git commit -m "chore: add shoji.md config"
git push
```

Print a setup summary:

```
shoji.md is ready 🌈

  Project:    https://github.com/orgs/<owner>/projects/<number>
  Build:      /<build_skill>
  Merge:      <merge_strategy>
  Refinement: <refinement_mode>
  Run mode:   <run_mode>

To start the board loop:
  /loop 5m /shoji

Or push any issue with the `shoji` label to trigger it.
```


## Phase 7: Tick Mode

*Runs on every `/shoji` call after setup. Also the handler for GitHub Actions triggers (`--issue <N>`) and `/shoji --status`.*

### 7a. Load config

```bash
cat .shoji.config.json
```

If missing: stop and tell the user to run `/shoji --setup`.

Extract: `owner`, `projectNumber`, `projectId`, `fields.status.*`, `settings.*`

### 7b. Sync untracked issues into the project

Fetch all open repo issues labeled `shoji` that are NOT yet in the project:

```bash
# All open issues with the shoji label
gh issue list --state open --label "shoji" --json number,url,title --limit 200

# All issues currently in the project
gh project item-list $PROJECT_NUM --owner $OWNER --format json --limit 200
```

Cross-reference. For each labeled issue not in the project, add it as Backlog:

```bash
ITEM_ID=$(gh project item-add $PROJECT_NUM --owner $OWNER --url $ISSUE_URL --format json | jq -r '.id')

gh project item-edit \
  --id "$ITEM_ID" \
  --project-id "$PROJECT_ID" \
  --field-id "$STATUS_FIELD_ID" \
  --single-select-option-id "$BACKLOG_OPTION_ID"
```

If `--issue <N>` was passed (GitHub Actions targeted mode): only process that one issue. Add it to the project if needed, then apply the pipeline below to it only.

### 7c. Handle Backlog items

Fetch all project items with status `Backlog`. For each:

**Skip** if labeled `building` (already in flight from a previous tick).

**If labeled `awaiting-clarification`:**
- Fetch comments: `gh issue view $NUM --json comments -q '.comments[].body'`
- Find the last `[shoji.md]` refinement comment (look for the marker string `**[shoji.md]**`)
- Check if a non-agent comment exists after that comment's timestamp
- If the user replied: read the reply, append it to the issue body as a "## Clarification" section, remove the `awaiting-clarification` label, move to Ready (see move command below)
- If no reply yet: skip (still waiting)

**No `awaiting-clarification` label — decide whether to refine:**

Load `settings.refinementMode`:
- `skip` → move to Ready immediately
- `always` → post refinement comment (see template below), add `awaiting-clarification` label
- `agent-decides`:
  - Fetch: `gh issue view $NUM --json title,body -q '{title: .title, body: .body}'`
  - The issue is **clear** if it has ALL of: body longer than 200 characters AND contains at least one acceptance criterion (`- [ ]` or a "## Acceptance Criteria" heading) AND mentions a specific file, component, or behaviour
  - If **clear**: move to Ready
  - If **vague**: post refinement comment, add `awaiting-clarification` label

**Refinement comment:**
```
**[shoji.md]** Before I start building, I want to make sure I get this right:

1. What's the expected outcome? What should work or look different after this is done?
2. Are there specific files, components, or behaviours I should focus on?
3. Any edge cases, constraints, or things I should NOT change?

Reply to this comment and I'll update the issue and start building.
```

```bash
gh issue comment $NUM --body "..."
gh issue edit $NUM --add-label "awaiting-clarification"
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

Fetch all project items with status `Ready`. Collect them into a build queue.

For each item in the queue:

1. Move to In Progress:
   ```bash
   gh project item-edit \
     --id "$ITEM_ID" \
     --project-id "$PROJECT_ID" \
     --field-id "$STATUS_FIELD_ID" \
     --single-select-option-id "$IN_PROGRESS_OPTION_ID"

   gh issue edit $NUM --add-label "building"
   ```

2. Derive a branch slug from the title: lowercase, replace spaces/special chars with `-`, max 40 chars.

3. Add to the parallel build batch.

After collecting all items, spawn **parallel build subagents** — one per issue — using the Agent tool. Wait for all to complete before continuing.

**Build subagent prompt** (fill in real values per issue):

> You are building the feature described in this GitHub issue: `<ISSUE_URL>`
>
> **Config:**
> - Build skill: `<settings.buildSkill>`  (`ship` | `ship-fast` | `basic`)
> - Merge strategy: `<settings.mergeStrategy>`  (`branch-pr-manual` | `branch-pr-auto` | `direct`)
> - Branch: `shoji/<ISSUE_NUM>-<slug>`
>
> **Steps:**
>
> 1. Read the full issue, including all comments:
>    `gh issue view <ISSUE_NUM> --json title,body,comments`
>
> 2. Create a branch (unless merge strategy is `direct`):
>    `git checkout -b shoji/<ISSUE_NUM>-<slug>`
>
> 3. Build using the configured skill:
>    - `ship` → `Skill({ skill: "ship", args: "<issue title + key requirements>" })`
>    - `ship-fast` → `Skill({ skill: "ship-fast", args: "<issue title + key requirements>" })`
>    - `basic` → read the issue, explore the relevant code, implement, commit
>
> 4. Push the branch:
>    `git push -u origin HEAD`
>
> 5. If merge strategy is NOT `direct`, open a PR:
>    ```bash
>    PR_URL=$(gh pr create \
>      --title "<issue title>" \
>      --body "Closes #<ISSUE_NUM>\n\n<one-paragraph summary of what changed>" \
>      --base main)
>    PR_NUM=$(echo "$PR_URL" | grep -oE '[0-9]+$')
>    ```
>
> 6. If merge strategy is `branch-pr-auto`: approve and merge immediately:
>    ```bash
>    gh pr review $PR_NUM --approve --body "**[shoji.md]** Auto-approved"
>    gh pr merge $PR_NUM --squash --auto
>    ```
>
> 7. If merge strategy is `direct`: commit directly to main (no branch, no PR):
>    ```bash
>    git checkout main
>    git push
>    ```
>
> 8. Post a completion comment on the issue:
>    ```bash
>    # For PR strategies:
>    gh issue comment <ISSUE_NUM> --body "**[shoji.md]** Built — PR #$PR_NUM ready for review."
>    # For direct strategy:
>    gh issue comment <ISSUE_NUM> --body "**[shoji.md]** Built — merged directly to main."
>    ```
>
> 9. Remove the `building` label:
>    `gh issue edit <ISSUE_NUM> --remove-label "building"`
>
> Do NOT close the issue — shoji handles that when the PR merges or direct commit is confirmed.
> Do NOT move the project status — shoji handles that in the next tick.

### 7e. Check In Progress items

For each project item with status `In Progress`:

```bash
# Check for a completion comment from shoji
gh issue view $NUM --json comments -q '.comments[].body' | grep "\[shoji\.md\].*Built"

# Check for a linked PR
gh pr list --search "closes:#$NUM" --json number,state,mergedAt --limit 5
```

- **PR found and merged**: move to Done, close issue:
  ```bash
  gh project item-edit ... --single-select-option-id "$DONE_OPTION_ID"
  gh issue close $NUM --reason completed
  ```

- **PR found and open**: move to In Review:
  ```bash
  gh project item-edit ... --single-select-option-id "$IN_REVIEW_OPTION_ID"
  ```

- **No PR, no `building` label, item last updated >60 minutes ago**: stale build detected. Post a comment and move back to Ready for retry:
  ```bash
  gh issue comment $NUM --body "**[shoji.md]** No build activity detected. Moving back to Ready for retry."
  gh project item-edit ... --single-select-option-id "$READY_OPTION_ID"
  ```

### 7f. Check In Review items

For each project item with status `In Review`:

```bash
gh pr list --search "closes:#$NUM" --json number,state,mergedAt,reviewDecision --limit 5
```

- **PR merged**: move to Done, close issue (same as 7e above)
- **`CHANGES_REQUESTED`**: move back to In Progress with a note:
  ```bash
  gh issue comment $NUM --body "**[shoji.md]** PR has changes requested. Moving back to In Progress."
  gh project item-edit ... --single-select-option-id "$IN_PROGRESS_OPTION_ID"
  ```
- **Merge strategy is `branch-pr-auto` and PR is open (no review yet)**: auto-approve and merge:
  ```bash
  gh pr review $PR_NUM --approve --body "**[shoji.md]** Auto-approved"
  gh pr merge $PR_NUM --squash --auto
  ```
- **Open, awaiting manual review**: leave in In Review, no action

### 7g. Print board snapshot

After all processing, print a summary:

```
shoji.md — <ISO timestamp>
────────────────────────────
Backlog      <N>
Ready        <N>
In Progress  <N>  (building: <N>)
In Review    <N>
Done         <N>  (all time)
Blocked      <N>
────────────────────────────
This tick:
  <list each action taken, e.g.:>
  → Moved #12 "add dark mode" to Ready (clear requirements)
  → Posted refinement comment on #14 "fix the thing"
  → Spawned build for #9 "user auth" (ship-fast)
  → Moved #7 "search" to In Review (PR #23 open)
  → Moved #5 "onboarding" to Done (PR #19 merged)

Next check in ~5 min  (/loop 5m /shoji)
```

If `--status` was passed (status-only mode): print the snapshot and exit without taking any actions.
