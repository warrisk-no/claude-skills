# tune-es-transform

Analyse a running or stopped Elasticsearch transform's live stats and recommend optimal `docs_per_second` and `max_page_search_size` settings, then optionally apply them.

## Trigger

User asks to tune, optimise, or find good settings for an Elasticsearch transform, or mentions `docs_per_second` / `max_page_search_size` in the context of a transform.

## What to ask if not provided

1. **Transform ID** — e.g. `ais-data-quality`. Required.
2. **Goal** — speed (minimise checkpoint time) or cluster protection (limit impact on other workloads). Default: balance both.

## Step 1 — Fetch live stats

```bash
uv run esctl transforms <ID> --stats
uv run esctl transform-status <ID>
```

Parse from the raw stats JSON:
- `pages_processed`
- `documents_processed`
- `search_time_in_ms`
- `index_time_in_ms`
- `processing_time_in_ms`
- `search_failures`
- `exponential_avg_checkpoint_duration_ms`
- `exponential_avg_documents_processed`
- `exponential_avg_documents_indexed`

Also read the transform's JSON config file (e.g. `elastic/transforms/<id>.json`) to get the **current** `docs_per_second` and `max_page_search_size` settings. These may differ from what is deployed.

**Important**: the deployed transform may differ from the JSON on disk. Confirm with the user which reflects the running config if there is doubt.

## Step 2 — Compute the per-page profile

```
docs_per_page  = documents_processed / pages_processed
avg_search_s   = search_time_in_ms / 1000 / pages_processed
breakeven_dps  = docs_per_page / avg_search_s
```

`breakeven_dps` is the key figure: any `docs_per_second` **above** this value adds negligible throttle overhead because the search itself is already slower. Any value **below** it starts meaningfully slowing the transform.

## Step 3 — Model docs_per_second options

For each candidate value, compute the throttle wait added per page and per recent checkpoint:

```
throttle_wait_per_page = docs_per_page / dps          (0 if unlimited)
throttle_per_checkpoint = exp_avg_docs_processed / dps (0 if unlimited)
overhead_pct = throttle_per_checkpoint / exp_avg_checkpoint_duration_s * 100
```

Show a table:

| docs_per_second | throttle/page | added per checkpoint | % overhead |
|---|---|---|---|
| 100 | … s | … s | …% |
| 500 | … s | … s | …% |
| 1000 | … s | … s | …% |
| … | | | |
| unlimited | 0 s | 0 s | 0% |

Mark the current setting and the breakeven point.

**Guidance**:
- If search dominates (avg_search_s >> throttle/page), the transform is search-bound — raising `docs_per_second` gives little speed gain, but setting a moderate value (e.g. 1000–2000) provides a light cluster-protection governor at low cost.
- If throttle dominates (docs_per_second << breakeven_dps), increasing it will speed up the transform.
- The `search_failures` rate is a cluster pressure signal: >1% warrants conservative throttle; <0.5% suggests headroom.

## Step 4 — Model max_page_search_size options

Each search request has fixed overhead (shard fan-out, network) plus variable cost scaling with the number of source documents scanned. Larger pages → fewer requests → less fixed overhead, but heavier individual queries.

Use a **40/60 model** (40% fixed overhead, 60% scales with page size) as a starting estimate:

```
current_page_sz = max_page_search_size from config (default 500)
for each candidate size S:
    scale        = S / current_page_sz
    new_pages    = pages_processed / scale
    est_avg_s    = avg_search_s * (0.4 + 0.6 * scale)
    est_total_s  = new_pages * est_avg_s
    delta_pct    = (est_total_s - total_search_s) / total_search_s * 100
```

Show a table across sizes 200, 500, 1000, 2000, 5000, 10000.

**Guidance**:
- Increasing page size reduces total search time until individual queries get long enough to risk timeouts or heavy memory pressure.
- A safe upper bound: keep `est_avg_s` below ~60–90 s per query. Beyond that you risk hitting cluster search timeouts and increasing GC pressure.
- The 40/60 model is an estimate. Measure before and after to confirm.
- ES transforms cap `max_page_search_size` at 10000.

## Step 5 — Recommend

State the recommended pair and the reasoning in 2–3 sentences. Example structure:

> **Recommended: `docs_per_second: 1000`, `max_page_search_size: 1000`**
>
> Search takes 26 s/page (breakeven at 59 docs/s), so the transform is search-bound — `docs_per_second: 1000` adds only 10% overhead while acting as a cluster governor. Doubling page size to 1000 halves the request count and cuts estimated total search time by ~20% with per-query times staying well under 60 s.

If the user's goal is cluster protection, recommend a lower `docs_per_second` (e.g. 500) even at the cost of some checkpoint speed.

## Step 6 — Apply (if requested)

Update the JSON config file and redeploy using the `deploy-es-transform` skill:

```
/deploy-es-transform elastic/transforms/<id>.json
```

If the user only wants to update the settings without a full stop/delete/recreate cycle, note that `docs_per_second` and `max_page_search_size` can be updated on a running transform via the Update Transform API — but the `deploy-es-transform` skill handles this correctly, so prefer it.

## Notes

- Stats accumulate over the transform's lifetime. If it has been reset or recreated recently, the `pages_processed` count may be small and estimates will be noisy — say so.
- `exponential_avg_*` fields reflect a weighted average of recent checkpoints and are more representative than lifetime totals for an ongoing transform.
- A stopped transform's stats are still valid for analysis; the last checkpoint's behaviour is captured in the exponential averages.
- If `pages_processed` is 0 (brand-new transform), skip modelling and advise starting with `max_page_search_size: 500` (default) and no throttle, then re-tuning after the first full checkpoint.
