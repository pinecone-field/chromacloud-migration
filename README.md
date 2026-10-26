# Chroma Cloud to Pinecone migration

Scripts that move a Chroma Cloud collection to a [Pinecone](https://www.pinecone.io/) serverless index during a planned write pause, and check every step before you switch your app over.

Chroma is [winding down Chroma Cloud](https://www.trychroma.com/cloud-wind-down):

- **November 22, 2026:** writes stop. Chroma Cloud becomes read-only.
- **December 22, 2026:** reads stop. Chroma Cloud no longer serves queries.

Open-source Chroma is not affected.

## Disclaimer

This toolkit is provided as-is, with **no guarantees**. It may not fit your collection, embedding setup, or cutover plan. Before you pause writes or flip production traffic, run the scripts yourself against your own data, review every finding, and confirm that validation matches what your app needs. You are responsible for verifying the migration works for your use case.

## What this does

The toolkit moves **one collection** that already has dense embeddings, document text, and metadata into a Pinecone document index. Your existing vectors are reused, so nothing is re-embedded and your app keeps its embedding model. To move several collections, run it once per collection.

| Script | What it does |
|---|---|
| `preflight.py` | Scans the collection and reports dimension, distance metric, metadata types, and anything Pinecone would reject |
| `create_index.py` | Creates a Pinecone index with the matching dimension and metric |
| `export.py` | Streams the collection to `export.jsonl` and checks the count didn't change |
| `convert.py` | Turns each record into a Pinecone document and logs every rename, remap, and drop |
| `load.py` | Upserts into a fresh namespace in size-bounded batches, safe to rerun |
| `validate.py` | Compares counts, the full ID set, sampled records, and query results between Chroma and Pinecone |

The cutover plan assumes your app pauses writes for the migration window. With writes paused, there are no updates or deletes to replay, so the export is a consistent snapshot and validation can compare the two stores exactly.

## Use it with a coding agent

The repo includes an [Agent Skills](https://agentskills.io) skill, [`skills/chroma-cloud-migration`](skills/chroma-cloud-migration/SKILL.md), that walks a coding agent such as Claude Code, Cursor, or Codex through these steps. It runs each script in order, stops on every failed check, and leaves the write pause and the cutover to you.

```bash
npx skills add pinecone-field/chromacloud-migration
```

Then ask your agent to migrate your Chroma Cloud collection to Pinecone.

## Requirements

- [uv](https://docs.astral.sh/uv/getting-started/installation/). It installs Python 3.13 (pinned in `.python-version`) if you don't have it
- A Pinecone API key ([get one here](https://app.pinecone.io/))
- Chroma Cloud credentials: API key, tenant, and database
- Your app's query-embedding function, for the golden-query check in `validate.py`

## Quickstart

```bash
git clone https://github.com/pinecone-field/chromacloud-migration.git
cd chromacloud-migration
uv sync

export PINECONE_API_KEY=...                         # all Pinecone steps
export CHROMA_API_KEY=... CHROMA_TENANT=... CHROMA_DATABASE=...   # Chroma Cloud
```

`uv sync` creates a virtual environment from `uv.lock`. Run each script with `uv run`, which uses that environment.

Every script takes `--workdir DIR` (default `migration_work`). Each step reads the previous step's files from that directory and checks them by checksum. Scripts that read Chroma take `--collection NAME` and connect with `chromadb.CloudClient()`, which reads the three `CHROMA_*` variables. To rehearse against a local copy instead, add `--chroma-path PATH` to use `chromadb.PersistentClient`.

All scripts share the same exit codes:

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | A check failed or preflight found a blocking issue |
| 2 | Usage or precondition error: wrong collection or index name, missing `PINECONE_API_KEY`, or missing workdir files |

## Migration steps

Replace `my_collection`, `my-index`, and `my-collection-v1` with your own names. Pass the **same policy flags** (for example `--array-policy stringify`) to `preflight.py` and `convert.py`.

```bash
# Before the pause: rehearse, then create the index
uv run preflight.py --collection my_collection --workdir migration_work
uv run preflight.py --collection my_collection --workdir migration_work --allow-transforms   # after reviewing TRANSFORM findings
uv run create_index.py --index my-index --workdir migration_work

# Pause app writes (reads stay on Chroma), then:
uv run preflight.py --collection my_collection --workdir migration_work --allow-transforms
uv run export.py    --collection my_collection --workdir migration_work
uv run convert.py   --workdir migration_work
uv run load.py      --index my-index --namespace my-collection-v1 --workdir migration_work
# golden.jsonl holds query embeddings from your app's model (see "Golden queries" below)
uv run validate.py  --collection my_collection --index my-index --namespace my-collection-v1 \
                    --workdir migration_work --golden-queries golden.jsonl
# optional: add --app-filters app_filters.json to compare your app's where filters

# Decision gate, then flip reads and writes together
```

The full sequence:

1. **Rehearse.** Run preflight, then run steps 2 through 6 on a copy or a small collection. Time the export and load to size your write-pause window.
2. **Create the index** and ship an app change that switches reads **and** writes from Chroma to Pinecone with one config flag. Leave it pointed at Chroma.
3. **Pause writes.** Reads stay on Chroma. Rerun preflight with the same flags.
4. **Export.** `export.py` fails if the count changes during the export.
5. **Convert and load** into a fresh, empty namespace.
6. **Validate.** If a data check fails, stop, fix the cause, and reload into a new namespace (for example `my-collection-v2`). Don't patch gaps with an upsert-only pass: upserts can't remove records deleted in Chroma. If only `app_filters` fails, fix your app's filter instead.
7. **Decision gate.** This is the last point where rollback is free. The data is in Pinecone, but your app hasn't read from it or written to it yet.
8. **Flip reads and writes together**, then resume writes.
9. **Monitor** errors, latency, and result quality. From here, rolling back means replaying Pinecone writes into Chroma.
10. **Retire Chroma Cloud** before December 22. Remove Chroma client code, rotate or delete Chroma Cloud credentials, and archive `export.jsonl` with `field_map.json`, `id_map.jsonl`, and `transforms.jsonl`. You need all four files to reverse renames and ID mappings.

If you reach November 22 first, Chroma enforces the write pause for you. Steps 4 through 8 still work until December 22, but your app's writes fail until you flip.

### Collections with long documents

Document text is stored as a metadata field by default and counts toward Pinecone's 40 KB metadata limit per record. If preflight reports `metadata_over_40kb` because of long documents, either split the text in the source or store it as a [full-text search](https://docs.pinecone.io/guides/search/full-text-search) field. Decide before you create the index, because the schema can't change later. Run these commands **instead of** the matching ones above:

```bash
uv run preflight.py   --collection my_collection --workdir migration_work --text-mode fts --allow-transforms
uv run create_index.py --index my-index --workdir migration_work --full-text-search
uv run convert.py     --workdir migration_work --text-mode fts
```

A full-text search field doesn't count toward the 40 KB limit, but each document must stay within 100 KB, 10,000 tokens, and 1,024 bytes per token. Preflight blocks documents over these limits. Unspaced CJK text tokenizes as one long token and can hit the per-token limit.

### Golden queries

`validate.py` compares top-k results from Chroma and Pinecone for a set of real queries. Build `golden.jsonl` with the same embedding call your app uses for Chroma queries. This repo does not embed text.

```python
import json
queries = ["how do refunds work", "reset my password"]          # real user queries
with open("golden.jsonl", "w") as f:
    for i, q in enumerate(queries):
        emb = my_app_embed(q)   # INTEGRATION POINT: your app's query-embedding function
        f.write(json.dumps({"query_id": f"q{i}", "embedding": list(map(float, emb))}) + "\n")
```

## Concept mapping

| Chroma | Pinecone |
|---|---|
| Tenant / database | Organization / project |
| Collection | Index, or a namespace in a shared index when collections share dimension and metric |
| `ids` | `_id`: 1 to 512 ASCII characters, no NUL. Other IDs are remapped |
| `embeddings` | A `dense_vector` field named `embedding` in the index schema |
| `documents` | A `text` field on the document |
| `metadatas` | Top-level document fields, filterable automatically |
| Distance `l2` (default) / `cosine` / `ip` | Metric `euclidean` / `cosine` / `dotproduct` |
| `where` filter | `filter` with `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$and`, `$or`. `$exists` and `$not` exist in Pinecone but aren't translated by the toolkit; rewrite those by hand |
| `$contains` / `$not_contains` on an array key | `{"$in": [value]}` / `{"$nin": [value]}` |
| `$contains` / `$not_contains` on a non-array key, `where_document` | No equivalent. Redesign these queries |

## What changes in your data

`convert.py` never drops or changes data silently. Every per-record change is logged to `transforms.jsonl`, and the mapping tables let you reverse renames and ID remaps.

- **Distance metric.** Chroma defaults to `l2`, which maps to `euclidean`. The validator's metric check catches a mismatch. Don't rely on search quality to reveal one: with normalized vectors, `l2` and `cosine` rank results the same way.
- **Document text.** By default, document text is a metadata field and counts toward Pinecone's 40 KB metadata limit per record. For longer documents, see [Collections with long documents](#collections-with-long-documents). Adding full-text search after loading means a new index and a reload.
- **IDs.** IDs that are empty, non-ASCII, contain NUL, or are over 512 characters become `chroma-sha256-<sha256 of the ID>`, recorded in `id_map.jsonl` ([Python SDK reference](https://sdk.pinecone.io/python/reference/sync-index.html)). The original ID is also stored in a `chroma_id` field. Your app must use the new IDs wherever it reads or writes by ID.
- **Metadata keys.** Keys that start with `_` or `$`, exceed 64 bytes, or equal a reserved field (`embedding`, `text`, `chroma_id`) become `meta_<key>`. If that name is taken or too long, it is truncated and gets a hash suffix. The mapping is deterministic and recorded in `field_map.json`. Update your app's filters and reads to use the new names.
- **Arrays.** Chroma [metadata arrays](https://www.trychroma.com/changelog/metadata-arrays) can hold strings, ints, floats, or booleans. Pinecone lists must be lists of strings ([data modeling](https://docs.pinecone.io/guides/index-data/data-modeling)). Numeric and boolean arrays are rejected by default. `--array-policy stringify` converts elements (`3` to `"3"`, `True` to `"true"`) and logs each record. Stringified fields lose numeric and boolean filtering, so range filters on them can't be translated.
- **None values.** `None` metadata values are dropped and logged.
- **Numbers.** Pinecone stores numbers as 64-bit floats, so ints read back as floats. Pinecone silently rounds ints beyond 2^53, so they are rejected by default. `--big-int-policy stringify` stores them as strings.
- **All-zero vectors.** Chroma accepts them, Pinecone rejects them. Preflight blocks them; fix them in Chroma before the pause.
- **Filters.** `$contains` and `$not_contains` on array keys become `$in` and `$nin` (see the table above), which matched Chroma in testing. On any other key they need redesign, because Chroma's `$contains` on a scalar matches nothing while `$in` matches equal values. Chroma's `where_document` substring and regex filters ([Chroma filtering](https://docs.trychroma.com/cloud/search-api/filtering)) also need redesign; Pinecone's text-match operators work only on full-text search fields. Run the parity check with `--app-filters` on preflight and validate. See [Pinecone filter docs](https://docs.pinecone.io/guides/search/filter-by-metadata).
- **Scores.** `cosine` scores are 1 minus Chroma's cosine distance. `dotproduct` scores are 1 minus Chroma's `ip` distance. `euclidean` scores approximate squared L2 distance, where lower is closer. Update any distance thresholds in your app.
- **Freshness.** Search can lag a load by several seconds even when counts and ID listings already match. `validate.py` polls before its search and filter checks.

## Script reference

### preflight.py

Scans the collection page by page. Reports `collection.count()`, embedding dimension (and whether all records agree), distance space, the embedding function recorded on the collection, every metadata key with its value types and target Pinecone field name, None and empty documents, and IDs Pinecone would reject. The distance space is read from the collection configuration (`hnsw` or `spann`), then the collection schema, then `hnsw:space` metadata; Chroma's default is `l2`.

| | |
|---|---|
| Inputs | `--collection`, `--chroma-path` (optional), policy flags shared with `convert.py` (`--array-policy`, `--big-int-policy`, `--text-mode`, `--vector-field`, `--text-field`, `--id-field`, `--id-prefix`), `--allow-transforms`, `--app-filters FILE` (optional JSON list of `{"name", "where"}` or `{"name", "where_document"}`), `--page-size` (100) |
| Outputs | `preflight.json`; findings printed as BLOCKING, TRANSFORM, WARNING, or INFO |
| Exit 0 | No blocking findings, and either no transforms are needed or `--allow-transforms` was given |
| Exit 1 | Any blocking finding (zero vectors, mixed dimensions, metadata over 40 KB, unsupported types, numeric or boolean arrays under `--array-policy reject`, and with `--text-mode fts`: text over 100 KB, over 10,000 estimated tokens, or a token over 1,024 bytes), or transforms needed without `--allow-transforms` |
| Exit 2 | Collection cannot be opened |

With `--app-filters`, `$contains` and `$not_contains` are rewritten to `$in` and `$nin` only for keys whose values are arrays in every record. On any other key they are reported as needing redesign, because Chroma's `$contains` on a scalar matches nothing while `$in` matches equal values. `where_document` filters are always reported as needing redesign.

### create_index.py

Creates a document index with one `dense_vector` field (`embedding` by default), the source dimension, and the mapped metric. `--full-text-search` also declares the text field with full-text search. If the index already exists, the script only verifies its schema.

| | |
|---|---|
| Inputs | `--index` (required), `--workdir`, `preflight.json` (or `--dimension N` and `--metric cosine\|euclidean\|dotproduct`), `--cloud` (default `aws`), `--region` (default `us-east-1`), `--full-text-search`, `--language` (default `en`), `--deletion-protection enabled\|disabled` (default `disabled`) |
| Outputs | A ready index |
| Exit 0 | Index ready with the expected schema |
| Exit 1 | Existing index has a different dimension, metric, or text field |
| Exit 2 | No dimension or metric available, or `PINECONE_API_KEY` not set |

Regions other than AWS `us-east-1` require a Standard or Enterprise plan.

### export.py

Streams `collection.get(include=["embeddings", "documents", "metadatas"], limit, offset)` to JSONL, holding only the set of IDs in memory. Metadata keys are written sorted so the file is reproducible.

| | |
|---|---|
| Inputs | `--collection`, `--chroma-path`, `--page-size` (100) |
| Outputs | `export.jsonl` (one `{"id", "embedding", "document", "metadata"}` per line; keep it as your archive), `export_manifest.json` (counts before and after, unique IDs, metric, SHA-256) |
| Exit 0 | Unique exported IDs equal `collection.count()`, no duplicates, and the count didn't change during the export |
| Exit 1 | Any of those checks failed (the manifest has `"ok": false`). Writes are probably not paused |
| Exit 2 | Collection cannot be opened |

The count check can't detect an update that leaves the count unchanged. The write pause is what prevents those, so enforce it in your app.

### convert.py

Makes two streaming passes over `export.jsonl`: the first builds the mapping tables, the second writes documents. See [What changes in your data](#what-changes-in-your-data) for the rules. If any record fails, no `documents.jsonl` is written.

| | |
|---|---|
| Inputs | `export.jsonl`, `export_manifest.json` (must be ok and match its checksum), policy flags |
| Outputs | `documents.jsonl`, `field_map.json`, `id_map.jsonl`, `transforms.jsonl`, `convert_summary.json` |
| Exit 0 | Every record converted |
| Exit 1 | Blocking issues or per-record errors. No `documents.jsonl` is written; `transforms.jsonl.failed` is kept for review |
| Exit 2 | Export missing, not ok, or checksum mismatch |

`transforms.jsonl` entries: `id_remapped`, `none_value_dropped`, `document_none_omitted`, `document_empty_omitted` (full-text search mode only), `array_stringified`, `big_int_stringified`.

### load.py

Streams `documents.jsonl` with `index.documents.upsert`, one batch in memory at a time. A batch is sent before it would exceed `--max-docs` (default and maximum 1,000) or `--max-request-bytes` (default 1,900,000, under the 2 MB API limit). Retries rate-limit, 5xx, connection, and timeout errors with backoff.

The namespace must be new or empty. A rerun into the same namespace is allowed only when `load_state.json` shows the same index, namespace, and `documents.jsonl` checksum. Upserts overwrite by `_id`, so a rerun leaves the same content. `--resume` skips lines a previous run confirmed.

| | |
|---|---|
| Inputs | `--index`, `--namespace`, `documents.jsonl`, `convert_summary.json`, `field_map.json` |
| Outputs | `load_state.json` |
| Exit 0 | Every document accepted |
| Exit 1 | An upsert failed, or the index dimension, metric, or text field doesn't match |
| Exit 2 | Convert output missing or modified, namespace not empty without a matching load state, index doesn't exist, or `PINECONE_API_KEY` not set |

### validate.py

| Check | Fails when |
|---|---|
| `source_unchanged` | `collection.count()` differs from the exported count |
| `metric_and_dimension` | Chroma's space or dimension doesn't match the index schema. For an empty collection, pass `--dimension N` or the dimension is reported as not checked |
| `namespace_count` | The namespace record count doesn't reach the exported count within `--poll-timeout` (600 s) |
| `id_set` | Any source ID (mapped through `id_map.jsonl`) is missing from `index.documents.list(...)`, or any extra ID is present |
| `sample_fields` | A sampled document differs field by field from its source record, or doesn't round-trip back to the Chroma record. Samples are `--sample` (200) random records plus up to 200 records with entries in `transforms.jsonl` |
| `search_visible` | Probe searches don't return `top_k` results within `--search-timeout` (180 s) |
| `golden_queries` | Tie-adjusted top-k overlap with Chroma's `collection.query` averages below `--min-mean-overlap` (0.9), or any query falls below `--min-query-overlap` (0.6) |
| `self_retrieval` | Never fails. Reports hit@1 and hit@k with a likely reason for each miss (duplicate vector, `dotproduct` with unnormalized vectors, near tie, approximate search, or freshness) |
| `app_filters` | With `--app-filters`, a translated filter returns a different ID set than Chroma's `where`. Fix the app's filter, not the data |

| | |
|---|---|
| Inputs | `--collection`, `--index`, `--namespace`, workdir files, `--golden-queries FILE`, optional `--app-filters`, `--dimension`, `--report`, `--sample`, `--top-k`, `--poll-timeout`, `--search-timeout` |
| Outputs | `validation_report.json` (or `--report PATH`) |
| Exit 0 | Every check passed |
| Exit 1 | Any check failed |
| Exit 2 | Missing workdir files, convert not ok, no golden queries for a non-empty collection, collection can't be opened, index doesn't exist, or `PINECONE_API_KEY` not set |

The overlap thresholds are heuristics; tune them for your data. `--golden-from-sample N` builds golden queries from perturbed stored vectors and is for testing only.

## Testing

```bash
uv run tests/test_convert_unit.py                     # offline conversion rules
PINECONE_API_KEY=... uv run tests/smoke_test.py --scratch /tmp/mig-smoke
```

`smoke_test.py` builds edge-case collections in a local Chroma (`tests/build_fixtures.py`), runs every script with expected exit codes, and creates Pinecone indexes named `chroma-mig-eng-<run>-*`, which it deletes at the end. Run it against a Pinecone project you use for testing.

The toolkit has been tested with Python 3.13, `pinecone==10.2.0`, and `chromadb==1.5.9` against local open-source Chroma and Pinecone serverless indexes. The Chroma Cloud connection (`chromadb.CloudClient()`) uses the same Collection API but has not been run against Chroma Cloud. Rehearse against your own collection before your write pause.

## Known limitations

- One collection per run.
- `export.py` and `validate.py` hold the set of IDs in memory, and listing IDs in Pinecone takes about one request per 100 IDs. Very large collections may need a different approach.
- The toolkit upserts over the API. For very large collections, [import from object storage](https://docs.pinecone.io/guides/index-data/import-data) (Standard and Enterprise plans) may be faster; the toolkit doesn't produce import files.
- Chroma Cloud page-size and rate limits have not been measured. If you see throttling during export, lower `--page-size`.
- A mistyped `--golden-queries` or `--app-filters` path exits 1 instead of 2.
- If the index dimension doesn't match the source, `validate.py` reports the mismatch and then stops without writing a report.

## After you migrate

Once you're on Pinecone, you can move to [integrated embedding](https://docs.pinecone.io/guides/search/integrated-embedding), where a Pinecone-hosted model embeds text at upsert and query time, or add [full-text search](https://docs.pinecone.io/guides/search/full-text-search). Both need a new index and a reload from your stored text.

## Resources

- [Chroma Cloud wind-down notice](https://www.trychroma.com/cloud-wind-down)
- [Pinecone documentation](https://docs.pinecone.io/guides/get-started/overview)
- [Filter by metadata](https://docs.pinecone.io/guides/search/filter-by-metadata)
- [Data modeling](https://docs.pinecone.io/guides/index-data/data-modeling)
- [Pinecone Standard trial](https://docs.pinecone.io/admin/billing/standard-trial)
- [Pricing](https://www.pinecone.io/pricing)
