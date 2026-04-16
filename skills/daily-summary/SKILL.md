# daily-summary

Produce a daily GitHub activity summary for a GitHub organisation: commits (all branches), PRs opened/updated/merged, issues opened/closed, and issue comments, grouped by repo.

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

## Output format

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
  - HH:MM commenter: first line of comment
```

End with a one-line **Summary** — e.g. *"3 repos active · 12 commits · 2 PRs merged · 4 issues updated"*.

## Notes

- Repos with no activity on the target date: skip entirely.
- If `pushed_at` is the day before but PRs or issues were updated today, still include those repos.
- For repos where the commits API returns `409 Git Repository is empty`, skip silently.
- Truncate long comment bodies at ~120 characters to keep the summary readable.
- If the org has many repos (50+), process in parallel using the Agent tool with multiple Bash calls.
