---
name: deploy-heroku-app
description: Deploy a git branch to a warrisk Heroku app, verify the release against the running slug, and monitor the logs afterwards — including recording a rollback target first, detecting that the deploy would roll back someone else's branch, and log monitoring that survives a dropped tail. Use when the user asks to deploy to Heroku, push a branch to a heroku remote, roll back a Heroku release, or monitor an app after a deploy.
---

# deploy-heroku-app

Deploy a branch to a warrisk Heroku app over git, verify it on the dyno that is actually running, and watch the logs afterwards.

## Trigger

User asks to deploy a branch to Heroku, push to a heroku remote, redeploy an app, roll back a release, or monitor an app after a deploy.

## What to ask if not provided

1. **App name** — e.g. `warrisk-ais-groundcontrol`. Required.
2. **Branch** — defaults to the current branch. A feature branch is fine: Heroku builds whatever lands on its `main`.

Do not ask whether to merge to GitHub `main` first. Deploying a branch and merging its PR are separate decisions, and production often runs an unmerged branch deliberately.

## Step 1 — Record what you are replacing

Before pushing anything:

```bash
heroku releases -a <app> -n 5
heroku ps -a <app>
```

Write the current release version and its commit into your notes and into the final report — that version **is** the rollback target (`heroku rollback vNN -a <app>`). Looking for it after a bad deploy is slower and easier to get wrong.

## Step 2 — Check whether this deploy rolls something back

Heroku's `main` is not GitHub's `main`. It holds whatever was pushed last, which is often an unmerged branch.

```bash
git ls-remote https://git.heroku.com/<app>.git main      # deployed SHA
git merge-base --is-ancestor <deployed-sha> HEAD && echo "fast-forward, safe" || echo "WOULD ROLL BACK"
```

If the deployed SHA is **not** an ancestor of what you are about to push, the deploy removes whatever is currently live. Do not push. Build a combined deploy branch instead:

```bash
git checkout -b deploy/<app>-<date> <your-branch>
git merge --no-ff <the-other-deployed-branch>
# run the full test suite on the merged tree before pushing
```

Push that branch to Heroku and to origin, so the deployed SHA is fetchable rather than existing only in Heroku's git remote.

## Step 3 — Pre-flight

Run the repo's full suite and formatter on the exact tree being deployed — not on the branch you were working on earlier:

```bash
uv run pytest tests/ -q
uv run black --check <changed dirs>
```

## Step 4 — Add the remote and push

```bash
git remote add heroku-<app> https://git.heroku.com/<app>.git   # once
git push heroku-<app> <branch>:main
```

Builds take a few minutes; use a long timeout (10–15 min). Confirm the tail says `Released vNN` and `Verifying deploy... done.`

## Step 5 — Verify on the running slug

Release created ≠ code running. Check the dynos came back:

```bash
heroku releases -a <app> -n 2
heroku ps -a <app>
```

Then smoke-test **the deployed code**, not your local copy — a one-off dyno runs the slug that is live:

```bash
heroku run --exit-code --no-tty -a <app> -- python -c "
import inspect, mypkg.mymodule as m
print('feature present:', 'some_new_symbol' in dir(m))"
```

Assert something specific to the change (a new constant, a registry's contents, a string in a rewritten function). "The build succeeded" is not verification.

## Step 6 — Monitor

Arm a log monitor whose filter covers **failure signatures, not just the happy path**, and which survives the tail dropping:

```bash
while true; do heroku logs --tail -n 1 -a <app> 2>&1 || true; sleep 5; done \
| awk '{ line=$0; gsub(/\033\[[0-9;]*m/,"",line); split(line,a," "); ts=a[1];
         if (line ~ /ERROR|Traceback|State changed from up to crashed|<app-specific warnings>/) {
           if (ts > last) { last=ts; print line; fflush() } } }'
```

Why each part:

- **`while true` + reconnect** — `heroku logs --tail` exits 0 on its own after a while. A monitor without this goes quiet, and a dead monitor looks exactly like a healthy app.
- **timestamp de-duplication** — every reconnect replays recent lines; without the `ts > last` guard you re-report old events as new.
- **no router lines** — `heroku[router]` logs the full query string, which on webhook endpoints contains the shared-secret token. Filter on the app's own log lines instead so secrets are not echoed into the transcript.

When updating the code's log messages, update the filter too — a filter that watches for a warning string you just renamed will never fire.

## Step 7 — Report

State the release version, the rollback command, what the smoke test asserted, and what is being monitored. If throughput matters, quote a metric from after the deploy (`Stream metrics: processed=N`) rather than saying it looks fine.

## Traps

- **Config var changes restart every dyno.** `heroku config:set` triggers a full restart, which rebalances every Kafka consumer group the app's dynos belong to — and in this org apps share group IDs (`KAFKA_GROUP_ID_SUFFIX` is unset), so other apps rebalance too. Never flip `LOG_LEVEL` to DEBUG casually for diagnosis; prefer a code change that logs what you need.
- **Graceful shutdown is observable.** A correct stop logs `Received signal 15` then `Process exited with status 0`. A non-zero exit or a missing SIGTERM line means the consumer did not leave its group cleanly.
- **`heroku run` costs a one-off dyno** and counts against dyno hours. It is worth it for verification; it is not worth it for browsing.
- **Deploying is not merging.** After the PRs merge, redeploy `main` and delete the `deploy/*` branch, or the next person reads Heroku's `main` and sees a branch that no longer exists.
