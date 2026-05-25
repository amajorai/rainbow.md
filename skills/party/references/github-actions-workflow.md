# GitHub Actions Workflow

Create `.github/workflows/party.yml`:

```yaml
name: party.md agent
on:
  issues:
    types: [opened, labeled]

jobs:
  party:
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
      - name: Run party agent
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          direct_prompt: "/party --issue ${{ github.event.issue.number }}"
          allowed_tools: "Bash,Read,Write,Edit,Glob,Grep"
```

After creating the file, tell the user:

> Workflow created at `.github/workflows/party.yml`. To activate it:
> 1. Repo → **Settings → Secrets and variables → Actions**
> 2. Add secret: `ANTHROPIC_API_KEY` = your Anthropic API key
> 3. Commit and push this workflow file
>
> party.md will trigger automatically whenever an issue is opened or labeled in your repo.
