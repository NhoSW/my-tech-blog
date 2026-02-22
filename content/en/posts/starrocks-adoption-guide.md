---
title: "StarRocks Adoption Story: Revolutionizing Data Pipelines with Real-time OLAP"
date: 2026-02-23
draft: false
author: "Seungwoo Noh"
categories: [StarRocks, Data Engineering]
tags: [starrocks, olap, real-time, kafka, performance-tuning, data-pipeline]
ShowToc: true
TocOpen: true
summary: "How we reduced pipeline latency from 5 minutes to sub-second by adopting StarRocks. Covers table model selection, data ingestion, performance tuning, and operational lessons from production."
cover:
  image: ""
---

## Background

If you run data pipelines long enough, you inevitably face one question: **How do you build real-time dashboards?**

Our team was no different. Our existing pipeline looked like this:

```
Service → Kafka → Iceberg → S3 → Trino → Airflow(5min) → Dashboard
```

On the surface it worked fine, but the pain points in practice were clear:

- **At least 5 minutes of latency**: The Airflow schedule interval was the bottleneck
- **Pipeline complexity**: Managing 5+ components chained as Kafka → Flink → Redis → API → Dashboard
- **Redundant I/O**: Trino full-scanned S3 on every query
- **High development cost**: Each new real-time dashboard took roughly 2 weeks to build

After adopting StarRocks, the architecture simplified to this:

```
Service → Kafka → StarRocks → Dashboard (sub-second latency)
```

Eliminating the intermediate components dramatically simplified the pipeline, and ingesting data directly from Kafka into StarRocks gave us the real-time capability we needed.

## Results

After approximately 3 months of PoC and 6 months of phased rollout, we achieved the following improvements:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Dashboard latency | 5 min | < 1 sec | ~300x |
| Dashboard dev time | ~2 weeks | ~1 week | 50% reduction |
| Pipeline components | 5+ | 2 | 60% reduction |
| Query response time | 30~50 sec | 5~10 sec | 5~10x |
| Hardware cost | 128 GB x 18 nodes | 64 GB x 3 nodes | ~75% savings |

> Trino is fast in terms of raw query time, but when factoring in Airflow schedule delays and **end-to-end latency**, along with hardware cost efficiency, StarRocks proved to be a better fit for real-time workloads.

## Table Model Selection Guide

Choosing the right table model is the most important decision when first adopting StarRocks. A wrong choice means you will have to recreate the table later.

### Decision Flow

```
┌─────────────────────────────┐
│  What data are you storing?  │
└──────────────┬──────────────┘
               │
       ┌───────▼────────┐
       │  Need UPDATE?   │
       └───────┬────────┘
               │
      ┌────────┴────────┐
      │                 │
     [No]              [Yes]
      │                 │
┌─────▼─────┐    ┌─────▼──────┐
│  Need      │    │ Primary Key │
│aggregation?│    │ (frequent   │
└─────┬─────┘    │  UPDATE)    │
      │          └────────────┘
  [No]    [Yes]
      │      │
┌─────▼───┐ ┌▼──────────┐
│Duplicate│ │Aggregate   │
│(raw data)│ │(auto-agg) ★│
└─────────┘ └────────────┘
```

### Model Comparison

| Model | Duplicates Allowed | UPDATE | Auto-aggregation | Best For |
|-------|--------------------|--------|------------------|----------|
| Duplicate Key | O | X | X | Logs, raw events |
| Aggregate Key | X | Auto | O | Real-time statistics ★ |
| Primary Key | X | O (fast) | X | Frequent UPDATEs |

### Duplicate Key: Storing Raw Data

Use this when you need to preserve original data as-is, such as click logs, API events, or sensor data.

```sql
CREATE TABLE order_events (
    event_id BIGINT,
    event_time DATETIME,
    order_id VARCHAR(50),
    user_id BIGINT,
    event_type VARCHAR(20),
    amount DECIMAL(10, 2)
)
DUPLICATE KEY(event_id, event_time)
PARTITION BY date_trunc('day', event_time)
DISTRIBUTED BY HASH(event_id) BUCKETS 10;
```

### Aggregate Key: Real-time Statistics ★

Data is **automatically aggregated at ingestion time**. This model was the key reason for adopting StarRocks.

```sql
CREATE TABLE order_stats (
    stat_time DATETIME NOT NULL COMMENT '5-minute intervals',
    region VARCHAR(20) NOT NULL,
    delivery_type VARCHAR(20) NOT NULL,
    -- Aggregate columns: aggregate functions applied automatically at ingestion
    order_count BIGINT SUM DEFAULT "0",
    total_amount DECIMAL(15, 2) SUM DEFAULT "0",
    user_bitmap BITMAP BITMAP_UNION,
    max_amount DECIMAL(10, 2) MAX
)
AGGREGATE KEY(stat_time, region, delivery_type)
PARTITION BY date_trunc('day', stat_time)
DISTRIBUTED BY HASH(stat_time) BUCKETS 10;
```

Available aggregate functions:

| Function | Purpose | Example |
|----------|---------|---------|
| `SUM` | Summation | Order count, total revenue |
| `MAX` / `MIN` | Maximum / minimum value | Highest price, lowest price |
| `REPLACE` | Overwrite with latest value | Latest status |
| `BITMAP_UNION` | Exact unique count | Unique visitors |
| `HLL_UNION` | Approximate unique count | High-cardinality sets |

> Unlike HyperLogLog, `BITMAP_UNION` provides **exact** unique counts. For business KPI dashboards where accuracy matters, always use this approach.

### Primary Key: Frequent UPDATEs

Best suited for scenarios where the same key is frequently updated, such as order status tracking or inventory management.

```sql
CREATE TABLE orders (
    order_id VARCHAR(50) NOT NULL,
    status VARCHAR(20),
    amount DECIMAL(10, 2),
    updated_at DATETIME
)
PRIMARY KEY(order_id)
DISTRIBUTED BY HASH(order_id) BUCKETS 10
PROPERTIES (
    "enable_persistent_index" = "true"
);
```

> Enabling `enable_persistent_index` significantly improves UPDATE performance.

## Data Ingestion

### Routine Load: Real-time Kafka Integration

This approach continuously ingests data from a Kafka topic. It is the method used in most real-time pipelines.

```sql
CREATE ROUTINE LOAD order_load ON orders
COLUMNS(
    order_id,
    user_id,
    timestamp_ms,
    amount,
    order_date = FROM_UNIXTIME(timestamp_ms / 1000)
)
PROPERTIES (
    "format" = "json",
    "jsonpaths" = "[\"$.orderId\",\"$.userId\",\"$.timestamp\",\"$.amount\"]"
)
FROM KAFKA (
    "kafka_broker_list" = "kafka-broker:9092",
    "kafka_topic" = "orders"
);
```

When combined with an Aggregate Key table, **transformation and aggregation happen simultaneously at ingestion time**.

```sql
CREATE ROUTINE LOAD order_stats_load ON order_stats
COLUMNS(
    timestamp_ms,
    region,
    amount,
    user_id,
    -- Round to 5-minute intervals
    stat_time = FROM_UNIXTIME(FLOOR(timestamp_ms / 1000 / 300) * 300),
    order_count = 1,
    total_amount = amount,
    user_bitmap = BITMAP_HASH(user_id)
)
WHERE amount > 0
PROPERTIES ("format" = "json")
FROM KAFKA (
    "kafka_broker_list" = "kafka-broker:9092",
    "kafka_topic" = "orders"
);
```

This single pattern replaced the aggregation logic that previously required Flink -- using nothing but SQL.

### Stream Load: Bulk Data Loading

Ideal for one-time bulk loads via files or APIs.

```bash
# CSV file loading
curl --location-trusted \
  -u user:password \
  -H "label:load_$(date +%Y%m%d%H%M%S)" \
  -H "column_separator:," \
  -T data.csv \
  http://starrocks-fe:8030/api/mydb/mytable/_stream_load
```

## Performance Tuning Tips

### Thread Pool Configuration

In high-load environments with 500+ RPS of concurrent connections, the default thread pool size is insufficient.

```properties
# be.conf
pipeline_scan_thread_pool_thread_num = 32   # default: 24
pipeline_exec_thread_pool_thread_num = 32   # default: 24
```

### Bucket Count Guidelines

| Data Size | Recommended Buckets |
|-----------|---------------------|
| < 10 GB | 10 |
| 10~50 GB | 20 |
| 50~100 GB | 30 |
| > 100 GB | 50+ |

Formula: `buckets = max(1, data_size_GB / 10)`

### Partitioning Strategy

Applying functions to partition columns prevents partition pruning. This is a more common mistake than you might think.

```sql
-- ✅ Correct: partition pruning works
WHERE event_time >= NOW() - INTERVAL 3 DAY

-- ❌ Incorrect: partition pruning disabled
WHERE DATE(event_time) >= CURRENT_DATE - 3
```

### TTL Configuration

To automatically drop old partitions, configure TTL.

```sql
PROPERTIES (
    "partition_live_number" = "3"  -- Keep only the 3 most recent partitions
)
```

## Operational Know-how

### Materialized View Management

ASYNC refresh can stop without warning. Periodically check the status and manually recover when issues arise.

```sql
-- Check status
SHOW MATERIALIZED VIEWS;

-- Force synchronous refresh
REFRESH MATERIALIZED VIEW db.mv_name WITH SYNC MODE;

-- Reactivate a deactivated MV
ALTER MATERIALIZED VIEW db.mv_name ACTIVE;
```

### Routine Load Monitoring

The status frequently transitions to `PAUSED`. Common causes include Kafka offset issues or malformed messages.

```sql
-- Check status
SHOW ROUTINE LOAD FOR db.load_job;

-- Resume
RESUME ROUTINE LOAD FOR db.load_job;
```

### Scale-in Precautions

When scaling down nodes, you **must perform a Decommission first**. Removing nodes without this procedure will result in data loss.

```sql
-- 1. Check current nodes
SHOW PROC '/backends';

-- 2. Start decommission
ALTER SYSTEM DECOMMISSION BACKEND "<BE_IP>:<HEARTBEAT_PORT>";

-- 3. Wait until TabletNum reaches 0, then remove
ALTER SYSTEM DROP BACKEND "<BE_IP>:<HEARTBEAT_PORT>";
```

## Things to Know Before Adopting

### Known Limitations

| Issue | Description | Workaround |
|-------|-------------|------------|
| Routine Load | Limited handling of malformed messages | Pre-validate on the Kafka side |
| datetime partitions | Compatibility issues with Iceberg datetime partitions | Use an alternative partitioning strategy |
| Version upgrades | Encountered bugs in 4.x releases | Always test in a staging environment |

> Always thoroughly validate version upgrades in a staging environment before applying them to production. We went through several rounds of upgrades and rollbacks ourselves. Have a rollback plan ready at all times.

### Adoption Checklist

**Pre-deployment**
- [ ] Define use cases and requirements
- [ ] Estimate data volume and growth rate
- [ ] Choose table models
- [ ] Design partitioning strategy

**Post-deployment**
- [ ] Create and verify Routine Load jobs
- [ ] Configure user permissions
- [ ] Set data retention policies (TTL)
- [ ] Document scale-in/out procedures
- [ ] Build monitoring dashboards

## Conclusion

Here are the key lessons we learned from adopting StarRocks:

1. **The Aggregate Key model is the centerpiece** -- Automatic aggregation at ingestion time optimizes both storage and query performance
2. **Use BITMAP_UNION for exact unique counts** -- Business KPIs demand precise numbers, not approximations
3. **Routine Load + Aggregate Key replaces Flink** -- You can build a real-time aggregation pipeline with SQL alone
4. **Invest in operational automation** -- Monitoring Materialized Views and Routine Load is essential

For real-time analytics workloads, StarRocks is a powerful option that dramatically reduces pipeline complexity. That said, it is still maturing in terms of version upgrade stability and operational robustness, so we recommend conducting thorough PoC testing and staging validation before adopting it.

**Reference**: [StarRocks Official Docs](https://docs.starrocks.io)
