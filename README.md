# warrisk Claude Code Skills

Shared [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for the warrisk-no org.

## Install

```
/plugin install github:warrisk-no/claude-skills
```

## Skills

| Skill | Description |
|---|---|
| `/warrisk:daily-summary` | Daily GitHub activity summary for an org, organised by project: commits, PRs, issues, comments, and Projects activity |
| `/warrisk:cto-summary` | Daily executive/CTO summary: team standup, strategic project status, and auto-detected blockers and flags |
| `/warrisk:gh-project-setup` | Set up a GitHub project board from scratch: labels, milestones, issues, and project items |
| `/warrisk:gh-project-roadmap-setup` | Add custom fields and roadmap view to a GitHub Projects v2 board via GraphQL |
| `/warrisk:issue-from-spec` | Convert a plain-language spec or workstream description into structured GitHub issues |
| `/warrisk:tune-es-transform` | Analyse a running/stopped Elasticsearch transform's stats and recommend optimal `docs_per_second` and `max_page_search_size` |
| `/warrisk:daily-summary` | Daily GitHub activity summary (commits, PRs, issues, comments) organised by project |
| `/warrisk:cto-summary` | Daily executive summary for the org: team standup, strategic project status, blockers |
| `/warrisk:db-review-app-e2e` | Point a portal review app at a database review app for end-to-end testing, verify routing, report the URL to the issue |
| `/warrisk:security-triage` | Sweep org Dependabot/secret-scanning/code-scanning alerts, check fix-PR health, file issues for criticals, produce a prioritized digest |
| `/warrisk:deploy-heroku-app` | Deploy a branch to a Heroku app over git: record the rollback target, detect a deploy that would roll back another branch, verify on the running slug, monitor the logs |
| `/warrisk:survey-kafka-topic` | Inventory what is really on a Kafka topic without joining a consumer group, then cross-check against Elasticsearch to find silently dropped messages |
| `/warrisk:pr-comment` | Write and post GitHub PR comments in house style: summary first, few words, concrete references, facts only; draft → approval → post/edit via `gh api` |

### Repo-local skills

Skills that only make sense in one repo live in that repo's `.claude/skills/`, so they change in the same PR as the conventions they describe. They load in sessions started inside that repo.

| Repo | Skills |
|---|---|
| `warrisk-no/database` | `/db-pr-review`, `/db-rc-perf`, `/sql-test-comments` |

## Update

```
/plugin update warrisk
```
