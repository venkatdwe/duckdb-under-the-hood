# DuckDB Under the Hood — Course Companion

Reading guides for the [DuckDB Under the Hood](<TODO: youtube playlist link>) YouTube series.

Twelve modules that open the black box of a modern analytical database. You'll understand how DuckDB stores data, plans and executes queries, manages memory, and picks join algorithms — and why each choice matters for real query performance.

> 📺 **YouTube playlist:** <TODO: link>
> 💻 **Code repository:** [github.com/venkatdwe/learning-duckdb](https://github.com/venkatdwe/learning-duckdb) — SQL walkthroughs, exercises, slides
> 🦆 **DuckDB docs:** [duckdb.org](https://duckdb.org)

---

## How to use this repo

1. **Watch a module's video on YouTube.**
2. **Read the corresponding companion guide here** — same numbered folder. Every module's `README.md` is the reading companion: TL;DR, key concepts, walkthrough recap, exercise hints, common misconceptions, links to deeper material.
3. **Run the code from the [code repo](https://github.com/venkatdwe/learning-duckdb)** — clone it, follow `walkthroughs/NN-<slug>/03-walkthrough.sql`, attempt `04-exercise.md` after the video.

The companion docs are designed to be skimmed before a video and read fully after.

---

## Modules

| # | Module | Companion | Video |
|---|---|---|---|
| 00 | Welcome & Setup | [📖 Read](00-welcome-setup/README.md) | [📺 Watch](<TODO>) |
| 01 | Why DuckDB Is Fast | [📖 Read](01-why-fast/README.md) | [📺 Watch](<TODO>) |
| 02 | Reading Query Plans | [📖 Read](02-reading-plans/README.md) | [📺 Watch](<TODO>) |
| 03 | Columnar Storage | [📖 Read](03-columnar-storage/README.md) | [📺 Watch](<TODO>) |
| 04 | Memory Buffer | [📖 Read](04-memory-buffer/README.md) | [📺 Watch](<TODO>) |
| 05 | Grouped Aggregation | [📖 Read](05-grouped-aggregation/README.md) | [📺 Watch](<TODO>) |
| 06 | Sorting Large Tables | [📖 Read](06-sorting/README.md) | [📺 Watch](<TODO>) |
| 07 | Zonemaps & Row-Group Pruning | [📖 Read](07-zonemaps/README.md) | [📺 Watch](<TODO>) |
| 08 | ART Indexes | [📖 Read](08-art/README.md) | [📺 Watch](<TODO>) |
| 09 | Query Execution Pipelines | [📖 Read](09-pipelines/README.md) | [📺 Watch](<TODO>) |
| 10 | Vectorization Deep Dive | _coming soon_ | _coming soon_ |
| 11 | Optimizer Internals | _coming soon_ | _coming soon_ |
| 12 | Real-World Patterns | _coming soon_ | _coming soon_ |

---

## What's NOT in this repo

This repo is companion-docs-only. The walkthrough SQL, exercises, and slide images live in the [code repo](https://github.com/venkatdwe/learning-duckdb). Each companion guide links directly to its module's code, exercise, and slides.

The course's authoring tools (TTS pipeline, slide rendering, animation scripts) are private — not part of either public repo.

---

## Contributing

Found a typo, an unclear explanation, or a broken link? PRs welcome on this repo for companion-doc fixes. For SQL/exercise issues, please open an issue on the [code repo](https://github.com/venkatdwe/learning-duckdb/issues) instead.

---

## License

Companion guides: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — copy, share, adapt with attribution.

Code (in the separate code repo): MIT.
