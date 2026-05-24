# 🌈 shoji.md

> **Shoji** (障子) — a traditional Japanese sliding screen made of a translucent paper panel in a wooden frame. It divides spaces while letting you see the activity on the other side as soft silhouettes through the paper.
>
> shoji.md is your translucent interface into the development process. Drop issues into Backlog. Watch their silhouettes move across the board — Refinement, In Progress, In Review, Done — while the agents do the work on the other side of the screen.

A GitHub Projects kanban board that builds itself. One skill. Your issues become features automatically.

```
Backlog → Ready → In Progress → In Review → Done
                                              ↑
                              agents handle everything in here
```

## Skills

| Skill | What it does |
|-------|-------------|
| [`/shoji`](skills/shoji/SKILL.md) | First run: sets up your GitHub Project kanban board, configures build skill, merge strategy, and refinement mode. Every run after: scans for new issues, refines vague ones via comments, spawns parallel build agents, tracks PRs, and moves cards automatically. |

## How it works

```mermaid
flowchart TD
    A["🆕 Issue opened\n(labeled 'shoji')"] --> B["📥 Backlog\nAdded to project"]
    B --> C{Refinement mode?}
    C -->|"always / agent decides\n(vague issue)"| D["💬 Refinement\nAgent posts comment\nwaits for reply"]
    D -->|"User replies"| E["✅ Ready\nIssue updated with clarification"]
    C -->|"skip / agent decides\n(clear issue)"| E
    E --> F["⚙️ In Progress\nBuild subagent spawned"]
    F --> G{Build skill}
    G -->|/ship| H["🚢 Full pipeline\ninterview → explore → plan\n→ implement → verify → harden"]
    G -->|/ship-fast| I["⚡ Streamlined\nexplore → plan\n→ implement → verify"]
    G -->|basic| J["🔨 Basic build\nexplore → implement → commit"]
    H & I & J --> K["🔀 PR opened\nbranch: shoji/<issue>-<slug>"]
    K --> L{Merge strategy}
    L -->|"manual review"| M["👀 In Review\nWaiting for you"]
    L -->|"auto-merge"| N["✅ Done\nMerged automatically"]
    M -->|"Approved"| N
    N --> O["🏁 Issue closed\nCard moves to Done"]
```

## Quickstart

```bash
npx skills add amajorai/shoji.md
```

Then in any repo:

```
/shoji
```

First run walks you through setup — takes about 2 minutes. After that, just keep your GitHub Project open in a browser tab and watch the board fill itself in.

### Run modes

**Local loop** (start here):
```
/loop 5m /shoji
```
Runs every 5 minutes in your terminal. Uses your local Claude Code credit pool. Stop it any time with `Ctrl+C`.

**GitHub Actions** (24/7, no terminal needed):

During setup, choose "GitHub Actions" and shoji.md creates `.github/workflows/shoji.yml` for you. Add `ANTHROPIC_API_KEY` to your repo secrets and it runs automatically whenever you open or label an issue.

**Both** — local loop for active sessions, Actions as a fallback.

### Auto-Update

Pass `--update` to get the latest version before running:
```
/shoji --update
```

Or set `SKILLS_AUTO_UPDATE: true` in your project CLAUDE.md to always auto-update.

### Claude Code plugin

```
/plugin marketplace add amajorai/shoji.md
/plugin install shojimd@amajorai
```

Invoke as `/shojimd:shoji`.

---

## Board columns

| Column | Meaning |
|--------|---------|
| **Backlog** | New issues land here. Agent picks them up on the next tick. |
| **Ready** | Requirements are clear. Agent will build on the next tick. |
| **In Progress** | A build subagent is working on this. |
| **In Review** | PR is open, waiting for review (or auto-merging). |
| **Done** | Merged and closed. |
| **Blocked** | Issue is stuck. Agent won't touch it until you move it. |

## Labels

| Label | Meaning |
|-------|---------|
| `shoji` | Tells shoji.md to track this issue |
| `awaiting-clarification` | Agent posted a question; waiting for your reply |
| `building` | A build subagent is actively working |
| `blocked` | Issue is blocked (manual) |

---

## Works great with

- **[ship.md](https://github.com/amajorai/ship.md)** — Use as the build skill inside shoji.md for full quality-gated feature delivery: interview → explore → plan → implement → verify → edge cases → E2E → simplify → security review.
- **[vibe.md](https://github.com/amajorai/vibe.md)** — Set up your production server and deployment pipeline first. shoji.md's setup phase will ask if you want to run vibe.md before enabling the board.

---

Part of [amajorai/skills](https://github.com/amajorai/skills). For more skills check out the full collection.
