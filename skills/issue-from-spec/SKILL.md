---
name: issue-from-spec
description: Convert a plain-language feature spec, bullet list, or workstream description into ready-to-create GitHub issues with consistent structure. Use when the user runs /issue-from-spec or provides a spec/plan and asks to turn it into GitHub issues, draft issues, or create issues.
---

# issue-from-spec

Convert a plain-language feature spec, bullet list, or workstream description into ready-to-create GitHub issues with consistent structure.

## Trigger

User says `/issue-from-spec` or provides a spec/description and asks to turn it into GitHub issues, draft issues, or create issues from a plan.

## What to ask if not provided

1. **The spec** — paste or describe the workstream, feature, or set of tasks
2. **Repo** — `owner/repo` to create issues in (or "draft only" to just output the issue bodies)
3. **Labels and milestones** — if known; otherwise derive from context or ask

## Output format

For each issue, produce:

```
### <Issue title>

**Labels:** <label1>, <label2>
**Milestone:** <milestone name>

## Context
<1–3 sentences: why this work exists, what it replaces or enables, who benefits>

## Acceptance criteria
- <specific, testable, unambiguous criterion>
- <specific, testable, unambiguous criterion>
- ...
```

## Guidelines for decomposing a spec into issues

- **One deployable unit per issue.** An issue should represent a thing that can be built, reviewed, and merged independently.
- **Spikes are separate from implementation.** If research must happen before building, create a spike issue (output: decision or doc) and a separate implementation issue.
- **Acceptance criteria must be testable.** Avoid vague criteria like "works correctly" — prefer observable outcomes ("deploys within 5 minutes", "posts to Slack #devops on failure").
- **Size check:** if an issue's acceptance criteria list exceeds ~6 items, consider splitting it.
- **Avoid over-decomposing:** three tightly related steps that always ship together belong in one issue.

## Work type classification

Tag each issue mentally as one of:
- **Spike** — research/investigation, output is knowledge (a decision, a doc, a proof-of-concept). No production code ships.
- **Implementation** — concrete delivery with a shippable artifact (code, config, infrastructure).

Include this in the `Labels` line as `spike` or `implementation` if a label convention exists, or note it in parentheses.

## Example

Input spec:
> We need to replace Heroku CI with GitHub Actions. It should run on every PR, fail on test errors, build the Docker image, and push to GHCR on merge.

Output:
```
### AIS pipeline: GitHub Actions CI

**Labels:** devops, pipeline, ci
**Milestone:** v1 — CI/CD parity

## Context
Replace Heroku CI with GitHub Actions. Runs on every PR and push to main, with image publishing on merge.

## Acceptance criteria
- CI runs automatically on every PR and every push to `main`
- A failing test or lint error blocks merge
- Docker image build is verified on every PR (build only, no push)
- Image is pushed to GHCR on merge to `main` and `staging`
- CI completes in under 10 minutes
```
