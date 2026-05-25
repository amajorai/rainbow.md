# GitHub Project Column Setup

## Rename an existing column

For each existing option that needs renaming (e.g., "Todo" → "Backlog"):

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

## Add a new column

For each column that does not yet exist:

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

## Required columns and colors

| Column | Color |
|--------|-------|
| Backlog | `GRAY` |
| Ready | `BLUE` |
| In Progress | `YELLOW` |
| In Review | `ORANGE` |
| Done | `GREEN` |
| Blocked | `RED` |

After all mutations, run `gh project field-list $PROJECT_NUM --owner $OWNER --format json` again to collect the final option IDs for all six columns — you need these to move items between columns.
