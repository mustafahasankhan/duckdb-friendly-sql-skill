# DuckDB CLI Reference

Complete reference for the DuckDB command-line interface.

---

## Starting DuckDB

```bash
duckdb                          # in-memory database
duckdb mydb.duckdb              # persistent database (created if missing)
duckdb :memory:                 # explicit in-memory
duckdb -readonly mydb.duckdb    # read-only mode
duckdb -init /dev/null          # skip ~/.duckdbrc initialization
```

---

## Command-Line Flags

### Execution
| Flag | Description |
|------|-------------|
| `-c "SQL"` | Execute SQL and exit |
| `-f file.sql` | Execute SQL from file and exit |
| `-init FILE` | Use alternate init file instead of `~/.duckdbrc` |
| `-readonly` | Open database in read-only mode |
| `-echo` | Print each SQL command before running it |
| `-bail` | Stop on first error |
| `-unsigned` | Allow loading unsigned extensions |

### Output Format at Startup
| Flag | Description |
|------|-------------|
| `-csv` | Comma-separated output |
| `-json` | JSON array output |
| `-table` | ASCII table output |
| `-markdown` | Markdown table output |
| `-html` | HTML table output |
| `-line` | One value per line |

### Display
| Flag | Description |
|------|-------------|
| `-header` / `-noheader` | Show/hide column headers |
| `-nullvalue TEXT` | Text to display for NULL values (default: empty) |
| `-separator SEP` | Column separator (for csv/line modes) |

---

## Unix Piping and stdin/stdout

```bash
# Read from stdin
cat data.csv | duckdb -c "SELECT * FROM read_csv('/dev/stdin')"

# Write to stdout (pipe to other tools)
duckdb -c "COPY (FROM my_table) TO '/dev/stdout' (FORMAT CSV, HEADER)"

# Pipeline
duckdb -csv -c "SELECT * FROM 'data.parquet'" | head -20
duckdb -csv -c "SELECT * FROM 'data.parquet'" | wc -l

# Execute a SQL file
duckdb mydb.duckdb -f analysis.sql
cat analysis.sql | duckdb mydb.duckdb
```

### One-Liner Data Conversion
```bash
# CSV → Parquet
duckdb -c "COPY (FROM 'input.csv') TO 'output.parquet' (FORMAT PARQUET)"

# Parquet → CSV
duckdb -c "COPY (FROM 'input.parquet') TO 'output.csv' (HEADER, DELIMITER ',')"

# JSON → Parquet
duckdb -c "COPY (FROM read_json_auto('input.json')) TO 'output.parquet' (FORMAT PARQUET)"

# Filter on convert
duckdb -c "COPY (FROM 'data.csv' WHERE amount > 1000) TO 'filtered.parquet' (FORMAT PARQUET)"

# S3 → local Parquet
duckdb -c "COPY (FROM 's3://bucket/data/*.csv') TO 'local.parquet' (FORMAT PARQUET)"

# Multiple files → merged output
duckdb -c "COPY (FROM 'logs/*.parquet') TO 'merged.csv' (HEADER)"
```

---

## Dot Commands

### Database and Schema
| Command | Description |
|---------|-------------|
| `.open <path>` | Open or switch to a database file |
| `.open --readonly <path>` | Open in read-only mode |
| `.tables [pattern]` | List tables (supports LIKE pattern) |
| `.schema [table]` | Show CREATE statements |
| `.databases` | List attached databases |

### Output Control
| Command | Description |
|---------|-------------|
| `.mode FORMAT` | Change output format (see formats below) |
| `.output <file>` | Redirect all output to file |
| `.once <file>` | Redirect next query's output to file only |
| `.headers on/off` | Show or hide column headers |
| `.separator COL [ROW]` | Set column and row separators |
| `.nullvalue TEXT` | Set NULL display text |
| `.maxrows N` | Limit rows displayed (0 = unlimited) |
| `.maxwidth N` | Limit column display width |

### Execution and Debugging
| Command | Description |
|---------|-------------|
| `.timer on/off` | Show query execution time |
| `.echo on/off` | Print commands before executing |
| `.bail on/off` | Stop on first error |
| `.read <file.sql>` | Execute SQL from file |
| `.highlight on/off` | Toggle syntax highlighting |
| `.prompt '<text>'` | Customize the CLI prompt |

### Editing
| Command | Description |
|---------|-------------|
| `.edit` or `\e` | Open current query in external editor |
| `.help [pattern]` | Show help for dot commands |
| `.exit` / `.quit` / `Ctrl+D` | Exit DuckDB |

---

## Output Formats (18 available)

Use `.mode FORMAT` inside the CLI or pass `-FORMAT` flag at startup.

| Format | Best For |
|--------|----------|
| `duckbox` | Default — pretty Unicode box-drawing table |
| `table` | Simple ASCII table |
| `csv` | Comma-separated (spreadsheets, pipelines) |
| `tabs` | Tab-separated |
| `json` | JSON array |
| `jsonlines` | Newline-delimited JSON (streaming-friendly) |
| `markdown` | Markdown table (docs, GitHub, Notion) |
| `html` | HTML table |
| `latex` | LaTeX tabular (academic papers) |
| `insert TABLE` | SQL INSERT statements |
| `column` | Fixed-width columns |
| `line` | One key=value pair per line |
| `list` | Pipe-separated list |
| `trash` | Discard output (benchmarking) |

---

## Configuration: ~/.duckdbrc

Auto-loaded on startup. Contains dot commands and SQL settings.

```sql
-- ~/.duckdbrc
.timer on
.mode duckbox
.maxrows 50
.highlight on

-- Syntax highlighting colors
.keyword green
.constant yellow
.comment brightblack
.error red

-- DuckDB settings
SET threads = 8;
SET memory_limit = '16GB';
```

---

## Keyboard Shortcuts

### History Navigation
| Shortcut | Action |
|----------|--------|
| `Ctrl+P` / `↑` | Previous command |
| `Ctrl+N` / `↓` | Next command |
| `Ctrl+R` | Reverse history search |
| `Alt+<` / `Alt+>` | First / last in history |

### Cursor Movement
| Shortcut | Action |
|----------|--------|
| `Ctrl+A` | Move to start of line |
| `Ctrl+E` | Move to end of line |
| `Alt+B` / `Ctrl+←` | Jump one word backward |
| `Alt+F` / `Ctrl+→` | Jump one word forward |
| `Home` / `End` | Start / end of buffer |

### Editing
| Shortcut | Action |
|----------|--------|
| `Ctrl+W` | Delete word backward |
| `Alt+D` | Delete word forward |
| `Ctrl+K` | Delete to end of line |
| `Ctrl+U` | Delete entire line |
| `Alt+U` / `Alt+L` | Uppercase / lowercase word |

### Autocomplete
| Shortcut | Action |
|----------|--------|
| `Tab` | Autocomplete / next suggestion |
| `Shift+Tab` | Previous suggestion |
| `Esc Esc` | Undo autocomplete |

DuckDB autocomplete is context-aware: SQL keywords, table names, column names, and local file paths.

---

## External Editor

Open the current query buffer in your preferred editor:

```sql
.edit
-- or
\e
```

Editor resolution order: `DUCKDB_EDITOR` → `EDITOR` → `VISUAL` → `vi`

---

## Safe Mode

Restricts file system access. **Cannot be disabled once enabled in the same session.**

When active:
- No external file reads or writes
- `.read`, `.output`, `.import`, `.sh` commands disabled
- `read_csv()`, `read_parquet()`, `COPY FROM/TO` disabled

---

## Performance Tips

```bash
# Parquet is faster than CSV for repeated queries — convert once
duckdb -c "COPY (FROM '*.csv') TO 'data.parquet' (FORMAT PARQUET)"

# Use Parquet count (reads metadata only — instant)
duckdb -c "SELECT count(*) FROM 'huge.parquet'"

# Benchmark without printing output
duckdb -c ".mode trash" -c ".timer on" -c "SELECT * FROM 'data.parquet'"

# Disable CSV sniffer for known-schema files (faster for many small files)
duckdb -c "SELECT * FROM read_csv('*.csv', auto_detect=false, columns={'a':'INTEGER','b':'VARCHAR'})"
```

---

## Useful One-Liners

```bash
# Quick stats on a file
duckdb -c "SUMMARIZE 'data.csv'"

# Schema of a file
duckdb -c "DESCRIBE SELECT * FROM 'data.parquet'"

# CSV sniffer
duckdb -c "FROM sniff_csv('mystery.csv')"

# First 5 rows
duckdb -c "SELECT * FROM 'data.parquet' LIMIT 5"

# Row count (instant for Parquet)
duckdb -c "SELECT count(*) FROM 'large.parquet'"

# Export to partitioned Parquet
duckdb -c "COPY (FROM 'data.csv') TO 'out/' (FORMAT PARQUET, PARTITION_BY (year, region))"

# Convert with filter
duckdb -c "COPY (FROM 'all_data.parquet' WHERE region='US') TO 'us_data.parquet' (FORMAT PARQUET)"
```
