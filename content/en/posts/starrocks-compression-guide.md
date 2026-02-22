---
title: "StarRocks Compression Guide: Optimizing Performance and Storage"
date: 2026-02-23
draft: false
author: "Seungwoo Noh"
categories: [StarRocks, Data Engineering]
tags: [starrocks, compression, optimization]
ShowToc: true
TocOpen: false
summary: "A practical guide to optimizing compression settings in StarRocks. Covers algorithm comparison, per-workload recommendations, benchmark results, and operational tips from production experience."
cover:
  image: ""
---

## Why Compression Settings Matter

When you operate StarRocks long enough, there will inevitably come a point where your data grows to tens of terabytes. I have seen, time and again, cases where a single compression setting made a 30 to 50 percent difference in storage costs. But this is not just about disk space. Higher compression ratios reduce disk I/O and improve scan performance, while overly aggressive compression burns CPU and increases latency. Ultimately, choosing the right compression algorithm for your workload is one of the most important tuning decisions in StarRocks operations.

## Comparing Supported Compression Algorithms

StarRocks supports several compression algorithms. Here is a comparison of the three most commonly used in practice.

| Algorithm | Compression Ratio | Compression Speed | Decompression Speed | Best-fit Workload |
|-----------|-------------------|-------------------|---------------------|-------------------|
| LZ4 | Moderate (2-3x) | Very fast | Very fast | Real-time analytics, low-latency queries |
| ZSTD | High (4-6x) | Moderate | Fast | Batch analytics, cold data |
| Snappy | Low (1.5-2x) | Fast | Fast | General purpose, legacy compatibility |
| ZLIB | High (4-5x) | Slow | Moderate | Archiving, infrequently accessed data |

Personally, the combination I use most often is LZ4 for hot data and ZSTD for cold data. I occasionally use Snappy when dealing with data migrated from the Hadoop ecosystem, but I do not recommend it for new tables.

## Setting Compression When Creating a Table

You can specify the compression algorithm via the `compression` property in `PROPERTIES` when creating a table. If no value is set, StarRocks defaults to LZ4.

### Real-time Analytics Table (LZ4)

```sql
CREATE TABLE analytics.realtime_events (
    event_id       BIGINT,
    user_id        BIGINT,
    event_type     VARCHAR(64),
    event_time     DATETIME,
    properties     JSON
)
ENGINE = OLAP
DUPLICATE KEY(event_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 32
PROPERTIES (
    "replication_num" = "3",
    "compression" = "LZ4"
);
```

LZ4 has overwhelmingly fast decompression speed, making it ideal for tables that need to serve dashboard queries with sub-second response times.

### Batch Analytics Table (ZSTD)

```sql
CREATE TABLE warehouse.order_history (
    order_id       BIGINT,
    customer_id    BIGINT,
    order_date     DATE,
    total_amount   DECIMAL(18, 2),
    status         VARCHAR(32),
    items          ARRAY<STRUCT<sku STRING, qty INT, price DECIMAL(10,2)>>
)
ENGINE = OLAP
DUPLICATE KEY(order_id)
PARTITION BY RANGE(order_date) (
    PARTITION p2025 VALUES LESS THAN ('2026-01-01'),
    PARTITION p2026 VALUES LESS THAN ('2027-01-01')
)
DISTRIBUTED BY HASH(customer_id) BUCKETS 16
PROPERTIES (
    "replication_num" = "2",
    "compression" = "ZSTD"
);
```

ZSTD achieves 1.5 to 2 times higher compression ratios than LZ4, which translates into significant storage savings on history tables where hundreds of millions of rows accumulate per partition.

### Changing Compression on an Existing Table

If you want to change the compression algorithm on a table that is already in production, you can use `ALTER TABLE`. Keep in mind, however, that the new setting only applies to data loaded after the change. Existing segments will not be recompressed until a compaction cycle runs against them.

```sql
ALTER TABLE warehouse.order_history
SET ("compression" = "ZSTD");
```

## Recommended Compression Settings by Workload

The following recommendations are based on patterns I have validated repeatedly in production environments.

- **Real-time dashboards / Ad-hoc queries:** Use LZ4. Its CPU overhead is negligible, minimizing the impact on P99 latency.
- **Nightly batch reports / ETL result tables:** Use ZSTD. When query frequency is low and data volume is high, the storage savings translate directly into cost reductions.
- **High-volume log ingestion:** Use ZSTD, but lower `zstd_compression_level` to 3 or below. This strikes a good balance between compression speed and compression ratio.

## Benchmark Results: Compression Ratio vs. Performance Tradeoffs

The following benchmark was run against an event log table containing approximately 5 billion rows (roughly 800 GB uncompressed).

| Metric | LZ4 | ZSTD (level 3) | ZSTD (level 9) |
|--------|-----|----------------|----------------|
| Compressed size | 320 GB | 195 GB | 170 GB |
| Compression ratio | 2.5x | 4.1x | 4.7x |
| Simple scan query (Avg) | 1.2 s | 1.5 s | 1.8 s |
| Aggregation query (Avg) | 3.4 s | 3.8 s | 4.5 s |
| Data ingestion throughput | 120 MB/s | 95 MB/s | 60 MB/s |

Compared to LZ4, ZSTD level 3 reduced storage by approximately 39% while only increasing query latency by about 10 to 15 percent. ZSTD level 9, on the other hand, offered diminishing compression gains at the cost of a steep drop in ingestion throughput, making level 3 the optimal choice in most environments.

## Operational Tips and Monitoring

Here are three tips from production experience:

1. **Always monitor compaction.** Keep an eye on the `compaction_score` metric. When compression settings change or data volumes spike, compaction can fall behind, leading to degraded query performance from an excessive number of small segments.
2. **Separate compression strategies at the table level.** Do not apply a single compression algorithm across your entire cluster. Match the algorithm to each table's access pattern -- LZ4 for frequently queried tables, ZSTD for archival ones.
3. **Track disk usage trends over time.** Use `SHOW DATA` to monitor how each table's storage footprint evolves after compression changes.

```sql
SHOW DATA FROM warehouse.order_history;
```

Compression is not a set-it-and-forget-it decision. As data volumes grow and query patterns shift, it is worth revisiting your compression settings periodically. Even a small adjustment can yield meaningful improvements in both performance and infrastructure costs over time.
