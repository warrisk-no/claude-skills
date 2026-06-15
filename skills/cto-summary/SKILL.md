---
name: cto-summary
description: Produce a daily executive/CTO summary for a GitHub organisation — team standup (who worked on what), strategic project status across open projects, and auto-detected blockers and flags. Use when the user runs /cto-summary or asks for an executive summary, CTO view, leadership summary, or standup summary.
---

# cto-summary

Produce a daily executive summary for a GitHub organisation: who worked on what (team standup), strategic project status across all open projects, and auto-detected blockers and flags. Synthesises GitHub activity into plain-language narrative — no raw SHAs or commit logs.

Complements `/daily-summary`, which is engineer-focused. This skill is for a CTO or team lead who wants the full picture in one screen.

## Trigger

User says `/cto-summary` or asks for an executive summary, CTO view, leadership summary, or standup summary.

## What to ask if not provided

1. **Org** — GitHub organisation slug. Default: `warrisk-no`.
2. **Date** — ISO date (e.g. `2026-04-16`). Default: today (`currentDate` from context).

Derive `<NEXT_DATE>` (= DATE + 1 day) before running any steps.

## Step 1 — Active repos and contributors

Find repos with pushes on the target date, then collect all commits grouped by author:

```bash
# Active repos
gh api /orgs/<ORG>/repos --paginate \
  --jq '.[] | select(.pushed_at >= "<DATE>T00:00:00Z" and .pushed_at < "<NEXT_DATE>T00:00:00Z") | .name'

# For each active repo: commits by author across all branches
# List branches
gh api /repos/<ORG>/<REPO>/branches --paginate --jq '.[].name'

# Commits on each branch (deduplicate by SHA across branches)
gh api --paginate "/repos/<ORG>/<REPO>/commits?sha=<BRANCH>&since=<DATE>T00:00:00Z&until=<NEXT_DATE>T00:00:00Z&per_page=100" \
  --jq '.[] | {sha: .sha, author: .commit.author.name, repo: "<REPO>", branch: "<BRANCH>", msg: (.commit.message | split("\n")[0])}'
```

Group deduplicated commits by author. For each author, summarise what they worked on in plain English (infer topic from commit messages and branch names — do not list raw messages).

## Step 2 — PRs merged and open

```bash
gh pr list --repo <ORG>/<REPO> --state all --limit 200 \
  --json number,title,author,state,createdAt,updatedAt,mergedAt,reviewDecision,reviews \
  --jq '.[] | select(.updatedAt >= "<DATE>T00:00:00Z" and .updatedAt < "<NEXT_DATE>T00:00:00Z")'
```

Flag as a blocker: any PR that is `OPEN` and has no review activity (no reviews submitted).

## Step 3 — Issues closed today (wins)

```bash
gh api --paginate "/repos/<ORG>/<REPO>/issues?state=closed&since=<DATE>T00:00:00Z&per_page=100" \
  --jq '.[] | select(.pull_request == null and .closed_at >= "<DATE>T00:00:00Z" and .closed_at < "<NEXT_DATE>T00:00:00Z") | "#\(.number) \(.title)"'
```

## Step 4 — All open strategic projects (full picture)

Fetch ALL open projects, not just those active today. This gives the complete strategic status table.

```bash
# All open projects with item counts and last-updated date
gh api graphql -f query='
  query {
    organization(login: "<ORG>") {
      projectsV2(first: 50) {
        nodes { number title closed updatedAt items(first: 1) { totalCount } }
      }
    }
  }' \
  --jq '.data.organization.projectsV2.nodes[] | select(.closed == false) |
    "\(.number)|\(.title)|\(.items.totalCount)|\(.updatedAt[0:10])"'

# For projects updated today: fetch In Progress / Testing items and detect stale ones
gh api graphql -f query='
  query($org: String!, $num: Int!) {
    organization(login: $org) {
      projectV2(number: $num) {
        items(first: 100) {
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
              ... on Issue { number title state updatedAt }
              ... on PullRequest { number title state updatedAt }
            }
          }
        }
      }
    }
  }' -f org="<ORG>" -F num=<PROJECT_NUMBER> \
  --jq '.data.organization.projectV2.items.nodes[] |
    select(.content != null) |
    {
      status: (.fieldValues.nodes[] | select(.field.name == "Status") | .name),
      title: .content.title,
      number: .content.number,
      updatedAt: .updatedAt
    }'
```

A project item is **stale** if its status is `In Progress` or `Testing` and `updatedAt` is more than 3 days before the target date.

## Output format

```
## <DATE> — Executive Summary

<2-3 sentences: how many people were active, which strategic areas moved, any standout outcome or concern>

## Team

- **<Name>** — <plain-English summary of what they worked on, which project/area, outcome>
- ...
(List every contributor who made commits or opened/merged PRs. Omit org members with no activity.)

## Strategic Projects

| Project | Status | Today |
|---|---|---|
| <Title> (#N) | 🔄 Active | <1-sentence summary of what moved> |
| <Title> (#N) | ✅ On track | <brief note, or "No activity — N items open"> |
| <Title> (#N) | ⏸ Idle | No activity — N items, last active YYYY-MM-DD |
| <Title> (#N) | 🚨 Stale | N items In Progress with no activity since YYYY-MM-DD |
```

Status assignment rules:
- **🔄 Active** — project had item updates or commits today
- **✅ On track** — no activity today but progressing normally (last active ≤ 3 days ago)
- **⏸ Idle** — no activity today, last active > 3 days ago
- **🚨 Stale** — has items in `In Progress` or `Testing` with `updatedAt` > 3 days ago

```
## Blockers / Flags

- ⚠️ <specific item needing attention — PR awaiting review, issue stuck in Testing, stale project>
- ✅ <win — issue closed, PR merged, milestone reached>
```

End with a one-line **Summary** — e.g. *"4 contributors active · 1 PR merged · 2 issues closed · 3 of 13 projects moved today"*.

## Notes

- Do NOT include raw commit SHAs, branch names, or per-line commit messages in the output.
- Translate technical commit messages into plain English relevant to a non-engineering audience where helpful.
- Show ALL open projects in the Strategic Projects table — not just those active today. This gives the CTO a complete status snapshot.
- If a project has 0 items, it is likely being set up — show as ⏸ Idle.
- Combine activity for the same person across multiple repos into one Team entry.
- If the org has many repos (50+), collect commits in parallel using the Agent tool with multiple Bash calls.
