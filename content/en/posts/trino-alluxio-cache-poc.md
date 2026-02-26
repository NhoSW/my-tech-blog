---
title: "Trino Alluxio Cache PoC: EBS Throughput Was the Bottleneck"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [trino, alluxio, cache, s3, ebs, kubernetes, performance]
showTableOfContents: true
summary: "We ran a PoC of Trino's Alluxio-based file system cache in a production OLAP environment. Adding the cache alone didn't help much. The default EBS throughput of 125 MiB/s was the bottleneck. After bumping it to 1000 MiB/s, query performance improved visibly and S3 API costs dropped by ~$3,200/month."
---

We were running Trino as the primary query engine for our OLAP workloads. While evaluating ClickHouse as an alternative, we decided that improving Trino was the more practical path — keeping the tech stack unified is worth a lot when you're the team maintaining it. So we upgraded Trino from v433 to v451. The main goal was to use the Alluxio-based file system cache.

The previous Rubix cache had been deprecated. Alluxio cache was added in v439 and had a race condition bug fixed in v445, making it production-ready.

This post covers the PoC process and the bottleneck we didn't expect.

---

## What Alluxio Cache Does

Every time Trino reads data from S3, it goes over the network. Even if the same file is read multiple times, each read triggers an S3 API call. Alluxio cache stores fetched data on the worker node's local disk so subsequent reads come from local storage.

A few lines in the catalog config enable it.

```properties
fs.cache.enabled=true
fs.cache.max-sizes=50GB
fs.cache.directories=/mnt/cache
```

It works with Hive, Iceberg, and Delta Lake connectors. We enabled it on two catalogs: hive_zeppelin and iceberg.

---

## Constraints

Before starting the PoC, there were several constraints worth noting.

### Cache Is Not Shared Across Catalogs or Clusters

Even if multiple catalogs point at the same S3 bucket, each catalog maintains its own cache. The connector architecture instantiates the file system independently per catalog. Cross-cluster sharing is out of the question.

A native Alluxio file system PR was merged in September 2024 (Trino 460). It enables cache sharing between catalogs and even between clusters. Whether accessing a shared cache file system is actually faster than hitting S3 directly still needs separate validation.

### Spot Instances and Caching Don't Mix Well

Cache data lifetime is tied to the worker node lifetime. Node goes away, cache goes with it.

Our environment runs nearly 99% on spot instances. Spot reclamation wipes the cache on that worker. Scale-in from autoscaling does the same. Cache warm-up takes time, and if nodes churn frequently, you barely get to benefit from it.

### No Schema-Level Control

Cache activation is all-or-nothing at the catalog level. Tables under `temp` schemas that are almost never reused still get cached. Selective caching for specific schemas would be ideal, but it's not supported yet.

A PR to add `fs.cache.skip-paths` existed but was closed due to design disagreements. Reviewers argued that control should happen at the schema/table level, not the file path level.

---

## PoC Setup

### Apply to One Side of Blue/Green Only

Building a staging environment identical to production wasn't feasible cost-wise. Instead, we applied the cache to only the Blue cluster in both our OLAP and BI setups, then compared query performance metrics over 1-2 weeks.

A Trino Gateway sits in front, distributing queries based on the current query count per backend. The queries hitting Blue vs. Green aren't identical, but over two weeks the sample size is large enough for a meaningful comparison.

### Volume Configuration

Mapping one worker pod to one node is the Trino best practice. You could use StatefulSet with volumeClaimTemplate for per-pod volumes, but we chose to mount a dedicated EBS cache volume on each node and access it via hostPath from the worker pod.

Initial cache volume: 50 GB per worker. 60% allocated to the hive_zeppelin catalog, 30% to iceberg.

### Rollout Order

1. OLAP Blue cluster (2024-07-30)
2. BI Blue cluster (2024-07-31)
3. Added Grafana panels for p25/p50/p75/p99/avg/max query execution time comparison
4. Monitored cache utilization via JMX metrics

BI clusters were expected to benefit more from caching. Dashboard queries tend to hit the same tables with the same date ranges across multiple charts.

---

## The EBS Throughput Bottleneck

We turned on the cache and watched for a few days. No dramatic difference. Cache hit rates looked fine, so why wasn't performance improving?

We dug into the cache volume metrics. **EBS write throughput was constantly hitting the configured maximum of 125 MiB/s.** The gp3 volume's baseline throughput is 125 MiB/s, and cache writes were saturating it.

It doesn't matter how fast the cache is if the disk can't keep up with writes.

### Bumping Throughput

We raised it to 1000 MiB/s, the maximum configurable throughput for gp3.

- BI cluster: applied around 2:45 PM
- OLAP cluster: applied around 3:45 PM

The effect was immediate. After the throughput bump, the cache-enabled cluster (Blue) showed clearly better query performance. Write throughput reached around 200 MiB/s at peak but had plenty of headroom under the 1000 MiB/s ceiling. Combined read + write throughput was roughly 300 MiB/s.

We confirmed that io2 volumes weren't necessary — io2 caps out at the same 1000 MiB/s maximum throughput, and we didn't need the high IOPS it offers.

---

## Cache Volume Sizing

The initial 50 GB filled up fast. The allocated maximum (hive 60% + iceberg 30% = 45 GB) was fully utilized. When cache space runs out, the oldest data gets evicted. If eviction happens too aggressively, cache effectiveness drops.

We scaled up the volumes.

| Cluster | Before | After |
|---------|--------|-------|
| BI workers | 50 GB | 150 GB → 1.5 TB |
| OLAP workers | 50 GB | 2 TB |
| OLAP coordinator | - | 1 TB (for iceberg metadata cache) |

We also adjusted the per-catalog allocation ratios based on actual usage. Hive was consuming far more cache than iceberg, so we shifted from hive 60% / iceberg 30% to hive 85% / iceberg 10%.

EBS storage cost was a concern, but it turned out to be small. The ~20% reduction in worker count from improved cache performance more than offset the additional EBS cost.

Our approach: overprovision initially, add Grafana charts for per-node cache volume utilization, observe, then trim if wasteful.

---

## FIFO Can Beat LRU

Trino's Alluxio cache doesn't use a sophisticated eviction algorithm like LRU. It uses FIFO — first cached, first evicted.

Intuitively LRU seems better, but FIFO has advantages in certain scenarios.

- **Removes one-hit wonders quickly.** Data read once and never again gets evicted fast. LRU would update the access timestamp, potentially keeping such data around longer.
- **SSD-friendly.** FIFO minimizes random access patterns. On SSD/flash storage, this means less write amplification.

The simple eviction policy reflects the feature's early stage, but for OLAP workloads, FIFO is not necessarily a bad choice.

---

## Results

### Query Performance

After enabling the cache and bumping EBS throughput, the Blue cluster was consistently faster than Green.

The interesting part: **Blue recorded higher query throughput with fewer workers.** Because Blue processed queries faster, the gateway routed more queries to it. Blue ended up handling more total queries while running ~20% fewer workers than Green.

Cache warm-up took about 3 hours. Performance differences only became visible after workers had been up long enough to fill their caches.

### S3 API Cost Reduction

S3 GetObject costs dropped by **roughly $3,200/month** after cache activation. Other factors like query volume changes may have contributed, but the cost reduction was substantial.

### Cost Summary

| Item | Change |
|------|--------|
| S3 API costs | ~$3,200/month reduction |
| EBS storage costs | Small increase (cache volumes) |
| Node costs | Reduction from ~20% fewer workers |

---

## Instance Store Consideration

EBS is remote block storage. Compared to direct S3 access, the cache performance improvement might not be dramatic. If EBS caching hadn't shown sufficient results, we planned to switch to node types with NVMe SSD instance stores.

| Instance Type | Instance Store |
|--------------|----------------|
| r7gd.4xlarge | 1 x 950 GB NVMe SSD |
| m7gd.8xlarge | 1 x 1,900 GB NVMe SSD |

The two types have different instance store sizes, but Trino's cache config supports percentage-based sizing relative to total disk capacity, so this isn't a problem. In the end, bumping EBS throughput alone was sufficient, and we didn't proceed with the instance store switch.

---

## Bonus: Project Hummingbird

While upgrading to v451, we noticed another development worth watching. Project Hummingbird is Trino's ongoing performance improvement initiative using Java 22's Vector API for vectorized execution.

Vectorized decoding for Parquet file reads landed in v448. It requires 256-bit or larger vector registers, so it's disabled on Graviton 2 instances (r6g, m6g). Graviton 3 and later enable it automatically.

Trino requiring Java 22 starting from v447 is part of this same initiative.

---

## Takeaways

The PoC taught us one clear lesson.

**Adding a cache isn't enough.** If disk I/O is the bottleneck, cache hit rate doesn't matter. The default EBS gp3 throughput of 125 MiB/s was the ceiling for cache performance. The moment we raised it to 1000 MiB/s, the difference showed.

**BI workloads benefit most from caching.** Dashboard queries that repeatedly hit the same tables see high cache hit rates.

**Caching on spot instances requires compromise.** Frequent node churn means cache warm-up time is effectively wasted. Still, after the 3-hour warm-up period, the effect was there. Not useless, but not ideal.

Looking ahead, the native Alluxio file system (Trino 460) will enable cross-catalog and cross-cluster cache sharing once it stabilizes. That could also mitigate cache loss from spot reclamation.

**References:**
- [A cache refresh for Trino](https://trino.io/blog/2024/03/08/cache-refresh.html)
- [Trino File System Cache Documentation](https://trino.io/docs/current/object-storage/file-system-cache.html)
- [Alluxio cache PR (v439)](https://github.com/trinodb/trino/pull/18719)
- [Cache race condition fix (v445)](https://github.com/trinodb/trino/pull/21350)
- [Cross-catalog cache sharing issue](https://github.com/trinodb/trino/issues/22815)
- [Native Alluxio file system PR (v460)](https://github.com/trinodb/trino/pull/21603)
- [Project Hummingbird](https://github.com/trinodb/trino/issues/14237)
- [Vectorized Parquet decoding PR (v448)](https://github.com/trinodb/trino/pull/21465)
