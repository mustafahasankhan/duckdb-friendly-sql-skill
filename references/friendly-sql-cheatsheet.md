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
