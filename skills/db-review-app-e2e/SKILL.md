---
name: db-review-app-e2e
description: Wire a portal review app to a database review app so a database PR can be tested end-to-end in the real UI, verify the routing, and report the test URL back to the originating issue.
---

# db-review-app-e2e

Test a `warrisk-no/database` PR end-to-end through the member portal: every database PR gets a Heroku review app (`warrisk-db-pr-<n>`, fresh prod copy + PII obfuscation + seeded test users + migrations), and a portal review app can be pointed at it. Deployed portals call same-origin `/api`, which nginx proxies to `https://${API_SERVER}` (`config/nginx.conf.erb`) — so the switch is one env var, with no CORS or Auth0 changes.

## Trigger

User says `/db-review-app-e2e` or asks to test a database PR/change end-to-end in the portal, or to point a portal review app at a database review app.

## Step 1 — Locate the database review app and wait for its release

```bash
heroku pipelines:info warrisk-db                 # find warrisk-db-pr-<n>
heroku releases -a warrisk-db-pr-<n>             # wait until the deploy release is no longer "executing"
```

The first release copies the production DB, obfuscates PII, seeds test users, and runs migrations — it takes several minutes. Optionally spot-check the fix is live, e.g.:

```bash
heroku pg:psql -a warrisk-db-pr-<n> -c "select p.prosecdef from pg_proc p join pg_namespace n on n.oid = p.pronamespace where n.nspname = '<schema>' and p.proname = '<function_name>' and pg_get_function_identity_arguments(p.oid) = '<arg_types>'"
```

## Step 2 — Open a DO NOT MERGE portal PR

In `warrisk-no/portal`, branch from `main` and change one line in `app.json`:

```json
"API_SERVER": { "value": "warrisk-db-pr-<n>.herokuapp.com" }
```

Hostname only — no scheme, no trailing slash (nginx adds `https://`). Open a **draft** PR titled `DO NOT MERGE: point review app at warrisk-db-pr-<n>`, linking the database PR/issue. Merging it would repoint every future portal review app, hence do-not-merge. Opening the PR makes Heroku create `warrisk-portal-ui-pr-<m>`.

## Step 3 — Set API_SERVER on the created review app (the gotcha)

The `app.json` edit does **not** take effect: pipeline-level review-app config vars override `app.json` env defaults at creation, so the app comes up pointing at `warrisk-portal-db-staging`. Once `warrisk-portal-ui-pr-<m>` exists (check `heroku pipelines:info warrisk-portal-ui`), fix it directly:

```bash
heroku config:set API_SERVER=warrisk-db-pr-<n>.herokuapp.com -a warrisk-portal-ui-pr-<m>
```

This restarts nginx and applies immediately. It is lost if Heroku recreates the review app (e.g. PR reopen) — just re-run it.

## Step 4 — Verify the chain before telling anyone it works

```bash
heroku config:get API_SERVER -a warrisk-portal-ui-pr-<m>       # the db review host
curl -s -o /dev/null -w "%{http_code}\n" https://warrisk-portal-ui-pr-<m>.herokuapp.com/   # 200, app shell
curl -s https://warrisk-portal-ui-pr-<m>.herokuapp.com/api/    # PostgREST JSON; {"message":"LOGGED OUT"} for anon is success
```

Prove routing (don't infer it): curl an `/api/<table>` path, then check the **database** app's router logs for the request:

```bash
heroku logs -a warrisk-db-pr-<n> --num 30 | grep router
```

## Step 5 — Report and tear down

- Comment on the originating issue (`gh issue comment <issue> --repo warrisk-no/database`) with: the portal URL, one line of context (which PR it tests, via which portal PR), concrete click-through steps for a test member, and the note that the portal PR gets closed unmerged afterwards.
- After testing: close the portal PR **unmerged** and delete its branch; Heroku tears the review app down automatically.
