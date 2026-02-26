---
title: "Kafka Rack Awareness and Spark: Not Supported Yet"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [kafka, spark, rack-awareness, kubernetes, aws, networking]
showTableOfContents: true
summary: "We tried to apply Kafka rack awareness to Spark jobs to reduce cross-AZ network costs. Getting the AZ information was solved easily via AWS IMDS, but Spark itself doesn't support rack-aware Kafka partition assignment. The related Jira ticket is open but the PR was closed."
---

After a Kafka cluster version upgrade, rack awareness became available for reducing network costs. When a consumer in the same AZ reads from a replica in the same AZ, cross-AZ traffic drops.

We wanted this for our EMR-based Spark jobs too. Each executor deployed in an AZ should consume from Kafka partitions in the same AZ. The short answer: Spark doesn't support this.

---

## What Rack Awareness Does

Kafka's rack awareness lets consumers tell brokers their location (rack or AZ). The broker then serves data from the nearest replica. KIP-881 extended this to factor rack information into consumer partition assignment.

Configuration is straightforward. Set `client.rack` on the consumer. For example, `ap-northeast-2a` routes reads to replicas in that AZ.

The question is how to inject this setting in Spark.

---

## Code Analysis: Where to Inject

We first identified where `client.rack` should be injected in the codebase.

The Kafka topic read flow: individual job → `LogStoreProcessor.getKafkaLogDataFrame()` → `SparkSessionManager.rowKafkaDF()`. The `rowKafkaDF` method reads Kafka topics into DataFrames. This seemed like the right place to inject `client.rack`.

We also checked the write side. Rack awareness is generally irrelevant for Kafka writes, but depending on broker configuration, it can influence partition leader election. So we planned to inject it on the write path too.

---

## Getting AZ Information

We evaluated two approaches to extract the current AZ for `client.rack`.

### Approach 1: Kubernetes Downward API

Inject node topology labels into the container via Kubernetes Downward API.

Problems surfaced quickly.

- AZ information exists only in node labels, not pod labels
- The Downward API currently supports injecting pod labels only, not node labels
- KEP-4742 proposes exposing node topology labels via the Downward API and has been released as an alpha feature, but it's not available on our EKS version

A workaround — extracting the node name from `spec.nodeName` and querying the K8s API for node labels — was possible but required an initContainer or code changes. For EMR-on-EKS, pod template files would need to be uploaded to S3. Too much effort for the benefit.

### Approach 2: AWS IMDS

A much simpler solution. AWS Instance Metadata Service (IMDS) provides the current AZ directly.

```bash
curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone
# ap-northeast-2b
```

Accessible from EC2 instances and containers running on them via a fixed IP (`169.254.169.254`). No Kubernetes configuration needed. Works identically on EMR-on-EC2.

We implemented a utility class to extract AZ information using this approach.

---

## But Spark Doesn't Support It

Getting the AZ information was the easy part. The problem was the next step.

We tried to pass the extracted AZ to the Kafka session as `client.rack` in `SparkSessionManager.rowKafkaDF()`. Checked Spark's Kafka integration guide — no mention of rack awareness.

Further searching confirmed that Spark does not support rack awareness for Kafka integration. Setting `client.rack` alone isn't enough. The Spark driver needs logic to consider rack information when assigning Kafka partitions to executors. That logic doesn't exist in Spark's code.

A Jira ticket (SPARK-46798) was opened and a PR was submitted. But the PR was closed — reviewers deemed it cloud vendor-specific. They argued that such a feature would need a formal SPIP (Spark Improvement Proposal) and a cloud-agnostic design.

---

## Current Status

Until the Spark community implements this feature, Kafka rack awareness cannot be used with Spark Structured Streaming. The SPARK-46798 ticket remains open but isn't actively progressing.

On the Kubernetes side, KEP-4742 is moving forward — automatic copying of node topology labels to pods. When available on EKS, it would make AZ extraction cleaner, but without Spark-side support, it doesn't matter.

---

## Takeaways

**Setting `client.rack` on a consumer and rack-aware partition assignment are separate things.** Kafka itself supports rack-aware partition assignment, but Spark's Kafka integration layer doesn't use this information.

**IMDS is the simplest way to get instance metadata in AWS.** It sidesteps Kubernetes Downward API limitations. Works the same on EKS and EMR-on-EC2.

**Check community discussions before building.** If we had searched the Spark community for existing discussions first, we could have avoided unnecessary implementation work.

**References:**
- [SPARK-46798: Kafka custom partition location assignment (rack awareness)](https://issues.apache.org/jira/browse/SPARK-46798)
- [SPARK-46798 PR (closed)](https://github.com/apache/spark/pull/46863)
- [Spark Kafka Rack Aware Consumer - Apache Mailing List](https://lists.apache.org/thread/t0m0hy3tl8t6kyy7vvsshvckvznsk5zs)
- [Structured Streaming + Kafka Integration Guide](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- [KEP-4742: Expose Node Topology Labels via Downward API](https://github.com/kubernetes/enhancements/issues/4742)
- [K8s Issue: Exposing node labels via Downward API](https://github.com/kubernetes/kubernetes/issues/40610)
- [KIP-881: Rack-aware Partition Assignment for Kafka Consumers](https://cwiki.apache.org/confluence/display/KAFKA/KIP-881:+Rack-aware+Partition+Assignment+for+Kafka+Consumers)
