# Module 04 — Companion Guide: Memory & the Buffer Pool

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/04-memory-buffer](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/04-memory-buffer) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/04-memory-buffer/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/04-memory-buffer/slides/deck.pptx)

A reading companion to the module video. Skim before, read after.

---

## TL;DR

Modules 01–03 explained how data sits on disk. Module 04 is about what happens between disk and CPU: the **buffer pool**.

- The buffer pool is a chunk of RAM (default: 80% of system memory) where DuckDB caches pages it has already read from disk.
- **Cold read** = first time a page is touched. Goes to disk. Slow.
- **Warm read** = subsequent reads. Served from buffer pool. 10–20× faster.
- For queries too big to buffer, DuckDB **streams** the data — processes one vector (~2,048 rows) at a time, discards it, moves on. Memory stays bounded even when the data doesn't fit.
- Some operations (grouped aggregation with millions of groups, sorting all rows, hash join build side) *can't* stream — they need to materialize state proportional to the input. When that state outgrows memory, DuckDB **spills** to temporary disk storage.

`duckdb_memory()` is your live introspection table — it shows what the buffer pool currently holds, broken down by category (`BASE_TABLE`, `COLUMN_DATA`, `HASH_TABLE`, etc.).

---

## Key Concepts

### The buffer pool
A region of RAM that holds **disk pages** after they've been read. Pages stay pinned until evicted by the LRU policy or manually flushed by `DETACH`. The pool sits between the file system and the query executor, so a page read once becomes free for the next query.

### `duckdb_memory()` — what's in the pool
A table-valued function that lets you see what the buffer pool is holding. Key tags:

| Tag | What it holds |
|---|---|
| `BASE_TABLE` | Raw compressed pages — the disk image in RAM |
| `COLUMN_DATA` | Decoded/decompressed column vectors ready for the executor |
| `HASH_TABLE` | Hash structures used by `HASH_GROUP_BY` and `HASH_JOIN` |
| `TEMPORARY_BUFFER` | Spilled state when memory pressure forces overflow |

Run `FROM duckdb_memory()` before and after a query to see what changed.

### Cold vs warm
The walkthrough's two runs of the same `GROUP BY l_orderkey` query on `lineitem` (60M rows) demonstrate the gap. The first run reads pages off disk; the second run finds them in the buffer pool and runs ~10–20× faster on the same hardware. The query plan is identical — only the I/O cost differs.

### Streaming execution
DuckDB doesn't load whole tables into memory. It pulls **vectors** (default 2,048 rows) up the query plan, processes each, and discards. A simple `SELECT col * 10 FROM huge_table` works at any memory budget because no operator needs to keep the whole dataset alive.

This is why `SET memory_limit = '64MB'` in the walkthrough still completes a projection over 60M rows — only one vector is alive at a time.

### When streaming breaks: pipeline breakers
Some operators *must* see all input before producing output:

- **`HASH_GROUP_BY` with many groups** — the hash table grows with the distinct-key count.
- **`HASH_JOIN` build side** — the build hash table holds the entire smaller table.
- **`ORDER_BY` (full sort)** — must materialize all rows before emitting any.

These are **pipeline breakers** (Module 09 is named after them). When their state outgrows `memory_limit`, DuckDB spills to a `temporary_directory` on disk.

### `memory_limit` and spilling
`SET memory_limit = '4GB'` caps total in-memory state. Beyond it, DuckDB writes intermediate results to disk and reads them back. Spilling is graceful — queries finish; they just slow down. The walkthrough's tight `64MB` limit triggers this for grouped aggregations with many distinct keys.

---

## Walkthrough Recap

The walkthrough attaches `tpch-sf10.db` (15M orders, 60M lineitems) and:

1. **Block 1** — checks `duckdb_memory()` before any query (almost empty)
2. **Block 2** — runs `GROUP BY l_orderkey` *cold*, times it
3. **Block 3** — checks `duckdb_memory()` again — `BASE_TABLE` and `COLUMN_DATA` now show GBs of cached pages
4. **Block 4** — re-runs the same query *warm*, ~10× faster
5. **Block 6** — sets `memory_limit = '64MB'` and runs simple streaming queries that still complete
6. **Block 7** — same tight limit, runs a grouped aggregation that triggers spilling

The progression: see the pool fill → see the speedup → see the streaming bypass → see the spill threshold.

---

## Exercise Hints

If you got stuck on `04-exercise.md`:

- **Cold-vs-warm timing** — the *first* time matters. If you've already touched the column in any query, subsequent runs are warm. Use `DETACH` + `ATTACH` to flush the pool between tests.
- **`duckdb_memory()` snapshots** — run it before, run it after, diff. Look at `memory_usage_bytes` per tag, not just whether the row exists.
- **Triggering a spill** — set `memory_limit` to a size smaller than the table's hash-table-of-distinct-values, then run a grouped aggregation. Watch `temporary_storage_bytes` go up.
- **Streaming evidence** — a query that finishes successfully with `memory_limit = '64MB'` on a 60M-row table is, by definition, streaming. There's no other way it could fit.

---

## Common Misconceptions

**"DuckDB loads tables into memory like SQLite's `:memory:` mode."** No. DuckDB reads pages *on demand* and caches them in the buffer pool. Tables can be far larger than RAM; queries still work via streaming + spilling.

**"`memory_limit` is the database size limit."** It's the *working memory* limit — how much DuckDB can use for hash tables, sort buffers, and the page cache combined. The on-disk database can be 100× the `memory_limit`.

**"If the buffer pool is 80% of RAM, DuckDB will fight my application for memory."** The pool grows lazily up to its cap as queries demand. It will release pages under memory pressure. For a Python/Node app embedding DuckDB, set `memory_limit` explicitly (e.g., `'4GB'`) to bound it predictably.

**"Spilling means my query failed."** Spilling means the query *succeeded* by trading memory for disk I/O. The query is slower but still correct. The error you actually want to avoid is `Out of Memory Error`, which fires if `memory_limit` is too tight even with spilling.

**"Streaming and the buffer pool are the same thing."** They're complementary. Streaming = vectors flowing up the plan, lifecycle = single query. Buffer pool = pages cached across queries, lifecycle = process. Streaming reduces peak memory inside one query; the pool reduces I/O across queries.

---

## Where This Connects

- **Module 03 — Columnar Storage:** the pages the buffer pool caches are row-group/column-segment-shaped — the layout you saw on disk.
- **Module 05 — Grouped Aggregation:** the `HASH_GROUP_BY` operator whose memory pressure triggered spilling here gets its full internal treatment.
- **Module 06 — Sorting Large Tables:** the other big spill-prone operator. External sort algorithms specifically designed for spill-friendly behavior.
- **Module 09 — Query Execution Pipelines:** the formal name for "operators that must see all input before emitting" — the pipeline breakers that defeat streaming.

---

## Further Reading

- [DuckDB memory management docs](https://duckdb.org/docs/configuration/pragmas#memory-related-settings) — `memory_limit`, `temp_directory`, `max_temp_directory_size`
- [`duckdb_memory()` reference](https://duckdb.org/docs/sql/duckdb_table_functions#duckdb_memory) — full column list
- Hannes Mühleisen, *Data Management for Data Science* lectures (CWI) — the classical buffer pool, LRU eviction, the ARC algorithm, why analytical engines often skip LRU entirely
- Daniel Abadi, [The Design and Implementation of Modern Column-Oriented Database Systems](https://stratos.seas.harvard.edu/sites/scholar.harvard.edu/files/stratos/files/columnstoresfntdbs.pdf) — chapter on buffer management for column stores
