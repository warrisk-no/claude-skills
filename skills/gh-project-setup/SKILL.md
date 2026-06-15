---
name: gh-project-setup
description: Set up a GitHub project board from scratch — create labels, milestones, issues, and add all items to a project. Use when the user runs /gh-project-setup or asks to set up a GitHub project, create issues for a workstream, or bootstrap a repo's project board.
---

# gh-project-setup

Set up a GitHub project board from scratch: create labels, milestones, issues, and add all items to a project.

## Trigger

User says `/gh-project-setup` or asks to set up a GitHub project, create issues for a workstream, or bootstrap a repo's project board.

## What to ask if not provided

Before running, confirm:
1. **Repo** — `owner/repo` (e.g. `warrisk-no/infrastructure`)
2. **Project number** — the integer from the project URL (`/projects/7`)
3. **Owner** — org or user that owns the project (may differ from repo owner)
4. **Labels** — space-separated list (e.g. `devops pipeline ci staging prod`)
5. **Milestones** — list of phase names (e.g. `"v0 — Research" "v1 — CI/CD parity"`)
6. **Issues** — title, body (Context + Acceptance criteria), milestone, and labels for each

## Steps

### 1. Create labels

```bash
for label in <labels>; do
  gh label create "$label" --repo <owner/repo> --force
done
```

`--force` makes this idempotent — safe to re-run if a label already exists.

### 2. Create milestones

```bash
for title in <milestones>; do
  gh api repos/<owner/repo>/milestones \
    --method POST \
    --field title="$title"
done
```

Note: if a milestone already exists this will create a duplicate. Check first with:
```bash
gh api repos/<owner/repo>/milestones --jq '.[].title'
```

### 3. Create issues and add to project

For each issue:
```bash
url=$(gh issue create --repo "<owner/repo>" \
  --milestone "<milestone name>" \
  --label "<label1>,<label2>" \
  --title "<title>" \
  --body "<body>")

gh project item-add <project-number> --owner <owner> --url "$url"
```

To add existing issues (by number) to the project:
```bash
gh project item-add <project-number> --owner <owner> \
  --url "https://github.com/<owner/repo>/issues/<number>"
```

## Issue body format

Use this structure for every issue:

```markdown
## Context
<1-3 sentences: why this work exists, what it replaces or enables>

## Acceptance criteria
- <concrete, testable criterion>
- <concrete, testable criterion>
- ...
```

## Notes

- Milestone names with dashes or special characters (em-dash `—`) must be quoted in bash
- If `gh issue create` fails mid-script due to a missing label or milestone, fix the issue and re-run from that step only — issues already created will duplicate if you re-run the full script
- Labels are repo-scoped; milestones are repo-scoped; project items are org/user-scoped
- Project number is the integer in the URL: `github.com/orgs/<org>/projects/<number>`
