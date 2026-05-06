# Module 03 — Companion Guide: Columnar Storage & Row Groups

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/03-columnar-storage](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/03-columnar-storage) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/03-columnar-storage/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/03-columnar-storage/slides/deck.pptx)

A reading companion to the module video. Skim before, read after.

---

## TL;DR

Module 01 said "DuckDB stores each column separately." That's true but incomplete. The full layout is **two-dimensional**:

- **Vertical split:** every column is stored as its own column-segment (the Module 01 story).
- **Horizontal split:** every table is sliced into **row groups** of ~122,880 rows each. A 100M-row table = ~814 row groups.

Each `(column, row_group)` cell on disk also carries a **zonemap** — just two numbers, the min and max value in that slice. Before reading any data, DuckDB checks the zonemap: if the predicate `c1 = 500000` doesn't fall inside `[min, max]`, the entire row group is *skipped*.

Two skipping mechanisms compose: **column pruning** (vertical, "don't read columns I don't need") + **zonemap filtering** (horizontal, "don't read row groups whose values can't match"). Together they're why a point-lookup on a 100M-row table can finish in milliseconds.

The only lever you control: **insert order**. Sort your inserts and zonemaps stay tight. Insert random data and zonemaps span the full domain — no skipping possible.

---

## Key Concepts

### Row groups
DuckDB chops every table horizontally into chunks of **122,880 rows** (=`120 × 1024`). Each chunk is called a **row group**. Inside a row group, each column is stored as its own contiguous segment.

```
table t (100M rows)
  ├── row group 0  (rows 0       … 122,879)
  │     ├── c1 segment  +  zonemap: {min: 0,       max: 122876}
  │     └── c2 segment  +  zonemap: {min: 1234,    max: 99999987}
  ├── row group 1  (rows 122,880 … 245,759)
  │     ├── c1 segment  +  zonemap: {min: 122880,  max: 245756}
  │     └── c2 segment  +  zonemap: {min: 51,      max: 99998765}
  ...
  └── row group 813
```

### Zonemaps — two numbers that gate everything
A zonemap is just `(min, max)` per column per row group. It's not an index — there are no B-trees, no hash tables, no extra storage to maintain. DuckDB writes it once at insert time and reads it once per query.

For `WHERE c1 = 500000`: DuckDB walks the row group list, checks each `c1` zonemap, and skips any group where `500000 ∉ [min, max]`. Only the matching groups (often just one) get their data read off disk.

### Sorted data → tight spans → fast scans
The walkthrough's `c1` column is monotonically increasing (`i & ~3`). After insertion, every row group's `c1` zonemap is **narrow** — `[0, 122876]`, `[122880, 245756]`, etc. A point-lookup skips ~813 of 814 row groups.

The `c2` column is hash-scrambled across `[0, 100M)`. Every row group's `c2` zonemap is **wide** — essentially `[~0, ~99999999]`. A point-lookup can't skip *any* row group; it must read all of them. Hence the **10× timing gap** between Query A (`WHERE c1 = 500000`) and Query B (`WHERE c2 = 500000`) — same selectivity, completely different physical effort.

### `pragma_storage_info`
The introspection table that lets you see all of this directly. One row per `(column, row_group, segment)`, with the `stats` column containing the zonemap's min/max as text.

```sql
SELECT row_group_id, column_name, stats
FROM   pragma_storage_info('t')
WHERE  segment_type = 'BIGINT'
ORDER BY row_group_id, column_name;
```

The walkthrough builds a `zonemap()` macro on top of this to render `(group, min, max, span)` cleanly. Span = `max - min`. Narrow span = skippable. Wide span = unskippable.

### `ATTACH` / `USE` / `DETACH`
DuckDB databases are single `.duckdb` files. You open one with `ATTACH 'path/file.duckdb' AS alias`, switch the active database with `USE alias`, and close cleanly with `DETACH alias`. The walkthrough closes with this pattern so subsequent modules (04+) can use named databases.

---

## Walkthrough Recap

The walkthrough builds one table with two columns telling opposite stories:

```sql
INSERT INTO t(c1, c2)
  SELECT i & ~3                                      AS c1,    -- ascending
         (i * 9876983769044::hugeint % 100000000)::bigint AS c2 -- random
  FROM range(100000000) AS _(i);
```

Then runs the same predicate on each:

```sql
EXPLAIN ANALYZE SELECT t.c1, t.c2 FROM t WHERE c1 = 500000;   -- ~10ms
EXPLAIN ANALYZE SELECT t.c2, t.c1 FROM t WHERE c2 = 500000;   -- ~100ms
```

10× gap. Identical selectivity (one matching row each). The difference is which row groups DuckDB has to actually open.

The remaining blocks introspect the layout: `pragma_storage_info` shows you the per-row-group metadata; the `zonemap()` macro distills it down to `(min, max, span)`; you can literally see the wide-vs-narrow span pattern by eyeballing a few rows.

---

## Exercise Hints

The exercise builds a `floats` table with one ascending and one random column, asks you to predict skippability, and compares. The exercise file has full answers — but the mental model to apply:

- **Row group count = `ceil(rows / 122_880)`.** A 1M-row table is 9 row groups; a 100M-row table is 814.
- **Skippable groups for a point predicate on sorted data ≈ row group count − 1.** All-but-one when the value falls inside one specific group.
- **Span is the diagnostic.** `max - min` per row group. Narrow span = sorted insert. Wide span = scrambled insert.
- **Stretch exercise — sorting on insert:** `CREATE TABLE x AS SELECT … ORDER BY col` rewrites the data physically sorted by `col`. Now `col`'s zonemaps become tight even if the source data was random. The flip side: the *other* columns' zonemaps may widen.

---

## Common Misconceptions

**"Zonemaps are an index."** No. They're metadata DuckDB writes once at insert time and uses for free during scans. There's no B-tree maintenance, no separate file, no manual rebuild. Module 08 introduces ART indexes — those are real indexes; zonemaps are not.

**"All my queries will be 10× faster if I sort the data."** Only queries whose predicates match the sort column get the speedup. Sorting by `c1` makes `c1` zonemaps tight; `c2` zonemaps stay wide. You sort by the column you filter on most.

**"Zonemaps make scans skip individual rows."** They skip entire row groups (~120K rows each). The granularity is the row group, not the row. For finer-grained skipping, you need an ART index (Module 08).

**"`SEQ_SCAN` reads the whole table."** With zonemaps, `SEQ_SCAN` can read a tiny fraction of the table while still being a sequential scan in name. The "whole table read" intuition from row-store databases doesn't carry over.

**"Zonemaps work on any column type."** They work for any orderable type (numeric, date, string). For columns where ordering doesn't make semantic sense (UUIDs, hashes), the zonemap exists but its span will always be effectively the full domain — useless for filtering.

---

## Where This Connects

- **Module 02 — Reading Plans:** the `~N rows` row-count estimates you saw in `EXPLAIN` come from these zonemap statistics.
- **Module 04 — Memory Buffer:** how DuckDB decides which row groups stay in RAM vs go back to disk after the scan finishes.
- **Module 06 — Sorting:** when you ask DuckDB to sort, it builds a new physical layout — and that new layout has new zonemaps.
- **Module 07 — Zonemaps & Row-Group Pruning:** the deep dive. Updates, `has_updates` flag, what happens to zonemaps when rows change.
- **Module 08 — ART Indexes:** what to reach for when zonemaps are wide and you need finer-grained skipping. The Index-Scan-vs-Sequential-Scan tradeoff.

---

## Further Reading

- [DuckDB internals: storage](https://duckdb.org/docs/internals/storage) — official docs on the on-disk format
- [`pragma_storage_info` reference](https://duckdb.org/docs/sql/pragmas#pragma_storage_info) — full column list of the introspection view
- Hannes Mühleisen, *DuckDB-Internals* talks (CMU DB Group on YouTube) — visual treatment of row groups and zonemaps from the co-author
- Daniel Lemire's blog: [zone maps and the like](https://lemire.me) — broader survey of min/max metadata across analytical engines (Parquet, Arrow, ORC use the same pattern)
