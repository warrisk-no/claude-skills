---
name: db-pr-review
description: Review a warrisk-no/database PR against the repo's real conventions — war-schema SECURITY DEFINER pattern, split-check mechanics, SQL test format, and the fact that tests do not run in CI.
---

# db-pr-review

Review a pull request in `warrisk-no/database` (PostgreSQL schema, dbmate migrations, PostgREST). This repo has conventions that generic review misses, and its test suite does **not** run in CI — committed tests may never have executed anywhere. This skill encodes what to check and how to verify claims empirically instead of trusting the diff.

## Trigger

User says /db-pr-review or asks to review a database-repo PR (e.g. "review database PR 384").

## Step 1 — Fetch the PR

```bash
gh pr view <url-or-number> --json title,body,author,baseRefName,headRefName,state,additions,deletions,changedFiles
gh pr diff <url-or-number>
```

Review `sql/` changes first and suppress feedback on `migrations/` that is already addressed in `sql/` — the `sql/` directory is generated from the migrated database by `make dumpschema split` and is the source of truth for the end state.

## Step 2 — Access-control conventions (most common failure mode)

- The `war` schema is deliberately locked down: not in PostgREST's `db-schema` (check `postgrest-portal*.conf`), and reachable by API roles (`_user`, `_uwr`) **only through `SECURITY DEFINER` functions** (`public.policy_files_access()`, `policies_organisations_access()`, and ~60 others, including trigger functions). A PR that adds `GRANT ... ON war.* TO _user/_uwr` is almost certainly the wrong fix:
  - Some war tables (e.g. `war.policy_files`) have **no RLS** — a grant exposes the whole table to every `_user` code path.
  - Where RLS exists (e.g. `war.policies_organisations`), a grant makes function bodies see the **intersection** of RLS policies and their own explicit `public.allowed()` checks — an undesigned filter that can silently drop rows. The definer pattern keeps `allowed()`/`uwr()` as the single source of truth.
- The right shape for "function X can't read war table Y" is `CREATE OR REPLACE ... SECURITY DEFINER` (bodies schema-qualify everything; the repo pins no `search_path` anywhere, consistently).
- Verify empirically, don't assume:

```bash
grep -n "<table>" sql/rls.sql sql/rls-policies.sql   # does it have RLS?
grep -n "db-schema" postgrest-portal*.conf            # is the schema API-exposed?
psql $DATABASE_URL -c "select prosecdef from pg_proc where proname = '<function>'"
```

## Step 3 — Migration + split-check mechanics

- `sql/` must exactly match `make dumpschema split` output after migrations apply — the `split.yml` CI check regenerates and diffs. Hand-edited grant **ordering** is the classic failure: pg_dump emits grants in its own order (e.g. `_uwr` before `_user` per table).
- Empty `-- migrate:down` is the repo norm; don't demand down-migrations.
- Regenerating `sql/` locally needs a **PostgreSQL 18 client**: `pg_dump` 17 drops named not-null constraints (four `cyberrisk_*` tables differ). If unrelated files drift after `make dumpschema split`, restore them — never commit environment noise.

## Step 4 — Test conventions

- Test files: first line literally `BEGIN;`, last line literally `ROLLBACK;` (`test/test.sh` greps for these), `DO $$` blocks with `assert`, `${DB_ROLE}` substituted via envsubst, fixed UUIDs from `test/pre-test.sql` (`tests.member_user_id` etc.), `refresh materialized view cache.access` after permission-relevant inserts.
- Schema trap: policies link to organisations via `war.policies_organisations (policy_id, organisation_id)` — `war.policies` has **no** `organisation_id` column.
- **Run every new/changed test locally** — CI won't:

```bash
TESTS="<pattern>" make test    # requires local dev DB per repo makefile/.env
```

## Step 5 — CI reality check

```bash
gh pr checks <number>
gh api repos/warrisk-no/database/actions/workflows --jq '.workflows[] | {name, state}'
```

- `split.yml` ("Split (clean)") is the only meaningful gate. The "Dummy Tests" workflow is `disabled_manually` — a green PR does **not** mean tests ran. Bot-authored branches (Copilot agent) additionally stall on `action_required` even for enabled workflows.
- Known pre-existing local failures from dataset count drift: `test-permissions-circulars`, `test-permissions-invoices-cyberrisk`. Don't attribute these to the PR.

## Step 6 — Copilot comments and verdict

- Fetch inline comments and address or acknowledge each:

```bash
gh api repos/warrisk-no/database/pulls/<number>/comments --jq '.[] | {user: .user.login, path, body}'
```

- Structure the review: overview, blocking issues, security assessment (grants/RLS/definer), test quality, verdict. Verify the PR's factual claims (error messages, column names, "tests pass") against the actual schema and a local run before repeating them.
