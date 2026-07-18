# QueryBuilder & Search

## QueryBuilder vs JCR-SQL2 vs XPath

| | QueryBuilder | JCR-SQL2 | XPath |
|---|---|---|---|
| AEM-specific? | Yes — AEM API only | No — JCR standard | No — JCR standard |
| Syntax | Key-value pairs | SQL-like | XPath |
| Joins | No | Yes | Limited |
| Custom predicates | Yes | No | No |
| Pagination | Built-in | Manual `LIMIT`/`OFFSET` | Manual |
| Best for | AEM content queries | Complex joins, migration scripts | Legacy code |

## Essential QueryBuilder Predicates

```java
Map<String, String> params = new LinkedHashMap<>();

// ALWAYS start with type — uses Oak nodetype index
params.put("type",           "cq:Page");

// Scope
params.put("path",           "/content/mysite/en");
params.put("path.exact",     "false");      // search all descendants

// Property filter
params.put("1_property",     "jcr:content/cq:template");
params.put("1_property.value", "/conf/mysite/settings/wcm/templates/article");

// Full-text search
params.put("fulltext",       "AEM performance");
params.put("fulltext.relPath", "jcr:content");  // scope to page content only

// Date range
params.put("daterange.property",  "jcr:content/publishDate");
params.put("daterange.lowerBound", "2024-01-01T00:00:00.000+05:30");
params.put("daterange.upperBound", "2024-12-31T23:59:59.000+05:30");
params.put("daterange.lowerOperation", ">=");
params.put("daterange.upperOperation", "<=");

// Sorting
params.put("orderby",       "@jcr:content/publishDate");
params.put("orderby.sort",  "desc");

// Pagination — ALWAYS set these
params.put("p.limit",       "10");
params.put("p.offset",      "0");

// Performance — NEVER omit on large repos
params.put("p.guessTotal",  "100");  // estimate, avoids full count traversal

Query query = queryBuilder.createQuery(
    PredicateGroup.create(params),
    resourceResolver.adaptTo(Session.class)
);
SearchResult result = query.getResult();
```

## p.guessTotal — Why It's Critical

Without `p.guessTotal`, QueryBuilder traverses **every matching node** to compute an exact total count. On a repo with 100,000 pages, this means 100,000 node reads for every search request.

With `p.guessTotal=100` (or any estimate), QueryBuilder stops counting after that number and returns an approximation. Use `result.getHits()` for actual results and `result.getTotalMatches()` for the (estimated) count.

## Explaining a Query — Debug Tool

Test queries in the Felix console at `/system/console/jmx` → `QueryStat`, or use:
```
/bin/querybuilder.json?type=cq:Page&path=/content&p.limit=10
```

## Common Interview Questions — QueryBuilder

**Q: What is the difference between `p.limit=-1` and a specific limit?**
`p.limit=-1` returns all results with no pagination. **Never use this in production.** On a large repository it loads every matching node into memory and will cause an `OutOfMemoryError`. Always use explicit pagination.

**Q: What does `path.exact=true` do?**
It restricts results to the exact path specified, not its descendants. Equivalent to querying for that single node. Rarely useful in practice; the default (`false`) searches all descendants.

**Q: How do you build an OR query in QueryBuilder?**
Use a `PredicateGroup` with `p.or=true`:
```java
PredicateGroup orGroup = new PredicateGroup();
orGroup.setAllRequired(false); // this makes it OR
orGroup.add(new Predicate("property").set("property", "jcr:content/category").set("value", "tech"));
orGroup.add(new Predicate("property").set("property", "jcr:content/category").set("value", "sport"));
```

**Q: How do you use QueryBuilder in a unit test?**
Mock `QueryBuilder` and `SearchResult` with Mockito, or use the `ResourceResolverMock` from wcm.io test helpers, which includes a basic in-memory query engine.

---

## AEM Search & Indexing — Advanced Deep Dive

This section bridges high-level concepts (layman's terms) with the deep architecture behind AEM search, Apache Oak indexing, Lucene scoring, and faceted search. *(For the full index-internals reference — inverted index structure, indexing pipeline, index types — see `lucene-oak-search-complete.md`.)*

### Lucene Relevance Scoring (TF-IDF)

When AEM executes a full-text search, the underlying engine (Lucene) uses a mathematical formula to rank results — most commonly **TF-IDF**.

A document is considered highly relevant if the search term appears in it *frequently*, but only if that term is relatively *rare* across the entire repository.

- **TF (Term Frequency):** How many times the search term appears in a specific document. Higher frequency → higher score.
- **IDF (Inverse Document Frequency):** How rare the term is across *all* indexed documents. Common words ("the", "page") are penalized; rare words ("Omnichannel") are heavily rewarded.

**Additional scoring factors:**
- **Field Length Normalization (lengthNorm):** Lucene penalizes long fields. A match in a short `jcr:title` scores higher than a match in a massive `jcr:description`.
- **Index-Time Boosts:** Developers can manually boost specific properties in the Oak index (e.g. making a match in `jcr:title` worth 4x more than a match in body text).
- **Coordination Factor (coord):** Rewards documents that contain *multiple* terms from a multi-word search query.

> **Interview tip:** Debug these scores using the Query Builder Debugger (`/libs/cq/search/content/querydebug.html`) — run a query and check "Extract explain plan" to see the exact math Lucene applied.

### Faceted Search — The "Smart Filters"

A facet is a **smart filter**: it categorizes search results *and* mathematically counts how many items match each category before you click it.

*Analogy:* Shopping for laptops online — the sidebar doesn't just say "Brands," it says "Apple (1,500), Asus (500)." Click Asus, and the Color filter instantly updates to remove colors Asus doesn't make — you never hit a dead-end "0 results."

**The technical handshake:**
1. **The request (QueryBuilder):** `1_property=jcr:content/author` + `p.facets=true`
2. **The permission (Oak index):** AEM only calculates this if the underlying Lucene index rule for that property has `propertyIndex=true` and `facets=true` set.

A faceted query response has two blocks: `hits` (the actual results) and `facets` (a dictionary of categories and counts, used to render the sidebar UI).

### Dynamic Facet Recalculation

AEM does **not** pre-calculate facet counts or read JCR nodes during a query to count them. It uses a two-step in-memory process:

1. **The Match:** A filtered query fires (e.g. "Marketing" + "Author: Jane"). Lucene finds the matching internal Document IDs (e.g. 400 pages).
2. **The Tally:** Lucene cross-references those 400 IDs against **DocValues** (an ultra-fast, in-memory columnar structure) to instantly tally counts for other facets, based only on those 400 IDs.

**Security and ACLs — the performance bottleneck:** AEM cannot show a count for a node a user isn't allowed to see, but checking ACLs for every node in a 50,000-result search would crash the server.
- **AEM 6.5 fix:** set `secure=statistical` on the facet index definition — Oak checks a random sample (e.g. 1,000 nodes) and applies that permission ratio to the total count, an accurate estimate without the performance hit.
- **AEMaaCS fix:** search is offloaded entirely to Elasticsearch, which handles these aggregations natively off the AEM JVM.

### The Oak Query Planner — Cost-Based Model

The Query Planner is AEM's "auctioneer," ensuring queries avoid full repository traversal:

1. **Parsing:** breaks the query into core restrictions (e.g. Node Type = `cq:Page`, Property = `jcr:title`).
2. **The Bid:** asks all indexes under `/oak:index` how much it would "cost" them to execute the query (cost = estimated number of nodes the index has to read).
3. **The Award:** the index with the lowest cost wins execution rights.

**The Traversal Nightmare:** if you query an unindexed property, no custom index can bid. The planner falls back to a basic Node Type index (or the root path), bidding a massive cost (e.g. 100,000) — AEM must manually traverse every node, causing a `TraversalWarning` and severe performance degradation.

### The Cardinality Trap & Dictionary Encoding

Setting `facets=true` makes Lucene build a separate in-memory "spreadsheet" (DocValues) mapping Document ID → Property Value. To save space it uses **Dictionary Encoding**: a dictionary of unique text values, plus a pointer array mapping Document IDs to dictionary integer IDs.

- **Low cardinality** (e.g. `status`: Draft/Published — 2 unique values): the dictionary holds 2 strings; the pointer array holds 100,000 tiny integers. Extremely fast and lightweight.
- **High cardinality** (e.g. `jcr:title` on 100,000 pages — 100,000 unique titles): the dictionary must load 100,000 unique, heavy text strings directly into the JVM heap.

**Interview takeaway:** setting `facets=true` on high-cardinality, free-text properties will bloat the JVM heap and eventually cause `OutOfMemoryError` crashes in AEM 6.5. Facets should strictly be used for categorical, low-cardinality data.

### The Inverted Index — Physical Structure

Lucene stores three things on disk:

1. **The Dictionary** — every unique word, sorted alphabetically. Sorted so binary search finds any word in ~20 steps regardless of dictionary size (log₂ of 10,000,000 ≈ 23) — like finding a name in a phone book by repeatedly opening to the middle, never reading it cover to cover.
2. **The Postings List** — for each word, which document IDs contain it (e.g. `"villa" → [doc_45, doc_123, doc_234]`), stored as compact delta-encoded integers.
3. **The Document Store** — stored field values per document, for fields marked `stored=true`, so results can be returned without re-reading JCR.

```
Query: "beachfront" AND "villa"
Step 1: Binary search dictionary → "beachfront" → [doc_45, doc_234, doc_567]
Step 2: Binary search dictionary → "villa"      → [doc_45, doc_123, doc_234]
Step 3: Intersect both lists                    → [doc_45, doc_234]
Total: ~3 operations. Same speed for 10 or 10 million documents.
```

Physical files: `crx-quickstart/repository/index/lucene-1234/` — `_0.cfs` (compound file segment), `_0.si` (segment info), `segments_N` (active segment list), `write.lock`.

### The Indexing Pipeline — How Content Gets INTO the Index

Oak doesn't iterate through all nodes to build the index. It **reacts to change events** — the MVCC/NodeState diff system tracks exactly what changed between commits, so the indexer only ever processes what actually changed:

```
Save in AEM → JCR saves the node (immediate)
    → Oak writes a checkpoint: "node X changed"
    → (~5s later) async indexer wakes up, reads the checkpoint
    → checks which index definitions cover this node type/path
    → Analyzer runs on text fields: tokenise → lowercase → stop-word removal → stem
    → IndexWriter adds tokens to postings, written to a new segment file
    → next query finds the document instantly
```

### Asynchronous Indexing — The Most Important Operational Fact

```
/oak:index/@async = "async"            → runs every ~5 seconds
/oak:index/@async = "fulltext-async"   → separate lane, also ~5 seconds
```

A page saved right now will **not** appear in a Lucene search for up to 5 seconds. Property indexes update synchronously (within the same commit) — always immediately consistent. Monitor lag at `/system/console/jmx` → "Async Indexing Statistics" (`LastIndexedTime`, `IndexingLag`, `FailedIndexCount`).

### Why Searching a Huge Index Isn't Slow

- "Going through millions of dictionary terms must be slow" — No, binary search: 23 comparisons for 10 million terms.
- "Iterating 50,000 matching document IDs must be slow" — No, a modern CPU processes ~1 billion integers/second; merging two sorted lists runs in O(n+m).
- "Oak must check every index definition for each query" — No, the query planner scores each index mathematically using stored statistics and picks exactly one winner — arithmetic, not data access.

```
p.explain=true shows the decision:
GET /bin/querybuilder.json?type=cq:Page&...&p.explain=true
→ "plan": "[propertyListingIndex] cost: 2.3"   ← picked over traversal cost: 500,000
```

### Four Oak Index Types

| Type | How it works | Use for |
|---|---|---|
| `lucene` | Full Lucene inverted index | Full-text search, complex multi-property queries, sorting, facets |
| `property` | B-tree on a single property | Simple exact equality / range queries |
| `nodetype` | Index by `jcr:primaryType`/`jcr:mixinTypes` | Always used first to narrow by node type |
| `counter` | Counts nodes in subtree | Rarely used directly |

### OOTB Indexes

| Index | Covers |
|---|---|
| `lucene` | Default full-text — all node types, all text |
| `cqPageLucene` | `cq:Page` nodes — most page queries use this |
| `damAssetLucene` | DAM assets — asset search in Touch UI |
| `workflowDataLucene` | Workflow instances |

### When to Create a Custom Index

Create one when: your query triggers a Traversal Warning in `error.log`, or you need to sort by a custom JCR property (`ordered=true` required).

**Q: How do you trigger reindexing after deploying a new index definition?**
Set `reindex=Boolean(true)` on the index node in CRXDE. Oak detects this, reindexes all existing content matching the index definition, then sets it back to `false`. For large repositories, use `oak-run.jar`'s offline reindex to avoid pausing the JVM.
