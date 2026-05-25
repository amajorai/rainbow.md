# Config File Template

Write `.party.config.json` (substitute real values from setup):

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
    "branchPrefix": "party/"
  }
}
```
