# DuckDB Friendly SQL — Complete Cheatsheet

Quick syntax reference for every DuckDB SQL extension. Use when you need to recall
exact syntax without reading the full SKILL.md.

---

## Table Operations

```sql
-- Create or replace (avoids DROP TABLE)
CREATE OR REPLACE TABLE t AS SELECT ...;
CREATE OR REPLACE TABLE t AS FROM 'file.csv';

-- CTAS (infer schema)
CREATE TABLE t AS SELECT ...;

-- Insert by column name (not position)
INSERT INTO t BY NAME SELECT 'val' AS col_name;

-- Conflict handling
INSERT OR IGNORE INTO t VALUES (...);
INSERT OR REPLACE INTO t VALUES (...);

-- Inspect
DESCRIBE t;                  -- schema
SUMMARIZE t;                 -- statistics
SUMMARIZE SELECT ... FROM t; -- statistics on query result
```

---

## FROM-First Syntax

```sql
FROM t;                      -- = SELECT * FROM t
FROM t SELECT col1, col2;    -- = SELECT col1, col2 FROM t
FROM 'file.csv';             -- = SELECT * FROM 'file.csv'
COPY (FROM t) TO 'out.parquet';
```

---

## SELECT Modifiers

```sql
SELECT * EXCLUDE (col1, col2) FROM t;
SELECT * REPLACE (expr1 AS col1, expr2 AS col2) FROM t;

-- COLUMNS() — dynamic selection
SELECT COLUMNS('regex') FROM t;
SELECT COLUMNS(col -> col LIKE '%pattern%') FROM t;
SELECT max(COLUMNS(* EXCLUDE id)) FROM t;
SELECT COLUMNS(* REPLACE (col::DOUBLE AS col)) FROM t;
```

---

## Aliases

```sql
-- Prefix alias (colon syntax)
SELECT alias: expression FROM t;

-- Column alias reuse in same SELECT
SELECT a + b AS sum, sum * 2 AS doubled FROM t;

-- Alias use in WHERE, GROUP BY, HAVING (not JOIN ON!)
SELECT date_trunc('month', ts) AS m, count() AS n
FROM t
WHERE m >= '2024-01-01'
GROUP BY m
HAVING n > 100;
```

---

## GROUP BY / ORDER BY

```sql
GROUP BY ALL;                -- group by all non-aggregated columns
ORDER BY ALL;                -- order by all selected columns
ORDER BY ALL DESC;
```

---

## UNION

```sql
q1 UNION ALL BY NAME q2;     -- match by column name, fill missing with NULL
q1 UNION BY NAME q2;         -- deduplicate
```

---

## Aggregation

```sql
count()                      -- = count(*)
agg() FILTER (WHERE cond)    -- conditional aggregate

-- Top-N functions
max(col, n)                  -- top N values as array
min(col, n)                  -- bottom N values as array
arg_max(arg, val, n)         -- arg for top N vals
arg_min(arg, val, n)
max_by(arg, val, n)
min_by(arg, val, n)

-- Multi-level grouping
GROUP BY ROLLUP (a, b)       -- (a,b), (a), ()
GROUP BY CUBE (a, b)         -- (a,b), (a), (b), ()
GROUP BY GROUPING SETS ((a), (b), (a,b), ())
```

---

## PIVOT / UNPIVOT

```sql
PIVOT t ON col USING agg(val) GROUP BY grp;
PIVOT t ON col IN ('v1','v2') USING sum(val) GROUP BY grp;

UNPIVOT t
ON COLUMNS(* EXCLUDE grp)
INTO NAME col_name VALUE val_name;
```

---

## Joins

```sql
-- ASOF: nearest-match (great for time-series)
t1 ASOF JOIN t2 ON t1.key = t2.key AND t1.ts >= t2.ts;

-- LATERAL: correlated subquery in FROM
FROM t1, LATERAL (SELECT ... WHERE col = t1.id) AS sub;

-- POSITIONAL: row-by-row match
t1 POSITIONAL JOIN t2;
```

---

## Lists

```sql
[1, 2, 3]                               -- list literal
list[1]                                 -- first element (1-indexed)
list[2:4]                               -- slice (inclusive)
list[-1]                                -- last element
list[-3:]                               -- last 3 elements

list_transform(l, x -> expr)            -- map
list_filter(l, x -> cond)               -- filter
list_filter(l, (x, i) -> i != 2)       -- filter with index
list_reduce(l, (acc, x) -> expr)        -- reduce
list_reduce(l, (acc, x, i) -> expr)    -- reduce with index

[expr FOR x IN l]                       -- list comprehension
[expr FOR x IN l IF cond]              -- comprehension with filter

-- Dot chaining on lists
l.list_filter(x -> x > 0).list_sort()
```

---

## Structs

```sql
{key: val, key2: val2}                  -- struct literal
s.field                                 -- field access
s."Field Name"                          -- quoted field (for names with spaces)
s.*                                     -- expand struct into columns
SELECT t FROM my_table t;               -- auto-create struct per row
```

---

## MAP

```sql
MAP([k1, k2], [v1, v2])                -- map literal
MAP {'k1': v1, 'k2': v2}               -- alternative syntax
```

---

## UNION Type

```sql
CREATE TABLE t (col UNION(tag1 TYPE1, tag2 TYPE2));
union_tag(col)                          -- get active type tag
col.tag1                                -- extract by tag
```

---

## Function Chaining

```sql
expr.func()                             -- = func(expr)
expr.func(arg2, arg3)                   -- = func(expr, arg2, arg3)
expr.func1().func2()                    -- chained

-- Examples
'hello'.upper()                         -- 'HELLO'
text.trim().lower().replace(' ', '_')
list.list_filter(x -> x > 0).list_sort()
```

---

## Strings

```sql
str[1:5]                                -- substring (1-indexed)
str[-3:]                                -- last 3 chars
str[:-2]                                -- all but last 2 chars

format('{} {}', a, b)                   -- Python-style format
printf('%s: %.2f', label, value)        -- C-style format
```

---

## Numbers

```sql
1_000_000                               -- underscore separator
0xFF                                    -- hex literal
0b1010                                  -- binary literal
```

---

## Direct File Queries

```sql
FROM 'file.csv';
FROM 'file.parquet';
FROM 'file.json';
FROM 'file.xlsx';
FROM 'dir/*.parquet';
FROM 'dir/**/*.csv';
FROM 's3://bucket/path/*.parquet';
FROM 'https://example.com/data.parquet';
FROM 'az://container/path/*.csv';
FROM 'gcs://bucket/data.parquet';
```

---

## Misc

```sql
LIMIT 10%                               -- percentage limit
SELECT col1, col2,  FROM t              -- trailing comma OK
GROUP BY a, b,;                         -- trailing comma OK in GROUP BY
[1, 2, 3,]                             -- trailing comma OK in lists
```

---

## Auto-Features

| Feature | Behavior |
|---|---|
| Case sensitivity | Identifiers are case-insensitive, original case preserved in output |
| Duplicate columns after JOIN | Auto-renamed to `col:1`, `col:2` |
| JSON parsing | Automatically converted to native LIST/STRUCT types |
| Implicit casts | `INTEGER` ↔ `VARCHAR` casts happen automatically in JOINs |
| CSV headers | Auto-detected |
| CSV types | Auto-inferred from 20,480 row sample |
| File schema | Auto-inferred for CSV, Parquet, JSON |
| Friendly errors | Suggests similar table/column names on typos |

---

## QUALIFY and DISTINCT ON

```sql
-- QUALIFY: filter on window function result (no subquery needed)
SELECT * FROM t
QUALIFY row_number() OVER (PARTITION BY grp ORDER BY ts DESC) = 1;

QUALIFY rank() OVER (ORDER BY score DESC) <= 10;
QUALIFY sum(revenue) OVER (PARTITION BY region) > 10000;

-- DISTINCT ON: one row per group
SELECT DISTINCT ON (user_id) user_id, event_type, ts
FROM events
ORDER BY user_id, ts DESC;    -- ORDER BY determines which row is kept
```

---

## Window Functions

```sql
-- Named WINDOW clause (reusable specs)
SELECT avg(v) OVER w7, sum(v) OVER w30
FROM t
WINDOW
    w7  AS (PARTITION BY grp ORDER BY date ROWS 6 PRECEDING),
    w30 AS (PARTITION BY grp ORDER BY date ROWS 29 PRECEDING);

-- GROUPS frame (peer-based, not row-count-based)
avg(v) OVER (ORDER BY score GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING)

-- EXCLUDE clause
avg(v) OVER (ROWS UNBOUNDED PRECEDING EXCLUDE CURRENT ROW)
-- Also: EXCLUDE GROUP, EXCLUDE TIES, EXCLUDE NO OTHERS

-- fill(): linear interpolation for NULLs (DuckDB-specific)
fill(value) OVER (ORDER BY date)

-- All aggregates work as window functions
list(product) OVER (PARTITION BY date)
string_agg(name, ', ') OVER (ORDER BY id ROWS 3 PRECEDING)
```

---

## unnest()

```sql
unnest(list_col)                          -- expand list to rows
SELECT id, unnest(tags) AS tag FROM t;    -- one row per tag per id

-- Parallel unnest (zip behavior)
SELECT unnest(names) AS name, unnest(scores) AS score FROM t;
```

---

## Struct Manipulation

```sql
-- Create
{name: 'Alice', age: 30}                  -- struct literal
struct_pack(name := 'Alice', age := 30)   -- named struct

-- Access
s.field_name                              -- dot notation
s['field_name']                           -- bracket notation
s[1]                                      -- by position (1-based)
s.*                                       -- expand all fields to columns

-- Modify
struct_insert(s, new_field := val)        -- add field
struct_update(s, field := new_val)        -- add or update field
struct_concat(s1, s2)                     -- merge two structs

-- Query
struct_contains(s, 'field')              -- field exists?
struct_keys(s)                            -- list of field names
struct_extract(s, 'field')               -- extract by name
```

---

## Vector Math (List Functions)

```sql
-- Similarity / distance (for embeddings, ML features)
list_cosine_similarity(v1, v2)            -- range [-1, 1]
list_cosine_distance(v1, v2)              -- 1 - cosine_similarity
list_inner_product(v1, v2)                -- dot product
list_distance(v1, v2)                     -- Euclidean distance

-- Sorting / ranking
list_grade_up(list)                       -- argsort (ascending index order)
list_select(values, indices)              -- fancy indexing by index list

-- Set operations
list_intersect(l1, l2)
list_has_all(list, sublist)               -- all elements present?
list_has_any(list, sublist)               -- any element present?
list_distinct(list)                       -- unique elements

-- Aggregation over list elements
list_sum(list)
list_avg(list)
list_min(list) / list_max(list)
list_string_agg(list, sep)
list_aggregate(list, 'any_agg_name')      -- apply any aggregate function
```

---

## Key Datetime Functions

```sql
-- Truncate / extract
date_trunc('month', ts)                   -- truncate to precision
date_part('year', ts)                     -- extract component (= extract)
extract('dow' FROM ts)                    -- day of week (0=Sun)

-- Arithmetic
DATE '2024-01-15' + 5                     -- add days (returns DATE)
DATE '2024-01-15' + INTERVAL 5 DAY        -- add interval (returns TIMESTAMP)
DATE '2024-01-20' - DATE '2024-01-15'     -- difference in days (integer)
date_diff('month', d1, d2)               -- count boundaries crossed
age(ts1, ts2)                             -- human-readable interval

-- Current time
today()  /  current_date                  -- date only
now()    /  current_timestamp             -- with time

-- Parse / format
strptime(str, '%d/%m/%Y')                -- string → timestamp
try_strptime(str, fmt)                   -- returns NULL instead of error
strftime(ts, '%Y-%m-%d')                 -- timestamp → string

-- Binning
time_bucket(INTERVAL '15 minutes', ts)   -- temporal bucketing
generate_series(d1, d2, INTERVAL 1 DAY) -- inclusive date range
range(d1, d2, INTERVAL 1 DAY)           -- exclusive date range

-- Helpers
last_day(date)                            -- last day of month
dayname(date)                             -- 'Monday', 'Tuesday', ...
monthname(date)                           -- 'January', 'February', ...
```

---

## CLI Quick Reference

```bash
duckdb                                    # in-memory
duckdb mydb.duckdb                        # persistent
duckdb -readonly mydb.duckdb              # read-only
duckdb -c "SELECT * FROM 'f.csv' LIMIT 5" # one-liner
duckdb -csv / -json / -markdown           # set output format
duckdb mydb.duckdb -f script.sql          # run SQL file

# Conversion
duckdb -c "COPY (FROM 'in.csv') TO 'out.parquet' (FORMAT PARQUET)"
duckdb -c "COPY (FROM 'in.parquet') TO 'out.csv' (HEADER)"

# Pipe
cat f.csv | duckdb -c "SELECT * FROM read_csv('/dev/stdin')"
duckdb -csv -c "FROM t" | head -20
```

Dot commands: `.tables`, `.schema`, `.mode FORMAT`, `.timer on`, `.output file`, `.once file`, `.read file.sql`, `.edit`

See `references/cli.md` for full reference.
