# DuckDB Visualization Query Patterns

SQL templates for generating chart-ready data. DuckDB doesn't render charts — it produces
the aggregated, shaped result sets that visualization libraries (Plotly, Vega, Chart.js,
Observable, matplotlib, etc.) consume.

---

## Schema → Chart Type Guide

Run `SUMMARIZE my_table` or `DESCRIBE my_table` first, then pick chart type by column types:

| Column Types | Recommended Chart |
|---|---|
| date/timestamp + numeric | Time series / Line chart |
| categorical + numeric | Bar chart |
| two categoricals | Heatmap / Frequency table |
| two numerics | Scatter plot |
| one numeric | Histogram / Distribution |
| numeric → cumulative | Area / Funnel chart |
| ranked numeric | Leaderboard / Top-N table |

---

## Time Series

```sql
-- Daily aggregation
SELECT
    date_trunc('day', event_time) AS date,
    count() AS events,
    sum(revenue) AS daily_revenue
FROM events
GROUP BY ALL
ORDER BY date;

-- Monthly with multiple metrics
SELECT
    date_trunc('month', created_at) AS month,
    count() FILTER (WHERE plan = 'free') AS free_signups,
    count() FILTER (WHERE plan = 'pro') AS pro_signups,
    count() FILTER (WHERE plan = 'enterprise') AS enterprise_signups
FROM users
GROUP BY ALL
ORDER BY month;

-- Rolling average (7-day window)
SELECT
    date_trunc('day', event_time) AS date,
    sum(revenue) AS daily_revenue,
    avg(sum(revenue)) OVER (
        ORDER BY date_trunc('day', event_time)
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS ma_7day
FROM events
GROUP BY ALL
ORDER BY date;

-- Indexed to first period (100 = baseline)
WITH monthly AS (
    SELECT date_trunc('month', ts) AS month, sum(revenue) AS rev
    FROM sales GROUP BY ALL ORDER BY month
)
SELECT
    month,
    rev,
    rev / first_value(rev) OVER (ORDER BY month) * 100 AS indexed
FROM monthly;

-- Fill missing dates so charts have no gaps
WITH spine AS (
    SELECT generate_series AS date
    FROM generate_series(DATE '2024-01-01', DATE '2024-12-31', INTERVAL 1 DAY)
),
daily AS (
    SELECT date_trunc('day', event_time) AS date, count() AS events
    FROM events GROUP BY ALL
)
SELECT d.date, coalesce(e.events, 0) AS events
FROM spine d
LEFT JOIN daily e USING (date)
ORDER BY date;
```

---

## Bar Charts

```sql
-- Simple bar: category → value (sorted descending)
SELECT
    category,
    count() AS n,
    sum(revenue) AS total_revenue
FROM sales
GROUP BY ALL
ORDER BY total_revenue DESC
LIMIT 20;

-- Grouped bar (two dimensions)
SELECT
    region,
    product_type,
    sum(revenue) AS revenue
FROM sales
GROUP BY ALL
ORDER BY region, revenue DESC;

-- Stacked bar: pivot to wide format (for libraries that need one column per stack)
PIVOT (
    SELECT region, product_type, sum(revenue) AS revenue
    FROM sales GROUP BY ALL
)
ON product_type
USING sum(revenue)
GROUP BY region
ORDER BY region;

-- Percent of total per group (normalized stacked bar)
SELECT
    region,
    product_type,
    sum(revenue) AS revenue,
    sum(revenue) / sum(sum(revenue)) OVER (PARTITION BY region) * 100 AS pct_of_region
FROM sales
GROUP BY ALL
ORDER BY region, revenue DESC;

-- Waterfall: running totals for variance charts
SELECT
    category,
    revenue,
    sum(revenue) OVER (ORDER BY category ROWS UNBOUNDED PRECEDING) AS running_total
FROM (SELECT category, sum(revenue) AS revenue FROM sales GROUP BY ALL)
ORDER BY category;
```

---

## Scatter Plots

```sql
-- Basic X vs Y
SELECT
    avg_session_length AS x,
    ltv AS y,
    user_segment AS color
FROM user_metrics
ORDER BY x;

-- Bubble chart (X, Y, size)
SELECT
    avg_revenue AS x,
    retention_rate AS y,
    customer_count AS size,
    segment AS label
FROM segment_stats
ORDER BY customer_count DESC;

-- Correlation coefficients (quick overview)
SELECT
    corr(price, rating)    AS price_vs_rating,
    corr(price, reviews)   AS price_vs_reviews,
    corr(rating, reviews)  AS rating_vs_reviews
FROM products;

-- Scatter with linear trend line
WITH stats AS (
    SELECT
        avg(x_col) AS mx, avg(y_col) AS my,
        corr(x_col, y_col) * stddev(y_col) / stddev(x_col) AS slope
    FROM data
),
reg AS (SELECT my - slope * mx AS intercept, slope FROM stats)
SELECT
    x_col AS x,
    y_col AS y,
    (SELECT intercept + slope * x_col FROM reg) AS trend
FROM data
ORDER BY x;
```

---

## Heatmaps

```sql
-- Frequency heatmap: two categoricals
SELECT
    category_a,
    category_b,
    count() AS frequency
FROM events
GROUP BY ALL
ORDER BY category_a, category_b;

-- Hour-of-day × day-of-week activity heatmap
PIVOT (
    SELECT
        hour(event_time) AS hour,
        dayname(event_time) AS day,
        count() AS events
    FROM events
    GROUP BY ALL
)
ON day IN ('Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday')
USING sum(events)
GROUP BY hour
ORDER BY hour;

-- Correlation matrix (long format for heatmap libraries)
SELECT column_a, column_b, correlation FROM (
    SELECT 'price'  AS column_a, 'rating'  AS column_b, corr(price, rating)   AS correlation FROM products
    UNION ALL BY NAME
    SELECT 'price',              'reviews',              corr(price, reviews)  FROM products
    UNION ALL BY NAME
    SELECT 'rating',             'reviews',              corr(rating, reviews) FROM products
);
```

---

## Histograms and Distributions

```sql
-- Equal-width histogram (20 buckets)
WITH bounds AS (SELECT min(amount) AS lo, max(amount) AS hi FROM transactions)
SELECT
    round(floor((amount - lo) / (hi - lo) * 20) / 20 * (hi - lo) + lo, 2) AS bucket_start,
    count() AS frequency
FROM transactions, bounds
GROUP BY ALL
ORDER BY bucket_start;

-- Fixed-width bins (e.g. $100 buckets)
SELECT
    floor(amount / 100) * 100 AS bucket,
    count() AS frequency
FROM transactions
GROUP BY ALL
ORDER BY bucket;

-- Percentiles / box plot data
SELECT
    quantile_cont(amount, 0.00) AS p0_min,
    quantile_cont(amount, 0.25) AS p25,
    quantile_cont(amount, 0.50) AS median,
    quantile_cont(amount, 0.75) AS p75,
    quantile_cont(amount, 1.00) AS p100_max,
    avg(amount) AS mean,
    stddev(amount) AS stddev
FROM transactions;

-- Box plot by group
SELECT
    category,
    quantile_cont(amount, [0.25, 0.50, 0.75]) AS quartiles,
    avg(amount) AS mean,
    stddev(amount) AS stddev
FROM transactions
GROUP BY ALL;
```

---

## Cumulative and Funnel Charts

```sql
-- Cumulative revenue over time (area chart)
SELECT
    date,
    daily_revenue,
    sum(daily_revenue) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING) AS cumulative
FROM daily_sales
ORDER BY date;

-- Conversion funnel
SELECT
    step,
    users,
    users::FLOAT / first_value(users) OVER (ORDER BY step_order) * 100 AS conversion_pct,
    users::FLOAT / lag(users) OVER (ORDER BY step_order) * 100 AS step_pct
FROM funnel_data
ORDER BY step_order;
```

---

## Ranking / Top-N Tables

```sql
-- Top 10 leaderboard (QUALIFY avoids subquery)
SELECT
    name,
    revenue,
    rank() OVER (ORDER BY revenue DESC) AS rank
FROM sales_by_rep
QUALIFY rank <= 10
ORDER BY rank;

-- Top 3 per category
SELECT category, product, revenue
FROM sales
QUALIFY row_number() OVER (PARTITION BY category ORDER BY revenue DESC) <= 3
ORDER BY category, revenue DESC;

-- Rank with percent of total
SELECT
    product,
    revenue,
    revenue / sum(revenue) OVER () * 100 AS pct_of_total,
    rank() OVER (ORDER BY revenue DESC) AS rank
FROM product_sales
QUALIFY rank <= 20
ORDER BY rank;
```

---

## Quick Reference

| Chart | Key SQL Techniques |
|-------|-------------------|
| Line / time series | `date_trunc()`, `GROUP BY ALL`, window `avg()` for smoothing |
| Bar | `GROUP BY ALL`, `ORDER BY metric DESC`, `LIMIT N` |
| Stacked / grouped bar | `PIVOT ... ON category` for wide format |
| Scatter | Select two numeric columns, optional third for bubble size |
| Heatmap | Two `GROUP BY` dimensions + `count()` or `PIVOT` for wide format |
| Histogram | `floor(val / bucket) * bucket` for bucketing |
| Box plot | `quantile_cont(col, [0.25, 0.5, 0.75])` |
| Funnel | `first_value()` window for normalization |
| Cumulative | Window `sum()` with `ROWS UNBOUNDED PRECEDING` |
| Top-N | `rank()` + `QUALIFY rank <= N` |
