# Module 01 — Companion Guide: Why DuckDB Is Fast

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/01-why-fast](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/01-why-fast) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/01-why-fast/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/01-why-fast/slides/deck.pptx)

A reading companion to the module video. Skim before, read after.

---

## TL;DR

DuckDB is fast on analytical queries for two structural reasons:

1. **Columnar storage** — each column is stored separately on disk, so a query that only touches three columns reads only those three columns' bytes. Row-stores read every column for every row touched, even unused ones.
2. **In-process execution** — DuckDB runs as a library inside your application, not as a separate server. No TCP, no serialization, no network hops between your code and the engine.

Together, these make a "small data" analytical pattern (`SUM`, `GROUP BY`, scans over a few columns out of many) ~10–100× faster than running the same query through a row-store like SQLite or a server-based DB like Postgres for the same hardware.

---

## Key Concepts

### Columnar storage
Data laid out by **column** rather than **row**. A table with columns `(id, revenue, age, name, …)` and 10M rows is stored as 4+ separate files (one per column), each holding 10M values of one type.

| Storage layout | Read pattern for `SUM(revenue)` | Bytes read |
|---|---|---|
| Row-store | Open every row, skip 9 columns, sum the 10th | ~10× the column's bytes |
| Column-store | Open `revenue` column, sum it | exactly the column's bytes |

### Analytical query pattern
Real-world analytical queries (dashboards, reports, ETL aggregates) typically touch **few columns of many** — three out of forty is common. Row-stores read all forty for every row. Column-stores read three.

### In-process execution
DuckDB runs in your application's address space — `import duckdb`, `duckdb.query(...)`, results land in your variables. No client/server protocol. Compare to Postgres where every query crosses a TCP connection, gets serialized, sent, parsed, executed, results serialized back, sent, deserialized.

For analytical queries on local data, the round-trip overhead alone often costs more than the query itself.

### EXPLAIN
DuckDB's plan-printing command. Prepend `EXPLAIN` to a query to see how the engine intends to execute it — which operators run, in what order, on which columns.

```sql
EXPLAIN SELECT SUM(revenue) FROM wide;
```

You'll see a tree with operators like `SEQ_SCAN(wide, [revenue])` — note that only `revenue` is listed as a projection, even though the table has many columns. That's the column-store advantage made visible.

---

## Walkthrough Recap

The walkthrough builds a `wide` table with 10 numeric columns and 10M rows, then runs two queries:

```sql
-- Query A: one column summed
SELECT SUM(revenue) FROM wide;

-- Query B: one row fetched, all columns
SELECT id, revenue, age, score, ratio, weight, factor, rate, metric, value
FROM wide WHERE id = 999999;
```

**Query A** scans the `revenue` column (10M values, ~80MB on disk for 8-byte integers).
**Query B** scans the `id` column (10M values) to find a match, then fetches all 10 columns for that single row.

Counterintuitively, Query A is *faster* despite returning a "summary" of 10M rows, because it only touches one column. Query B touches one row but in *ten* separate column files.

---

## Exercise Hints

If you got stuck on `04-exercise.md`:

- Both queries return "one row of output" but do dramatically different work under the hood.
- The amount of *data scanned* is what determines speed for analytical queries, not the size of the result.
- Use `EXPLAIN ANALYZE` (covered in module 02) to see actual rows-read counts. Or just time them with `.timer on`.

---

## Common Misconceptions

**"Column stores are slow for OLTP."** True for single-row inserts/updates — DuckDB is not a transactional database. It's optimized for analytical reads, batch loads, and append patterns.

**"Column stores compress better, that's the secret."** Compression helps but isn't the point. The fundamental win is *not reading bytes you don't need*. Compression compounds on top.

**"In-process means no networking — so it can't scale."** DuckDB is single-node by design. For analytical workloads on data that fits a server's RAM (often hundreds of GBs), single-node is faster than distributed because you skip the coordination overhead.

---

## Where This Connects

- **Module 02 — Reading Plans**: How to read the `EXPLAIN` output you saw glimpses of here. You'll learn what each operator does and how to spot inefficient plans.
- **Module 03 — Columnar Storage**: How the columnar layout actually works on disk — row groups, segments, dictionary encoding. Today's "open one file per column" was a simplification; the real structure is finer-grained.
- **Module 04 — Memory Buffer**: How DuckDB decides which columns to keep in RAM and which to leave on disk.

---

## Further Reading

- DuckDB docs: [Why DuckDB?](https://duckdb.org/why_duckdb)
- The original column-store paper: Stonebraker et al., "C-Store: A Column-oriented DBMS" (VLDB 2005) — establishes the foundational arguments.
- Hannes Mühleisen (DuckDB co-creator): [The Case for In-Process Analytics](https://duckdb.org/2024/06/26/the-case-for-in-process-analytics) — short, opinionated, persuasive.
