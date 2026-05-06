# Module 00 — Companion Guide: Welcome & Setup

> 📺 **Video:** [Watch on YouTube](<TODO: youtube link>)
> 💻 **Code:** [walkthroughs/00-welcome-setup](https://github.com/venkatdwe/learning-duckdb/tree/main/walkthroughs/00-welcome-setup) — `03-walkthrough.sql`
> ✍️ **Exercise:** [04-exercise.md](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/00-welcome-setup/04-exercise.md)
> 📊 **Slides:** [Open the deck](https://github.com/venkatdwe/learning-duckdb/blob/main/walkthroughs/00-welcome-setup/slides/deck.pptx)

This guide covers everything you need to set up before watching the module video.

---

## Prerequisites

- A terminal (macOS, Linux, or Windows)
- Basic SQL knowledge: SELECT, WHERE, GROUP BY

No database internals background required.

---

## Installation

### DuckDB CLI

**macOS (Homebrew):**
```bash
brew install duckdb
```

**Linux / Windows:**
Download the single-file binary from [duckdb.org](https://duckdb.org). No installer, no dependencies. Drop it on your PATH.

**Verify:**
```bash
duckdb --version
```

### Python Package

Use whatever package manager you prefer. A virtual environment is strongly recommended.

```bash
uv pip install duckdb
# or: pip install duckdb
```

**Verify:**
```bash
python3 -c "import duckdb; print(duckdb.__version__)"
```

Both commands should print the same version. If they don't, something is mixed up in your environment — stick with one installation method.

---

## Quick Setup Check

Start the CLI:
```bash
duckdb
```

You should see the `D` prompt. Run:
```sql
SELECT 'hello' AS greeting;
```

Then enable the timer (do this in every session):
```sql
.timer on
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `duckdb` | Start in-memory session |
| `duckdb myfile.db` | Open (or create) a persistent database |
| `.timer on` | Show query runtime after every result |
| `.quit` or Ctrl-D | Exit the CLI |
| `EXPLAIN ...` | Show estimated query plan |
| `EXPLAIN ANALYZE ...` | Show actual query plan with timing |

---

## Python Basics

```python
import duckdb

# In-memory query
result = duckdb.sql("SELECT 'hello' AS greeting").fetchall()

# Return as DataFrame
df = duckdb.sql("SELECT * FROM range(10)").df()

# Return as Arrow table
arrow = duckdb.sql("SELECT * FROM range(10)").arrow()
```

---

## Going Deeper

- [DuckDB documentation](https://duckdb.org/docs/) — official reference
- [DuckDB internals blog](https://duckdb.org/news/) — architecture posts from the team
- [DuckDB GitHub](https://github.com/duckdb/duckdb) — source and issues

---

## Source Material

This course is based on the *DiDi* (Design and Implementation of DuckDB Internals) lecture series developed by Prof. Torsten Grust at the University of Tübingen.

- DiDi course material: https://github.com/torsten-grust/didi
- License: MIT
- Course author: Prof. Torsten Grust, University of Tübingen
