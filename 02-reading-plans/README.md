# Module 02 — Companion Guide: Reading Query Plans

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/02-reading-plans](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/02-reading-plans) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/02-reading-plans/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/02-reading-plans/slides/deck.pptx)

A reading companion to the module video. Skim before, read after.

---

## TL;DR

A DuckDB query plan is an **upside-down tree** drawn bottom-up. Leaves at the bottom read data from disk; the root at the top emits the result; rows flow upward. To read a plan: start at the bottom-most operator, follow the `│` connectors upward, and ask at each step *what does this operator add?*

Two commands give you plans:
- `EXPLAIN` — shows the **shape** of the plan (operators, estimated row counts) without running the query
- `EXPLAIN ANALYZE` — runs the query, then shows the same plan annotated with **actual** row counts and per-operator wall-clock time

After this module, you can sketch a plan on paper from a SQL string, run `EXPLAIN`, and read the `~N rows` annotations to spot where the optimizer is guessing wrong.

---

## Key Concepts

### Plan trees flow upward
DuckDB's `EXPLAIN` output draws operators stacked vertically. **Sources** (`SEQ_SCAN`, `INDEX_SCAN`) are at the bottom — they produce rows from storage. **Sinks** (`RESULT`, `EXPLAIN`) are at the top — they consume the final rows. Everything in between is a transform.

```
┌─────────────┐   ← top: query result
│  PROJECTION │
└──────┬──────┘
┌──────┴──────┐
│ HASH_GROUP_BY│  ← aggregator
└──────┬──────┘
┌──────┴──────┐
│   SEQ_SCAN  │   ← bottom: reads disk
└─────────────┘
```

### EXPLAIN vs EXPLAIN ANALYZE
| | `EXPLAIN` | `EXPLAIN ANALYZE` |
|---|---|---|
| Runs the query? | No | Yes |
| Shows row estimates? | Yes | Yes |
| Shows actual row counts? | No | Yes |
| Shows per-operator timing? | No | Yes |
| Use it for | "Will this plan look reasonable?" | "Why is this query slow?" |

### Operator glossary (the five you'll meet most)
- **`SEQ_SCAN`** — reads a table sequentially. The walkthrough's `EXPLAIN SELECT SUM(revenue) FROM wide` produces a single `SEQ_SCAN` over the `revenue` column.
- **`PROJECTION`** — computes derived values or selects a subset of columns.
- **`FILTER`** — applies a `WHERE` predicate. Often *fused* into `SEQ_SCAN` rather than appearing as a separate operator (an optimization called **filter pushdown**).
- **`HASH_GROUP_BY`** — groups rows by a key and aggregates. Builds a hash table; emits one row per group.
- **`HASH_JOIN`** — joins two tables. The smaller table is the **build side** (used to construct the hash table); the larger table is the **probe side** (each row is looked up in the hash table).

### Estimated vs actual row counts
The optimizer uses statistics to *estimate* how many rows will flow through each operator. `EXPLAIN ANALYZE` shows both estimate and actual.

When estimate ≈ actual: optimizer is happy, plan is probably good.
When estimate ≫ actual or estimate ≪ actual: the optimizer chose its plan based on bad information. The plan might still be fine — or it might be the wrong shape entirely (e.g., picked the wrong build side for a join).

---

## Walkthrough Recap

The walkthrough builds two tables — `wide` (5M rows, single-column aggregates) and `sales` (1M rows with a region column for `GROUP BY` / `JOIN` examples) — then runs a series of `EXPLAIN` and `EXPLAIN ANALYZE` queries:

```sql
-- Block 1: simplest tree
EXPLAIN SELECT SUM(revenue) FROM wide;
-- → SEQ_SCAN → UNGROUPED_AGGREGATE

-- Block 3: GROUP BY adds a hash aggregator
EXPLAIN SELECT region, SUM(revenue) FROM sales GROUP BY region;
-- → SEQ_SCAN → HASH_GROUP_BY → PROJECTION

-- Block 4: ANALYZE adds timing + actual row counts
EXPLAIN ANALYZE SELECT region, SUM(revenue) FROM sales GROUP BY region;
-- Same shape; now you can see HASH_GROUP_BY took 80% of the time
```

The progression: **simple → grouped → joined → ordered**. Each step adds one operator. By the end you can read a 4-operator plan top-to-bottom or bottom-to-top fluently.

---

## Exercise Hints

If you got stuck on `04-exercise.md`:

- **Challenge 1 (sketching joins).** Look at the SQL: which table is small (`regions` = 10 rows) and which is large (`sales` = 1M rows)? DuckDB's optimizer almost always picks the small one as the join's build side. The build side is rendered as the **right child** of `HASH_JOIN` in the `EXPLAIN` output.
- **Challenge 1 (TOP_N).** `ORDER BY … LIMIT N` doesn't compile to `ORDER_BY` in DuckDB — it compiles to `TOP_N`, which maintains a heap of N rows instead of sorting everything. Module 06 explores this in detail.
- **Challenge 2 (heavy operator).** In `EXPLAIN ANALYZE` output, look at the timing column. The operator with the biggest wall-clock time isn't always the one you'd expect — `HASH_GROUP_BY` is often surprisingly expensive because it has to maintain a hash table across all input rows.
- **Challenge 3 (estimate vs actual).** Run with two different `WHERE` predicates that select wildly different fractions of rows. The optimizer's estimate uses table statistics — when those statistics are stale or the predicate is unusual, the estimate diverges from reality.

---

## Common Misconceptions

**"Reading the plan top-down feels natural — top is the start, right?"** No. The top of the plan tree is the **end** of execution: the final result. Data starts at the leaves (bottom) and flows up. A common confusion because written documents read top-to-bottom — query plans flip that.

**"`EXPLAIN ANALYZE` is just `EXPLAIN` plus timing, so use it always."** It actually *runs the query*. For an expensive query you don't want to wait for, `EXPLAIN` (without `ANALYZE`) gives you the plan shape instantly. Use `ANALYZE` when you specifically need the timing.

**"If the row estimate is wrong, the plan is broken."** Often the optimizer's plan is robust enough that bad estimates still produce a reasonable plan. The estimate matters most when it changes which operator the optimizer picks (e.g., choosing the wrong build side, picking nested-loop over hash join). Bad estimate ≠ bad plan automatically.

**"`SEQ_SCAN` always reads the whole table."** With **filter pushdown** (Module 03 + 07), DuckDB can skip entire row groups based on column statistics. The `SEQ_SCAN` operator name makes it sound exhaustive but the actual bytes read can be far less.

---

## Where This Connects

- **Module 03 — Columnar Storage:** the on-disk layout that makes `SEQ_SCAN` fast. Row groups, column segments, and how the optimizer's row estimates come from segment statistics.
- **Module 04 — Memory Buffer:** how DuckDB decides which `SEQ_SCAN`'d data stays in RAM.
- **Module 05 — Grouped Aggregation:** the internals of the `HASH_GROUP_BY` you saw here.
- **Module 06 — Sorting:** when `ORDER BY` becomes `TOP_N` and when it stays `ORDER_BY` — the LIMIT-aware optimization.
- **Module 09 — Pipelines:** how DuckDB groups operators into execution pipelines and runs them in parallel. Pipeline breakers like `HASH_GROUP_BY` and `HASH_JOIN` get their full treatment.

Every module from here on assumes you can read an `EXPLAIN` plan. Module 02 is the most foundational module in the course.

---

## Further Reading

- [DuckDB EXPLAIN docs](https://duckdb.org/docs/guides/meta/explain) — official reference for the operators
- [DuckDB EXPLAIN ANALYZE docs](https://duckdb.org/docs/guides/meta/explain_analyze) — what each annotation means
- [Use The Index, Luke](https://use-the-index-luke.com/) — Markus Winand's classic on reading execution plans (Postgres/MySQL examples but the mental model is identical)
- The DuckDB optimizer paper: Hannes Mühleisen et al., *DuckDB: an Embeddable Analytical Database* (SIGMOD 2019) — short, readable.
