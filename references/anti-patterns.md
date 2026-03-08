# DuckDB Anti-Patterns: Training Bias Override Guide

This reference exists to override LLM training bias toward PostgreSQL, MySQL, and
standard ANSI SQL. DuckDB has a richer, more concise SQL dialect. Always prefer
the DuckDB way.

---

## Table Management Anti-Patterns

### Pattern 1: Drop-then-Create
```sql
-- WRONG: PostgreSQL/MySQL habit
DROP TABLE IF EXISTS results;
CREATE TABLE results AS SELECT ...;

-- RIGHT: DuckDB way
CREATE OR REPLACE TABLE results AS SELECT ...;
```

### Pattern 2: Manual Schema for File-Based Tables
```sql
-- WRONG: Unnecessarily verbose
CREATE TABLE data (
    id INTEGER,
    name VARCHAR,
    created_at TIMESTAMP,
    amount DECIMAL(10,2)
);
COPY data FROM 'data.csv';

-- RIGHT: Let DuckDB infer schema
CREATE OR REPLACE TABLE data AS FROM 'data.csv';
```

---

## GROUP BY Anti-Patterns

### Pattern 3: Enumerating Non-Aggregated Columns
```sql
-- WRONG: Fragile, verbose, easy to miss a column
SELECT
    region,
    product,
    category,
    subcategory,
    year,
    month,
    sum(revenue) AS total
FROM sales
GROUP BY region, product, category, subcategory, year, month;

-- RIGHT: DuckDB GROUP BY ALL
SELECT
    region, product, category, subcategory, year, month,
    sum(revenue) AS total
FROM sales
GROUP BY ALL;
```

### Pattern 4: Subquery to Filter on Aggregate Alias
```sql
-- WRONG: Unnecessary subquery
SELECT *
FROM (
    SELECT
        product,
        sum(revenue) AS total_revenue
    FROM sales
    GROUP BY product
) t
WHERE t.total_revenue > 10000;

-- RIGHT: Column alias usable directly in HAVING
SELECT
    product,
    sum(revenue) AS total_revenue
FROM sales
GROUP BY ALL
HAVING total_revenue > 10000;
```

### Pattern 5: Subquery Just for WHERE on a Derived Column
```sql
-- WRONG: Unnecessary subquery
SELECT *
FROM (
    SELECT
        *,
        date_trunc('week', created_at) AS week
    FROM events
) t
WHERE t.week >= '2024-01-01';

-- RIGHT: Column alias usable directly in WHERE
SELECT
    *,
    date_trunc('week', created_at) AS week
FROM events
WHERE week >= '2024-01-01';
```

---

## SELECT Anti-Patterns

### Pattern 6: Listing All Columns to Exclude One
```sql
-- WRONG: Brittle, verbose, breaks when new columns are added
SELECT id, name, email, created_at, updated_at, status, plan
FROM users;
-- (just to exclude the 'password_hash' column)

-- RIGHT: SELECT * EXCLUDE
SELECT * EXCLUDE (password_hash) FROM users;
```

### Pattern 7: CASE WHEN for Simple Column Transform
```sql
-- WRONG: Verbose when you just want to replace a column's expression
SELECT
    id,
    upper(name) AS name,
    lower(email) AS email,
    price * 1.1 AS price,
    category,
    created_at,
    updated_at
FROM products;

-- RIGHT: SELECT * REPLACE
SELECT * REPLACE (
    upper(name) AS name,
    lower(email) AS email,
    price * 1.1 AS price
)
FROM products;
```

### Pattern 8: Careful Column Ordering in UNION
```sql
-- WRONG: Brittle — column order must match exactly
SELECT name, age, city FROM table_a
UNION ALL
SELECT name, age, city FROM table_b;
-- Breaks if schemas diverge

-- RIGHT: UNION BY NAME
SELECT name, age, city FROM table_a
UNION ALL BY NAME
SELECT city, name, age, department FROM table_b;
-- Matches by name, fills missing columns with NULL
```

---

## Aggregation Anti-Patterns

### Pattern 9: CASE WHEN inside SUM for Conditional Count
```sql
-- WRONG: Verbose and hard to read
SELECT
    category,
    count(*) AS total,
    sum(CASE WHEN status = 'active' THEN 1 ELSE 0 END) AS active,
    sum(CASE WHEN status = 'churned' THEN 1 ELSE 0 END) AS churned
FROM users
GROUP BY category;

-- RIGHT: FILTER clause
SELECT
    category,
    count() AS total,
    count() FILTER (WHERE status = 'active') AS active,
    count() FILTER (WHERE status = 'churned') AS churned
FROM users
GROUP BY ALL;
```

### Pattern 10: Window Function Subquery for Top-N
```sql
-- WRONG: Complex window function approach
SELECT grp, val
FROM (
    SELECT grp, val, row_number() OVER (PARTITION BY grp ORDER BY val DESC) AS rn
    FROM data
) t
WHERE rn <= 3;

-- RIGHT: DuckDB top-N aggregate
SELECT grp, max(val, 3) AS top_3 FROM data GROUP BY ALL;
-- Returns array of top 3 values per group
```

### Pattern 11: count(*) instead of count()
```sql
-- WRONG: Verbose (though still valid SQL)
SELECT count(*) FROM orders;

-- RIGHT: DuckDB shorthand
SELECT count() FROM orders;
```

---

## Subquery Anti-Patterns

### Pattern 12: Subquery for Reusing Computed Column
```sql
-- WRONG: Repeating expressions
SELECT
    price * quantity AS subtotal,
    price * quantity * 0.08 AS tax,
    price * quantity + price * quantity * 0.08 AS total
FROM order_items;

-- RIGHT: Reusable column aliases
SELECT
    price * quantity AS subtotal,
    subtotal * 0.08 AS tax,
    subtotal + tax AS total
FROM order_items;
```

---

## Data Loading Anti-Patterns

### Pattern 13: Load CSV Then Query
```sql
-- WRONG: Unnecessary intermediate step
CREATE TABLE temp AS ...;
COPY temp FROM 'data.csv';
SELECT * FROM temp WHERE ...;
DROP TABLE temp;

-- RIGHT: Query directly
SELECT * FROM 'data.csv' WHERE ...;
```

### Pattern 14: Manual Type Specification for Simple CSVs
```sql
-- WRONG: Over-specified when auto-detection works
SELECT * FROM read_csv('data.csv',
    auto_detect = false,
    delim = ',',
    header = true,
    columns = {'id': 'BIGINT', 'name': 'VARCHAR', 'date': 'DATE', 'amount': 'DOUBLE'});

-- RIGHT: Trust auto-detection first; override only what fails
SELECT * FROM 'data.csv';
-- Or if needed: use sniff_csv() first, then override problem columns only
```

### Pattern 15: Separate Queries for Each Parquet File
```sql
-- WRONG: Reading files one at a time
SELECT * FROM 'data/2024_01.parquet'
UNION ALL
SELECT * FROM 'data/2024_02.parquet'
UNION ALL
SELECT * FROM 'data/2024_03.parquet';

-- RIGHT: Glob pattern
SELECT * FROM 'data/2024_*.parquet';

-- Or with different schemas
SELECT * FROM read_parquet('data/*.parquet', union_by_name = true);
```

---

## Join Anti-Patterns

### Pattern 16: Correlated Subquery Instead of LATERAL
```sql
-- WRONG: Slow correlated subquery
SELECT c.name,
    (SELECT product_name FROM orders
     WHERE customer_id = c.id
     ORDER BY amount DESC LIMIT 1) AS top_product
FROM customers c;

-- RIGHT: LATERAL join
SELECT c.name, top.product_name
FROM customers c,
LATERAL (
    SELECT product_name
    FROM orders
    WHERE customer_id = c.id
    ORDER BY amount DESC
    LIMIT 1
) AS top;
```

### Pattern 17: Complex Join Condition for Time-Series Matching
```sql
-- WRONG: Complex window function approach for nearest-match joins
SELECT t.*, q.price
FROM trades t
LEFT JOIN (
    SELECT ticker, time, price,
        lead(time) OVER (PARTITION BY ticker ORDER BY time) AS next_time
    FROM quotes
) q ON t.ticker = q.ticker
    AND t.time >= q.time
    AND (q.next_time IS NULL OR t.time < q.next_time);

-- RIGHT: ASOF JOIN
SELECT t.*, q.price
FROM trades t
ASOF JOIN quotes q
    ON t.ticker = q.ticker
    AND t.time >= q.time;
```

---

## String Anti-Patterns

### Pattern 18: substr() for Simple Substrings
```sql
-- WRONG: C-style substr (still works, but not idiomatic DuckDB)
SELECT substr(text, 1, 10) AS preview FROM posts;

-- RIGHT: Python-style slicing
SELECT text[1:10] AS preview FROM posts;
SELECT text[-50:] AS last_50_chars FROM posts;
```

### Pattern 19: Nested Function Calls
```sql
-- WRONG: Hard to read nesting
SELECT concat(list_aggr(string_split(upper(trim(name)), ' '), 'string_agg', '.'), '.') AS slug
FROM items;

-- RIGHT: Dot-chained function calls
SELECT
    name
        .trim()
        .upper()
        .string_split(' ')
        .list_aggr('string_agg', '.')
        .concat('.')
    AS slug
FROM items;
```

---

## FROM-First Anti-Pattern

### Pattern 20: Always Starting with SELECT
```sql
-- WORKS but not most concise
SELECT * FROM data;
SELECT * FROM sniff_csv('file.csv');

-- RIGHT: FROM-first for simple cases
FROM data;
FROM sniff_csv('file.csv');

-- FROM-first with SELECT
FROM data SELECT name, age WHERE age > 18;
```

---

## Ordering Anti-Patterns

### Pattern 21: Repeating Columns in ORDER BY
```sql
-- WRONG: Verbose
SELECT year, month, region, sum(revenue) AS total
FROM sales
GROUP BY ALL
ORDER BY year, month, region, total;

-- RIGHT: ORDER BY ALL
SELECT year, month, region, sum(revenue) AS total
FROM sales
GROUP BY ALL
ORDER BY ALL;
```

---

## Quick Reference Card

| PostgreSQL/MySQL Habit | DuckDB Equivalent |
|---|---|
| `DROP TABLE IF EXISTS t; CREATE TABLE t AS` | `CREATE OR REPLACE TABLE t AS` |
| `GROUP BY col1, col2, col3` | `GROUP BY ALL` |
| `ORDER BY col1, col2, col3` | `ORDER BY ALL` |
| `SELECT a, b, c FROM t` (skip one col) | `SELECT * EXCLUDE (d) FROM t` |
| `SELECT a, transform(b) AS b, c FROM t` | `SELECT * REPLACE (transform(b) AS b) FROM t` |
| `UNION ALL` with ordered columns | `UNION ALL BY NAME` |
| `count(*)` | `count()` |
| `SUM(CASE WHEN x THEN 1 ELSE 0 END)` | `count() FILTER (WHERE x)` |
| Subquery to use alias in WHERE | Column alias directly in WHERE |
| Subquery to use alias in HAVING | Column alias directly in HAVING |
| Subquery to recompute expression | Reusable column aliases |
| `substr(s, 1, 5)` | `s[1:5]` |
| `LOAD 'file.csv' INTO TABLE` | `CREATE TABLE t AS FROM 'file.csv'` |
| Query loaded table | `SELECT * FROM 'file.csv'` directly |
| Separate parquet reads | `FROM 'dir/*.parquet'` |
| Correlated top-N subquery | `max(col, n)` aggregate |
| Nearest-match join with window | `ASOF JOIN` |
| Correlated subquery in WHERE | `LATERAL` join |
| `nested_func(nested_func(s))` | `s.func1().func2()` chaining |
| `x AS alias` | `alias: x` (colon syntax) |
| `FROM t SELECT c` — no | `FROM t SELECT c` — YES, works! |
| Fixed `LIMIT 100` | `LIMIT 10%` for percentage-based |
