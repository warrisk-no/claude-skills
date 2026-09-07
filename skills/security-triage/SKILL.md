---
name: security-triage
description: Sweep GitHub security alerts across an organisation — Dependabot, secret scanning, and code scanning — triage them by severity, check the state of Dependabot fix PRs, file issues for unhandled criticals, flag stale secret-scanning alerts for rotation, and produce a prioritized digest. Use when the user runs /security-triage or asks for a security-alert sweep, security digest, or vulnerability triage across the org.
---

# security-triage

Sweep GitHub security alerts across an organisation — Dependabot, secret scanning, and code scanning — triage them by severity, check the state of Dependabot fix PRs, file issues for unhandled criticals, flag stale secret-scanning alerts for rotation, and produce a prioritized digest. Designed to run weekly on a schedule and on demand.

## Trigger

User says `/security-triage`, or asks for a security-alert sweep, security digest, or vulnerability triage across the org.

## What to ask if not provided

1. **Org** — GitHub organisation slug. Default: `warrisk-no`.
2. **Mode** — `digest` (report only, default) or `act` (also file issues for criticals and comment on stale items).

## Environment note — tokens and no `gh` CLI

**Token for the alert endpoints (Steps 1–2):** cloud sessions proxy GitHub API traffic, and the proxied session token (`GH_TOKEN`/`GITHUB_TOKEN`) is blocked for the `dependabot/alerts`, `secret-scanning/alerts`, and `code-scanning/alerts` paths. A dedicated read-only PAT is provided as `SECURITY_TRIAGE_GH_PAT` in the scheduled routine's environment. Prefer it whenever it is set, and call the alert endpoints with direct `curl` (not `gh api`, which routes through the blocking proxy):

```bash
TOKEN="${SECURITY_TRIAGE_GH_PAT:-${GITHUB_TOKEN:-$GH_TOKEN}}"
curl -sf -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  "https://api.github.com<path>?per_page=100&page=N"
```

paginating manually (loop `page=N` until an empty array comes back). The `gh api` commands in Steps 1–2 below are the canonical endpoint reference — in cloud sessions translate them to this curl form even when `gh` is installed. If the alert calls still return 403 with the PAT set, say so explicitly in the digest (that means the sandbox proxy intercepts direct calls to api.github.com too).

Cloud/sandboxed sessions may also lack `gh` entirely. If `command -v gh` fails, replace every remaining `gh api <path>` below with the same curl form, and use a GitHub MCP server (if available) or the REST API for issue creation/search instead of `gh issue` / `gh search` / `gh pr`. The endpoints and jq filters are identical. Non-alert calls (issues, PRs, search) work fine with the session token — the PAT is only needed for the alert endpoints.

## Step 1 — Dependabot alerts, org-wide

```bash
gh api "/orgs/<ORG>/dependabot/alerts?state=open&per_page=100" --paginate > /tmp/dep_alerts_pages.json
jq -s 'add' /tmp/dep_alerts_pages.json > /tmp/dep_open.json   # pages are separate arrays; merge first

# Totals by severity
jq -r 'sort_by(.security_advisory.severity) | group_by(.security_advisory.severity) | .[] | [.[0].security_advisory.severity, length] | @tsv' /tmp/dep_open.json

# Per-repo: total / critical / high, sorted worst-first
jq -r 'sort_by(.repository.full_name) | group_by(.repository.full_name) | sort_by(-length) | .[] |
  [.[0].repository.full_name, length,
   ([.[] | select(.security_advisory.severity=="critical")] | length),
   ([.[] | select(.security_advisory.severity=="high")] | length)] | @tsv' /tmp/dep_open.json

# Critical detail (package, patched version, age)
jq -r '.[] | select(.security_advisory.severity=="critical") |
  [.repository.full_name, .dependency.package.name, .dependency.scope,
   .security_vulnerability.first_patched_version.identifier // "no patch", .created_at[0:10],
   .security_advisory.summary] | @tsv' /tmp/dep_open.json
```

Note `dependency.scope`: `development`-scoped criticals (test/build tooling) are real but lower urgency than `runtime`.

## Step 2 — Secret-scanning and code-scanning alerts

```bash
gh api "/orgs/<ORG>/secret-scanning/alerts?state=open" --paginate \
  --jq '.[] | [.repository.full_name, .secret_type_display_name, .created_at[0:10], .html_url] | @tsv'

gh api "/orgs/<ORG>/code-scanning/alerts?state=open&severity=critical" --paginate \
  --jq '.[] | [.repository.full_name, .rule.id, .html_url] | @tsv'
gh api "/orgs/<ORG>/code-scanning/alerts?state=open&per_page=100" --paginate > /tmp/code_alerts_pages.json
jq -s 'add' /tmp/code_alerts_pages.json > /tmp/code_open.json
jq -r 'sort_by(.repository.full_name) | group_by(.repository.full_name) | .[] | [.[0].repository.full_name, length] | @tsv' /tmp/code_open.json
```

Any secret-scanning alert older than 14 days is **stale** — secrets cannot be auto-fixed; each one needs rotation + revocation by a human. Always surface these at the top of the digest with owner; use known repo ownership metadata when available, otherwise mark the owner as `unknown` and call out the follow-up needed.

## Step 3 — Dependabot fix-PR health

Security-update PRs are enabled org-wide (since 2026-08-18) and the noisiest repos have grouped `dependabot.yml` configs. Check what Dependabot has opened and whether anything is stuck:

```bash
gh search prs --owner <ORG> --author "app/dependabot" --state open \
  --json repository,title,url,createdAt,isDraft --limit 500 \
  --jq '.[] | [.repository.nameWithOwner, .createdAt[0:10], .title, .url] | @tsv'
```

For each open Dependabot PR, check CI status (`gh pr checks <url>`). Classify:
- **green + security-labelled** → list as "ready to merge" (do not auto-merge; merging follows the org review conventions)
- **red** → list as "needs attention" with the failing check name
- **older than 14 days** → flag as stuck

## Step 4 — Act (only in `act` mode)

For each **critical** Dependabot alert with no open fix PR and no existing tracking issue in that repo (search issues for the package name + "security" before creating):

```bash
gh issue create -R <ORG>/<REPO> \
  --title "Security: bump <package> to >=<first_patched_version> (critical)" \
  --body "<advisory summary, GHSA link, alert link>. Per org convention use a >=<patched> minimum-version constraint for security-sensitive packages."
```

For each stale secret-scanning alert, comment on (or create, if missing) a rotation issue in the affected repo — never paste the secret value or its location; link the alert URL only.

Never dismiss alerts, close issues, or merge PRs from this skill.

## Step 5 — Digest

Produce the digest in this order (worst first):

1. **Secrets needing rotation** — repo, type, age, owner
2. **Critical Dependabot alerts** — with fix-PR/issue status
3. **Dependabot PRs ready to merge** (green CI)
4. **Dependabot PRs stuck/red**
5. **Trend line** — total open alerts by severity vs. previous run if a previous digest is available
6. **Code-scanning summary** — per-repo counts, criticals called out

Keep it short enough to read in two minutes; link everything. When run on a schedule, the digest is the routine's report output.
