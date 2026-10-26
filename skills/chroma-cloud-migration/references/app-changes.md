# App changes for the Pinecone side

Load this file at Step 4 of the workflow, when helping the user write the switch that moves reads and writes from Chroma to Pinecone. Read `migration_work/preflight.json` and, after conversion, `field_map.json` and `id_map.jsonl` first. Those files say which names changed for this collection.

## The switch

One config flag selects the store for both reads and writes. Ship it pointed at Chroma. Flip it only at Step 9 of the workflow, with the user's go-ahead.

```python
import os
USE_PINECONE = os.environ.get("VECTOR_STORE") == "pinecone"
```

## Reading and writing with Pinecone

The index is a document index. Use the Documents API in Python SDK 10 or later. The classic `index.query` and `index.upsert` calls are refused on a document index.

```python
from pinecone import Pinecone

pc = Pinecone()                      # reads PINECONE_API_KEY
index = pc.index(name="my-index")
NAMESPACE = "my-collection-v1"

# Search, the replacement for collection.query(query_embeddings=[vec], n_results=10, where=...)
resp = index.documents.search(
    namespace=NAMESPACE,
    top_k=10,
    score_by=[{"type": "dense_vector", "fields": ["embedding"], "values": query_vector}],
    filter={"category": {"$eq": "billing"}},   # optional, translated from the Chroma where filter
    include_fields=["text", "category"],
)
for match in resp.matches:
    print(match._id, match._score)

# Write, the replacement for collection.upsert / collection.add
index.documents.upsert(
    namespace=NAMESPACE,
    documents=[{"_id": "doc-1", "embedding": vector, "text": "...", "category": "billing"}],
)
```

Use the field names from `field_map.json`. The defaults are `embedding` for the vector and `text` for the document. A metadata key that collided with a reserved name was renamed (for example `meta_text_982d9e3e`).

## What the app has to handle

- **IDs.** Chroma IDs that Pinecone can't accept got new IDs, listed in `id_map.jsonl`. Most look like `chroma-sha256-<hash>`, but one gets a `-2` suffix when the plain form is already taken. Never rebuild an ID by hashing. Always look it up in `id_map.jsonl`. Each remapped document also stores the original ID in the `chroma_id` field. Anywhere the app reads, updates, or deletes by ID, it must use the new ID or look it up through that field.
- **Renamed metadata keys.** Update filters and reads to the names in `field_map.json`. Watch for keys that collided with a reserved name. If the collection already had a metadata key named `chroma_id`, `text`, or `embedding`, it was renamed (for example to `meta_chroma_id`). The reserved name now means something else, with no error to warn you.
- **Filters.** The toolkit translates `where` filters that use `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$and`, and `$or`, and checks them in the parity check. It does not translate `$exists` or `$not`, and reports them as needing redesign. Pinecone has both operators. The app can rewrite those filters by hand, but the parity check won't verify them. Take the original entry out of `app_filters.json` (or replace it with one the toolkit translates) and test the rewritten query yourself. `$contains` on an array key becomes `{"$in": [value]}`, and `$not_contains` becomes `{"$nin": [value]}`. `$contains` on a non-array key, and any `where_document` substring or regex filter, has no equivalent. Those queries need redesign. Pinecone's text-match operators work only on a full-text-search field, which exists only if the index was created with `--full-text-search`.
- **Stringified arrays.** With `--array-policy stringify`, numbers in arrays became strings. Range filters on those fields no longer work.
- **Numbers.** Pinecone stores numbers as 64-bit floats. An integer reads back as a float (1 becomes 1.0). Compare accordingly.
- **Score thresholds.** For `l2` collections, leave thresholds as they are. Chroma's `l2` distance and Pinecone's `euclidean` score are the same number, squared L2, where lower is closer. For `cosine` and `ip`, Pinecone returns a similarity score equal to 1 minus Chroma's distance, where higher is closer. A distance threshold `d` becomes a score threshold `1 - d`, with the comparison flipped: "distance below 0.3" becomes "score above 0.7".
- **Freshness.** A new write can take several seconds to show up in search, even after it's acknowledged. Code that writes and immediately reads back by search should allow for that.
