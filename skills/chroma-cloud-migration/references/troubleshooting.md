# Troubleshooting

Load this file when a script exits with anything other than 0.

## Exit codes

| Code | Meaning | What to do |
|---|---|---|
| 0 | Success | Go to the next step |
| 1 | A check failed, or preflight found a blocking issue | Read the finding, explain it, fix the cause, rerun the same step |
| 2 | Usage or precondition error | Check the collection and index names, `PINECONE_API_KEY`, and that the earlier steps' files are in the workdir |

## Preflight findings

**`zero_vector`.** Chroma accepts them and Pinecone rejects them. Fix or remove those records in Chroma before the write pause.

**`inconsistent_dimension`.** Records in one collection have vectors of different lengths. The collection needs cleanup in Chroma first. Usually that means re-embedding the outliers with the same model as the rest.

**`duplicate_ids_in_scan`.** The same ID came back twice while scanning the collection. Writes may still be landing, or the collection changed mid-scan. Confirm writes are paused and rerun preflight.

**`metadata_over_40kb`.** Document text counts toward the 40 KB metadata limit by default. Switch to the full-text-search variant (`--text-mode fts`, `--full-text-search`) or split the text in the source. Decide before creating the index. In full-text-search mode, each document must stay within 100 KB, 10,000 tokens, and 1,024 bytes per token. Unspaced CJK text can tokenize as one long token and hit the per-token limit. Split it in the source.

**`non_string_arrays`.** Pinecone lists hold strings only. Fix the data, or use `--array-policy stringify` on both preflight and convert.

**`unsupported_value_types`.** A metadata value has a type Pinecone can't store. Fix it in Chroma before the pause.

**`text_over_100kb`, `text_over_10000_tokens`, `text_token_over_1024_bytes`.** Full-text-search mode limits, covered under `metadata_over_40kb` above. Split the text in the source.

**`metadata_near_40kb`, `text_near_10k_tokens` (warnings).** Records close to a limit. Nothing to do now, but new records written after the migration could cross it.

**TRANSFORM findings (`invalid_ids`, `renamed_keys`, `none_values`) with exit 1.** Expected. Preflight exits 1 until the user accepts the transforms. Rerun with `--allow-transforms` once they have.

**Exit 2: collection can't be opened.** Check the collection name and the three `CHROMA_*` variables. For a local copy, check the `--chroma-path` directory.

## Export

**Exit 1.** The count changed during the export, or an ID repeated. Writes are almost certainly not paused. Confirm the pause with the user and rerun. If Chroma Cloud throttles the export, lower `--page-size`.

## Load

**Exit 2: namespace not empty.** `load.py` only writes to a new or empty namespace, unless `load_state.json` shows the same index, namespace, and `documents.jsonl` checksum. Use a new namespace name, such as `my-collection-v2`.

**Exit 1: dimension, metric, or text field mismatch.** The index doesn't match the converted data. This usually means the index was created before a policy or text-mode change. Create a new index from the current `preflight.json`.

## Validation

**`namespace_count` or `id_set` fails.** Records are missing or extra. Don't patch with an upsert. Find the cause (often writes that landed during export), then reload into a fresh namespace and validate again.

**`metric_and_dimension` fails.** The index metric doesn't match the collection's space. With normalized vectors, `l2` and `cosine` rank results identically. Search results can look right on the wrong metric. Trust this check. Recreate the index with the right metric.

**`golden_queries` overlap is low.** The usual causes are a metric mismatch, or golden queries embedded with a different model than the collection used. Confirm the app's embedding function and model, then rebuild `golden.jsonl`. The 0.9 mean and 0.6 per-query thresholds are heuristics. Tune them with `--min-mean-overlap` and `--min-query-overlap` only after ruling out those two causes.

**`search_visible` times out.** Search can lag a load by several seconds. The validator waits up to 180 seconds (`--search-timeout`). Rerun once before treating it as a real failure.

**`app_filters` is `warn`.** At least one filter needs redesign and wasn't compared. The run still exits 0, but don't flip until the app's query is rewritten (SKILL.md critical rule 3).

**`app_filters` fails, everything else passes.** The data is fine. The translated filter returns different IDs than Chroma's. Redesign that filter in the app and don't reload.

**`self_retrieval` misses.** A diagnostic that never fails the run. Common reasons: duplicate vectors, `dotproduct` with unnormalized vectors, near ties, approximate search, or freshness.

## Known limits

- One collection per run.
- `export.py` and `validate.py` keep the set of IDs in memory. Listing IDs in Pinecone takes about one request per 100 IDs. Very large collections may need a different approach, such as Pinecone's import from object storage on Standard and Enterprise.
- The Chroma Cloud connection uses the same Collection API as local Chroma but hasn't been run against Chroma Cloud by the toolkit's authors. That's one more reason to rehearse.
