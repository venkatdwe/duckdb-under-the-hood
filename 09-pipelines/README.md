# Module 09 — Companion Guide: Query Execution Pipelines

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/09-pipelines](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/09-pipelines) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/09-pipelines/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/09-pipelines/slides/deck.pptx)

A reading companion to the module video. Skim before, read after.

---

## TL;DR

A **pipeline** is a chain of operators where rows flow one at a time from a source, through transforms, into a sink. Most simple queries are one pipeline start-to-finish.

Some operators block this streaming flow:

- **`HASH_GROUP_BY`** must accumulate every row before emitting any group totals.
- **`HASH_JOIN`** (build side) must fully load the build hash table before the probe side can begin.
- **`ORDER_BY`** must sort every row before emitting any.

These are **pipeline breakers**. Each one terminates a pipeline and starts the next one above it. A `JOIN` followed by `GROUP BY` typically creates 3 pipelines that run in strict dependency order — pipeline 2 can't start until pipeline 1 finishes; pipeline 3 can't start until pipeline 2 finishes.

Within each pipeline, DuckDB runs work in parallel across threads — the **morsel-driven** model. A morsel is a chunk of row groups (~120K rows). Each thread picks up morsels independently and runs them through the pipeline. Threads finish at different times based on data and load.

The consequence of parallel morsels: **rows arrive at the sink in thread-completion order, not table order**. Run the same `GROUP BY` query twice with 4 threads and you get the same data in different orders.

The fix: `ORDER BY`. Without it, DuckDB makes no promise about result ordering — and that includes `GROUP BY` results.

---

## Key Concepts

### Pipeline = streaming chain of operators
Source → Filter → Projection → Sink. Each row enters at the source, passes through each operator's transform, and exits at the sink. No row is ever materialized in full — only one vector (~2,048 rows) is alive at any time.

This is why simple queries finish in bounded memory regardless of table size. Streaming bypasses the buffer pool's "fits in RAM?" question entirely.

### Pipeline breakers
Operators that *must* see all input before they can emit any output:

| Operator | Why it breaks streaming |
|---|---|
| `HASH_GROUP_BY` | Hash table must contain every input row's key before group totals are correct |
| `HASH_JOIN` (build side) | Probe side can't lookup keys until the build hash table is fully loaded |
| `ORDER_BY` | Sort needs all rows in memory (or spilled) before emitting the smallest |
| `DISTINCT`, `WINDOW`, `LIMIT N` (sometimes) | Same accumulation requirement |

When you spot a breaker in `EXPLAIN`, a new pipeline starts above it. The pipeline below produces *into* the breaker; the pipeline above consumes *from* the breaker's accumulated output.

### Multi-pipeline execution
A `JOIN` followed by `GROUP BY`:

```
Pipeline 1:   scan smaller table → build join hash table
Pipeline 2:   scan larger table → probe join hash table → build group-by hash table
Pipeline 3:   read group-by hash table → emit grouped results
```

Each pipeline must complete before the next can start. Pipeline 2 can't begin probing until pipeline 1 has fully built the join hash table. Pipeline 3 can't start emitting groups until pipeline 2 has fully populated the group-by hash table.

The bottleneck is usually the pipeline with the most scanning. In the example above, pipeline 2 scans the larger table — that's where most wall-clock time goes.

### Morsel-driven parallelism
Within a single pipeline, DuckDB parallelizes by handing each thread a **morsel** (chunk of row groups, ~120K rows). Threads grab morsels from a shared queue, run them through the pipeline independently, and return for more.

This is the core of how DuckDB scales with cores:
- No shared state during scanning (each thread reads its own morsels)
- Each thread's hash-aggregation produces a *partial* hash table
- Threads coordinate at pipeline breakers via partition-wise combine (Module 05's two-phase aggregation)

### Non-deterministic ordering — and why it's not a bug
With 4 threads each finishing morsels at different times, the rows reach the sink in **thread-completion order**. That's not the same as table order. Same query, same data, different order each run.

This applies to:
- Raw `SELECT * FROM t` — rows come in thread-finish order, not insertion order.
- `GROUP BY` results — DuckDB makes no promise about which group rows come first.
- Join outputs — probe-side ordering is wrecked by parallel probing.

The fix is simple: add `ORDER BY` whenever you need a stable result. SQL's standard says ordering is undefined without it — DuckDB takes that seriously and uses parallelism aggressively.

---

## Walkthrough Recap

The walkthrough attaches `tpch-sf1.db` and demonstrates pipeline structure progressively:

1. **Block 1** — single pipeline: `EXPLAIN SELECT o_orderkey FROM orders WHERE o_totalprice > 400000;`. One source, one operator. No breakers.
2. **Block 2** — two pipelines: add `GROUP BY`. The `HASH_GROUP_BY` is a breaker. Pipeline 1 builds the hash table; pipeline 2 emits results.
3. **Block 3** — three pipelines: add a `JOIN`. Now 3 pipelines: build join hash table → probe + build group-by hash table → emit groups.
4. **Block 4** — parallelism demo: same `SELECT list(o_orderkey)` query run with `SET threads = 1` (sorted result) and `SET threads = 4` (scrambled result). Same data; different order. The smoking gun.

---

## Exercise Hints

If you got stuck on `04-exercise.md`:

- **Counting pipelines** — count breakers in the plan. N breakers = N+1 pipelines.
- **Identifying the bottleneck pipeline** — the one with the largest table being scanned. In a join+aggregate, usually pipeline 2.
- **Predicting determinism** — if there's no `ORDER BY` in the outermost SELECT, the result order is non-deterministic with multi-threading. Even `GROUP BY` doesn't guarantee order.
- **The 1-thread vs 4-thread test** — sort ordering vanishes the moment you go from 1 to 4 threads. There's no in-between behavior.
- **Adding ORDER BY** — locks the output order without changing pipeline structure. The earlier pipelines still run in parallel; only the very top of the plan emits in sorted order.

---

## Common Misconceptions

**"Pipelines are about how queries are written, not how they execute."** Pipelines are an *execution* concept — how the engine groups operators that can run together as a streaming unit. The same SQL can compile to different pipeline counts depending on which operators it contains.

**"Adding `ORDER BY` makes the whole query slower."** It only adds a sort at the *very top* of the plan. The earlier pipelines (joins, aggregations, scans) still run in parallel. The `ORDER BY` cost is bounded by the size of the final result — usually small relative to the work below it.

**"`GROUP BY` produces ordered output."** Many databases happen to emit grouped results in some natural order (often the order keys were inserted into the hash table). DuckDB does not. With multi-threaded execution, you get whatever order the threads finished in.

**"More threads is always faster."** Up to a point. Each pipeline parallelizes well within itself, but pipeline-to-pipeline coordination is sequential. Heavy data skew (one morsel that's 10× the others) bottlenecks the whole pipeline. For small queries, thread-startup overhead can dominate.

**"Pipelines are unique to DuckDB."** The morsel-driven model originated with **Hyper** (Neumann + Leis, TUM) and is now standard in modern analytical engines (DuckDB, Apache DataFusion, Velox). The names ("pipeline breaker," "morsel") come from the academic literature.

---

## Where This Connects

- **Module 02 — Reading Plans:** the operators you saw in `EXPLAIN` are the building blocks of pipelines. Every plan can be decomposed into pipelines by finding the breakers.
- **Module 05 — Grouped Aggregation:** the **two-phase external aggregation** is exactly how `HASH_GROUP_BY` runs across pipelines — Phase 1 within the producing pipeline, Phase 2 at the breaker boundary.
- **Module 06 — Sorting:** `ORDER_BY` is a pipeline breaker; `TOP_N` is *not*. That distinction is one of the most consequential in the model.
- **Module 04 — Memory Buffer:** pipeline breakers are precisely the operators that defeat streaming and force memory pressure → spilling.

---

## Further Reading

- [Neumann + Leis, *Morsel-Driven Parallelism*](https://db.in.tum.de/~leis/papers/morsels.pdf) — the foundational paper. Short, dense, worth the time.
- [Andy Pavlo's CMU lecture on parallel query execution](https://15445.courses.cs.cmu.edu/) — broader context: pipeline parallelism vs intra-operator parallelism vs distribution.
- [DuckDB blog: Parallel Execution](https://duckdb.org/2022/05/04/friendlier-sql.html) — DuckDB-specific implementation notes
- [Mark Raasveldt's CWI lectures](https://www.cwi.nl/en/groups/database-architectures/) — the educational treatment from one of DuckDB's authors
