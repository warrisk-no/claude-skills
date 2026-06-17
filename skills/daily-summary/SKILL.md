---
name: daily-summary
description: Produce a daily GitHub activity summary for an organisation — commits across all branches, PRs opened/updated/merged, issues opened/closed, comments, and Projects activity, organised by project. Use when the user runs /daily-summary or asks for a daily summary, today's activity, or what happened today on GitHub.
---

# daily-summary

Produce a daily GitHub activity summary for a GitHub organisation: commits (all branches), PRs opened/updated/merged, issues opened/closed, issue comments, and GitHub Projects activity — organised by project first, with commits and PRs nested under their linked issues, and unaffiliated activity grouped separately at the end.

## Trigger

User says `/daily-summary` or asks for a daily summary, today's activity, or what happened today on GitHub.

## What to ask if not provided

1. **Org** — GitHub organisation slug (e.g. `warrisk-no`). Default: `warrisk-no`.
2. **Date** — ISO date (e.g. `2026-04-16`). Default: today (`currentDate` from context).

Derive `<NEXT_DATE>` (= DATE + 1 day in `YYYY-MM-DD` format) before running any steps — it is used as the exclusive upper bound throughout.

## Step 1 — Find active repos

Fetch all repos pushed on the target date. Use a `[DATE, NEXT_DATE)` window so past-date summaries don't bleed into later days. Repos with only issue/PR activity (no push) are picked up naturally in Steps 3–5 — don't drop them if they appear there.

```bash
gh api /orgs/<ORG>/repos --paginate \
  --jq 'sort_by(.pushed_at) | reverse | .[] |
    select(.pushed_at >= "<DATE>T00:00:00Z" and .pushed_at < "<NEXT_DATE>T00:00:00Z") | .name'
```

## Step 2 — Commits (all branches)

For each active repo, iterate branches and collect commits within the `[DATE, NEXT_DATE)` window. Skip empty results.

```bash
# List branches (paginate — repos with many branches truncate at one page)
gh api /repos/<ORG>/<REPO>/branches --paginate --jq '.[].name'

# Commits on a branch
gh api --paginate "/repos/<ORG>/<REPO>/commits?sha=<BRANCH>&since=<DATE>T00:00:00Z&until=<NEXT_DATE>T00:00:00Z&per_page=100" \
  --jq '.[] | "\(.commit.author.date[11:16]) \(.sha[0:7]) [\(.commit.author.name)] \(.commit.message | split("\n")[0])"'
```

Deduplicate commits that appear on multiple branches (same SHA).

## Step 3 — Pull requests

Fetch PRs updated during the target date. Include open, merged, and closed states.

```bash
gh pr list --repo <ORG>/<REPO> --state all --limit 200 \
  --json number,title,author,state,createdAt,updatedAt,mergedAt \
  --jq '.[] | select(.updatedAt >= "<DATE>T00:00:00Z" and .updatedAt < "<NEXT_DATE>T00:00:00Z") | "#\(.number) [\(.state)] \(.author.login): \(.title)"'
```

For each PR returned, fetch its branch name and closing-issue references. This is used to link commits (which carry the branch name) back to PRs, and PRs back to issues:

```bash
gh pr view --repo <ORG>/<REPO> <PR_NUMBER> \
  --json number,headRefName,closingIssuesReferences \
  --jq '{pr: .number, branch: .headRefName, closes: [.closingIssuesReferences[].number]}'
```

Build a linkage map from this data: branch name → PR number → list of closing issue numbers.

## Step 4 — Issues

Fetch issues updated during the target date. Exclude PRs (they appear in the issues API too).

```bash
gh api --paginate "/repos/<ORG>/<REPO>/issues?state=all&since=<DATE>T00:00:00Z&per_page=100" \
  --jq '.[] | select(.pull_request == null and .updated_at >= "<DATE>T00:00:00Z" and .updated_at < "<NEXT_DATE>T00:00:00Z") | "#\(.number) [\(.state)] \(.user.login): \(.title)"'
```

## Step 5 — Issue comments

```bash
gh api --paginate "/repos/<ORG>/<REPO>/issues/comments?since=<DATE>T00:00:00Z&per_page=100" \
  --jq '.[] | select(.created_at >= "<DATE>T00:00:00Z" and .created_at < "<NEXT_DATE>T00:00:00Z") | "\(.created_at[11:16]) \(.user.login) on #\(.issue_url | split("/") | last): \(.body | split("\n")[0] | .[0:120])"'
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
    select(.closed == false and .updatedAt >= \"<DATE>T00:00:00Z\" and .updatedAt < \"<NEXT_DATE>T00:00:00Z\") |
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
    select(
      ((.updatedAt >= "<DATE>T00:00:00Z") and (.updatedAt < "<NEXT_DATE>T00:00:00Z")) or
      ((.content.updatedAt >= "<DATE>T00:00:00Z") and (.content.updatedAt < "<NEXT_DATE>T00:00:00Z"))
    ) |
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
  --jq ".[] | select(.created_at >= \"<DATE>T00:00:00Z\" and .created_at < \"<NEXT_DATE>T00:00:00Z\") |
    \"\(.created_at[11:16]) \(.user.login): \(.body | split(\"\n\")[0] | .[0:120])\""

# Timeline events (closed, assigned, labeled, reopened)
gh api "/repos/<ORG>/<REPO>/issues/<NUMBER>/events" \
  --jq ".[] | select(.created_at >= \"<DATE>T00:00:00Z\" and .created_at < \"<NEXT_DATE>T00:00:00Z\") |
    select(.event == \"closed\" or .event == \"reopened\" or .event == \"labeled\" or .event == \"assigned\") |
    \"\(.created_at[11:16]) [\(.event)] by \(.actor.login)\""
```

## Output format

Projects are the primary grouping. Issues sit under their project, with commits, PRs, and comments nested under each issue. Activity not linked to any project goes in a separate section at the end.

### Linking activity to issues

Before rendering, build a linkage map using the PR data from Step 3:
- **Commits → PR**: match a commit's branch name to the PR whose `headRefName` matches.
- **PR → issues**: use `closingIssuesReferences`; also recognise `#N` patterns in the PR title.
- **Issue → project**: from the project items fetched in Step 6.

Anything not reachable through this chain is unaffiliated.

### Projects section (rendered first)

```
## Projects

### `<Project Title>` (#N)

**#<number> — <title>** · <Status> · assignees
- HH:MM [event] by actor
- **PR #N** [state] author: title
  - HH:MM `sha` commit message
  - HH:MM `sha` commit message
- HH:MM commenter: comment body
```

Rules:
- Only include project items with activity today (events, linked PR/commit updates, or comments).
- List events (assigned, closed, labeled, reopened) first in chronological order.
- Under each event list, show any PRs that close this issue, with their commits indented beneath.
- After PRs, show any comments on the issue.
- If a project item is itself a PR (not an issue), list its commits directly under it.
- Omit projects with no activity on the target date.

### Unaffiliated activity (rendered after projects)

Commits, PRs, and issues that have no link to any project item:

```
## Unaffiliated activity

### `<repo>`

**Commits** (`<branch>`)
- HH:MM `sha` [Author] commit message

**Pull requests**
- #N [state] author: title

**Issues**
- #N [state] author: title
  - HH:MM commenter: comment
```

Omit this section entirely if all activity is covered by projects.

---

End with a one-line **Summary** — e.g. *"3 repos active · 12 commits · 2 PRs merged · 4 issues updated · 3 projects touched"*.

## Notes

- Repos with no activity on the target date: skip entirely.
- If `pushed_at` is the day before but PRs or issues were updated today, still include those repos.
- For repos where the commits API returns `409 Git Repository is empty`, skip silently.
- Truncate long comment bodies at ~120 characters to keep the summary readable.
- If the org has many repos (50+), process in parallel using the Agent tool with multiple Bash calls.
- Skip closed projects when fetching project activity.
- Items tracked in a project appear only in the Projects section — never duplicated in Unaffiliated activity.
