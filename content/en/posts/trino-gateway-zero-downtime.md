---
title: "Adopting Trino Gateway: Zero-Downtime Deployments and Multi-Cluster Routing"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [trino, trino-gateway, kubernetes, blue-green, routing, karpenter, argocd, helm]
showTableOfContents: true
summary: "Trino doesn't support coordinator HA. Redeploying the coordinator means downtime. We adopted Trino Gateway to enable Blue/Green deployments with zero downtime and route BI/OLAP queries to separate clusters based on HTTP headers."
---

The Trino coordinator is a single point of failure. It doesn't support HA. When the coordinator goes down, the entire cluster stops accepting queries. In production, this becomes a problem in two scenarios.

First, **node rotation**. We manage nodes with Karpenter on Kubernetes. Long-running nodes can develop network issues. We want to rotate them periodically with `ttlSecondsUntilExpired`, but there's no good way to control when the coordinator's node gets replaced.

Second, **deployments**. Redeploying the coordinator pod causes downtime until the new pod is up. Running a CronJob at night is an option, but batch queries running at that hour will fail.

We learned from a Deview conference talk that Lyft solved this with presto-gateway. Put a gateway in front of multiple Trino clusters and you get Blue/Green deployments. Coordinators can be replaced without dropping a single query.

---

## What Trino Gateway Does

Trino Gateway is a load balancer and routing proxy that sits in front of multiple Trino clusters. Originally developed by Lyft as presto-gateway, it's now actively maintained under the trinodb organization as trino-gateway.

Three core capabilities:

- **Multi-cluster routing**: Route queries to different cluster groups based on conditions
- **Backend health checks**: Periodically verify that backend clusters are healthy and automatically exclude failed ones
- **Queue checks**: Distribute load based on the current query count on each backend

From the client's perspective, there's only one endpoint. They don't need to know how many clusters exist or which ones are alive.

---

## Architecture

The target architecture looks like this.

```
Trino Gateway
  ├── BI Cluster Group
  │     ├── BI Cluster 1 (Coordinator: AZ-B)
  │     └── BI Cluster 2 (Coordinator: AZ-C)
  │
  └── OLAP Cluster Group
        ├── OLAP Cluster 1 (Coordinator: AZ-A)
        └── OLAP Cluster 2 (Coordinator: AZ-B)
```

Spreading coordinators across different availability zones was intentional. We had a past incident where node provisioning in a single AZ failed and the coordinator couldn't start. With AZ-spread coordinators, one AZ going down doesn't take out all query processing.

### Operational Model

To minimize resource waste, only one cluster per group is active at a time. Both clusters run simultaneously only during rolling deployments.

```
Normal:   Gateway → Cluster 1 (active)     Cluster 2 (inactive)
Rolling:  Gateway → Cluster 1 (active) + Cluster 2 (booting)
After:    Gateway → Cluster 2 (active)     Cluster 1 (inactive)
```

The gateway confirms the new cluster is ready via health checks, shifts traffic over, waits for running queries on the old cluster to finish, then deactivates it.

---

## Header-Based Routing Rules

Queries are routed to the appropriate cluster group based on their source. The gateway inspects HTTP headers sent by Trino clients.

```yaml
# Superset → BI cluster
- name: "superset"
  condition: >
    request.getHeader("X-Trino-Source") == "Apache Superset"
    && request.getHeader("X-Trino-Client-Tags") == null
  actions:
    - "result.put(\"routingGroup\", \"bi\")"

# Querybook → OLAP cluster
- name: "querybook"
  condition: >
    request.getHeader("X-Trino-Source") == "trino-python-client"
    && request.getHeader("X-Trino-Client-Tags") == null
  actions:
    - "result.put(\"routingGroup\", \"olap\")"

# Zeppelin → OLAP cluster
- name: "zeppelin"
  condition: >
    request.getHeader("X-Trino-Source") ~= "^zeppelin-.+"
  actions:
    - "result.put(\"routingGroup\", \"olap\")"
```

The `X-Trino-Source` header is set automatically by Trino clients. Superset sends `Apache Superset`. Querybook sends `trino-python-client`. Zeppelin uses a source name starting with `zeppelin-`.

The `X-Trino-Client-Tags == null` condition reserves room for future routing overrides via client tags.

---

## Fixing Backend Health and Queue Checks

After deploying the gateway, we discovered issues with both the health check and queue check logic.

### Health Check Issue

The gateway checks backend cluster status by hitting Trino's `/v1/info` endpoint. The problem: this endpoint sometimes returned 200 while the coordinator was still starting up. The gateway would mark the cluster as ready and route queries to it, but it couldn't actually process queries yet.

### Queue Check Issue

The logic for distributing load based on running and queued query counts wasn't working correctly. Queries were piling up on one cluster while the other sat idle.

### Fixes

We modified both the gateway code and the Trino cluster configuration.

- **Gateway side**: Strengthened health check logic to verify the coordinator is fully ready. Fixed queue-based load balancing to distribute queries accurately.
- **Trino cluster side**: Updated Helm chart templates and settings to ensure proper integration with the gateway.

---

## Daily Rolling Restart Batch

The biggest win from adopting the gateway: **zero-downtime cluster replacement, even during business hours.**

We built an Airflow DAG that rolls clusters daily. The flow:

1. Spin up new coordinator and workers on the inactive cluster
2. Wait until the gateway health check marks the new cluster as healthy
3. Gateway starts routing new queries to the new cluster
4. Wait for running queries on the old cluster to complete
5. Deactivate the old cluster

This prevents network issues from long-running nodes while never disrupting in-flight queries.

---

## Rollout Process

We took a phased approach.

### Phase 1: PoC in Test Environment

Deployed the gateway in a test environment first. Validated basic routing and health check behavior.

### Phase 2: Stage Environment in Production

Built a staging setup in the production account. Verified end-to-end integration with real clients — Superset, Querybook, etc. This is where we discovered and fixed the health check and queue check issues.

### Phase 3: Production Rollout

Applied to Superset, Querybook, and Zeppelin first. Switched their endpoints to the gateway address and monitored routing behavior.

### Phase 4: Daily Rolling Batch

Started operating the daily cluster rolling batch as an Airflow DAG. Validated in beta first, then applied to production.

---

## Gateway Internals

### Stateful Routing and MetaDB

The gateway is not a simple reverse proxy. After a Trino query is submitted, follow-up requests (status checks, result fetching) must go to the same coordinator. The gateway stores the `query_id`-to-backend mapping in a MetaDB (MySQL). When a follow-up request arrives, it looks up the MetaDB and routes it to the correct backend.

### Authentication Integration

We added nginx and nginx-ldap as sidecars to the gateway pod for LDAP authentication. An authentication layer sits in front of the gateway. Users authenticate through a dedicated auth endpoint before requests reach the gateway.

---

## Helm Chart Structure

### Base + Cluster-Specific Values

Managing Blue/Green clusters requires a clean split of Helm values. We borrowed a pattern from our CI runner setup: separate base yaml from cluster-specific yaml.

```
values.prod.base.yaml     # Shared settings
values.prod.blue.yaml     # Blue cluster only (AZ, node selectors, etc.)
values.prod.green.yaml    # Green cluster only
```

At deploy time, the base and cluster-specific yamls are merged. Changes to shared settings only require editing the base file. Cluster-specific differences are isolated in their own files.

### Independent ArgoCD Apps

Blue and Green clusters are registered as separate ArgoCD applications. Since each app is managed independently, updating or deactivating one side is straightforward. During rolling deployments, spinning up Green first and then taking down Blue is controllable directly from the ArgoCD UI.

---

## Pre-Production Prerequisites

### Audit Log Separation and Unification

With Blue/Green clusters running as independent query engines, audit logs are split per cluster. Previously we only had to look at one cluster's logs. Now we need a combined view across both.

Two approaches were considered:

1. Separate audit log tables per cluster with a union view on top
2. A single audit log table with an added source cluster field

The existing Airflow audit log dump DAG needed modification, and additional EFS volumes (including Exchange Manager volumes) had to be provisioned per cluster.

### Load Testing

Adding a gateway in front introduces routing overhead. To verify it could handle production workloads, we ran load tests by replaying dumped production queries through the gateway.

---

## Active-Active Consideration

The current setup is Active-Standby within each cluster group. One active cluster at a time. This is sufficient for the original goal of zero-downtime deployments.

We also evaluated Active-Active — running multiple coordinators simultaneously within the same cluster group. This becomes relevant when workloads exceed what a single coordinator can handle.

With default settings, queries are distributed randomly. Not strict round-robin. For production Active-Active, routing needs to account for each cluster's current load. The queue check provides query-count-based distribution, but more sophisticated load-aware routing would require additional work.

For now we operate in Active-Standby mode, with Active-Active as a future option if workload demands grow.

---

## Takeaways

Here's what Trino Gateway solved for us.

**Zero-downtime deployments.** Coordinator redeployments no longer cause downtime. Blue/Green switching after the new cluster is confirmed ready means not a single query is dropped.

**AZ fault tolerance.** Coordinators spread across different AZs ensure query processing survives a single-AZ failure.

**Workload isolation.** BI queries route to BI clusters. OLAP queries route to OLAP clusters. Clients only know one endpoint.

**Node freshness.** Daily rolling restarts prevent network issues that accumulate on long-running nodes.

Fixing the health check and queue check logic took some effort, but the gateway has been stable since. If you're running Trino in production, a gateway isn't optional — it's close to essential.

**References:**
- [Trino Gateway (trinodb)](https://github.com/trinodb/trino-gateway)
- [Presto Infrastructure at Lyft](https://eng.lyft.com/presto-infrastructure-at-lyft-b10adb9db01)
- [Trino Open Source Infrastructure Upgrading at Lyft](https://eng.lyft.com/trino-open-source-infrastructure-upgrading-at-lyft-83f26b099fa)
- [Presto Gateway (Lyft, legacy)](https://github.com/lyft/presto-gateway)
