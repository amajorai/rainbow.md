# Build Subagent Prompt Template

Use this template per issue. Substitute real values for each field.

---

> You are building the feature described in this GitHub issue: `<ISSUE_URL>`
>
> **Config:**
> - Build skill: `<settings.buildSkill>` (`ship` | `ship-fast` | `basic`)
> - Merge strategy: `<settings.mergeStrategy>` (`branch-pr-manual` | `branch-pr-auto` | `direct`)
> - Branch: `party/<ISSUE_NUM>-<slug>`
> - Owner/repo: `<OWNER>/<REPO>`
>
> **Steps:**
>
> 1. Read the full issue including all comments:
>    ```bash
>    gh issue view <ISSUE_NUM> --json title,body,comments
>    ```
>
> 2. Create a branch (unless merge strategy is `direct`):
>    ```bash
>    git checkout -b party/<ISSUE_NUM>-<slug>
>    ```
>
> 2.5. Take a **before screenshot** (best-effort — skip silently if no dev server is running):
>    ```bash
>    # Detect a running dev server on common ports
>    SCREENSHOT_PORT=""
>    for port in 3000 5173 4321 8080 8000 4200; do
>      if curl -s --max-time 1 http://localhost:$port > /dev/null 2>&1; then
>        SCREENSHOT_PORT=$port; break
>      fi
>    done
>
>    if [ -n "$SCREENSHOT_PORT" ]; then
>      mkdir -p .party/screenshots
>      npx --yes playwright screenshot \
>        --browser chromium \
>        --full-page \
>        http://localhost:$SCREENSHOT_PORT \
>        .party/screenshots/issue-<ISSUE_NUM>-before.png 2>/dev/null || true
>    fi
>    ```
>
> 3. Build using the configured skill:
>    - `ship` → `Skill({ skill: "ship", args: "<issue title + key requirements>" })`
>    - `ship-fast` → `Skill({ skill: "ship-fast", args: "<issue title + key requirements>" })`
>    - `basic` → read the issue, explore the relevant code, implement (do not commit yet — verify first in step 3.1)
>
> 3.1. **If build skill is `basic`:** after implementing, spawn an Opus agent to verify before committing:
>    ```
>    Agent({
>      subagent_type: "claude",
>      model: "opus",
>      prompt: "Verify this implementation against the acceptance criteria from issue #<ISSUE_NUM>: <AC list from issue body>. Run the project build, then the full test suite + lint + typecheck. All pass → report success. Failures remain → fix directly and run again. After 3 passes with failures still present → report what failed and stop.",
>    })
>    ```
>    If the agent reports success, commit the changes. If it reports failure after 3 passes, post a comment on the issue explaining what failed and stop — do not push broken code.
>
> 3.5. Take an **after screenshot** (same port detected above — skip if none):
>    ```bash
>    if [ -n "$SCREENSHOT_PORT" ]; then
>      # Give the dev server a moment to reload after changes
>      sleep 2
>      npx --yes playwright screenshot \
>        --browser chromium \
>        --full-page \
>        http://localhost:$SCREENSHOT_PORT \
>        .party/screenshots/issue-<ISSUE_NUM>-after.png 2>/dev/null || true
>
>      # Commit screenshots to the branch so they're accessible via raw GitHub URL
>      git add .party/screenshots/issue-<ISSUE_NUM>-*.png 2>/dev/null || true
>      git diff --cached --quiet || git commit -m "chore: add before/after screenshots for #<ISSUE_NUM>"
>    fi
>    ```
>
> 4. Push the branch:
>    ```bash
>    git push -u origin HEAD
>    ```
>
> 5. If merge strategy is NOT `direct`, open a PR:
>    ```bash
>    PR_URL=$(gh pr create \
>      --title "<issue title>" \
>      --body "Closes #<ISSUE_NUM>
>
>    <one-paragraph summary of what changed>" \
>      --base main)
>    PR_NUM=$(echo "$PR_URL" | grep -oE '[0-9]+$')
>    ```
>
> 6. If merge strategy is `branch-pr-auto`: approve and merge immediately:
>    ```bash
>    gh pr review $PR_NUM --approve --body "**[party.md]** Auto-approved"
>    gh pr merge $PR_NUM --squash --auto
>    ```
>
> 7. If merge strategy is `direct`: push directly to main (no branch, no PR):
>    ```bash
>    git checkout main
>    git push
>    ```
>
> 8. Post a completion comment with before/after screenshots if available:
>    ```bash
>    BRANCH="party/<ISSUE_NUM>-<slug>"
>    BEFORE_IMG=".party/screenshots/issue-<ISSUE_NUM>-before.png"
>    AFTER_IMG=".party/screenshots/issue-<ISSUE_NUM>-after.png"
>
>    # Build screenshot markdown if both images were captured
>    SCREENSHOT_MD=""
>    if [ -f "$BEFORE_IMG" ] && [ -f "$AFTER_IMG" ]; then
>      RAW_BASE="https://raw.githubusercontent.com/<OWNER>/<REPO>/$BRANCH"
>      SCREENSHOT_MD="
>
>    | Before | After |
>    |--------|-------|
>    | ![]($RAW_BASE/.party/screenshots/issue-<ISSUE_NUM>-before.png) | ![]($RAW_BASE/.party/screenshots/issue-<ISSUE_NUM>-after.png) |"
>    fi
>
>    # For PR strategies:
>    gh issue comment <ISSUE_NUM> --body "**[party.md]** Built — PR #$PR_NUM ready for review.$SCREENSHOT_MD"
>    # For direct strategy:
>    # gh issue comment <ISSUE_NUM> --body "**[party.md]** Built — merged directly to main.$SCREENSHOT_MD"
>    ```
>
> 9. Remove the `building` label:
>    ```bash
>    gh issue edit <ISSUE_NUM> --remove-label "building"
>    ```
>
> Do NOT close the issue — party.md handles that when the PR merges or the direct commit is confirmed.
> Do NOT move the project board status — party.md handles that in the next tick.
>
> **Screenshot notes:**
> - Screenshots are best-effort. If no dev server is detected or Playwright fails, skip and continue.
> - Playwright is invoked via `npx --yes` so no pre-install is needed.
> - Screenshots are committed to the feature branch and referenced via raw.githubusercontent.com so they render inline in the GitHub issue comment.
> - For non-UI changes (backend, config, migrations) there will be no dev server and no screenshots — that is expected.
