---
title: "Flink on EKS In-place Scaling: Scaling TaskManagers Without Restarting the Job"
date: 2026-03-03
draft: false
categories: [Data Engineering]
tags: [flink, kubernetes, eks, autoscaling, spot-instance, kafka, streaming]
showTableOfContents: true
summary: "Our recommendation system's Flink application required sub-1-minute latency, which prevented us from using autoscaling or spot instances. Autoscaling or spot reclamation triggered full Flink restarts that took 2-3 minutes. Using Flink 1.18's adaptive scheduler and K8s Operator 1.8, we enabled in-place scaling — reducing consumer lag peaks to 1/5 and lag duration from 5-7 minutes to 2-3 minutes during scale events."
---

We were running Flink on EKS for the recommendation system. The requirement was sub-1-minute response latency, which meant we couldn't use autoscaling or spot instances. We were running on-demand instances with fixed allocation. The cost was significant.

The core problem was Flink's restart time. When the application restarted due to autoscaling or spot reclamation, checkpoint recovery and Kafka consumer group reorganization took 2-3+ minutes before actual processing resumed. That violated the latency requirement.

We set out to find whether Flink could scale without restarting.

---

## Two Approaches: Reactive Scheduler and Adaptive Scheduler

Flink has two mechanisms for in-place scaling.

**Reactive Scheduler** was introduced in Flink K8s Operator 1.2. It detects TaskManager pod count changes and automatically adjusts job parallelism. The limitation: it only works in standalone mode.

We tested standalone mode, but the JobManager failed to find the JAR file at startup.

```
org.apache.flink.util.FlinkException: An error occurred while access the provided classpath.
Caused by: java.nio.file.NoSuchFileException: /opt/flink/usrlib/ds-stream-assembly.jar
```

Same Docker image, same JAR path as native mode — but standalone mode couldn't locate the file. `StandaloneApplicationClusterEntryPoint` appears to resolve classpath differently. Quick resolution wasn't feasible, so we pivoted.

**Adaptive Scheduler** was introduced in Flink 1.18. It exposes REST APIs to adjust each vertex's parallelism in the job graph, and Flink K8s Operator 1.8 integrates this with its autoscaler. No standalone mode requirement. We went with this approach.

---

## Flink K8s Operator Upgrade: 1.5 → 1.8

Using adaptive scheduler-based in-place scaling required Flink K8s Operator 1.8. We upgraded from 1.5.

Flink itself also needed to go from 1.17 to 1.18. This surfaced two build issues.

### jackson-module-scala Version Conflict

```
com.fasterxml.jackson.databind.JsonMappingException:
Scala module 2.13.2 requires Jackson Databind version >= 2.13.0 and < 2.14.0
- Found jackson-databind version 2.14.3
```

`jackson-module-scala` requires matching major/minor versions with `jackson-databind`. But `jackson-databind` 2.14.3 was being loaded from somewhere, causing a conflict.

`sbt dependencyTree` showed no direct dependency on 2.14.3. The version appeared to come from an internal Flink library transitive dependency, but we never identified the exact path. Reverting Flink core libraries to 1.17 made the error disappear, confirming it was introduced by the 1.18 upgrade.

We resolved it by aligning all jackson library versions to 2.14.3.

```scala
val jacksonVersion = "2.14.3"   // Only 2.14.x is compatible with Flink 1.18.x
```

### ExecutionConfig ClassNotFoundException

After fixing the jackson issue, a different error appeared.

```
Cause: java.lang.ClassNotFoundException: org.apache.flink.api.common.ExecutionConfig
```

`ExecutionConfig` lives in `flink-core` and has existed since Flink's earliest versions. This wasn't a missing dependency — it was a ClassLoader isolation issue.

We found the exact same problem on the Flink mailing list. When sbt runs tests in the same JVM, ClassLoader isolation breaks in specific scenarios. Forking tests into a separate JVM process resolves it.

```scala
Test / fork := true
```

The sbt documentation mentions that "deserialization and class loading may behave unexpectedly" without forking, but doesn't elaborate further.

---

## Manual Test: The Scale Button

After deploying Flink 1.18.1 and K8s Operator 1.8 to beta, we ran manual tests first.

Flink 1.18 adds a Scale button to the Flink Web UI. Clicking it lets you adjust each vertex's parallelism. The same operation is available via REST API.

We clicked the button. New TaskManager pods spawned and started processing immediately — without job restart. Manual in-place scaling worked.

The question was whether the K8s Operator's autoscaler would trigger it automatically. According to the Operator 1.8 documentation, the autoscaler handles this. But beta didn't have enough traffic to trigger autoscaling, so we moved testing to the staging environment.

---

## Autoscaling Test: The Memory Autotuning Trap

In staging, we launched two Flink apps with `consumer.offset=earliest` to generate heavy traffic. Starting with 1 TM each, the autoscaler detected massive consumer lag and scaled out to 120 and 64 TMs respectively.

But instead of in-place scaling, the jobs fully restarted — same behavior as 1.17.

The culprit was **memory autotuning**, a new feature in Operator 1.8. We'd enabled it alongside in-place scaling. When autoscaling triggered, the operator also recalculated memory settings, which forced a full restart.

After disabling memory autotuning, we re-tested. The difference was immediately visible in the graphs:

- **Before disabling**: TM count dropped to 0 during scaling, creating processing gaps
- **After disabling**: TM count changed incrementally with no processing gaps

In-place scaling was working correctly.

---

## Performance Comparison: Consumer Lag

We quantified the improvement by comparing consumer lag during scale events.

### In-place Disabled (Baseline)

Scaling triggers a full app restart. TM count drops to 0 temporarily. Kafka consumer lag spikes.

- Peak consumer lag: **1.3M–1.6M**
- Spike duration: **5–7 minutes**

### In-place Enabled (Improved)

TM count never drops to 0 during scaling. However, Kafka consumer rebalancing still occurs — consumers disconnect and reconnect, causing some processing delay.

- Peak consumer lag: **250K–500K**
- Spike duration: **2–3 minutes**

| Metric | In-place Disabled | In-place Enabled | Improvement |
|--------|-------------------|------------------|-------------|
| Peak consumer lag | 1.3M–1.6M | 250K–500K | ~1/5 |
| Spike duration | 5–7 min | 2–3 min | ~1/2 |

The result wasn't "zero latency" as we'd initially hoped. Even though in-place scaling prevents Flink job restarts, Kafka consumer group rebalancing is unavoidable. But reducing peak lag by 5x and spike duration by half is a meaningful improvement.

---

## Node Allocation Strategy Changes

With in-place scaling available, we could simplify the node allocation strategy.

Previously, we ran a complex setup to mitigate spot reclamation impact: spot and on-demand nodes simultaneously with spot priority, periodic eviction of pods on on-demand nodes, and over-provisioning pods to buffer node spin-up time.

In-place scaling meant individual TM pod termination no longer caused a full job restart. The complexity could be reduced.

**Updated strategy:**

- **JobManager**: Always on on-demand nodes. JM death means full restart, so it shouldn't be exposed to spot reclamation.
- **TaskManager**: Spot preferred, on-demand fallback. If a TM is reclaimed, remaining TMs continue processing via in-place scaling.
- **Over-provisioning pods**: Removed. With JM stable on on-demand, full app restarts are rare outside of deployments. Node spin-up latency is tolerable.
- **Eviction logic**: Simplified from evicting both JM and TM to TM-only.

### Spot Termination Delay Unnecessary

We had initially planned to apply the same approach we used for Trino workers — delaying pod termination during spot reclamation until replacement pods are ready. But even with this delay, Kafka consumer rebalancing would still cause 2–3 minutes of processing lag. In-place scaling already provides the same benefit, making the additional mechanism redundant.

---

## Takeaways

**In-place scaling means "no restart," not "no delay."** Flink job restarts are prevented, but Kafka consumer rebalancing is unavoidable. Consumers redistribute partitions during scale events, causing 2–3 minutes of processing delay. Setting accurate expectations matters.

**Don't enable multiple new features simultaneously.** Memory autotuning and in-place scaling were activated together, and in-place scaling appeared broken. Enable new features one at a time and verify each independently.

**Understand the bottleneck before applying a solution.** We planned to apply Trino's spot termination delay pattern to Flink, but Flink's bottleneck wasn't pod termination — it was consumer rebalancing. Assuming the same fix works across different systems is risky.

**Complex operational structures simplify when root causes are resolved.** Over-provisioning, dual eviction, mixed spot/on-demand logic — all built on the premise that "restarts are expensive." Once restart cost dropped, these structures became unnecessary. Solving root causes eliminates derived complexity.

**References:**
- [Flink K8s Operator 1.2: Improved Upgrade Flow](https://flink.apache.org/2022/10/07/apache-flink-kubernetes-operator-1.2.0-release-announcement/)
- [Flink K8s Operator 1.8: In-place Scaling Support](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-release-1.8/docs/custom-resource/autoscaler/)
- [Flink Mailing List: ClassNotFoundException with Flink 1.18](https://www.mail-archive.com/user@flink.apache.org/msg52040.html)
- [sbt: Running Project Code](https://www.scala-sbt.org/1.x/docs/Running-Project-Code.html)
