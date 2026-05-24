# 🌈 rainbow.md

> A rainbow moves through a full spectrum before it's complete. So does your code: Backlog, Ready, In Progress, In Review, Done. Each stage a different color on the board.
>
> rainbow.md is the agent that moves your issues through that spectrum. Drop a feature into Backlog. Watch it travel through each color until it ships.

A GitHub Projects kanban board that builds itself. One skill. Your issues become features automatically.

```
Backlog → Ready → In Progress → In Review → Done
                                              ↑
                              agents handle everything in here
```

## Why

Most AI coding tools are tied to your laptop. You open a chat, describe what you want, watch it build, and close the tab when you're done. The moment you walk away, the work stops.

rainbow.md flips that model. Your interface is GitHub, not Claude Code. You file issues the same way you always have. You check the kanban board the same way your team always has. The agents run in the background on whatever machine you point them at, whether you're at your desk or not.

This means you can:

- **File an issue from your phone** and come back to a merged PR
- **Run the agent on a cheap cloud server, VPS, or Raspberry Pi.** Any machine with `gh` and `npx` installed will work
- **Keep your team's existing GitHub workflow.** Issues, projects, PRs, labels, all unchanged. The board is the interface; the agents are the implementation detail
- **Build 24/7 without a terminal open.** GitHub Actions triggers rainbow.md automatically on new issues. A long-running loop on a server handles the rest
- **Scale to a team.** Everyone files issues; the agent picks them up in order. No one needs to know Claude Code exists

It follows the same structure software development has always used: backlog, refinement, in progress, review, done. The difference is agents do the building instead of humans typing the code.

### Where to run rainbow.md

| Environment | How | Best for |
|-------------|-----|----------|
| **Your laptop** | `/loop 5m /rainbow` in a Claude Code terminal | Getting started, active dev sessions |
| **Cloud server / VPS** (Hetzner, OVH, AWS, DigitalOcean) | SSH in, install Claude Code + `gh`, run `/loop` in `tmux` | 24/7 unattended operation |
| **Raspberry Pi** | Same as VPS. Claude Code runs fine on arm64 | Low-cost always-on home server |
| **GitHub Actions** | rainbow.md creates the workflow for you during setup | Serverless, event-driven, no machine to maintain |
| **Any CI/CD runner** | Trigger `/rainbow --issue <N>` as a job step | Custom pipelines, self-hosted runners |

**Recommended path:**
1. Start locally with `/loop 5m /rainbow` to test your setup
2. Once it's working, SSH into a server (or use [vibe.md](https://github.com/amajorai/vibe.md) to provision one), clone your repo, and run the loop there permanently
3. Optionally add GitHub Actions as a fallback for when the server is down

#### Setting up on a server or Pi

```bash
# 1. Install Claude Code
npm install -g @anthropic-ai/claude-code

# 2. Authenticate
claude auth login

# 3. Install GitHub CLI and authenticate
# Ubuntu/Debian:
sudo apt install gh
# macOS / others: https://cli.github.com/manual/installation
gh auth login

# 4. Clone your project repo
git clone https://github.com/your-org/your-repo.git
cd your-repo

# 5. Install rainbow.md
npx skills add amajorai/rainbow.md

# 6. Run setup (first time only)
claude "/rainbow --setup"

# 7. Start the board loop in a persistent session
tmux new -s rainbow
claude "/loop 5m /rainbow"
# Detach: Ctrl+B then D
# The loop keeps running after you close SSH
```

To reconnect later: `tmux attach -t rainbow`

## Works great with

- 📦 **[ship.md](https://github.com/amajorai/ship.md)** as the build skill inside rainbow.md for full quality-gated feature delivery: interview → explore → plan → implement → verify → edge cases → E2E → simplify → security review.
- 🪅 **[vibe.md](https://github.com/amajorai/vibe.md)** to provision your production server and deployment pipeline before enabling the board. rainbow.md's setup phase will offer to run it for you.

## Skills

| Skill | What it does |
|-------|-------------|
| [`/rainbow`](skills/rainbow/SKILL.md) | First run: sets up your GitHub Project kanban board, configures build skill, merge strategy, and refinement mode. Every run after: scans for new issues, refines vague ones via comments, spawns parallel build agents, tracks PRs, and moves cards automatically. |

## How it works

```mermaid
flowchart TD
    A["🆕 Issue opened\n(labeled 'rainbow')"] --> B["📥 Backlog\nAdded to project"]
    B --> C{Refinement mode?}
    C -->|"always / agent decides\n(vague issue)"| D["💬 Refinement\nAgent posts comment\nwaits for reply"]
    D -->|"User replies"| E["✅ Ready\nIssue updated with clarification"]
    C -->|"skip / agent decides\n(clear issue)"| E
    E --> F["⚙️ In Progress\nBuild subagent spawned"]
    F --> G{Build skill}
    G -->|/ship| H["🚢 Full pipeline\ninterview → explore → plan\n→ implement → verify → harden"]
    G -->|/ship-fast| I["⚡ Streamlined\nexplore → plan\n→ implement → verify"]
    G -->|basic| J["🔨 Basic build\nexplore → implement → commit"]
    H & I & J --> K["🔀 PR opened\nbranch: rainbow/<issue>-<slug>"]
    K --> L{Merge strategy}
    L -->|"manual review"| M["👀 In Review\nWaiting for you"]
    L -->|"auto-merge"| N["✅ Done\nMerged automatically"]
    M -->|"Approved"| N
    N --> O["🏁 Issue closed\nCard moves to Done"]
```

## Quickstart

```bash
npx skills add amajorai/rainbow.md
```

Then in any repo:

```
/rainbow
```

First run walks you through setup. Takes about 2 minutes. After that, keep your GitHub Project open in a browser tab and watch the board fill itself in.

### Run modes

**Local loop** (start here):
```
/loop 5m /rainbow
```
Runs every 5 minutes in your terminal. Uses your local Claude Code credit pool. Stop it any time with `Ctrl+C`.

**GitHub Actions** (24/7, no terminal needed):

During setup, choose "GitHub Actions" and rainbow.md creates `.github/workflows/rainbow.yml` for you. Add `ANTHROPIC_API_KEY` to your repo secrets and it runs automatically whenever you open or label an issue.

**Both** combines the local loop for active sessions with Actions as a fallback for when your terminal is closed.

### Auto-Update

Pass `--update` to get the latest version before running:
```
/rainbow --update
```

Or set `SKILLS_AUTO_UPDATE: true` in your project CLAUDE.md to always auto-update.

### Claude Code plugin

```
/plugin marketplace add amajorai/rainbow.md
/plugin install rainbowmd@amajorai
```

Invoke as `/rainbowmd:rainbow`.

## Board columns

| Column | Meaning |
|--------|---------|
| **Backlog** | New issues land here. Agent picks them up on the next tick. |
| **Ready** | Requirements are clear. Agent will build on the next tick. |
| **In Progress** | A build subagent is working on this. |
| **In Review** | PR is open, waiting for review or auto-merging. |
| **Done** | Merged and closed. |
| **Blocked** | Issue is stuck. Agent won't touch it until you move it. |

## Labels

| Label | Meaning |
|-------|---------|
| `rainbow` | Tells rainbow.md to track this issue |
| `awaiting-clarification` | Agent posted a question; waiting for your reply |
| `building` | A build subagent is actively working |
| `blocked` | Issue is blocked (manual) |

---

Part of [amajorai/skills](https://github.com/amajorai/skills). For more skills check out the full collection.
