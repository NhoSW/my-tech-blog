---
title: "EKS Topology Aware Hints: Why They Had No Effect on Our Cluster"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [kubernetes, eks, networking, aws, cost-optimization]
showTableOfContents: true
summary: "We evaluated Kubernetes Topology Aware Hints to reduce cross-AZ network costs on EKS. Hints were correctly applied to EndpointSlices, but had no actual effect. AWS Load Balancer Controller's IP target mode bypasses kube-proxy entirely, and our primary internal workloads — Spark, Trino, Airflow — are all single-zone or stateful, meaning the traffic paths where hints get referenced simply don't exist in our environment."
---

Cross-AZ network costs on EKS are not trivial. AWS charges $0.01 per GB for traffic between AZs within the same region. With workloads distributed across multiple AZs, this adds up quickly.

Topology Aware Hints (TAH), available since Kubernetes 1.23, aim to mitigate this. The feature attaches zone (AZ) information as hints to a Service's EndpointSlice, guiding kube-proxy to prefer endpoints in the same AZ when routing traffic.

AWS's official blog had a post demonstrating TAH for cross-AZ cost reduction on EKS. Could we apply it to our cluster? We ran the tests.

The short answer: TAH had no meaningful effect in our environment. Here's why.

---

## Test Setup

Following the AWS blog guide, we deployed a simple echo server across multiple AZs and enabled TAH.

Adding the `service.kubernetes.io/topology-aware-hints: auto` annotation to a Service causes the EndpointSlice controller to attach `hints.forZones` fields to each endpoint. This hint means "this endpoint is suitable for handling traffic from this AZ."

The EndpointSlice controller correctly applied hints to individual endpoints. The feature itself was working as designed.

However, when accessing the service from a dummy pod's shell, traffic wasn't routed exclusively to same-AZ endpoints as the hints suggested. For TAH to actually influence traffic routing, several prerequisites must be met — and our environment didn't satisfy them.

---

## External Traffic: AWS Load Balancer Controller Bypasses kube-proxy

Let's start with external traffic entering the cluster.

We use AWS Load Balancer Controller (not nginx-ingress) as our ingress controller. Following the AWS EKS best practices recommendation, we create ingress resources with IP target type:

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

With IP target type, the ALB registers pod IPs directly as targets. Traffic reaches pods without passing through the node's kube-proxy. This is the key point.

TAH is a hint that kube-proxy references when routing traffic. When kube-proxy sets up iptables or IPVS rules, it factors in the zone hints to prefer same-AZ endpoints. But with IP target type ingress, kube-proxy is never in the traffic path — there's no point where the hint gets referenced.

nginx-ingress controller is different. Traffic between the nginx pod and upstream pods flows through kube-proxy, making TAH effective. But our cluster removed the previously-used nginx-ingress and runs AWS Load Balancer Controller exclusively.

For external traffic, TAH is simply not applicable.

---

## Internal Traffic: No Workloads Meet the Requirements

What about intra-cluster communication? When pods communicate through Services, kube-proxy is in the path, so TAH could theoretically work.

But TAH requires several conditions to be effective.

### Constraints

**Multi-AZ many-to-many deployment:** Both client and server sides must have multiple pods distributed across several AZs.

**Balanced endpoint distribution per AZ:** The server side must be deployed across all availability zones (A, B, C) with sufficient endpoints relative to each AZ's total node core count. The EndpointSlice controller distributes hints based on per-AZ core ratios. If any AZ has insufficient endpoints, hints won't work properly.

**Even client AZ distribution:** If clients are skewed toward a specific zone, hints break down. Additionally, node load metrics from a single zone won't represent overall cluster load, causing compatibility issues with autoscalers like HPA.

**Stateless communication:** Clients must be able to reach any endpoint pod behind the server service without issues — purely stateless communication.

### Our Workload Analysis

We mapped these constraints against our cluster's primary internal communication patterns.

| Workload | Communication Pattern | TAH Applicable? |
|---------|----------------------|----------------|
| Spark driver ↔ executor | Single-zone deployment, stateful | No |
| Trino coordinator ↔ worker | Single-zone deployment, stateful | No |
| Trino Gateway ↔ Trino clusters | Individual clusters are single-zone, stateful (specific queries route to specific backends) | No |
| Airflow scheduler ↔ worker ↔ metaDB (RDS) ↔ Redis (ElastiCache) | Stateful | No |

Spark and Trino are intentionally deployed in a single AZ. Communication between driver and executor, coordinator and worker is extremely frequent and high-volume. To eliminate cross-AZ costs at the source, all components are placed in the same AZ. Since they're already single-zone, the problem TAH tries to solve doesn't exist.

Trino Gateway to backend cluster communication is stateful. Once the gateway routes a query to a specific backend cluster, all subsequent communication for that query goes to that cluster. The concept of "pick the nearest endpoint" from TAH doesn't apply to this communication pattern.

Airflow communication between scheduler, worker, metaDB, and Redis is also stateful. When the scheduler assigns a task to a specific worker, communication with that worker is maintained. MetaDB and Redis have fixed endpoints.

The conclusion: no internal communication patterns in our cluster met the TAH requirements.

---

## Where TAH Would Be Valuable

TAH was ineffective in our cluster, but there are environments where it clearly adds value:

- Clusters using nginx-ingress controller, where traffic between nginx pods and upstream pods flows through kube-proxy
- Classic microservice architectures with stateless services deployed across multiple AZs
- Recommendation API servers or similar workloads where both clients and servers have many pods distributed across multiple AZs

In fact, after sharing these findings internally, the recommendation team indicated TAH could be valuable for their workloads. The decision depends entirely on workload characteristics.

---

## Notes

A few additional considerations.

**Annotation key change:** Starting with Kubernetes v1.27, the TAH annotation key changed from `service.kubernetes.io/topology-aware-hints` to `service.kubernetes.io/topology-mode`. Our EKS version (v1.24 at the time) wasn't affected, but this is relevant for future version upgrades.

**Custom heuristics:** A feature for attaching custom heuristic rules to routing hints is under development. When available, it would enable user-defined routing logic beyond the current AZ-based hints, potentially broadening applicability.

---

## Takeaways

**A feature working correctly and a feature being effective are different things.** Hints being properly applied to EndpointSlices doesn't mean they influence traffic routing. You must first verify that traffic actually passes through the path where hints are referenced (kube-proxy).

**AWS Load Balancer Controller's IP target mode completely bypasses kube-proxy.** This affects not just TAH, but all kube-proxy-based network features (including some NetworkPolicy implementations). When applying network-level features, understanding the exact traffic path is essential.

**For cross-AZ cost reduction, placement strategy can be more effective than network features.** Deploying Spark and Trino in a single zone is the most reliable way to eliminate cross-AZ traffic. Controlling AZ at the workload placement level is simpler and more effective than trying to manage it at the network routing level.

**Analyze workload communication patterns before applying.** If no workloads meet TAH's constraints (multi-AZ, balanced distribution, stateless), enabling the feature has no effect. Skipping the step of verifying whether a blog post's use case applies to your environment leads to wasted time.

**References:**
- [Amazon EKS: Reduce cross-AZ traffic costs with Topology Aware Hints](https://aws.amazon.com/ko/blogs/tech/amazon-eks-reduce-cross-az-traffic-costs-with-topology-aware-hints/)
- [Kubernetes: Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [Kubernetes: Topology Aware Routing Constraints](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/#constraints)
- [EKS Best Practices: Use IP target type load balancers](https://aws.github.io/aws-eks-best-practices/networking/loadbalancing/loadbalancing/#use-ip-target-type-load-balancers)
- [Kubernetes: Custom Heuristics for Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/#custom-heuristics)
