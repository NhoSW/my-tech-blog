---
title: "Trino Query Optimization: Practical Tips for Better Performance"
date: 2026-02-23
draft: false
author: "Seungwoo Noh"
categories: [Trino, Data Engineering]
tags: [trino, sql, optimization, performance]
ShowToc: true
TocOpen: false
summary: "Practical tips for optimizing Trino queries in production. Covers partition pruning, join strategies, data types, and query profiling techniques."
cover:
  image: ""
---

Trino is a powerful distributed SQL engine, but throwing queries at it without thinking about how it executes them is a fast track to slow dashboards, failed queries, and frustrated stakeholders. After running Trino in production against multi-terabyte data lakes for several years, I have collected a set of optimization patterns that consistently make a real difference. This post covers the ones I reach for most often.

## 1. Partition Pruning and Predicate Pushdown

The single highest-impact optimization is making sure Trino reads as little data as possible. If your tables are partitioned -- and they should be -- always filter on the partition column explicitly.

**Bad -- full table scan:**

```sql
SELECT user_id, event_type, created_at
FROM hive.analytics.events
WHERE created_at >= TIMESTAMP '2026-01-01 00:00:00';
```

If `dt` is the partition column (a date string like `2026-01-15`) and `created_at` is a timestamp column inside the data files, Trino cannot use the `created_at` filter to skip partitions. It has to open every partition and scan.

**Optimized -- partition column in the WHERE clause:**

```sql
SELECT user_id, event_type, created_at
FROM hive.analytics.events
WHERE dt >= '2026-01-01'
  AND created_at >= TIMESTAMP '2026-01-01 00:00:00';
```

Adding the `dt` predicate lets Trino prune partitions at the metastore level before reading a single byte of data. I have seen this alone reduce query times from 10+ minutes to under 30 seconds.

The same principle applies to predicate pushdown into connectors. Filters on columns that exist in file-level metadata (Parquet min/max statistics, for example) allow Trino to skip entire row groups. Keep your predicates simple and on the columns the storage layer knows about.

## 2. Join Optimization

Trino supports two join distribution strategies: **broadcast** and **partitioned (distributed)**. The choice matters enormously.

- **Broadcast join:** The smaller table is copied to every worker node. Great when one side is small (< a few hundred MB).
- **Distributed join:** Both sides are hash-partitioned across workers. Required when both tables are large.

Trino's cost-based optimizer usually picks well, but when statistics are stale or absent, it guesses wrong. You can guide it:

```sql
-- Force a broadcast join when you know the dimension table is small
SELECT /*+ BROADCAST(d) */
    f.order_id, f.amount, d.region_name
FROM hive.warehouse.fact_orders f
JOIN hive.warehouse.dim_regions d
    ON f.region_id = d.region_id;
```

**Join ordering** also matters. Filter early, join late. Push filters and aggregations as close to the base tables as possible so that intermediate result sets are small before they hit the join.

**Bad -- join first, filter later:**

```sql
SELECT o.order_id, o.amount, c.segment
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.dt = '2026-02-01' AND c.segment = 'enterprise';
```

**Better -- pre-filter with CTEs for clarity and to help the optimizer:**

```sql
WITH filtered_orders AS (
    SELECT order_id, amount, customer_id
    FROM orders
    WHERE dt = '2026-02-01'
),
enterprise_customers AS (
    SELECT customer_id, segment
    FROM customers
    WHERE segment = 'enterprise'
)
SELECT o.order_id, o.amount, c.segment
FROM filtered_orders o
JOIN enterprise_customers c ON o.customer_id = c.customer_id;
```

In practice, Trino's optimizer can often push predicates down through joins on its own. But making the intent explicit with CTEs both helps readability and gives the planner a clearer picture, especially in complex multi-join queries.

## 3. Data Type and Format Considerations

Your file format and schema design decisions are made long before query time, but they have a direct impact on performance.

- **Use columnar formats.** Parquet and ORC both support column projection and predicate pushdown via min/max statistics. Avoid JSON and CSV for analytical tables.
- **Select only the columns you need.** With columnar formats, every column you add to your SELECT is an additional I/O cost.

**Bad:**

```sql
SELECT *
FROM hive.analytics.events
WHERE dt = '2026-02-01';
```

**Optimized:**

```sql
SELECT user_id, event_type, created_at
FROM hive.analytics.events
WHERE dt = '2026-02-01';
```

This is basic, but I still see `SELECT *` in production dashboards that read 200-column tables when only 4 columns are used. On a wide Parquet table the difference can be 10-20x in I/O.

Also watch your data types. Joining on a `VARCHAR` column when both sides could be `BIGINT` adds unnecessary overhead from hashing and comparison. If you control the pipeline, cast to the most efficient type upstream.

## 4. Query Profiling with EXPLAIN and Query Stats

Before optimizing blindly, measure. Trino gives you two essential tools:

**EXPLAIN** shows the logical and distributed plan:

```sql
EXPLAIN
SELECT user_id, COUNT(*) AS cnt
FROM hive.analytics.events
WHERE dt = '2026-02-01'
GROUP BY user_id;
```

Look for `ScanFilterProject` to verify predicate pushdown is happening. Check whether the join strategy is `REPLICATED` (broadcast) or `PARTITIONED` (distributed). If the plan shows a full `TableScan` with no filter, your predicates are not being pushed down.

**EXPLAIN ANALYZE** actually runs the query and gives you runtime statistics:

```sql
EXPLAIN ANALYZE
SELECT user_id, COUNT(*) AS cnt
FROM hive.analytics.events
WHERE dt = '2026-02-01'
GROUP BY user_id;
```

This shows wall time, rows processed, data read per stage, and memory usage. I make it a habit to run `EXPLAIN ANALYZE` on any new query before it goes into a scheduled pipeline. The Trino Web UI also exposes per-query stats, stage-level timing, and memory breakdowns -- all valuable for diagnosing bottlenecks.

## 5. Resource Management Tips

A single expensive query can starve an entire cluster. A few configuration and usage practices help:

- **Use resource groups** to isolate workloads. Give ETL pipelines, ad-hoc analysts, and dashboards separate concurrency and memory limits so they do not interfere with each other.
- **Limit concurrent queries** per user or group. Even well-optimized queries add up when 50 of them run simultaneously.
- **Set session-level memory limits** for exploratory work:

```sql
SET SESSION query_max_memory = '2GB';
```

This prevents an accidental cross join from consuming the entire cluster's memory before it gets killed.

- **Avoid `ORDER BY` without `LIMIT`** on large result sets. Sorting requires materializing the full result in memory. If you just need the top N rows, always add `LIMIT`.

## Final Thoughts

Most Trino performance wins come from reading less data: prune partitions, select fewer columns, filter early. After that, getting join strategies right and profiling with `EXPLAIN ANALYZE` covers the majority of remaining issues. None of these tips are exotic -- they are fundamentals that compound. Getting them right consistently is what separates a query that finishes in 3 seconds from one that times out.
