---
title: "EKS AI Serving Node Cost Reduction: Instance Diversification, Consolidation, and Scheduled Scaling"
date: 2026-03-05
draft: false
categories: [Data Engineering]
tags: [eks, karpenter, cost-optimization, autoscaling, keda, kubernetes, consolidation]
showTableOfContents: true
summary: "An AI platform team's serving API was running 500 fixed pods on only two on-demand instance types (c6i.2xlarge, m6i.2xlarge). We applied instance type diversification and Karpenter consolidation as phase 1, then KEDA cron-trigger scheduled scaling as phase 2. Key finding: consolidation alone has limited effect when pod count is fixed — it needs scale-in to actually reduce node count."
---

We needed to reduce node costs for the AI platform team's serving workloads. The largest workload was the dispatch-travel-time-prediction API — 500 pods deployed at all times. HPA was configured with CPU metrics, but min replicas was set to 500, so actual scale-in never occurred.

Grafana showed pod count fixed at 500 for an entire week. Resources sized for weekend evening peaks were maintained through the early morning hours.

We split cost reduction into two phases: phase 1 for instance type diversification and consolidation, phase 2 for schedule-based scaling.

---

## Phase 1: Instance Type Diversification

The existing NodePool only allowed two instance types: `c6i.2xlarge` and `m6i.2xlarge`. Different serving API pods have different CPU-to-memory ratios, but with only two instance types, bin-packing efficiency suffers. The gap between pod resource requests and instance capacity is wasted.

We changed the Karpenter NodePool instance selection criteria.

```yaml
# Before: specific instance types
requirements:
- key: node.kubernetes.io/instance-type
  operator: In
  values:
  - m6i.2xlarge
  - c6i.2xlarge

# After: category/generation/size-based criteria
requirements:
- key: karpenter.k8s.aws/instance-category
  operator: In
  values: ["r", "m", "c"]
- key: karpenter.k8s.aws/instance-generation
  operator: Gt
  values: ["5"]  # 6th generation and above
- key: karpenter.k8s.aws/instance-size
  operator: In
  values:
  - 2xlarge
  - 4xlarge
```

Instead of specifying exact instance types via `node.kubernetes.io/instance-type`, we split the criteria into `instance-category`, `instance-generation`, and `instance-size`. This allows r, m, and c categories, 6th generation and above, in 2xlarge and 4xlarge sizes. Karpenter can now automatically select the optimal instance for each pod's resource request.

---

## Phase 1: Enabling Consolidation

Instance type diversification alone doesn't automatically replace existing nodes. Karpenter's consolidation must be enabled for nodes to be redistributed onto more efficient instances.

To avoid impacting existing serving APIs, we created a separate NodePool with consolidation enabled and applied it to dispatch-travel-time-prediction first.

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: ds-eks-prod-mlops-serving-with-consolidation
spec:
  weight: 50
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 480h0m0s  # 20 days
    budgets:
    - nodes: "2"
    - nodes: "1"              # limit to 1 concurrent disruption during business hours
      schedule: "0 1 * * *"   # UTC
      duration: 14h           # KST 10:00 ~ 24:00
```

A dedicated taint was added to the new NodePool, and the dispatch-travel-time-prediction deployment received the corresponding toleration for gradual migration.

### Disruption Budget Tuning

Initially, we completely blocked consolidation-driven node disruptions during business hours (`nodes: "0"`). But this prevented empty nodes from being drained during daytime redeployments.

The service already had PDB (PodDisruptionBudget) configured, and each pod had a `sleep 30` preStop hook. With on-demand instances only and explicit resource requests, consolidation rarely triggers during business hours anyway. We relaxed the budget to `nodes: "1"`.

### The Limitation of Consolidation Alone

Here was the critical discovery: when pod count is fixed at 500, consolidation has almost no cost-saving effect.

Consolidation works by repacking pods onto fewer nodes when there's unused capacity. But if pod count doesn't change, total required resources don't change either. Instance type diversification provides marginal bin-packing improvement, but meaningful cost reduction requires actually reducing the number of pods.

---

## Phase 2: KEDA Cron-Trigger Scheduled Scaling

500 pods weren't needed around the clock. The count was sized for weekend evening peak traffic (3000+ RPS) but maintained through early morning hours. The API had a characteristic where timeouts started occurring at just 20% CPU utilization, making HPA alone insufficient for proper scale-in.

We used KEDA's cron trigger to adjust pod count by time of day.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: dispatch-travel-time-prediction-scheduled
spec:
  scaleTargetRef:
    kind: Deployment
    name: dispatch-travel-time-prediction-scheduled
  minReplicaCount: 200
  maxReplicaCount: 400
  triggers:
  # Morning (08:30~10:30)
  - type: cron
    metadata:
      timezone: Asia/Seoul
      desiredReplicas: "250"
      start: 30 8 * * 0-6
      end: 0 1 * * 0-6
  # Midday weekday (10:30~16:00)
  - type: cron
    metadata:
      timezone: Asia/Seoul
      desiredReplicas: "300"
      start: 30 10 * * 1-5
      end: 30 14 * * 1-5
  # Weekday evening peak (16:00~23:30)
  - type: cron
    metadata:
      timezone: Asia/Seoul
      desiredReplicas: "350"
      start: 0 16 * * 1-5
      end: 30 23 * * 1-5
  # Weekend evening peak (16:00~23:30) — higher order volume
  - type: cron
    metadata:
      timezone: Asia/Seoul
      desiredReplicas: "400"
      start: 0 16 * * 0,6
      end: 30 23 * * 0,6
```

During early morning hours (01:00~08:30), only the minimum 200 pods are maintained. Pod count scales with traffic patterns: 250 → 300 → 350 (weekdays) / 400 (weekends). Compared to the fixed 500, this eliminates 300 pods during off-peak hours.

### Coexistence with Existing HPA

We didn't remove the existing CPU-based HPA. Instead, we separated the KEDA-managed scheduled deployment from the existing HPA-managed deployment using `autoscaling/scheduled: "true"` / `"false"` labels.

The existing HPA deployment's min replicas was reduced from 500 to 100, max from 1000 to 500. Scheduled scaling handles most capacity, while HPA serves as a safety net for unexpected load spikes.

### Label Selector Hotfix

One issue emerged right after deployment. The scheduled deployment's selector was missing the `autoscaling/scheduled` label, causing it to select pods from the existing deployment as well. A hotfix added the label selector to resolve it.

---

## Result: Consolidation + Scale-In Synergy

After applying scheduled scaling, the intended synergy with consolidation materialized.

When pods scaled in, consolidation triggered and repacked remaining pods onto fewer nodes. Node count rose and fell in tandem with pod count.

Pod redistribution during consolidation caused minor fluctuations in running pod count, but node-level disruption budgets, pod-level PDB, and `sleep 30` preStop hooks collectively ensured zero service disruption.

---

## Why We Didn't Use Spot Instances

Spot instances were discussed during cost reduction planning. Karpenter supports spot/on-demand ratio control via topologySpreadConstraint, and spot disruption events trigger pod rescheduling to other nodes.

However, company policy prohibits spot instances for API services. A delivery time prediction API handling 3000+ RPS couldn't risk replica count instability from spot reclamation. We proceeded with instance diversification + consolidation + scheduled scaling without spot.

---

## Takeaways

**Consolidation alone has limited effect.** When pod count doesn't change, total required resources don't change. Consolidation optimizes bin-packing, but meaningful savings require reducing pod count. Apply in order — instance diversification → consolidation → scheduled scaling — but the real impact comes from scale-in.

**Apply changes one at a time, in sequence.** Instance diversification, consolidation, and scheduled scaling applied simultaneously make root cause analysis difficult when problems arise. Separate each phase, verify its effect, then proceed to the next. This minimizes service impact.

**Cost reduction isn't just an infrastructure team's job.** The min replica 500 setting came from peak-time latency requirements. The API characteristic of timing out at 20% CPU, the scheduling window design, whether to use spot — all required discussion with the service team. Configuration changes alone don't solve it.

**Layer safety mechanisms.** Consolidation-driven node redistribution involves temporary pod disruption. PDB, `sleep 30` preStop hooks, and disruption budget concurrent node limits — all three work together to guarantee zero-downtime. Missing any one could cause traffic loss during redistribution.
