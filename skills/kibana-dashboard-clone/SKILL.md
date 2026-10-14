---
name: kibana-dashboard-clone
description: Copy a Kibana dashboard onto a different index or data view (e.g. a v2 index that keeps the v1 field names), optionally add panels, import it as new objects into a Kibana space, verify every reference, and keep the export in the repo. Use when the user asks to copy, clone, duplicate or "switch the source of" a Kibana dashboard, point its panels at another index, or make a v2 of a customer dashboard.
---

# kibana-dashboard-clone

Copy a dashboard so it reads another index, without touching the original. The method is: export with all
references, swap the data view id everywhere it is referenced, import as **new** objects, verify, and save the
export in the repo.

## Trigger

The user asks to copy or clone a dashboard, switch its data source or index, or build "v2" of a dashboard for a
new index, in any Kibana space (customer spaces such as `nr`, `dnk`, `brit`, `gard`).

## What to ask if not provided

1. **Source dashboard:** space and title or id. List them if unsure (Step 1).
2. **Target index pattern** (e.g. `port-calls-v2`) and its time field (default: the source data view's).
3. **Target space:** default = the source's.
4. **New title**, and whether it should show as a draft until the data is complete.
5. **Extra panels**, if any (e.g. a new field the target index adds).
6. **Where to keep the export** in the repo (warrisk-no/ais: `elastic/kibana/<space>-<slug>.ndjson`).

**Customer spaces are visible to the customer.** Anything created in `nr`, `dnk`, `brit`, … is visible to their
users immediately. Confirm the space and the title before importing; offer a `(draft …)` title while the target
index is still being filled, or build in an internal space first.

## Access

Kibana for the `warrisk` deployment is the Elasticsearch URL with `.es.` replaced by `.kb.` (same port). The
ES API key in `.env` (`ELASTIC_API_KEY`) works on the Kibana API:

```python
import os, re, json, requests
from dotenv import load_dotenv; load_dotenv()
KB = re.sub(r"\.es\.", ".kb.", os.environ["ELASTIC_URL"].rstrip("/"))
H = {"Authorization": f"ApiKey {os.environ['ELASTIC_API_KEY']}", "kbn-xsrf": "true"}
def base(space): return KB if space == "default" else f"{KB}/s/{space}"
```

Every write needs the `kbn-xsrf` header. All API paths below are relative to `base(space)`.

## Step 1 — Find the dashboard

```python
spaces = [s["id"] for s in requests.get(f"{KB}/api/spaces/space", headers=H).json()]
r = requests.get(f"{base(space)}/api/saved_objects/_find", headers=H,
                 params={"type": "dashboard", "per_page": 500, "fields": "title"})
```

## Step 2 — Export with all references

```python
r = requests.post(f"{base(space)}/api/saved_objects/_export", headers=H, json={
    "objects": [{"type": "dashboard", "id": DASH_ID}], "includeReferencesDeep": True})
objs = [json.loads(l) for l in r.text.splitlines() if l.strip()]
```

The last line is a summary (`exportedCount`, `missingRefCount`), not an object. Keep the raw export as a backup
in the job's scratch directory. Note every `index-pattern` object: its `id` and `attributes.title` (the pattern)
and `timeFieldName`.

Then inventory **every** field the copy will use. Field names do not only sit in `sourceField`; they also appear in:
- Lens formulas (`sum(call.feeds.sat)`), Lens layer and column filters;
- saved-search `columns`, `sort` and query;
- the dashboard's KQL or Lucene query and its filters (`meta.key` and the query body), and its controls;
- map layers: `geoField`, metrics, joins, tooltips and layer filters.

A list of paths always misses some. Take every field name of the **old** index instead, and find which of them
occur anywhere in the export's strings:

```python
from elasticsearch import Elasticsearch
es = Elasticsearch(os.environ["ELASTIC_URL"], api_key=os.environ["ELASTIC_API_KEY"])
caps_old = es.field_caps(index=OLD_PATTERN, fields="*").body["fields"]
old_fields = {f for f, t in caps_old.items() if not f.startswith("_") and not set(t) <= {"object", "nested"}}  # leaf fields only

def strings(node):
    if isinstance(node, str):
        yield node
        if node[:1] in "[{":                       # Kibana stores much of its state as JSON inside strings
            try: yield from strings(json.loads(node))
            except ValueError: pass
    elif isinstance(node, dict):
        for k, v in node.items(): yield k; yield from strings(v)
    elif isinstance(node, list):
        for v in node: yield from strings(v)

text = "\n".join(strings([o for o in objs if "type" in o]))
FIELDS = sorted(f for f in old_fields if re.search(rf"(?<![\w.]){re.escape(f)}(?![\w])", text))
```

Read through `FIELDS` once. A common word that happens to be a field name (e.g. `name`) can show up as a false
match, which is harmless: it only adds a field to check. Note any other data views the export references (a
second one, or a control's). Those are not swapped, but in a copy to another space they must exist there (Step 6).

## Step 3 — Check the fields exist in the target

```python
for idx in (OLD_PATTERN, NEW_PATTERN):
    caps = es.field_caps(index=idx, fields=FIELDS).body["fields"]   # pass a list, not a joined string
    # missing: f not in caps; retyped: set(caps[f]) differs between the two indices
```

Every field must exist in the target **with the same type**. A missing or retyped field means that panel needs
rebuilding, not a reference swap: stop and tell the user which panels are affected. Also compare the values a
filter depends on (e.g. `fleet: nr` must exist in the new index).

## Step 4 — Create or reuse the target data view, in the target space

Data views belong to a space. Create the new one in the space the copy will be imported into, which is not
necessarily the source's:

```python
dvs = requests.get(f"{base(target_space)}/api/data_views", headers=H).json()["data_view"]
dv = next((d for d in dvs if d["title"] == NEW_PATTERN), None) or requests.post(
    f"{base(target_space)}/api/data_views/data_view", headers=H,
    json={"data_view": {"title": NEW_PATTERN, "name": NEW_PATTERN, "timeFieldName": TIME_FIELD}}).json()["data_view"]
NEW_ID = dv["id"]
```

## Step 5 — Swap the references

The old data view id appears in many places: the dashboard's `references` (one per panel layer, one per
filter, map layers through `indexPatternRefName`), each Lens panel's own `references` **and** its
`indexPatternId` / `currentIndexPatternId`, and each saved search. A string replace over the serialised objects
catches them all; nothing else carries a UUID that could collide.

```python
keep = [o for o in objs if "type" in o and not (o["type"] == "index-pattern" and o["id"] == OLD_ID)]
keep = [json.loads(json.dumps(o).replace(OLD_ID, NEW_ID)) for o in keep]
assert OLD_ID not in json.dumps(keep)
```

Only the swapped data view is left out (it would otherwise be imported again). Everything else stays: a
second data view, tags, a control's data view. Retitle the dashboard and
every saved search (e.g. "… v4", "port calls v2"), and set a description that says what the copy reads.

**Adding a panel:** copy an existing Lens panel of the same kind as a template; it is the most reliable shape.
Give it a new `panelIndex` (uuid4) and new layer and column ids, and place it below the others
(`gridData.y` = max(y + h)). Set `references: [{type: "index-pattern", id: NEW_ID, name: "indexpattern-datasource-layer-<layerId>"}]`
in its attributes, and add the same reference to the dashboard with the name prefixed `"<panelIndex>:"`.
`bar_percentage_stacked` with one `sum` column per field shows shares over time.

## Step 6 — Import as new copies

**Copying into another space:** check first that every object the copy references is either in `keep` or
already exists in the target space. A reference to an object that is in neither shows up as a
`missing_references` error, and the panel would be broken:

```python
ids_in_copy = {(o["type"], o["id"]) for o in keep}
refs = {(r["type"], r["id"]) for o in keep for r in o.get("references", [])} - ids_in_copy
missing = [t for t in refs if requests.get(f"{base(target_space)}/api/saved_objects/{t[0]}/{t[1]}", headers=H).status_code == 404]
assert not missing, missing     # add those objects to the export (or create them in the target space) first
```

```python
nd = "\n".join(json.dumps(o) for o in keep) + "\n"
r = requests.post(f"{base(target_space)}/api/saved_objects/_import", headers=H,
                  params={"createNewCopies": "true"}, files={"file": ("copy.ndjson", nd, "application/ndjson")})
```

`createNewCopies` gives every object a new id and rewires the references between the copies, so nothing
existing is overwritten. Read `successResults[].destinationId` for the new dashboard's id. Its URL is
`<base>/app/dashboards#/view/<id>`.

## Step 7 — Verify

Export the **new** dashboard with `includeReferencesDeep` and check that:
- the summary says `missingRefCount: 0`;
- the old data view id does not appear;
- the panel count = the original's, plus what you added;
- the filters and time range are as intended;
- the target index answers the panels' aggregations (run one with the dashboard's filters).

Tell the user the dashboard can be opened, and that the original is untouched.

## Step 8 — Keep the export in the repo

Export again with `excludeExportDetails: true`. Strip per-instance fields (`updated_at`, `updated_by`,
`created_at`, `created_by`, `version`) and write with sorted keys, so a later re-export diffs only on real
changes. Commit it with a README entry: what it is, the space, what it reads, how it differs from the original,
and the import command:

```bash
curl -X POST "$KIBANA_URL/s/<space>/api/saved_objects/_import?overwrite=true" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey $ELASTIC_API_KEY" --form file=@elastic/kibana/<file>.ndjson
```

After a later change in Kibana (e.g. renaming a draft), export again and commit, so the file stays the source of
truth. Rename with `PUT /api/saved_objects/dashboard/<id>`, sending `attributes` and the current `references`.

## Gotchas

- **`PUT` saved object:** send the current `references` with the attributes, or the panels lose their data views.
- **Map layers** reference their data view through `indexPatternRefName` (resolved through the dashboard's
  references), not an id in `layerListJSON`. The string replace still covers them; check them in Step 7.
- **Lens by value vs by reference:** panels embedded by value carry their own `attributes.references`.
  Panels by reference (`type: lens` objects in the export) are separate objects, so include them in `keep`.
- **`field_caps`** wants a list of fields; a joined string can come back empty and look like "missing in both".
- Never delete or overwrite the original dashboard or data view unless the user asks.

## Reference

First done for "Ports of call v4" in space `nr` (warrisk-no/ais#196, 2026-10-13): a copy of v3 from `port-calls`
to `port-calls-v2`, with a feeds-over-time panel added. Export: `warrisk-no/ais` `elastic/kibana/nr-ports-of-call-v4.ndjson`.
