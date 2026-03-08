---
title: "AWS EC2 Instance Architecture Comparison: ARM Graviton4 vs AMD Turin — Is the Fastest Instance the Best Choice?"
date: 2026-03-09
draft: false
categories: [Data Engineering]
tags: [aws, ec2, graviton, arm, amd, spot-instance, trino, spark, karpenter, cost-optimization]
showTableOfContents: true
summary: "In the 2026 cloud VM benchmarks, AMD EPYC Turin (C8a) dominated both single-threaded and multi-threaded performance. Should we migrate our Graviton (ARM) data platform infrastructure to C8a? After analyzing the vCPU vs physical core distinction, Spot price-to-core efficiency, and workload characteristics — the conclusion is that a hybrid strategy beats a full migration."
---

Our data platform team runs Trino, Spark, Flink, StarRocks, and other large-scale distributed processing workloads on AWS ap-northeast-2 (Seoul region). We've been using Graviton (ARM) based EC2 instances as our primary compute, achieving 10–20% lower cost than x86 with comparable or better performance.

The 2026 cloud VM benchmarks showed AMD EPYC Turin-based AWS C8a instances taking a decisive first place in both single-threaded and multi-threaded performance. This raised the question: should we migrate our ARM infrastructure to C8a?

The short answer: a full migration isn't cost-optimal. Here's why.

---

## vCPU ≠ Physical Core: The Most Important, Most Overlooked Distinction

The first concept to understand when comparing instances is the difference between vCPUs and physical cores. Without this understanding, you can buy two instances with the same 32 vCPU count and get 2x difference in actual compute capacity.

Most Intel and AMD instances enable SMT (Simultaneous Multi-Threading). This technology splits one physical core into 2 logical threads (vCPUs). It improves throughput by utilizing idle time during memory access waits, but since both threads share the same physical resources, CPU-bound workloads don't see 2x performance gains.

| Instance | Architecture | SMT | Physical Cores at 32 vCPU |
|----------|-------------|-----|--------------------------|
| C8g (Graviton4) | ARM | None | **32 Cores** |
| C8a (AMD Turin) | x86 | **Disabled** | **32 Cores** |
| C7i (Intel Sapphire Rapids) | x86 | Enabled | **16 Cores** |
| C7g (Graviton3) | ARM | None | **32 Cores** |

C8a has SMT intentionally disabled by AWS — 1 vCPU = 1 physical core. Graviton never had SMT. But C7i (Intel) with 32 vCPUs only provides 16 physical cores.

Spark executors receive vCPUs via `--executor-cores`. On an HT instance, 4 vCPUs means only 2 physical cores — half the actual parallel compute capacity. Trino worker task parallelism also scales with physical core count. For CPU-bound operations like shuffle, sort, hash join, and aggregation, physical core count directly determines throughput.

---

## Benchmarks: Turin Dominates — But Context Matters

In the 2026 benchmarks, AMD Turin (C8a) took first place in nearly every category. Relative performance normalized to Graviton3 = 100:

| Instance | Single-Thread | Multi-Thread |
|----------|--------------|-------------|
| C8a (AMD Turin) | **145** | **160** |
| C8g (Graviton4) | 125 | 130 |
| C7i (Intel Sapphire Rapids) | 110 | 105 |
| C7g (Graviton3) | 100 | 100 |

Turin is 16% faster than Graviton4 in single-thread and 23% faster in multi-thread. Compared to Graviton3, the gap is 45–60%. Looking at numbers alone, switching to C8a seems obvious.

### Workload-Specific Nuances

Turin doesn't dominate every benchmark uniformly.

**7-zip decompression**: Graviton4 and Graviton3 can outperform Turin in decompression scenarios. For Iceberg file scans heavy on Parquet decompression, ARM can actually be faster.

**OpenSSL RSA4096 (AVX512)**: Turin > Genoa > Intel. ARM doesn't support AVX512 and ranks lower. However, Trino and Spark have low AVX512 dependency. StarRocks BE vector aggregation is an exception that can benefit from AVX512.

**NGINX single-thread**: C8a shows nearly 2x performance over 2nd place and 3x over 3rd. Meaningful for services like Trino coordinator where single-thread response time directly impacts query latency.

---

## Pricing: Cost Per Core Is What Matters, Not Raw Performance

We've confirmed C8a is the fastest. The question is price.

### 8xlarge (32 vCPU) Spot Price Comparison

| Instance | Spot Monthly Cost | Physical Cores | Cost Per Core |
|----------|------------------|---------------|--------------|
| C7g (Graviton3) | ~$220 | 32 | **$6.9** |
| C8g (Graviton4) | ~$237 | 32 | **$7.4** |
| C8a (AMD Turin)* | ~$340 | 32 | **$10.6** |
| C7i (Intel) | ~$290 | 16 | **$18.1** |

*C8a Seoul region price estimated from us-east-1 actuals + ~12% Seoul premium.

C8a delivers superior per-core performance, but Spot pricing is 43% higher than Graviton4. The cost gap exceeds the performance gap.

### Cores Available on a Fixed Budget ($500/month Spot)

| Instance | Cores Available | Relative Perf/Core | Effective Throughput |
|----------|----------------|--------------------|--------------------|
| C7g | ~72 | 1.0x | 72 |
| C8g | ~67 | 1.3x | **87** |
| C8a* | ~47 | 1.6x | 75 |
| C7i | ~28 | 1.05x | 29 |

Graviton4 (C8g) delivers the highest effective throughput per dollar. C8a has excellent per-core performance but secures fewer cores, resulting in lower effective throughput than Graviton4. Intel's HT penalty puts it last.

---

## Workload Analysis

### Spark Batch/ETL

Spark batch processing is fundamentally about massive parallelism. `executor-cores` drives task parallelism, so more physical cores means near-linear throughput gains. Core count takes priority over single-thread clock speed.

On EKS Spot, interruptions are handled by executor retries (even more robust with Celeborn RSS). Running many cheap instances in parallel outperforms running fewer expensive ones. Graviton4 offers the best Spot price-to-core ratio.

**Recommendation: C8g (Graviton4) Spot**

### Trino Interactive Queries

Trino's workload profile differs from Spark. Query response latency matters, and the coordinator's single-threaded query plan processing can be a bottleneck.

C8a's dominant single-thread performance directly reduces coordinator query planning and parsing time. Worker nodes need high core counts for parallel scans, aggregation, and joins.

**Recommendation: Coordinator on C8a (On-Demand), Workers on C8g Spot**

### Iceberg Maintenance

Compaction, OPTIMIZE, and REWRITE MANIFESTS are driven by file I/O and parallel sorting. Managing 220+ Iceberg tables requires maximizing core count to reduce completion time. These jobs handle Spot interruptions with retries, making cost-effective Graviton ideal.

**Recommendation: C8g (Graviton4) Spot**

### StarRocks BE

StarRocks Backend performs vectorized aggregation and can leverage AVX512 instruction sets. Turin's latest AVX512 implementation dominates Intel in benchmarks. For dedicated StarRocks clusters, C8a migration offers the clearest benefit.

**Recommendation: C8a (On-Demand or Reserved)**

---

## Migration Decision Framework

Even when C8a launches in Seoul region, switching isn't always the right call.

### Consider C8a When

- Trino query latency consistently exceeds SLA targets and CPU is the clear bottleneck
- Single query processing time (especially coordinator plan generation) isn't solved by adding more cores
- StarRocks BE aggregation is CPU-bound and AVX512 utilization matters

### Stay on Graviton When

- Spot-based Spark batch is running smoothly and core shortage isn't the bottleneck
- Maximizing cores per budget is the strategic priority (scale-out first)
- ARM native builds (Docker images, libraries) are already in place with zero migration cost
- C8a Spot availability in Seoul is low (common with newly launched instances)

---

## Practical Migration Considerations

### ARM → x86 (C8a) Migration

- **Docker images**: ARM64 → AMD64 rebuild required. ECR multi-architecture image management recommended
- **JVM flags**: Review ARM optimization flags like `-XX:+UseNUMA`, `-XX:+UseTransparentHugePages`
- **Native libraries**: Verify AMD64 binaries for JNI-based libraries (Arrow, Snappy, LZ4)
- **Spot availability**: New instances have smaller Spot pools initially — higher interruption rates. Observe for 3–6 months after launch

### Graviton3 → Graviton4 Upgrade

- Same ARM architecture (arm64). No image, library, or config changes needed
- ~8% Spot price increase for ~30% performance improvement
- Update EKS nodegroup AMI — the lowest-risk upgrade path available

---

## Conclusion

The 2026 benchmarks confirmed AMD Turin's dominant performance. But "fastest instance ≠ best choice."

For distributed data platforms running Trino and Spark, three factors must be weighed together:

1. **Physical core count**: In Spot parallel workloads, core count determines throughput
2. **Spot price efficiency**: If the same budget buys more cores, distributed processing wins
3. **Workload characteristics**: Distinguish between single-thread latency needs and parallel throughput needs

We recommend maintaining the Graviton4 ARM strategy as the core infrastructure, while selectively leveraging C8a's dominant single-thread performance for latency-critical components (Trino coordinator, StarRocks BE).

**References:**
- [Cloud VM benchmarks 2026: performance / price](https://dev.to/dkechag/cloud-vm-benchmarks-2026-performance-price-p0p)
- [AWS EC2 C8g Instance Types](https://aws.amazon.com/ec2/instance-types/c8g/)
- [AWS EC2 Spot Pricing](https://aws.amazon.com/ec2/spot/pricing/)
- [AMD EPYC 9004 Series (Turin)](https://www.amd.com/en/products/processors/server/epyc/9005-series.html)
