# Module 07 — Companion Guide: Zonemaps & Row-Group Pruning

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/07-zonemaps](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/07-zonemaps) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/07-zonemaps/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/07-zonemaps/slides/deck.pptx)

A reading companion to the module video. Skim before, read after.

---

## TL;DR

Module 03 introduced zonemaps as a single sentence: "two numbers per row group." Module 07 is the deep dive. The hook:

```sql
WHERE t.c1 = 500_000   -- 0.001s  → 1 row group touched out of 814
WHERE t.c2 = 500_000   -- 0.010s  → all 814 row groups touched
```

Same table, same 100M rows, same predicate, **10× difference**. Why? `c1` is monotonically increasing, so each row group's `(min, max)` zonemap is narrow and the optimizer can skip 813 of 814 groups. `c2` is hash-scrambled, so every row group's zonemap spans the full `[0, 100M)` domain — nothing is skippable.

The mechanism is automatic. There's no `CREATE INDEX`, no maintenance, no separate file. Zonemaps are written once at insert time and queried once per scan via `pragma_storage_info`. They cost almost nothing to maintain — and on sorted data they're a 100–1000× speedup.

The catch: **zonemaps reflect insert order**, not column semantics. The same column can be highly skippable in one table and useless in another, depending on how rows were loaded.

---

## Key Concepts

### Zonemap = `(min, max)` per column per row group
For every row group (default ~120K rows) and every column, DuckDB records the minimum and maximum value. Total cost: 16 bytes per `(column, row_group)` for fixed-width types, slightly more for variable-width.

When a query has `WHERE col = X` (or `BETWEEN`, `<`, `>`, etc.), DuckDB walks the zonemap list before reading data:

```
for each row group g:
    if zonemap(g, col).min ≤ X ≤ zonemap(g, col).max:
        scan g for matching rows
    else:
        skip g entirely (no I/O)
```

Skip = the row group's data pages never get loaded. This is the fastest possible filter — zero work for excluded data.

### Insert order is the only lever
Zonemaps are computed at insert time and never recomputed. If you insert rows sorted by `c1`, you get tight `c1` zonemaps and wide zonemaps for everything else. Insert by `c2` and the situation flips.

The sort lever:
```sql
-- Tight c1 zonemaps:
INSERT INTO t SELECT … FROM source ORDER BY c1;

-- Tight c2 zonemaps (same data, different layout):
CREATE TABLE t_by_c2 AS SELECT … FROM source ORDER BY c2;
```

For point-lookup-heavy workloads, choose the column you filter on most as the sort key.

### `pragma_storage_info` — reading zonemaps from SQL
DuckDB exposes its row-group metadata through `pragma_storage_info('table_name')`. One row per `(column, row_group, segment)`, with the `stats` column containing the zonemap as text:

```
"Min: 0  Max: 122876  [Has Null: false]"
```

The walkthrough builds a `zonemap()` macro that parses this into clean `(min, max, span)` columns. You can run it on any table to see exactly which row groups are skippable.

### `has_updates` — what happens when rows change
When you `UPDATE` rows in a row group, DuckDB does *not* rewrite the existing zonemap. Instead:

1. The original zonemap stays — it still reflects the *original* min/max.
2. The row group's `has_updates` flag flips to `true`.
3. Updated values are tracked in separate **update-statistics storage** (visible in `pragma_storage_info` with the `TRANSACTION` tag).

When DuckDB scans this row group, it has to consult both stores. Once `has_updates = true`, the zonemap can no longer guarantee the original tight bounds — it might still skip in many cases, but the optimizer is more conservative.

### When zonemaps stop being enough
Zonemaps work great for:
- Point lookups on sorted columns (the walkthrough's `c1` case)
- Range queries on sorted columns
- Any predicate that aligns with insert order

They fail for:
- Random/hashed columns (every zonemap spans the full domain)
- Columns where insert order doesn't match query patterns
- Predicates with low selectivity even on sorted columns

When zonemaps are wide and your queries are still selective, the next tool is **ART indexes** (Module 08) — proper indexes that map keys to row IDs.

---

## Walkthrough Recap

The walkthrough builds a 100M-row table with one ordered (`c1`) and one random (`c2`) column, then:

1. **Block 2** — runs `EXPLAIN ANALYZE` for `WHERE c1 = 500_000` (~0.001s, 1 row group touched)
2. **Block 3** — runs the same predicate on `c2` (~0.010s, all 814 row groups touched)
3. **Block 4** — uses `pragma_storage_info` to inspect the actual zonemap stats
4. **Block 5** — builds the `zonemap()` macro that distills the stats to `(group, min, max, span)`
5. **Block 6** — counts skippable row groups for various predicates programmatically
6. Later blocks demonstrate `has_updates` after an `UPDATE` and how it affects subsequent scans

The progression: see the timing gap → understand why → see the metadata directly → see what an UPDATE does to it.

---

## Exercise Hints

If you got stuck on `04-exercise.md`:

- **Predicting skips for sorted columns** — for `c1` ascending in [0, 100M), a point predicate `c1 = X` matches exactly one row group: `floor(X / 122_880)`. The other 813 are skippable.
- **Predicting skips for random columns** — every row group spans roughly the full domain. Skip count is approximately zero unless the predicate value is *outside* the table's overall min/max.
- **Range queries** — `BETWEEN 1000 AND 2000` on a sorted column might span 0–1 row groups (very tight). On a random column, it can't skip any (every row group's zonemap likely overlaps that range).
- **Counting skippable row groups in SQL** — use the `zonemap()` macro: `SELECT count(*) FROM zonemap('c1') WHERE min > 500_000 OR max < 500_000;` gives the skip count for an equality predicate.
- **`has_updates` after an `UPDATE`** — the new value might be *outside* the original zonemap's `[min, max]`. After the update, the row group's effective bounds widen — fewer skips possible on subsequent queries.

---

## Common Misconceptions

**"Zonemaps are an index."** No — they're free-of-charge metadata, written once at insert time, with no maintenance. ART indexes (Module 08) are real indexes; zonemaps are not.

**"Sorting my data once will make zonemaps work for any column."** Sorting by `c1` makes `c1` zonemaps tight; the other columns' zonemaps remain whatever they happen to be. You can only sort by one ordering at a time. (You *can* maintain multiple sorted copies of the same table — at the cost of duplicate storage.)

**"Random data means zonemaps are useless, period."** They're useless for predicates *on the random column*. Predicates on other columns (or compound predicates that include the sort column) can still benefit. Insert order is the lever — pick it deliberately based on your query mix.

**"`has_updates = true` makes the row group slow."** It just makes the optimizer more conservative. The row group is still scanned in the same way; the optimizer just can't promise as much skippability. For workloads with frequent updates, you may want to periodically `VACUUM` or `CHECKPOINT` to rewrite the row groups and refresh the zonemaps.

**"DuckDB will choose the right sort order for me."** It won't. DuckDB inserts in the order rows arrive. If you want sorted layout, you have to use `ORDER BY` in your `CREATE TABLE AS SELECT` or `INSERT INTO ... SELECT`.

---

## Where This Connects

- **Module 02 — Reading Plans:** the `~N rows` row-count estimates in `EXPLAIN` are computed from these zonemap stats.
- **Module 03 — Columnar Storage:** the row-group / segment layout that hosts the zonemap metadata. Module 03 was the introduction; Module 07 is the deep dive.
- **Module 08 — ART Indexes:** what to use when zonemaps are wide and your queries are still selective. The Index-Scan-vs-Sequential-Scan tradeoff.
- **Module 09 — Pipelines:** zonemap filtering happens *inside* the `SEQ_SCAN` operator, before any rows enter the pipeline. It's not a separate operator — it's a property of how scans work.

---

## Further Reading

- [DuckDB internals: storage](https://duckdb.org/docs/internals/storage) — official docs on row groups + segment headers
- [`pragma_storage_info` reference](https://duckdb.org/docs/sql/pragmas#pragma_storage_info)
- [Apache Parquet stats spec](https://parquet.apache.org/docs/file-format/data-pages/encodings/) — Parquet uses the same min/max metadata pattern; reading the spec sharpens the mental model
- [Daniel Abadi's *Modern Column-Oriented DBs*](https://stratos.seas.harvard.edu/sites/scholar.harvard.edu/files/stratos/files/columnstoresfntdbs.pdf) — chapter on lightweight indexing
