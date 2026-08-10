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
| `/warrisk:db-pr-review` | Review a `warrisk-no/database` PR against repo conventions: war-schema SECURITY DEFINER pattern, split check, SQL test format, disabled CI tests |
| `/warrisk:db-review-app-e2e` | Point a portal review app at a database review app for end-to-end testing, verify routing, report the URL to the issue |

## Update

```
/plugin update warrisk
```
