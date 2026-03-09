---
name: duckdb-friendly-sql
description: >
  Guides writing idiomatic DuckDB SQL using DuckDB's "friendly SQL" dialect and
  easy data import/export patterns. Use when writing DuckDB queries, analyzing data
  with DuckDB, importing CSV/Parquet/JSON/Excel files, working with S3/GCS/Azure
  storage, or when a user asks about DuckDB syntax, functions, or data loading.
  Key trigger phrases: "DuckDB query", "query a CSV/Parquet/JSON file", "load data
  into DuckDB", "DuckDB syntax", "GROUP BY ALL", "FROM-first", "DuckDB import",
  "read_csv", "read_parquet", "duckdb pivot", "duckdb lambda", "duckdb join",
  "duckdb cli", "duckdb shell", "convert parquet to csv", "duckdb export data".
metadata:
  author: duckdb-friendly-sql-skill
  version: 1.0.0
---

# DuckDB Friendly SQL Skill

## Instructions

You are an expert in DuckDB's SQL dialect. DuckDB has a rich set of SQL extensions
("friendly SQL") that make queries more concise, readable, and powerful. **Always
prefer these DuckDB-native patterns over verbose standard SQL equivalents.** Many
LLMs default to PostgreSQL or ANSI SQL patterns — override that bias and use DuckDB's
actual capabilities.

### Critical Principle: Prefer DuckDB-Native Over Verbose SQL

When writing DuckDB SQL, always check: is there a DuckDB shorthand for this?
Before writing a subquery, check if `GROUP BY ALL` or a column alias in `WHERE` works.
Before writing `DROP TABLE IF EXISTS; CREATE TABLE`, use `CREATE OR REPLACE TABLE`.
Before listing every non-aggregated column in GROUP BY, use `GROUP BY ALL`.

---

## Step 1: Table Creation and Management

### CREATE OR REPLACE TABLE
**Always use this instead of `DROP TABLE IF EXISTS` + `CREATE TABLE`.**

```sql
-- DuckDB way (preferred)
CREATE OR REPLACE TABLE my_table AS SELECT * FROM 'data.csv';

-- Anti-pattern (verbose, old-style)
DROP TABLE IF EXISTS my_table;
CREATE TABLE my_table AS SELECT * FROM 'data.csv';
```

### CREATE TABLE AS SELECT (CTAS)
Schema is auto-inferred — no need to define columns manually:

```sql
CREATE TABLE analytics AS
SELECT date_trunc('month', event_date) AS month, count() AS events
FROM 'events.parquet'
GROUP BY ALL;
```

### INSERT INTO ... BY NAME
Match by column name, not position — more robust and readable:

```sql
INSERT INTO users BY NAME
SELECT 'alice@example.com' AS email, 'Alice' AS name;
```

### INSERT OR IGNORE / INSERT OR REPLACE
Handle constraint conflicts gracefully:

```sql
INSERT OR IGNORE INTO users BY NAME
    (SELECT 1 AS id, 'alice' AS name);              -- skip on conflict

INSERT OR REPLACE INTO users BY NAME
    (SELECT 1 AS id, 'alice_new' AS name);           -- overwrite on conflict
```

### DESCRIBE and SUMMARIZE
Inspect tables without writing metadata queries:

```sql
DESCRIBE my_table;     -- column names, types, nullability
SUMMARIZE my_table;    -- min, max, avg, null counts, unique counts per column
```

---

## Step 2: Query Syntax Enhancements

### FROM-First Syntax
DuckDB supports starting queries with FROM — aligns with logical execution order:

```sql
-- Equivalent to SELECT * FROM my_table
FROM my_table;

-- With column selection
FROM my_table SELECT name, age;

-- Works with COPY too
COPY (FROM sales WHERE year = 2024) TO 'sales_2024.parquet';
```

### GROUP BY ALL
**This is DuckDB's killer feature.** Auto-groups by all non-aggregated SELECT columns.
Never list GROUP BY columns manually when GROUP BY ALL works:

```sql
-- DuckDB way
SELECT region, product_category, month, sum(revenue) AS total
FROM sales
GROUP BY ALL;

-- Anti-pattern (verbose, error-prone)
SELECT region, product_category, month, sum(revenue) AS total
FROM sales
GROUP BY region, product_category, month;
```

### ORDER BY ALL
Order by all SELECT columns left-to-right for deterministic results:

```sql
SELECT year, month, count() AS events
FROM logs
GROUP BY ALL
ORDER BY ALL;

-- Descending
ORDER BY ALL DESC;
```

### SELECT * EXCLUDE
Return all columns except unwanted ones — no need to list every column:

```sql
-- Remove PII or irrelevant columns
SELECT * EXCLUDE (ssn, password_hash, internal_notes)
FROM users;

-- Works with aliases too
SELECT * EXCLUDE (jar_jar_binks) FROM star_wars_data;
```

### SELECT * REPLACE
Return all columns but transform specific ones in-place:

```sql
-- Normalize a column without writing every other column
SELECT * REPLACE (
    lower(email) AS email,
    price * 1.1 AS price
)
FROM products;
```

### UNION BY NAME
Combine result sets by column name rather than position — handles schema differences:

```sql
-- Tables with different column orders or counts
SELECT name, age FROM employees_us
UNION ALL BY NAME
SELECT age, name, department FROM employees_eu;
-- Missing columns filled with NULL automatically
```

### Prefix Aliases (Colon Syntax)
More concise alias syntax:

```sql
SELECT
    revenue: sum(amount),
    month: date_trunc('month', created_at),
    user_count: count(DISTINCT user_id)
FROM orders
GROUP BY ALL;
```

### Percentage LIMIT
```sql
-- Get top 10% of rows
SELECT * FROM large_table LIMIT 10%;
```

### QUALIFY — Filter on Window Function Results
**Eliminates subqueries for filtering by window functions.** `QUALIFY` is to window functions what `HAVING` is to aggregates.

```sql
-- DuckDB way: keep the full row for each group (no subquery!)
SELECT * FROM events
QUALIFY row_number() OVER (PARTITION BY user_id ORDER BY created_at DESC) = 1;

-- Anti-pattern (unnecessary subquery)
SELECT * FROM (
    SELECT *, row_number() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM events
) t WHERE rn = 1;
```

### arg_max / max_by — Simplest "Best Per Group"
When you only need specific columns (not `SELECT *`), `arg_max` is simpler than QUALIFY:

```sql
-- Best product per customer (no window function needed)
SELECT
    customer_id,
    arg_max(product, amount) AS top_product,
    max(amount) AS max_amount
FROM orders
GROUP BY ALL;

-- Multiple values: arg_max returns the arg at the row where val is max
-- Use max_by as an alias (identical behavior)
SELECT
    region,
    max_by(product, revenue) AS top_product,
    max_by(salesperson, revenue) AS top_seller,
    max(revenue) AS max_revenue
FROM sales
GROUP BY ALL;
```

**When to use which:**
- `arg_max` / `max_by` → need specific columns from the "best" row per group
- `QUALIFY` → need `SELECT *` (entire row) or complex ranking logic (top-N, dense_rank, etc.)

### DISTINCT ON — One Row Per Group
Cleaner alternative to `ROW_NUMBER() = 1` for latest-record-per-group queries:

```sql
-- Latest event per user (ORDER BY determines which row is kept)
SELECT DISTINCT ON (user_id)
    user_id, event_type, created_at
FROM events
ORDER BY user_id, created_at DESC;
```

---

## Step 3: Column Operations

### Column Aliases in WHERE, GROUP BY, HAVING
**DuckDB allows using SELECT aliases in WHERE, GROUP BY, and HAVING — no subquery needed.**

```sql
-- DuckDB way (no subquery needed!)
SELECT
    date_trunc('week', created_at) AS week,
    count() AS signups,
    sum(revenue) AS total_revenue
FROM events
WHERE week >= '2024-01-01'    -- uses alias directly
GROUP BY week                  -- uses alias directly
HAVING total_revenue > 1000;  -- uses alias directly

-- Anti-pattern (subquery workaround)
SELECT * FROM (
    SELECT date_trunc('week', created_at) AS week, count() AS signups, ...
) t WHERE t.week >= '2024-01-01';
```

**Note:** Column aliases cannot be used in JOIN ON clauses.

### Reusable Column Aliases
Reference an alias defined earlier in the SAME SELECT clause:

```sql
SELECT
    price * quantity AS subtotal,
    subtotal * 0.08 AS tax,         -- reuses subtotal
    subtotal + tax AS total          -- reuses both
FROM order_items;
```

### COLUMNS() — Dynamic Column Selection
Apply operations across columns matching a pattern:

```sql
-- Select columns matching a regex
SELECT id, COLUMNS('.*_amount') FROM transactions;

-- Apply aggregate to matching columns
SELECT max(COLUMNS('sales_.*')) FROM monthly_data;

-- Exclude from COLUMNS
SELECT max(COLUMNS(* EXCLUDE id)) FROM products;

-- REPLACE within COLUMNS
SELECT COLUMNS(* REPLACE (price::DECIMAL(10,2) AS price)) FROM catalog;

-- Lambda-based selection
SELECT COLUMNS(col -> col LIKE '%_date') FROM events;
```

---

## Step 4: Data Import — Direct File Querying

**DuckDB's superpower: query files directly without loading them first.**

### Direct File Queries
```sql
-- CSV (auto-detects headers, types, delimiter)
SELECT * FROM 'data.csv';

-- Parquet (columnar — very fast)
SELECT * FROM 'data.parquet';

-- JSON
SELECT * FROM 'data.json';

-- Excel
SELECT * FROM 'report.xlsx';

-- Multiple files with glob
SELECT * FROM 'data/part-*.parquet';
SELECT * FROM 'logs/2024/*.csv';

-- Nested glob
SELECT * FROM 'warehouse/**/*.parquet';
```

### Create Table from File
```sql
-- Fastest pattern: CTAS from file
CREATE OR REPLACE TABLE sales AS FROM 'sales_data.csv';

-- With transformations
CREATE OR REPLACE TABLE clean_users AS
SELECT * EXCLUDE (tmp_col)
REPLACE (trim(lower(email)) AS email)
FROM 'users.csv'
WHERE email IS NOT NULL;
```

### read_csv with Options
```sql
SELECT * FROM read_csv('data.csv',
    delim = '|',
    header = true,
    columns = {'date': 'DATE', 'amount': 'DECIMAL(10,2)', 'name': 'VARCHAR'},
    dateformat = '%d/%m/%Y',
    sample_size = -1    -- scan entire file for type detection
);
```

### CSV Sniffer — Inspect Before Loading
```sql
-- See what DuckDB auto-detects about a CSV
FROM sniff_csv('mystery_file.csv');
-- Returns: delimiter, quote, escape, header, column types, and a ready-to-use SQL prompt
```

### COPY Statement
```sql
-- Import
COPY users FROM 'users.csv';
COPY users FROM 'users.csv' (FORMAT csv, HEADER true, DELIMITER ',');

-- Export
COPY users TO 'users_export.parquet' (FORMAT parquet);
COPY (FROM sales WHERE year = 2024) TO 'sales_2024.csv' (HEADER, DELIMITER ',');
```

Consult `references/data-import.md` for S3, GCS, Azure, HTTP, and Excel details.

---

## Step 5: Advanced Aggregation

### FILTER Clause on Aggregates
Conditional aggregation without CASE WHEN in every aggregate:

```sql
-- DuckDB way
SELECT
    category,
    count() AS total,
    count() FILTER (WHERE status = 'active') AS active_count,
    sum(revenue) FILTER (WHERE channel = 'web') AS web_revenue,
    sum(revenue) FILTER (WHERE channel = 'mobile') AS mobile_revenue
FROM sales
GROUP BY ALL;

-- Anti-pattern (verbose)
SELECT
    category,
    count(*) AS total,
    sum(CASE WHEN status = 'active' THEN 1 ELSE 0 END) AS active_count,
    sum(CASE WHEN channel = 'web' THEN revenue ELSE 0 END) AS web_revenue,
    sum(CASE WHEN channel = 'mobile' THEN revenue ELSE 0 END) AS mobile_revenue
FROM sales
GROUP BY category;
```

### count() Shorthand
```sql
SELECT count() FROM orders;  -- same as count(*)
```

### Top-N Per Group
No window function subquery needed:

```sql
-- Get top 3 values per group
SELECT grp, max(value, 3) AS top_3_values FROM data GROUP BY ALL;
-- Returns: [3rd, 2nd, 1st] as an array

-- Also: min(col, n), arg_max(arg, val, n), arg_min(arg, val, n)
-- max_by(arg, val, n), min_by(arg, val, n)
```

### GROUPING SETS / ROLLUP / CUBE
Multi-level aggregation in one query:

```sql
-- Subtotals at multiple levels
SELECT region, product, sum(sales)
FROM data
GROUP BY ROLLUP (region, product);
-- Gives: (region, product), (region), () — grand total

-- All combinations
GROUP BY CUBE (region, product);

-- Custom grouping sets
GROUP BY GROUPING SETS ((region), (product), (region, product), ());
```

---

## Step 6: Table Transformation — PIVOT and UNPIVOT

### PIVOT (Long → Wide)
```sql
-- Transform year values into columns
PIVOT sales
ON year
USING sum(amount)
GROUP BY product;

-- With explicit values
PIVOT sales
ON quarter IN ('Q1', 'Q2', 'Q3', 'Q4')
USING sum(revenue)
GROUP BY region;
```

### UNPIVOT (Wide → Long)
```sql
-- Convert year columns back to rows
UNPIVOT pivoted_sales
ON COLUMNS(* EXCLUDE product)
INTO
    NAME year
    VALUE amount;
```

---

## Step 7: Nested Types — Lists and Structs

### List Creation and Operations
```sql
-- Create lists
SELECT ['apple', 'banana', 'cherry'] AS fruits;

-- List slicing (1-indexed)
SELECT fruits[1:2] FROM data;        -- first 2 elements
SELECT fruits[-2:] FROM data;        -- last 2 elements
SELECT fruits[2] FROM data;          -- single element

-- List comprehension
SELECT [x * 2 FOR x IN numbers IF x > 0] AS doubled_positives
FROM data;

-- list_transform
SELECT list_transform(prices, x -> x * 1.1) AS prices_with_tax FROM data;

-- list_filter
SELECT list_filter(tags, t -> t != 'spam') AS clean_tags FROM data;

-- Chained with dot syntax
SELECT tags.list_filter(t -> t.contains('urgent')).list_aggr('string_agg', ',') AS urgent_tags
FROM tickets;
```

### Struct Creation and Access
```sql
-- Create struct
SELECT {name: 'Alice', age: 30, city: 'NYC'} AS person;

-- Dot notation access
SELECT person.name, person.age FROM data;

-- Expand struct into columns
SELECT person.* FROM data;

-- Auto-struct from table alias
FROM users SELECT users;  -- returns each row as a struct
```

### MAP Type
```sql
SELECT MAP(['key1', 'key2'], [100, 200]) AS metrics;
```

### unnest() — Expand Lists to Rows
```sql
-- Flatten a list column into individual rows
SELECT id, unnest(tags) AS tag FROM posts;

-- Unnest multiple columns in parallel (row-aligned)
SELECT order_id, unnest(items) AS item, unnest(quantities) AS qty FROM orders;
```

---

## Step 8: Function Chaining

DuckDB's dot operator passes the left side as the first argument to the right function:

```sql
-- Chain string operations
SELECT
    ('  Hello World  ')
        .trim()
        .lower()
        .replace(' ', '_') AS slug;

-- Chain list operations
SELECT
    tags
        .list_filter(t -> t != '')
        .list_transform(t -> upper(t))
        .list_sort()
    AS clean_tags
FROM posts;

-- Chain on columns
SELECT
    description
        .regexp_extract('(\d+)', 1)::INTEGER AS extracted_number
FROM products;
```

---

## Step 9: Special Join Types

### ASOF Join — Time-Series Matching
For each row in the left table, find the closest (but not exceeding) matching row:

```sql
SELECT
    t.ticker,
    t.trade_time,
    t.shares,
    q.price,
    t.shares * q.price AS trade_value
FROM trades t
ASOF JOIN quotes q
    ON t.ticker = q.ticker
    AND t.trade_time >= q.quote_time;
```

### LATERAL Join — Correlated Subquery in FROM
```sql
-- Top 3 products per customer
SELECT c.name, top_prods.product_name, top_prods.total_spent
FROM customers c,
LATERAL (
    SELECT product_name, sum(amount) AS total_spent
    FROM orders
    WHERE customer_id = c.id
    GROUP BY ALL
    ORDER BY total_spent DESC
    LIMIT 3
) AS top_prods;
```

### POSITIONAL Join — Row-by-Row
```sql
-- Combine two tables of equal length by row position
SELECT a.*, b.score FROM predictions a POSITIONAL JOIN actuals b;
```

---

## Window Function Enhancements

### Named WINDOW Clause
Define reusable window specs to avoid repetition across multiple window functions:

```sql
SELECT
    date, symbol, price,
    avg(price) OVER w7  AS ma_7day,
    avg(price) OVER w30 AS ma_30day,
    sum(price) OVER w7  AS sum_7day
FROM prices
WINDOW
    w7  AS (PARTITION BY symbol ORDER BY date ROWS 6 PRECEDING),
    w30 AS (PARTITION BY symbol ORDER BY date ROWS 29 PRECEDING)
ORDER BY symbol, date;
```

### GROUPS Frame — Peer-Based Windows
Include all rows tied on the ORDER BY value (useful for rankings and tied scores):

```sql
SELECT name, score,
    avg(score) OVER (ORDER BY score GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS neighborhood_avg
FROM leaderboard;
```

### EXCLUDE Clause
Exclude specific rows from the window frame — removes the need for `WHERE id != self` tricks:

```sql
SELECT id, value,
    avg(value) OVER (
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        EXCLUDE CURRENT ROW   -- exclude self from the average
    ) AS avg_of_others
FROM data;
-- Also: EXCLUDE GROUP (all peers), EXCLUDE TIES, EXCLUDE NO OTHERS (default)
```

### fill() — Linear Interpolation for NULLs
DuckDB-specific function that fills NULL values via linear interpolation (extrapolates at edges):

```sql
-- Fill gaps in a sparse time series
SELECT date, fill(price) OVER (ORDER BY date) AS interpolated_price
FROM sparse_prices;
```

### Any Aggregate as a Window Function
All DuckDB aggregate functions work in window context, including list aggregates:

```sql
SELECT
    date, product, revenue,
    list(product) OVER (PARTITION BY date) AS all_products_that_day,
    string_agg(product, ', ') OVER (ORDER BY date ROWS 2 PRECEDING) AS recent_3_products
FROM sales;
```

---

## Step 10: Data Types and Identifiers

### Readable Number Literals
```sql
SELECT 1_000_000 AS one_million, 3_14159 AS approx_pi;
```

### Case-Insensitive Identifiers
DuckDB is case-insensitive but preserves original case for display:

```sql
CREATE TABLE MyTable AS SELECT 1 AS MyColumn;
SELECT mycolumn FROM mytable;  -- works fine
-- Displays as: MyColumn
```

### UNION Type — Multi-Type Column
```sql
CREATE TABLE events (
    payload UNION(
        click STRUCT(x INT, y INT),
        text  VARCHAR,
        count BIGINT
    )
);
```

### Trailing Commas
DuckDB allows trailing commas everywhere — useful when commenting out lines:

```sql
SELECT
    revenue,
    cost,
    -- margin,   -- commented out, trailing comma on cost is fine
FROM financials
GROUP BY ALL
ORDER BY ALL,
;
```

---

## Step 11: String Operations

### String Slicing
```sql
SELECT 'Hello World'[1:5];      -- 'Hello'
SELECT 'Hello World'[-5:];      -- 'World'
SELECT 'Hello World'[7:11];     -- 'World'
```

### String Formatting
```sql
SELECT format('{} has {} items', category, count()) FROM inventory GROUP BY ALL;
SELECT printf('%-20s: $%.2f', name, price) FROM products;
```

### Useful String Utilities
```sql
-- Filesystem path parsing (built-in, no regex needed)
SELECT
    parse_filename(file_path) AS filename,   -- 'report.csv'
    parse_dirname(file_path) AS dir,         -- '/data/2024'
    parse_dirpath(file_path) AS dirpath      -- '/data/2024/'
FROM file_log;

-- Human-readable sizes
SELECT format_bytes(file_size_bytes) AS size FROM files;
-- e.g. '1.5 GiB', '842.3 MiB'

-- URL encoding
SELECT url_encode('hello world & more') AS encoded;  -- 'hello+world+%26+more'

-- Unicode / accent handling
SELECT strip_accents('café naïve résumé');           -- 'cafe naive resume'
SELECT nfc_normalize(text) FROM documents;            -- normalize Unicode form
```

---

## Exploratory Analysis Workflow

When working with unfamiliar data, use this sequence:

### 1. Inspect the File
```sql
FROM sniff_csv('mystery.csv');              -- delimiter, types, sample SQL
DESCRIBE SELECT * FROM 'data.parquet';      -- schema for any file format
```

### 2. Cache Remote or Large Files First
**Always cache S3/GCS/HTTPS files before repeated queries.** Re-reading a remote file
on each query wastes time and money.

```sql
-- Cache once, query many times
CREATE OR REPLACE TABLE raw AS FROM 's3://bucket/large-dataset.parquet';

-- Multi-file with different schemas
CREATE OR REPLACE TABLE raw AS
FROM read_parquet('s3://bucket/data/**/*.parquet', union_by_name = true);
```

### 3. Profile the Data
```sql
SUMMARIZE raw;   -- min, max, avg, null count, unique count per column

-- Distribution of a key column
SELECT region, count() AS n,
    count() / sum(count()) OVER () * 100 AS pct
FROM raw GROUP BY ALL ORDER BY n DESC;
```

### 4. Iterate on the Cached Table
```sql
SELECT date_trunc('month', created_at) AS month,
    count() AS events, count(DISTINCT user_id) AS users
FROM raw
GROUP BY ALL
ORDER BY month;
```

### 5. Export Results
```sql
COPY (SELECT * FROM raw WHERE status = 'active') TO 'active.parquet' (FORMAT PARQUET);
COPY (FROM final_query) TO 'report.csv' (HEADER);
```

Consult `references/visualization.md` for SQL patterns to shape data for charts.

---

## CLI Quick Reference

See `references/cli.md` for the full CLI reference. Key patterns:

### One-Liner Queries
```bash
duckdb -c "SELECT * FROM 'data.csv' LIMIT 10"
duckdb -c "SUMMARIZE 'large.parquet'"
duckdb mydb.duckdb -f analysis.sql
duckdb -readonly mydb.duckdb
```

### Data Conversion
```bash
# CSV → Parquet
duckdb -c "COPY (FROM 'in.csv') TO 'out.parquet' (FORMAT PARQUET)"

# Parquet → CSV
duckdb -c "COPY (FROM 'in.parquet') TO 'out.csv' (HEADER)"

# Multiple files → merged
duckdb -c "COPY (FROM 'logs/*.csv') TO 'all.parquet' (FORMAT PARQUET)"
```

### Pipe to/from Unix Tools
```bash
cat data.csv | duckdb -c "SELECT * FROM read_csv('/dev/stdin') WHERE amount > 1000"
duckdb -csv -c "SELECT * FROM 'data.parquet'" | head -20
```

### Output Formats
```bash
duckdb -csv       # comma-separated
duckdb -json      # JSON array
duckdb -markdown  # Markdown table
```

### In-Shell Settings
```sql
.timer on          -- show query time
.mode markdown     -- switch output format
.maxrows 100       -- limit displayed rows
.once out.csv      -- next query to file
```

---

## Common Anti-Patterns to Avoid

Consult `references/anti-patterns.md` for a full list. Key ones:

| Anti-Pattern | DuckDB Way |
|---|---|
| `DROP TABLE IF EXISTS t; CREATE TABLE t AS ...` | `CREATE OR REPLACE TABLE t AS ...` |
| `GROUP BY col1, col2, col3` (all non-agg cols) | `GROUP BY ALL` |
| Subquery just to use an alias in WHERE | Column alias directly in WHERE |
| `SELECT col1, col2, col3, ..., colN` (all except one) | `SELECT * EXCLUDE (unwanted_col)` |
| `UNION` with careful column ordering | `UNION ALL BY NAME` |
| `count(*)` | `count()` |
| `CASE WHEN condition THEN 1 ELSE 0 END` in SUM | `count() FILTER (WHERE condition)` |
| Loading CSV then querying | `SELECT * FROM 'file.csv'` directly |
| Explicit GROUP BY listing all select columns | `GROUP BY ALL` |
| Window subquery just to get "best" column per group | `arg_max(col, val)` / `max_by(col, val)` |
| Subquery to filter window function result (full row) | `QUALIFY` clause |
| `ROW_NUMBER() = 1` subquery for dedup | `DISTINCT ON (col)` |
| Re-reading remote file in every query | `CREATE TABLE AS FROM 's3://...'` cache first |
| `CREATE INDEX` for range scans / analytics | Automatic zonemaps handle it |

---

## Examples

### Example 1: Analytics Query on CSV Files
User says: "Analyze sales by region and product from multiple CSV files"

```sql
-- Query multiple files directly, use GROUP BY ALL, count() shorthand
SELECT
    region,
    product_category,
    date_trunc('month', sale_date) AS month,
    count() AS num_sales,
    sum(amount) AS total_revenue,
    avg(amount) AS avg_order_value,
    count() FILTER (WHERE status = 'returned') AS returns
FROM 'sales/2024/*.csv'
GROUP BY ALL
ORDER BY total_revenue DESC;
```

### Example 2: Data Cleaning Pipeline
User says: "Clean and load a messy CSV into a table"

```sql
-- Inspect first
FROM sniff_csv('messy_data.csv');

-- Load with transformations
CREATE OR REPLACE TABLE clean_data AS
SELECT
    * EXCLUDE (tmp_id, raw_date),
    REPLACE (
        trim(lower(email)) AS email,
        strptime(raw_date, '%d/%m/%Y') AS created_date
    )
FROM read_csv('messy_data.csv', sample_size = -1)
WHERE email IS NOT NULL
    AND email LIKE '%@%';
```

### Example 3: Time-Series Join
User says: "Join trade data with the most recent quote prices"

```sql
CREATE OR REPLACE TABLE enriched_trades AS
SELECT
    t.*,
    q.bid_price,
    q.ask_price,
    (q.bid_price + q.ask_price) / 2 AS mid_price
FROM trades t
ASOF JOIN quotes q
    ON t.symbol = q.symbol
    AND t.timestamp >= q.timestamp;
```

### Example 4: Pivot Report
User says: "Show monthly revenue by product as a wide table"

```sql
PIVOT (
    SELECT product, date_trunc('month', sale_date)::VARCHAR AS month, amount
    FROM sales
)
ON month
USING sum(amount)
GROUP BY product
ORDER BY product;
```

### Example 5: List Processing with Lambdas
User says: "Filter and transform a list column"

```sql
SELECT
    order_id,
    items
        .list_filter(item -> item.quantity > 0)
        .list_transform(item -> {
            name: item.name,
            total: item.price * item.quantity
        }) AS active_line_items
FROM orders;
```

---

## Troubleshooting

**Error: CSV type mismatch**
Cause: Auto-detected type doesn't match actual data
Solution: Use `sniff_csv()` to inspect, then override with `types = {'col': 'VARCHAR'}`

**Error: Skill produces standard SQL GROUP BY instead of GROUP BY ALL**
Solution: Always explicitly use `GROUP BY ALL` — never enumerate non-aggregated columns

**Error: Duplicate column names after JOIN**
Cause: Both tables have same column name
Solution: DuckDB auto-renames to `col:1`, `col:2` — use `SELECT * EXCLUDE` or explicit aliases

**Error: Column alias not working in WHERE**
Cause: Using a JOIN ON clause (not supported) vs WHERE/GROUP BY/HAVING (supported)
Solution: Column aliases work in WHERE, GROUP BY, HAVING but NOT in JOIN ON conditions

**Error: Cannot use window function result in WHERE**
Cause: WHERE is evaluated before window functions
Solution: Use `QUALIFY` instead — it is evaluated after window functions

**Error: ART index creation fails on large table (OOM)**
Cause: ART indexes must fit entirely in memory during creation
Solution: Don't create indexes for analytics — DuckDB's automatic zonemaps handle range scans; ART indexes are only useful for highly selective point lookups (< 0.1% of rows)
