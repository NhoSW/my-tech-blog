---
title: "EMR on EKS VPA Review: 18 Months of Debugging and the Switch to ScaleOps"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [emr, eks, vpa, spark, kubernetes, scaleops, autoscaling]
showTableOfContents: true
summary: "We tried using AWS's built-in VPA integration for EMR on EKS to auto-optimize Spark executor resources. After 18 months of PoC work, multiple AWS support cases, and a custom manifest bundle rebuild, the operator still didn't work. We abandoned it and switched to ScaleOps."
---

When you run Spark jobs on EMR on EKS, executor pods land on Kubernetes. The question is how much CPU and memory to give each executor. It varies by job and by time of day. Over-provision and you waste money. Under-provision and you get OOM kills.

VPA (Vertical Pod Autoscaler) can recommend and auto-adjust resources based on historical execution data. AWS provides a VPA integration specifically for EMR on EKS, so we decided to evaluate it. We started in January 2024 and gave up in July 2025. Eighteen months.

This post covers what we tried and what we learned.

---

## VPA Modes

EMR on EKS VPA supports three modes.

| Mode | Behavior | Impact |
|------|----------|--------|
| Off | Computes recommendations only. No actual changes (dry-run) | None |
| Initial | Applies recommendations when the job starts | No executor restarts |
| Auto | Evicts running executors and restarts them with recommended resources | Possible latency |

Initial mode looked safest. No executor restart overhead, so it seemed safe to apply to production jobs directly.

The plan: turn on Off mode across all Spark jobs to collect recommendations first. Verify the recommendations make sense. Then switch to Initial mode.

---

## Job Signature Key Design

For VPA to manage recommendations per job, each job needs an identifying key. EMR VPA uses the concept of a job signature key. As execution history accumulates for a given signature, it computes resource recommendations from that data.

Job name alone isn't enough. The same job can have very different resource profiles during peak hours vs. off-hours. Day of week matters too.

We designed the signature key like this:

```python
f"{app_name}-day-{target_date.day_of_week}-hour-{target_date.hour}-"
f"{'holiday-' if is_holiday else ''}{resource_spec}"
```

Components:
- `app_name`: Spark job name
- `day_of_week`: Day of week (Friday traffic differs from midweek)
- `hour`: Hour of day (peak time differentiation)
- `holiday`: Whether it's a public holiday
- `resource_spec`: DRA settings and executor count spec (VPA scales individual pods vertically, so executor count is a significant factor)

### K8s Label Length Limitation

The signature key is stored as a Kubernetes label, which has constraints:

- Maximum 63 characters
- Only alphanumeric, `-`, `_`, `.` allowed
- Must start and end with alphanumeric

The raw signature key can exceed these limits. We converted it to a deterministic hash and stored the hash-to-original mapping in Airflow XCom for traceability.

---

## YuniKorn Compatibility Issue

The first wall was scheduler compatibility.

We were using the YuniKorn scheduler with EMR on EKS. YuniKorn doesn't support VPA. In gang scheduling environments, VPA causes problems. When an executor is restarted by VPA, the resource request changes from the original submission value. This mismatch can cause the job to fail or enter a zombie state.

Auto mode was ruled out for this reason alone. Initial mode only intervenes at startup, so we judged it safe to proceed.

---

## PoC Execution

We deployed the EMR VPA operator to the test environment and set up three test batches:

- Coupon mart batch (existing job running on EMR-on-EC2)
- Delivery mart batch
- DW blog batch (hourly batch, ~20 min at peak — ideal for comparison)

Each batch ran in both Initial and Auto modes for comparison against the same job without VPA.

VPA resources were created successfully. But the VPA status showed `NoPodsMatched`. It wasn't collecting resource usage metrics from executor pods.

---

## A Cascade of Operator Issues

Troubleshooting opened a can of worms.

### 1. kube-apiserver Overload

The EMR VPA recommender had no option to limit its target namespace. It was scanning all resources across all namespaces. Looking at the recommender logs, it was periodically hitting every endpoint on the kube-apiserver.

The admission controller's target namespace could be restricted by manually editing the OperatorGroup resource spec. But there was no equivalent for the recommender.

### 2. VPA Resource Recognition Failure

The EMR VPA controller (`emr-dynamic-sizing-controller-manager`) created VPA resources correctly. However, the admission controller and recommender — both installed from the same bundle — couldn't find the VPAs. The admission controller failed to locate existing VPAs by signature key. The recommender couldn't write recommendations to the VPA resources.

### 3. Black Box Operator

The operator is an AWS-bundled package installed onto EKS. We couldn't inspect or modify the code directly. Debugging was limited to reading logs and guessing.

---

## AWS Support Cases

We opened support cases twice.

The first time, we documented the namespace limitation and VPA recognition issues in detail. AWS responded by sharing the complete Kubernetes manifest bundle for the EMR VPA operator. The guidance was to customize and reinstall it ourselves.

We took the bundle, modified it, reinstalled, and tested again. Same issues. We filed another case with detailed component logs.

---

## Conclusion: The Feature Was Effectively Abandoned

After 18 months of evaluation, our conclusion:

The EMR VPA integration feature **appears to no longer be actively developed or maintained by AWS.** Two pieces of evidence:

1. Beyond AWS's own documentation, we couldn't find a single real-world usage report of this feature
2. In-place Pod Resize, introduced in Kubernetes 1.27, is a strictly better alternative. AWS had mentioned plans to integrate it directly into EMR on EKS at a summit event

The fundamental weakness of VPA is that resizing requires pod restarts. In-place Pod Resize changes resources without killing the pod. A PR to bring in-place resize support to the Kubernetes VPA itself has been opened on the autoscaler repo.

---

## Switching to ScaleOps

We ultimately adopted ScaleOps to fill the role that EMR VPA was supposed to play. By adding proper labels and grouping rules to Spark jobs submitted from Airflow, ScaleOps auto-optimizes resources based on execution history.

ScaleOps is a third-party solution, but unlike the EMR VPA black box, its behavior is transparent and support is responsive.

---

## Lessons Learned

**Not every AWS feature is production-ready.** Being in the official docs doesn't guarantee it works. If you can't find real usage reports for a feature, be skeptical.

**Black box operators are a debugging nightmare.** Without access to the code, you're guessing from logs. Support cases take time, and the answer is often "here's the manifests, fix it yourself."

**Read the direction of the K8s ecosystem.** Once In-place Pod Resize arrived, VPA's evict-and-recreate approach was destined to become legacy. Check upstream project roadmaps before committing to a technology.

**Starting with dry-run mode was the right call.** If we had applied VPA to production jobs directly, the operator issues could have disrupted the jobs themselves. Dry-run let us identify problems without any production impact.

**References:**
- [EMR on EKS Vertical Autoscaling - AWS Documentation](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/jobruns-vas-configure.html)
- [YuniKorn VPA Support Issue](https://issues.apache.org/jira/browse/YUNIKORN-1765)
- [VPA In-place Resize PR](https://github.com/kubernetes/autoscaler/pull/6652)
