# Module 08 — Companion Guide: ART Indexes

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/08-art](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/08-art) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/08-art/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/08-art/slides/deck.pptx)

A reading companion to the module video. Skim before, read after.

---

## TL;DR

When zonemaps (Module 07) are too wide and your queries are still selective, you reach for an index. DuckDB's primary index type is the **ART** — Adaptive Radix Tree.

The hook for the module:

```
WHERE o_clerk = 'Clerk#000051887'   -- 1,518 matching rows of 15M
With ART index:    0.004s   ← fast
Without ART index: 0.109s   ← 27× slower
```

But ART has a **duplicate trap**: when a key matches *many* rows, an Index Scan produces a long list of row IDs and DuckDB has to do many random reads across column chunks. A Sequential Scan reads chunks linearly with a CPU-friendly access pattern. For low-selectivity predicates, sequential wins.

DuckDB picks Index Scan only when it expects **few matches**:

```
threshold = MAX(index_scan_max_count, index_scan_percentage × row_count)
defaults: max_count = 2048, percentage = 0.001   → up to 0.1% of rows
```

So for a 15M-row table, the threshold is ~15,000 rows. Below that, Index Scan. Above, Sequential Scan.

What ART *doesn't* help with: `ORDER BY`, `min`/`max` aggregates, `LIKE`, and `AND`-combined predicates. These are known gaps — the plans simply don't use the index.

---

## Key Concepts

### ART = Adaptive Radix Tree
A trie variant that adapts node sizes to the local key distribution. Where a B-tree has fixed-fan-out nodes, ART uses 4 different node types (Node4, Node16, Node48, Node256) and picks the smallest one that fits the actual children. This makes it cache-friendly across diverse key spaces — sparse keys use small nodes; dense keys use large nodes.

For DuckDB, ART maps a key (e.g., `o_clerk = 'Clerk#000051887'`) to a list of **row IDs** in the table. Lookups are O(k) where k = key length in bytes, regardless of table size.

### The threshold rule
DuckDB only chooses Index Scan when it estimates the result will fit under:

```
threshold = MAX(
    current_setting('index_scan_max_count'),
    current_setting('index_scan_percentage') * (SELECT count(*) FROM table)
)
```

Defaults: `max_count = 2048`, `percentage = 0.001` (0.1%). Above the threshold, DuckDB falls back to Sequential Scan.

You can change these to force a choice for benchmarking:
```sql
SET index_scan_max_count = 0;
SET index_scan_percentage = 0.0;   -- always Sequential Scan
```

### The duplicate trap
This is the counterintuitive failure mode. ART lookup returns a **list of row IDs**. With few matches (small list), random access into the column data is cheap — a few I/Os, mostly cache hits. With many matches (long list), random access becomes the bottleneck — each row ID is a random jump into a column chunk, and you blow the cache.

A Sequential Scan, in contrast, reads chunks linearly. The CPU prefetcher loves linear access; SIMD loads work on contiguous data. Even though Sequential Scan reads *every* row, it does so so efficiently that for low-selectivity queries it beats Index Scan.

The walkthrough demonstrates this directly: with 40K duplicates per key, the ART index actually makes the query slower.

### Index maintenance is expensive
- **`CREATE INDEX` is not free.** DuckDB walks every row and builds the trie. For a 15M-row table this can take seconds.
- **Every `INSERT` and `UPDATE` on an indexed column requires ART maintenance.** Inserts find the right path; updates remove the old key and insert the new one. Both involve pointer juggling.
- **Bulk updates against an indexed column should drop the index first**, then recreate after. The drop+recreate is often faster than incremental maintenance for thousands of changes.

### What ART *doesn't* help with
This is the part most people miss:

| Operation | ART helps? | Why not |
|---|---|---|
| `WHERE col = X` (selective) | ✅ Yes | Direct lookup |
| `WHERE col BETWEEN X AND Y` | ✅ If selective | Range scan in trie |
| `ORDER BY col` | ❌ No | DuckDB doesn't walk leaves in order |
| `min(col)` / `max(col)` | ❌ No | DuckDB doesn't go to leftmost/rightmost leaf |
| `WHERE col LIKE 'prefix%'` | ❌ No | Rewritten as conjunctions internally |
| `WHERE col1 = X AND col2 = Y` (two indexes) | ❌ No | DuckDB doesn't intersect two index results |

These are known gaps in DuckDB's plan generation, not bugs. The plans just don't exist yet (as of late 2025). For these patterns, fall back to clever schema design or sorted layouts.

---

## Walkthrough Recap

The walkthrough attaches `tpch-sf10.db` (15M orders) and:

1. **Block 1** — creates an ART index on `o_clerk`
2. **Block 2** — checks the index's memory footprint and threshold
3. **Block 3** — runs a selective query (1,518 matching rows) → Index Scan fires, ~0.004s
4. **Block 4** — disables index (`SET index_scan_max_count = 0; SET index_scan_percentage = 0.0;`) and re-runs → Sequential Scan, ~0.109s. Compare.
5. Later blocks demonstrate the duplicate trap and `UPDATE` cost

---

## Exercise Hints

If you got stuck on `04-exercise.md`:

- **Selective query** — `o_clerk = 'Clerk#000051887'` matches ~1,500 rows out of 15M. Way under the 15K threshold → Index Scan.
- **`LIKE` predicate** — `o_clerk LIKE 'Clerk#00005188%'` doesn't use the index even with the explicit prefix. DuckDB rewrites internally and falls back to Sequential Scan.
- **Compound predicate** — `o_clerk = X AND o_orderkey < Y`. DuckDB will use *one* of the two indexes (whichever is more selective for its predicate), then filter the result. It does not intersect two index results.
- **`ORDER BY`** — never uses the index. Always materializes via the sort operators from Module 06 (`ORDER_BY` or `TOP_N`).
- **The duplicate trap demo** — to reproduce, build a column with 40K+ duplicates per value, then watch Index Scan get *slower* than Sequential Scan despite matching the threshold rule.

---

## Common Misconceptions

**"More indexes = faster queries."** Each index costs RAM (the trie itself, ~3 bytes/row in this scenario) and slows down `INSERT`/`UPDATE`/`DELETE` on the indexed column. Two indexes used by the same query don't get intersected. Add indexes deliberately for the queries that actually need them.

**"Index Scan is always faster than Sequential Scan."** False — and the walkthrough demonstrates the inverse. The duplicate trap is real. The threshold rule exists precisely because DuckDB knows Index Scan can lose.

**"ART is just like a B-tree."** Same outer interface (`CREATE INDEX`, lookup, range scan) but very different internals. ART's adaptive node sizes make it consistently faster than B-trees in dense-key regions. For DuckDB, the practical difference is mostly memory footprint — ART is more compact.

**"`CREATE INDEX` is instant."** It's a full table scan + tree build. For a 15M-row table on a single column, expect single-digit seconds. For multi-billion-row tables, expect minutes.

**"If `EXPLAIN` shows `INDEX_SCAN`, the index is being used."** Yes, but check `EXPLAIN ANALYZE` to see if it's actually fast. Index Scan can fire and still be slower than Sequential Scan would have been. The optimizer is using cost estimates; the estimates can be wrong.

---

## Where This Connects

- **Module 02 — Reading Plans:** look for `INDEX_SCAN` vs `SEQ_SCAN` in your `EXPLAIN` output.
- **Module 04 — Memory Buffer:** ART nodes themselves live in the buffer pool; their pages can be evicted under pressure.
- **Module 07 — Zonemaps:** the first thing to try for selective predicates. ART comes in only when zonemaps aren't enough.
- **Module 09 — Pipelines:** Index Scan is a pipeline source operator just like Sequential Scan, with one critical difference — it can produce a tiny row stream from a tiny lookup, where SEQ_SCAN at minimum opens row groups.

---

## Further Reading

- [Leis, Kemper, Neumann — *The Adaptive Radix Tree*](https://db.in.tum.de/~leis/papers/ART.pdf) — the original 2013 paper, very readable
- [DuckDB blog: Indexes](https://duckdb.org/2024/06/01/indexes.html) — DuckDB-specific implementation notes
- [DuckDB docs: indexes](https://duckdb.org/docs/sql/indexes) — official reference
- [Andy Pavlo's CMU lecture on tree indexing](https://15445.courses.cs.cmu.edu/) — wider context: B-tree vs B+tree vs ART, when each shines
