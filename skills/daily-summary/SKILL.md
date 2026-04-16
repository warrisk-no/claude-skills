# daily-summary

Produce a daily GitHub activity summary for a GitHub organisation: commits (all branches), PRs opened/updated/merged, issues opened/closed, issue comments, and GitHub Projects activity — grouped by repo and project.

## Trigger

User says `/daily-summary` or asks for a daily summary, today's activity, or what happened today on GitHub.

## What to ask if not provided

1. **Org** — GitHub organisation slug (e.g. `warrisk-no`). Default: `warrisk-no`.
2. **Date** — ISO date (e.g. `2026-04-16`). Default: today (`currentDate` from context).

## Step 1 — Find active repos

Fetch all repos in the org sorted by `pushed_at`, then filter to those pushed on the target date:

```bash
gh api /orgs/<ORG>/repos --paginate \
  --jq 'sort_by(.pushed_at) | reverse | .[] | select(.pushed_at >= "<DATE>T00:00:00Z") | .name'
```

## Step 2 — Commits (all branches)

For each active repo, iterate branches and collect commits since midnight on the target date. Skip empty results.

```bash
# List branches
gh api /repos/<ORG>/<REPO>/branches --jq '.[].name'

# Commits on a branch
gh api "/repos/<ORG>/<REPO>/commits?sha=<BRANCH>&since=<DATE>T00:00:00Z&per_page=20" \
  --jq '.[] | "\(.commit.author.date[11:16]) \(.sha[0:7]) [\(.commit.author.name)] \(.commit.message | split("\n")[0])"'
```

Deduplicate commits that appear on multiple branches (same SHA).

## Step 3 — Pull requests

Fetch PRs updated on or after the target date. Include open, merged, and closed states.

```bash
gh pr list --repo <ORG>/<REPO> --state all \
  --json number,title,author,state,createdAt,updatedAt,mergedAt \
  --jq '.[] | select(.updatedAt >= "<DATE>T00:00:00Z") | "#\(.number) [\(.state)] \(.author.login): \(.title)"'
```

## Step 4 — Issues

Fetch issues updated on or after the target date. Exclude PRs (they appear in the issues API too).

```bash
gh api "/repos/<ORG>/<REPO>/issues?state=all&since=<DATE>T00:00:00Z&per_page=30" \
  --jq '.[] | select(.pull_request == null) | "#\(.number) [\(.state)] \(.user.login): \(.title)"'
```

## Step 5 — Issue comments

```bash
gh api "/repos/<ORG>/<REPO>/issues/comments?since=<DATE>T00:00:00Z&per_page=30" \
  --jq '.[] | "\(.created_at[11:16]) \(.user.login) on #\(.issue_url | split("/") | last): \(.body | split("\n")[0] | .[0:120])"'
```

## Step 6 — GitHub Projects activity

Find projects updated on the target date, fetch their items (with Status field), and filter to items whose `updatedAt` or `content.updatedAt` falls on the target date. Then for each such item, fetch today's issue comments and timeline events (assigned, closed, reopened, labeled).

```bash
# Find projects updated today
gh api graphql -f query='
  query {
    organization(login: "<ORG>") {
      projectsV2(first: 20) {
        nodes { number title updatedAt closed }
      }
    }
  }' \
  --jq ".data.organization.projectsV2.nodes[] |
    select(.closed == false and .updatedAt >= \"<DATE>T00:00:00Z\") |
    \"\(.number) \(.title)\""

# Items updated today within a project
gh api graphql -f query='
  query($org: String!, $num: Int!) {
    organization(login: $org) {
      projectV2(number: $num) {
        title
        items(first: 50) {
          nodes {
            updatedAt
            fieldValues(first: 10) {
              nodes {
                ... on ProjectV2ItemFieldSingleSelectValue {
                  name
                  field { ... on ProjectV2SingleSelectField { name } }
                }
              }
            }
            content {
              ... on Issue      { number title state updatedAt repository { name } assignees(first: 3) { nodes { login } } }
              ... on PullRequest { number title state updatedAt repository { name } }
              ... on DraftIssue  { title updatedAt }
            }
          }
        }
      }
    }
  }' -f org="<ORG>" -F num=<PROJECT_NUMBER> \
  --jq '.data.organization.projectV2 | .title as $proj |
    .items.nodes[] |
    select((.updatedAt >= "<DATE>T00:00:00Z") or (.content.updatedAt >= "<DATE>T00:00:00Z")) |
    {
      project: $proj,
      itemUpdated: .updatedAt[0:16],
      status: (.fieldValues.nodes[] | select(.field.name == "Status") | .name),
      repo: .content.repository.name,
      number: .content.number,
      title: .content.title,
      state: .content.state,
      assignees: [.content.assignees.nodes[]?.login]
    }'

# Comments on a project item's issue today
gh api "/repos/<ORG>/<REPO>/issues/<NUMBER>/comments" \
  --jq ".[] | select(.created_at >= \"<DATE>T00:00:00Z\") |
    \"\(.created_at[11:16]) \(.user.login): \(.body | split(\"\n\")[0] | .[0:120])\""

# Timeline events (closed, assigned, labeled, reopened)
gh api "/repos/<ORG>/<REPO>/issues/<NUMBER>/events" \
  --jq ".[] | select(.created_at >= \"<DATE>T00:00:00Z\") |
    select(.event == \"closed\" or .event == \"reopened\" or .event == \"labeled\" or .event == \"assigned\") |
    \"\(.created_at[11:16]) [\(.event)] by \(.actor.login)\""
```

## Output format

### Repo section

Group by repo. Within each repo, use these sections (omit empty ones):

```
### `<repo>`

**Commits** (`<branch>`)
- HH:MM `<sha>` [Author] commit message

**Pull requests**
- #N [open|merged|closed] author: title

**Issues**
- #N [open|closed] author: title
  - HH:MM commenter: first line of comment
```

### Projects section

After the repo sections, add a `## Projects` section. Group by project. For each project, list items active today with their status, assignees, and a chronological activity log:

```
## Projects

### `<Project Title>` (#N)

**#<number> — <title>** · <Status> · assignees
- HH:MM event or comment
- HH:MM event or comment
```

Omit projects with no activity on the target date.

End with a one-line **Summary** — e.g. *"3 repos active · 12 commits · 2 PRs merged · 4 issues updated · 3 projects touched"*.

## Notes

- Repos with no activity on the target date: skip entirely.
- If `pushed_at` is the day before but PRs or issues were updated today, still include those repos.
- For repos where the commits API returns `409 Git Repository is empty`, skip silently.
- Truncate long comment bodies at ~120 characters to keep the summary readable.
- If the org has many repos (50+), process in parallel using the Agent tool with multiple Bash calls.
- Skip closed projects when fetching project activity.
- An item may appear in both the repo section (as an issue) and the projects section — this is intentional; the repo section shows what changed in the code, the projects section shows board/workflow movement.
