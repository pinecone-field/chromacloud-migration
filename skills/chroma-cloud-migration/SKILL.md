---
name: chroma-cloud-migration
description: Moves a Chroma Cloud collection to a Pinecone serverless index with the chromacloud-migration toolkit, reusing the existing embeddings. Runs preflight, index creation, export, conversion, load, and validation in order and stops at every failed check. Use when the user says "migrate off Chroma Cloud", "move my Chroma collection to Pinecone", "Chroma Cloud is shutting down", "export my Chroma Cloud data", or "replace Chroma with Pinecone". Also works on a local open-source Chroma directory. Not for moving Chroma into Postgres, and not for re-embedding data with a new model.
license: Apache-2.0
compatibility: Needs uv and network access to Pinecone. A Pinecone API key for the index, load, and validation steps. Chroma Cloud credentials (API key, tenant, database) for the real migration; a local rehearsal needs none. Tested with Python 3.13, pinecone 10.2.0, and chromadb 1.5.9.
metadata:
  author: Pinecone
  version: 1.2.0
  repository: https://github.com/pinecone-field/chromacloud-migration
  guide: https://www.pinecone.io/blog/migrate-chroma-cloud-to-pinecone/
---

# Chroma Cloud to Pinecone migration

Chroma Cloud stops accepting writes on November 22, 2026 and stops serving reads on December 22, 2026. This skill moves one collection at a time into a Pinecone document index during a planned write pause. It runs the six scripts in this repository and checks each step's output before going on. The collection's vectors are copied as they are. The app keeps its embedding model.

## Critical rules

Read these before running anything.

1. **The user pauses writes and flips traffic. You never do.** Step 5 and Step 9 change production. Stop before each one, say exactly what the user needs to do, and wait for them to confirm it's done.
2. **A non-zero exit means stop.** Every script exits 0 on success, 1 when a check fails, and 2 on a usage or precondition error. Explain the finding and fix the cause before the next step. The one expected exit 1 is preflight before the user has accepted its TRANSFORM findings (Step 1).
3. **A filter that needs redesign blocks the flip, even on exit 0.** Preflight reports these as the warning `filters_need_redesign`, and `validate.py` marks `app_filters` as `warn`. Both still exit 0. Don't move to Step 9 until the app's query is redesigned and the parity check passes. Some rewrites can't be parity-checked, such as a hand-written Pinecone filter for `$exists`, `$not`, or a `where_document` query. For those, remove or replace the entry in `app_filters.json`, rerun preflight and validate, and have the user test the rewritten query by hand. Otherwise the warning never clears, even after a correct rewrite.
4. **Never patch a failed load.** If a data check fails, fix the cause and reload into a new, empty namespace (for example `my-collection-v2`). An upsert-only pass for missing IDs can't remove records that were deleted in Chroma.
5. **Repeat the policy flags on every preflight and convert run. Carry the field flags into `create_index.py` too.** The policy flags are `--array-policy`, `--big-int-policy`, `--text-mode`, `--vector-field`, `--text-field`, `--id-field`, and `--id-prefix`. The scripts don't compare their flags with each other. A preflight run without them overwrites `preflight.json` with the defaults. The policies in use are recorded under `policies` in `preflight.json`; read them from there before each run. `create_index.py` reads only the dimension and metric from that file. Step 3 has to pass the matching field flags itself.
6. **Keep rehearsal and production apart.** A rehearsal uses its own workdir, index, and namespace. It never touches the production names.
7. **Don't print secrets.** Check that environment variables are set without echoing their values. Use `your-key` as a placeholder in anything you show.

In the commands below, `POLICY_FLAGS` stands for the policy flags chosen in Step 1, such as `--array-policy stringify`. Write them out in full each time. Don't put them in a shell variable: zsh doesn't split an unquoted variable into separate arguments.

## Before you start

Ask the user for anything you don't already know:

- The Chroma collection name, plus any other collections to move later.
- A Pinecone index name and a namespace name. Index names allow lowercase letters, digits, and hyphens only, up to 45 characters. Turn underscores into hyphens: collection `support_docs` becomes index `support-docs` and namespace `support-docs-v1`.
- The cloud and region. AWS `us-east-1` works on every plan. Other regions need Standard or Enterprise.
- Whether the app filters with `where` or `where_document`. If it does, collect the real filters for the parity check (see Step 1).
- The function and model the app uses to embed search queries. Step 8 needs it to build golden queries.
- Where the app's vector-store code lives, for the switch in Step 4.
- Today's date against November 22. After that date Chroma Cloud is already read-only (see "Past November 22" below).

If the user only wants an archive of their data, not a migration, run `export.py` alone (Step 6). It writes `export.jsonl` and a manifest with counts and a checksum, and needs no Pinecone key.

Set up from the repository root. Skip the clone if the repository is already checked out:

```bash
git clone https://github.com/pinecone-field/chromacloud-migration.git
cd chromacloud-migration
uv sync
```

Check credentials without printing them:

```bash
for v in PINECONE_API_KEY CHROMA_API_KEY CHROMA_TENANT CHROMA_DATABASE; do
  printenv "$v" > /dev/null && echo "$v set" || echo "$v MISSING"
done
```

The three `CHROMA_*` variables are needed only when reading from Chroma Cloud. A local rehearsal doesn't use them. `PINECONE_API_KEY` is needed from Step 3 on. If one that's needed is missing, ask the user to `export` it in their terminal, or to put it in a `.env` file and run every script as `uv run --env-file .env <script>`. Don't assume your shell inherited it.

The production commands below use `--workdir migration_work`, `my_collection`, `my-index`, and `my-collection-v1`. Replace those with the user's names.

## Workflow

### Step 1: Preflight

```bash
uv run preflight.py --collection my_collection --workdir migration_work
```

For a local open-source Chroma directory, add `--chroma-path PATH`. Only `preflight.py`, `export.py`, and `validate.py` read Chroma, and only they take that flag.

Preflight reads the whole collection and writes `migration_work/preflight.json`. Findings come in four levels:

- **BLOCKING** findings stop the migration until they're fixed. Some need a fix in Chroma before the write pause: all-zero vectors (`zero_vector`), mixed dimensions (`inconsistent_dimension`), values Pinecone can't store (`unsupported_value_types`). Others have a policy flag instead, listed below. With the flag, preflight reports them as TRANSFORM.
- **TRANSFORM** findings are changes the conversion will make: new IDs, renamed metadata keys, dropped `None` values, and anything a policy flag converts. Explain each one and what the app has to change because of it (see `references/app-changes.md`).
- **WARNING** findings need a decision from the user even though preflight still exits 0. Two matter most. `metadata_near_40kb` means a record is close to the limit. Settle it before Step 3, because the index schema can't change after that. A filter reported as needing redesign has no Pinecone equivalent. The app's query has to change before the cutover.
- **INFO** findings need no action. Mention them briefly.

Choose policies with the user rather than for them:

- `non_string_arrays`: Pinecone lists hold strings only. Either fix the arrays in Chroma or add `--array-policy stringify`, which turns `3` into `"3"`. Range filters on those fields stop working.
- Integers above 2^53: Pinecone would round them. `--big-int-policy stringify` stores them as strings.
- `metadata_over_40kb` from long documents: use the full-text-search variant in Step 3. It moves the text out of the 40 KB metadata limit but adds its own limits: 100 KB and 10,000 tokens per document, and 1,024 bytes per token. A document that's one very long word still fails. Decide now, because the schema can't change later.

If the app uses filters, write them to `app_filters.json` as a JSON list of `{"name": ..., "where": ...}` or `{"name": ..., "where_document": ...}` objects. Add `--app-filters app_filters.json` to the preflight run. It reports which filters translate and which need redesign.

Preflight exits 1 until the user accepts the TRANSFORM findings. That exit is expected. Once they accept, rerun with the chosen flags plus `--allow-transforms`:

```bash
uv run preflight.py --collection my_collection --workdir migration_work POLICY_FLAGS --allow-transforms
```

Exit 0 here is the gate for Step 2.

### Step 2: Rehearse and size the window

Rehearse Steps 3, 6, 7, and 8 before the real run. Keep everything separate from production:

- Workdir `migration_rehearsal`, index `my-index-rehearsal`, namespace `my-collection-rehearsal`.
- Run preflight again in that workdir first, with the same `POLICY_FLAGS` and `--allow-transforms`.
- The source can be a local copy (`--chroma-path PATH`) or a small collection in Chroma Cloud.
- If the app's embedding function isn't available yet, validate with `--golden-from-sample 20` instead of `--golden-queries`. It builds queries from perturbed stored vectors. Use it for rehearsal only, never to approve the real cutover.
- Skip Step 4. The app switch isn't part of a rehearsal.

Time the export and the load. Only a rehearsal that reads from Chroma Cloud sizes the write pause, because network transfer dominates a cloud export. A local export finishes in seconds and says nothing about the real window. Don't estimate a duration from anything else. When the rehearsal is done, ask the user whether to delete the rehearsal index.

### Step 3: Create the index

```bash
uv run create_index.py --index my-index --workdir migration_work
```

It reads only the dimension and metric from `preflight.json`. Map the other policies onto its flags yourself:

- `vector_field` other than `embedding`: pass `--vector-field` with the same name.
- `text_field` other than `text`: pass `--text-field` with the same name.
- `text_mode` of `fts`: pass `--full-text-search`, plus `--language` if the text isn't English.

A mismatch makes `load.py` exit 1 later. Chroma defaults to `l2`, which becomes `euclidean`. Add `--cloud` and `--region` for other locations, and suggest `--deletion-protection enabled` for production. If the index already exists, the script only checks its schema and exits 1 on a mismatch.

For long documents, use the full-text-search variant for preflight, index creation, and conversion. Use one path, not both:

```bash
uv run preflight.py    --collection my_collection --workdir migration_work --text-mode fts POLICY_FLAGS --allow-transforms
uv run create_index.py --index my-index --workdir migration_work --full-text-search   # plus --vector-field / --text-field if not the defaults
uv run convert.py      --workdir migration_work --text-mode fts POLICY_FLAGS
```

### Step 4: Ship the switch, pointed at Chroma

The app needs one config flag that moves reads and writes from Chroma to Pinecone together. Help the user write it using `references/app-changes.md`. It ships now and stays pointed at Chroma until Step 9.

### Step 5: Pause writes (the user does this)

Stop and ask the user to set the app to read-only. Reads stay on Chroma. Once they confirm, rerun preflight with the same flags as Step 1:

```bash
uv run preflight.py --collection my_collection --workdir migration_work POLICY_FLAGS --allow-transforms
```

### Step 6: Export

```bash
uv run export.py --collection my_collection --workdir migration_work
```

Exit 1 almost always means writes are still landing: the count changed during the export. Ask the user to confirm the pause, then rerun. An update that leaves the count unchanged can't be detected by this check. Only the pause prevents that. Keep `export.jsonl` as the archive copy.

### Step 7: Convert and load

```bash
uv run convert.py --workdir migration_work POLICY_FLAGS
uv run load.py    --index my-index --namespace my-collection-v1 --workdir migration_work
```

If any record fails, convert writes no `documents.jsonl` and keeps `transforms.jsonl.failed` for review. `load.py` needs a new or empty namespace and can resume with `--resume` after an interruption.

### Step 8: Validate

Build `golden.jsonl` from real user queries, embedded with the app's own query-embedding function. The toolkit doesn't embed text.

```python
import json
queries = ["how do refunds work", "reset my password"]   # real queries from the app
with open("golden.jsonl", "w") as f:
    for i, q in enumerate(queries):
        emb = my_app_embed(q)   # the app's query-embedding function
        f.write(json.dumps({"query_id": f"q{i}", "embedding": list(map(float, emb))}) + "\n")
```

```bash
uv run validate.py --collection my_collection --index my-index --namespace my-collection-v1 \
                   --workdir migration_work --golden-queries golden.jsonl
```

Add `--app-filters app_filters.json` if the app filters. Read `migration_work/validation_report.json` and report each check to the user by name. If a data check fails, follow critical rule 4. If only `app_filters` fails, the data is fine: fix the app's filter instead of reloading.

### Step 9: Decision gate, then flip (the user does this)

Summarize the validation report in a few lines. If any filter still needs redesign (critical rule 3), say so and don't offer the flip yet. Otherwise ask the user whether to flip. Rollback is still free at this point: the data is in Pinecone, but the app hasn't read or written there yet. If they flip, reads and writes move together through the Step 4 flag, then writes resume. Moving both at once means no write lands in a store the reads can't see.

### Step 10: Watch, then retire Chroma Cloud

After the flip, rolling back means replaying Pinecone writes into Chroma. Have the user watch errors, latency, and result quality for a while. Before December 22, they should remove the Chroma client code, rotate or delete the Chroma Cloud credentials, and archive `export.jsonl` with `field_map.json`, `id_map.jsonl`, and `transforms.jsonl`. All four files are needed to reverse the renames and ID mappings.

## Past November 22

Chroma Cloud is read-only from November 22, which enforces the write pause for you. Skip Step 5 and nothing else. Run Steps 1 through 4 first: preflight and policies, a short rehearsal, the index, and the shipped switch. Then run Steps 6 through 9 before December 22. The app's writes fail until it flips to Pinecone. Say this plainly if the user is starting late, and don't let the deadline justify skipping the index or the switch.

## Several collections

Run the whole workflow once per collection, each with its own workdir. Collections that share a dimension and distance metric can share one index, each in its own namespace.

## Reporting

After each step, tell the user the exit code, the numbers that matter from that step's JSON output (records counted, exported, converted, loaded, and checks passed), and what comes next. Keep each update short. The user is deciding whether to continue, not reading a log.

## Examples

**A small collection with no filters.** The user says "Chroma Cloud is shutting down, move my `support_docs` collection to Pinecone." Preflight reports a 1,536-dimension `cosine` collection with two TRANSFORM findings, both renamed keys (`text` becomes `meta_text`, or a hashed name if `meta_text` is taken; `field_map.json` has the final names). Explain the rename, rerun with `--allow-transforms`, rehearse, create the index `support-docs`, and help the user add the switch. Stop for the write pause. Then export, convert, load into `support-docs-v1`, and validate with golden queries the user generates from their app. Report that every check passed and ask whether to flip.

**Numeric arrays.** Preflight blocks with `non_string_arrays` on 75 records. Ask whether to fix them in Chroma or store them as strings. If the user picks strings, every later preflight and convert run carries `--array-policy stringify`.

**Long documents.** Preflight blocks with `metadata_over_40kb`. Explain that document text counts toward the 40 KB metadata limit by default. Offer the full-text-search variant, which also makes the text keyword-searchable, and say what its own limits are. Rerun preflight with `--text-mode fts`, create the index with `--full-text-search`, and convert with `--text-mode fts`.

**Starting after the cutoff.** On November 25 the user asks for help. Chroma Cloud is already read-only. Tell them the app's writes are failing until the flip. Run Steps 1 through 4 as usual, skip Step 5, then run Steps 6 through 9.

## Troubleshooting

See `references/troubleshooting.md` for each exit code, the common preflight findings, and the validation failures with their usual causes. The repository README has the full script reference.
