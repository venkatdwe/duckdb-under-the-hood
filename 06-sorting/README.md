# Module 06 — Companion Guide: Sorting Large Tables

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/06-sorting](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/06-sorting) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/06-sorting/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/06-sorting/slides/deck.pptx)

A reading companion to the module video. Skim before, read after.

---

## TL;DR

The hook for this module: same query, same data, different operator → 1000× difference.

```sql
SELECT l_orderkey FROM lineitem ORDER BY l_orderkey;            -- 3.2s
SELECT l_orderkey FROM lineitem ORDER BY l_orderkey LIMIT 10;   -- 0.06s
```

The reason: adding `LIMIT 10` switches the executor from a full **`ORDER_BY`** (must sort all 60M rows before emitting any) to a **`TOP_N`** (maintains a heap of 10 rows; never materializes the full sorted output).

For the unbounded sort case, DuckDB picks an algorithm based on the key type:
- **Fixed-width keys** (int, float, date) → **radix sort** — bucketed by byte values, no element-to-element comparisons. Cost is O(n·k) where k = key width in bytes.
- **Variable-width keys** (varchar, blob) → **comparison-based sort** — falls back to a comparator-driven algorithm.

Both run as **two-phase external sort**: each thread sorts its morsel locally, then a k-way merge combines all local runs. Sorted runs live on paged data, so the buffer pool handles spilling transparently — same story as `HASH_GROUP_BY` from Module 05.

---

## Key Concepts

### `ORDER_BY` vs `TOP_N` — the LIMIT-aware optimization
DuckDB rewrites `ORDER BY ... LIMIT N` (where N is a small literal) into the `TOP_N` operator. The semantics are identical to "sort then take first N", but the implementation is fundamentally different:

| | `ORDER_BY` | `TOP_N` |
|---|---|---|
| State | All input rows materialized | Heap of N rows |
| Memory | O(input size) | O(N) |
| Spill risk | High (full sort needs it all) | Zero for small N |
| Time complexity | O(n log n) (or O(n·k) for radix) | O(n log N) |

`EXPLAIN` shows the difference: top of the plan reads `ORDER_BY` for unbounded sorts and `TOP_N` for bounded ones. **Always look** when you write `ORDER BY ... LIMIT`.

### Radix sort — no comparisons
For fixed-width sort keys, DuckDB encodes the key as a binary-comparable byte sequence and runs **radix sort** in passes — once per byte of the key.

`l_orderkey` is `int64` — 8 bytes — so 8 passes over 60M rows. Each pass distributes rows into 256 buckets based on the current byte. Cost is **O(n·k)** with k = 8 here. This beats comparison-based O(n log n) for large n.

The byte-encoding trick: signed integers, floats, and dates all get transformed so that lexicographic byte comparison preserves the sort order. Once encoded, the algorithm is identical regardless of source type.

### Two-phase external sort
Same shape as `HASH_GROUP_BY` from Module 05:

**Phase 1 — local sorted runs.**
Each thread takes morsels (~120K-row chunks) and sorts them into local sorted runs. No coordination between threads. If a thread's run grows too large, the buffer pool spills cold pages.

**Phase 2 — k-way merge.**
A single coordinator thread merges all local runs into the final sorted stream. K-way merge keeps a small min-heap of "next row from each run"; emits the smallest, advances that run, repeats. This is the classic external merge sort pattern.

The benefit of two phases: Phase 1 parallelizes perfectly across cores; only Phase 2 is sequential, and even then it's a streaming merge — no random I/O.

### Spilling sort runs is invisible
Sorted runs live on **paged data**, the same memory-managed storage that backs `HASH_GROUP_BY` and the buffer pool. The sort operator never calls a `spill()` function. Cold pages of cold runs migrate to disk under memory pressure; the operator just keeps reading and writing pages, oblivious to which ones are RAM-resident.

`duckdb_memory()` shows `temporary_storage_bytes` lighting up when a sort spills. That's how you tell.

---

## Walkthrough Recap

The walkthrough attaches `tpch-sf10.db` (60M lineitems) and demonstrates the operator switch:

```sql
-- Block 2: full sort → ORDER_BY operator
EXPLAIN SELECT l_orderkey FROM lineitem ORDER BY l_orderkey;
-- → top of plan: ORDER_BY

-- Block 3: with LIMIT → TOP_N operator
EXPLAIN SELECT l_orderkey FROM lineitem ORDER BY l_orderkey LIMIT 10;
-- → top of plan: TOP_N

-- Block 4: see actual row flow
EXPLAIN ANALYZE SELECT l_orderkey FROM lineitem ORDER BY l_orderkey LIMIT 100;
-- → still 60M rows flow into TOP_N (it sees every row to know which 100 to keep)
-- → only 100 rows flow OUT

-- Block 5: memory check after sort
SELECT tag, temporary_storage_bytes FROM duckdb_memory();
-- → temporary_storage_bytes shows sort spill (if memory was tight)
```

The progression: see the operator name change → see that `TOP_N` still touches every row but emits few → see how spill shows up in memory introspection.

---

## Exercise Hints

If you got stuck on `04-exercise.md`:

- **Predicting `ORDER_BY` vs `TOP_N`** — the rule is simple: `LIMIT N` where N is a literal small number → `TOP_N`. No `LIMIT`, or `LIMIT` so large it might as well not exist (e.g., `LIMIT 60_000_000` on a 60M-row table) → `ORDER_BY`. Verify in `EXPLAIN`.
- **Variable-width keys (varchar)** — `ORDER BY l_comment LIMIT 100` still uses `TOP_N` (the LIMIT-rewrite is independent of key type), but the sort *within* the heap uses comparison-based sort, not radix.
- **`LIMIT 60_000_000` on a 60M-row table** — DuckDB sees that LIMIT covers the whole table and falls back to `ORDER_BY`. The threshold is set conservatively.
- **Spill check** — for the unbounded `ORDER_BY` queries, run with `SET memory_limit = '64MB'` and watch `temporary_storage_bytes` after the query. Sorting 60M `int64` values (=480MB minimum just for the keys) at 64MB *guarantees* spill.

---

## Common Misconceptions

**"`TOP_N` skips rows it doesn't need."** No — it sees every input row. The optimization is in *what it stores* (heap of N) and *what it emits* (N rows), not in which inputs it touches. Filter pushdown (Module 07's zonemaps) can skip rows; `TOP_N` cannot.

**"Radix sort is always faster than comparison sort."** For fixed-width keys with k bytes, radix is O(n·k). For comparison-based sort, it's O(n log n) but with simpler per-row work. For very small n or very long keys, comparison sort can win. DuckDB picks based on key type, not size.

**"Sorting parallelizes badly because of the merge step."** Phase 1 (local sorting) parallelizes perfectly. Phase 2 is a streaming merge with bounded memory — its cost is roughly linear in output size, and well-amortized across input. Sorting scales nearly linearly with cores in DuckDB up to single-digit threads, then plateaus on memory bandwidth, not coordination.

**"`ORDER BY` always materializes everything."** True for the operator named `ORDER_BY`. The trick is that `ORDER BY ... LIMIT N` doesn't compile to `ORDER_BY` — it compiles to `TOP_N` and never materializes anything but the heap. Same SQL, different operator.

**"Spilling sorts means you should add a `LIMIT` to avoid it."** If your application *needs* the full sorted result, no `LIMIT` will help. The right move is to either give DuckDB more memory or accept the spill cost. Adding `LIMIT 10000000` to "avoid spilling" doesn't help — the heap of 10M rows is bigger than your memory_limit too.

---

## Where This Connects

- **Module 02 — Reading Plans:** the `ORDER_BY` and `TOP_N` operator names you'll see in `EXPLAIN` are exactly what the optimizer picks.
- **Module 04 — Memory Buffer:** sorts use the same paged storage and the same memory manager. The spill behavior is identical.
- **Module 05 — Grouped Aggregation:** the two-phase morsel-local-then-merge pattern is shared. If you understood Phase 1/Phase 2 there, you understand it here.
- **Module 09 — Pipelines:** `ORDER_BY` is a **pipeline breaker** (must see all input before emitting any). `TOP_N` is *not* a breaker — it can stream rows through and only emits at the end. That difference is one of the most consequential parts of the pipeline model.

---

## Further Reading

- [Goetz Graefe, *Sorting And Indexing With Partitioned B-Trees*](https://www.cidrdb.org/cidr2003/program/p1.pdf) — modern external sort design, the source of many ideas DuckDB uses
- [DuckDB blog: Order by](https://duckdb.org/2021/08/27/external-sorting.html) — Mark Raasveldt walks through the actual implementation
- [DuckDB blog: Faster Sorting](https://duckdb.org/2024/03/01/faster-aggregation.html) — radix sort details and the binary key encoding
- [Bingmann's *radix sort* visualization](https://panthema.net/2013/sound-of-sorting/) — see radix vs comparison sorts in motion
