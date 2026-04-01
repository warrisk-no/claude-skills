# gh-project-roadmap-setup

Set up custom fields and roadmap view in a GitHub Projects v2 board. Covers the manual steps and the GraphQL workarounds for what `gh project` CLI cannot do yet.

## Trigger

User says `/gh-project-roadmap-setup` or asks to configure roadmap dates, add custom fields to a project, or set up a GitHub Projects v2 roadmap view.

## What the gh CLI can and cannot do

| Task | gh CLI | GraphQL API |
|---|---|---|
| Add items to project | `gh project item-add` | yes |
| Create single-select field | no | yes |
| Create date field | no | yes |
| Set field value on item | no | yes |
| Switch to roadmap view | no (UI only) | no |

For field creation and value-setting, use GraphQL via `gh api graphql`.

## Step 1: Get the project node ID

```bash
gh api graphql -f query='
  query($org: String!, $number: Int!) {
    organization(login: $org) {
      projectV2(number: $number) {
        id
        title
      }
    }
  }' -f org=<owner> -F number=<project-number> \
  --jq '.data.organization.projectV2.id'
```

Save the result as `PROJECT_ID`.

## Step 2: Create a date field

```bash
gh api graphql -f query='
  mutation($projectId: ID!, $name: String!) {
    createProjectV2Field(input: {
      projectId: $projectId,
      dataType: DATE,
      name: $name
    }) {
      projectV2Field { id name }
    }
  }' -f projectId="$PROJECT_ID" -f name="Start"
```

Repeat with `name="Target"` for the end date field.

## Step 3: Create a single-select field

Avoid `Type` — it is a reserved built-in field name in GitHub Projects. Use `Work type` instead.

```bash
gh api graphql -f query='
  mutation($projectId: ID!) {
    createProjectV2Field(input: {
      projectId: $projectId,
      dataType: SINGLE_SELECT,
      name: "Work type",
      singleSelectOptions: [
        {name: "Spike", color: YELLOW, description: "Research/investigation"},
        {name: "Implementation", color: BLUE, description: "Delivery with shippable output"}
      ]
    }) {
      projectV2Field {
        ... on ProjectV2SingleSelectField {
          id
          name
          options { id name }
        }
      }
    }
  }' -f projectId="$PROJECT_ID"
```

## Step 4: Set field values on items

First, get item node IDs and field IDs:

```bash
# List items with node IDs
gh api graphql -f query='
  query($projectId: ID!) {
    node(id: $projectId) {
      ... on ProjectV2 {
        items(first: 50) {
          nodes {
            id
            content {
              ... on Issue { number title }
            }
          }
        }
      }
    }
  }' -f projectId="$PROJECT_ID"

# List fields with IDs
gh api graphql -f query='
  query($projectId: ID!) {
    node(id: $projectId) {
      ... on ProjectV2 {
        fields(first: 20) {
          nodes {
            ... on ProjectV2Field { id name }
            ... on ProjectV2SingleSelectField { id name options { id name } }
            ... on ProjectV2IterationField { id name }
          }
        }
      }
    }
  }' -f projectId="$PROJECT_ID"
```

Set a date value:

```bash
gh api graphql -f query='
  mutation($projectId: ID!, $itemId: ID!, $fieldId: ID!, $date: Date!) {
    updateProjectV2ItemFieldValue(input: {
      projectId: $projectId,
      itemId: $itemId,
      fieldId: $fieldId,
      value: { date: $date }
    }) { projectV2Item { id } }
  }' \
  -f projectId="$PROJECT_ID" \
  -f itemId="<item-node-id>" \
  -f fieldId="<start-field-id>" \
  -f date="2026-04-07"
```

Set a single-select value (use the option's `id`, not its name):

```bash
gh api graphql -f query='
  mutation($projectId: ID!, $itemId: ID!, $fieldId: ID!, $optionId: String!) {
    updateProjectV2ItemFieldValue(input: {
      projectId: $projectId,
      itemId: $itemId,
      fieldId: $fieldId,
      value: { singleSelectOptionId: $optionId }
    }) { projectV2Item { id } }
  }' \
  -f projectId="$PROJECT_ID" \
  -f itemId="<item-node-id>" \
  -f fieldId="<work-type-field-id>" \
  -f optionId="<spike-or-implementation-option-id>"
```

## Step 5: Switch to Roadmap view (manual)

The roadmap view can only be added via the UI:

1. Open the project in the browser
2. Click **+ New view** (tab row, top left)
3. Select **Roadmap**
4. In the new view, click **Date fields** and set:
   - Start date → `Start`
   - Target date → `Target`
5. Click **Group by** → `Milestone` for swim lanes

## Reserved field names to avoid

GitHub Projects v2 has built-in fields that cannot be overridden with custom fields:

| Reserved name | Built-in purpose |
|---|---|
| `Title` | Issue/PR title |
| `Assignees` | Issue assignees |
| `Status` | Project status column |
| `Labels` | Issue labels |
| `Linked pull requests` | PR links |
| `Milestone` | Issue milestone |
| `Repository` | Source repo |
| `Type` | Issue type (Bug, Feature, etc.) — added ~2024 |
| `Reviewers` | PR reviewers |
| `Parent issue` | Issue hierarchy |

When naming custom fields, use descriptive alternatives: `Work type` instead of `Type`, `Sprint start` instead of `Start` if it conflicts.
