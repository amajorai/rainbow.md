# Build Subagent Prompt Template

Use this template per issue. Substitute real values for each field.

---

> You are building the feature described in this GitHub issue: `<ISSUE_URL>`
>
> **Config:**
> - Build skill: `<settings.buildSkill>` (`ship` | `ship-fast` | `basic`)
> - Merge strategy: `<settings.mergeStrategy>` (`branch-pr-manual` | `branch-pr-auto` | `direct`)
> - Branch: `party/<ISSUE_NUM>-<slug>`
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
> 3. Build using the configured skill:
>    - `ship` → `Skill({ skill: "ship", args: "<issue title + key requirements>" })`
>    - `ship-fast` → `Skill({ skill: "ship-fast", args: "<issue title + key requirements>" })`
>    - `basic` → read the issue, explore the relevant code, implement, commit
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
>      --body "Closes #<ISSUE_NUM>\n\n<one-paragraph summary of what changed>" \
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
> 8. Post a completion comment:
>    ```bash
>    # For PR strategies:
>    gh issue comment <ISSUE_NUM> --body "**[party.md]** Built — PR #$PR_NUM ready for review."
>    # For direct strategy:
>    gh issue comment <ISSUE_NUM> --body "**[party.md]** Built — merged directly to main."
>    ```
>
> 9. Remove the `building` label:
>    ```bash
>    gh issue edit <ISSUE_NUM> --remove-label "building"
>    ```
>
> Do NOT close the issue — party.md handles that when the PR merges or the direct commit is confirmed.
> Do NOT move the project board status — party.md handles that in the next tick.
