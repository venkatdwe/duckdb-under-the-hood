# Module 05 — Companion Guide: Grouped Aggregation

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/05-grouped-aggregation](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/05-grouped-aggregation) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/05-grouped-aggregation/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/05-grouped-aggregation/slides/deck.pptx)

A reading companion to the module video. Skim before, read after.

---

## TL;DR

`GROUP BY` is one of the most expensive operators in analytics. DuckDB has two fast paths:

- **`PERFECT_HASH_GROUP_BY`** — when column statistics show the grouping key has few distinct values, DuckDB skips the hash function entirely and uses a **direct-addressed table** (one slot per possible value). No hashing, no probing, no collision handling. `GROUP BY l_returnflag` (3 distinct values) compiles to this.
- **`HASH_GROUP_BY`** — the general case. Builds a real hash table, one entry per distinct key. `GROUP BY l_orderkey` over 60M lineitem rows builds a ~15M-entry table.

Both store their state on **paged data structures** that share the memory manager with the buffer pool. When RAM gets tight, the memory manager spills cold pages to disk *transparently* — the aggregation operator never sees the spill happen.

In parallel execution (Module 09), DuckDB runs **two-phase external aggregation**: each thread pre-aggregates its morsel into a thread-local hash table, then threads coordinate via 12-bit partition keys to combine results. This is what lets `GROUP BY` scale linearly with cores.

---

## Key Concepts

### `HASH_GROUP_BY` — the general aggregator
For each input row, DuckDB:
1. Computes a hash of the grouping key.
2. Looks up that hash in a hash table; on collision, **linear-probes** to the next slot.
3. Updates the running aggregate (sum, count, min, max, …) at that slot.

After all input rows are consumed, the operator emits one row per occupied slot. The hash function uses the slide deck's "salt (16) · partition (12) · index (17)" split — the bits aren't random; partition bits enable parallel partition-wise aggregation.

### `PERFECT_HASH_GROUP_BY` — when the cardinality is tiny
If the optimizer knows the grouping key has only N distinct values (column stats: range of an integer, enum-like string), it allocates a flat array of size N and uses the value itself as the index. No hashing, no collisions, no probing. The plan operator is named differently so you can spot the optimization in `EXPLAIN`.

Triggers: `GROUP BY` on small-domain columns (status flags, types, boolean-ish ints). For `lineitem.l_returnflag` ('A', 'N', 'R'), DuckDB knows the cardinality is 3 and uses an array of size 3.

### Paged data structures — spill is free
Hash table entries don't live in a contiguous block. They live in **paged data** — 32KB-256KB pages managed by the same memory manager that runs the buffer pool. Variable-length values (strings, lists) live in separate **Variable Data Pages** referenced from the main hash table.

The win: the aggregation operator just asks for pages and writes to them. The memory manager handles eviction. When `memory_limit` is tight, cold hash pages spill to disk and get faulted back as needed. The operator code doesn't have a "spill mode" — spilling is structural to the storage layer.

### Two-phase external aggregation
Single-threaded `HASH_GROUP_BY` is rare. With multiple threads:

**Phase 1 — thread-local pre-aggregation.**
Each thread takes a morsel of input rows and builds its own private hash table. Within Phase 1, threads partition their entries by 12 bits of the hash (4,096 partitions). Each thread's output is 4,096 partial aggregations.

**Phase 2 — partition-wise combine.**
DuckDB redistributes the 4,096 × N (threads) partial aggregations so that each partition is owned by exactly one thread for the final combine. That thread runs a smaller, single-partition `HASH_GROUP_BY` to merge the partial sums.

Two phases means: no shared hash table contention, parallelism scales linearly, spilling can happen per-partition (cold partitions go to disk while hot ones stay in RAM).

---

## Walkthrough Recap

The walkthrough attaches `tpch-sf10.db` and runs:

```sql
-- Block 2: tiny cardinality → PERFECT_HASH_GROUP_BY
EXPLAIN SELECT l_returnflag, count(*) FROM lineitem GROUP BY l_returnflag;
-- → operator: PERFECT_HASH_GROUP_BY

-- Block 3: 15M distinct keys → HASH_GROUP_BY
EXPLAIN SELECT l_orderkey, count(*) FROM lineitem GROUP BY l_orderkey;
-- → operator: HASH_GROUP_BY

-- Block 4: see the cost
FROM duckdb_memory();
-- → HASH_TABLE shows hundreds of MB

-- Block 5: tight memory_limit forces spill
SET memory_limit = '64MB';
SELECT l_orderkey, count(*) FROM lineitem GROUP BY l_orderkey;
-- → query completes, but TEMPORARY_BUFFER lights up (spilled pages)
```

The point is to see all three behaviors back-to-back: perfect path, general path, spill path. Same operator name in plan output for the latter two — only the memory tells you which one is happening.

---

## Exercise Hints

If you got stuck on `04-exercise.md`:

- **Predict the spill** — the rule of thumb: a hash table needs roughly *(distinct keys) × (per-entry size, ~40 bytes typical)* of RAM. 15M keys × 40B ≈ 600MB. With `memory_limit = 64MB`, that has to spill.
- **count(*) is special.** A single-row aggregate doesn't need a hash table — DuckDB streams a single accumulator. No spill possible.
- **Composite keys** — `GROUP BY (l_orderkey, l_linenumber)` widens the key but not necessarily the cardinality. lineitem's `(l_orderkey, l_linenumber)` is the primary key — exactly 60M distinct values. That's worse than `l_orderkey` alone (~15M).
- **Verify with `duckdb_memory()`** — look for `TEMPORARY_BUFFER` after a tight-memory run. Non-zero means spilling happened.

---

## Common Misconceptions

**"`HASH_GROUP_BY` always spills with low memory."** Only when the hash table itself can't fit. Streaming aggregates (no `GROUP BY`, or `GROUP BY` with few groups) don't build a hash table large enough to spill.

**"`PERFECT_HASH_GROUP_BY` is just a faster `HASH_GROUP_BY`."** It's a structurally different operator. No hash function, no probing, no collisions — just direct array indexing. Different code path, different runtime characteristics.

**"Spilling is a bug to fix with more memory."** It's a feature. Without it, queries that exceed RAM would fail outright. Spilling lets DuckDB run aggregations bigger than memory by trading wall-clock time for I/O. Setting `memory_limit` larger only helps if your data actually fits.

**"More threads = faster `GROUP BY`."** Up to a point. Phase 1 scales linearly with cores. Phase 2 is bottlenecked by the slowest partition (data skew). For `GROUP BY` with one extremely hot key, more threads can hurt because contention on that partition dominates.

**"Hash tables are tied to a specific RAM region."** The hash entries live on paged storage that the memory manager moves between RAM and disk freely. There's no "hash table memory" — there's "memory manager pages, some of which happen to hold hash entries."

---

## Where This Connects

- **Module 02 — Reading Plans:** the `HASH_GROUP_BY` and `PERFECT_HASH_GROUP_BY` operator names you see in `EXPLAIN`.
- **Module 04 — Memory Buffer:** the same memory manager that backs the buffer pool also backs hash table pages. Same eviction logic.
- **Module 06 — Sorting Large Tables:** the *other* big spill-prone operator. Same external-spill story but in sorted form.
- **Module 09 — Pipelines:** `HASH_GROUP_BY` is a **pipeline breaker**. The two-phase external aggregation you read about here is what runs inside one pipeline; the breaker boundary is at the start of phase 2.

---

## Further Reading

- [Mark Raasveldt and Hannes Mühleisen, *Data Management for Data Science*](https://www.cwi.nl/en/groups/database-architectures/) — the CWI lectures cover hash table design end-to-end
- [Goetz Graefe, *Query Evaluation Techniques for Large Databases*](https://dl.acm.org/doi/10.1145/152610.152611) — the canonical 1993 survey, still the best treatment of two-phase external aggregation
- [DuckDB blog: Aggregate Function Internals](https://duckdb.org/2022/03/07/aggregate-hashtable.html) — Mark Raasveldt walks through the actual data structure
- [`duckdb_memory()` reference](https://duckdb.org/docs/sql/duckdb_table_functions#duckdb_memory) — see HASH_TABLE and TEMPORARY_BUFFER tags
