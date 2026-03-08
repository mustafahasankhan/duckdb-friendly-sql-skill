# DuckDB Friendly SQL Skill for Claude

A Claude skill that teaches idiomatic DuckDB SQL — overriding common LLM training bias
toward PostgreSQL/MySQL patterns and enabling Claude to produce concise, correct DuckDB
queries using DuckDB's full "friendly SQL" dialect.

## What this skill does

- Guides Claude to use **DuckDB-native SQL extensions** instead of verbose standard SQL
- Teaches direct file querying (CSV, Parquet, JSON, Excel — no loading step needed)
- Covers all DuckDB data import patterns: S3, GCS, Azure, HTTP, local files
- Corrects 21 common anti-patterns LLMs fall into when writing DuckDB SQL

## Key capabilities

| DuckDB Feature | What it replaces |
|---|---|
| `GROUP BY ALL` | Listing every non-aggregated column |
| `CREATE OR REPLACE TABLE` | `DROP TABLE IF EXISTS` + `CREATE TABLE` |
| `SELECT * EXCLUDE (col)` | Listing all other columns by hand |
| `SELECT * REPLACE (expr AS col)` | Rewriting entire SELECT for one transform |
| `UNION ALL BY NAME` | Careful column ordering in UNION |
| `count()` | `count(*)` |
| `FILTER (WHERE cond)` on aggregates | `SUM(CASE WHEN ... END)` |
| Column alias in WHERE/HAVING | Subquery wrapper |
| `FROM 'file.csv'` directly | Load → table → query cycle |
| `ASOF JOIN` | Complex window function time-series joins |
| `LATERAL` join | Correlated subqueries |
| Function chaining with `.` | Deeply nested function calls |
| Reusable column aliases | Re-computing expressions |

## Installation

### Claude.ai
1. Download or clone this repo
2. Zip the `duckdb-friendly-sql/` folder
3. Go to Claude.ai → Settings → Capabilities → Skills → Upload skill
4. Select the zip file

### Claude Code
Place the `duckdb-friendly-sql/` folder in your Claude Code skills directory:
```bash
# macOS/Linux
cp -r duckdb-friendly-sql/ ~/.claude/skills/

# Or clone directly
git clone https://github.com/yourname/duckdb-friendly-sql-skill ~/.claude/skills/duckdb-friendly-sql
```

## Skill structure

```
duckdb-friendly-sql/
├── SKILL.md                              # Main skill — loaded when DuckDB task detected
└── references/
    ├── data-import.md                    # Full data import reference (S3, Azure, CSV, etc.)
    ├── anti-patterns.md                  # 21 common anti-patterns with DuckDB alternatives
    └── friendly-sql-cheatsheet.md        # Complete syntax quick reference
```

## Trigger phrases

The skill auto-loads when you ask about:
- "DuckDB query" / "write a DuckDB SQL query"
- "query a CSV/Parquet/JSON file"
- "load data into DuckDB" / "DuckDB import"
- "GROUP BY ALL" / "FROM-first syntax"
- `read_csv`, `read_parquet`, `read_json`
- "DuckDB pivot" / "DuckDB lambda" / "DuckDB join"

## Coverage

### Friendly SQL Features
- Table creation: `CREATE OR REPLACE`, CTAS, `INSERT BY NAME`, `INSERT OR IGNORE/REPLACE`, `DESCRIBE`, `SUMMARIZE`
- Query syntax: FROM-first, `GROUP BY ALL`, `ORDER BY ALL`, `SELECT * EXCLUDE`, `SELECT * REPLACE`, `UNION BY NAME`, prefix aliases, `LIMIT 10%`, trailing commas
- Column operations: aliases in WHERE/GROUP BY/HAVING, reusable aliases, `COLUMNS()` with regex/lambda/EXCLUDE/REPLACE
- Aggregation: `FILTER` clause, `count()` shorthand, top-N functions, `GROUPING SETS`/`ROLLUP`/`CUBE`
- Transformation: `PIVOT`, `UNPIVOT`
- Nested types: LIST creation/slicing/comprehension, lambda functions (`list_transform`, `list_filter`, `list_reduce`), STRUCT dot notation, `struct.*` expansion, auto-struct, MAP type, UNION type
- Function chaining with dot operator
- Joins: `ASOF JOIN`, `LATERAL JOIN`, `POSITIONAL JOIN`
- Strings: slicing, `format()`, `printf()`
- Data types: case insensitivity, auto-duplicate-rename, underscore number separators, implicit casts

### Data Import
- CSV: auto-detection, `sniff_csv()`, `read_csv` options, faulty CSV handling
- Parquet: direct query, column pruning, glob patterns, `union_by_name`, Hive partitioning
- JSON: auto-detection, nested type conversion
- Excel: `read_xlsx`, write xlsx
- S3/GCS/R2: secrets management, all auth methods
- Azure: Blob + ADLS, connection string, credential chain, service principal
- HTTP(S): direct query, partial Parquet reads, bearer token auth
- Secrets: persistent vs temporary, scoped secrets

## License

MIT

## Support

Feel free to ping on linkedin.com/in/mustafahasankhan

