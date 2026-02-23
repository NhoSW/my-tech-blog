---
title: "Starburst's AI Pivot: Is Trino Open Source Going to Be Okay?"
date: 2026-02-23
draft: false
categories: [Data Engineering]
tags: [trino, starburst, ai, open-source, lakehouse, iceberg, vector-store]
showTableOfContents: true
summary: "Trino open-source releases dropped 63% as Starburst pivoted from a query engine company to an AI platform. From the perspective of a team running Trino in production, here's what this shift means and what to do about it."
---

If you run Trino as a production query engine, you've probably felt it throughout 2025. **Releases have been getting sparse.** This isn't just a vague feeling — the numbers tell the story. Trino open-source releases dropped from 30 in 2024 to just 11 in 2025. A 63% decline.

In this post, I analyze why and how Starburst pivoted to become an AI-centric platform company, and lay out how Trino open-source users should think about this shift.

---

## What Happened to Trino Releases

### A Sharp Drop in Release Frequency

Looking at Trino open-source releases by quarter, the downward trend is unmistakable.

| Period | Releases | Avg. Cadence | Notes |
|--------|----------|-------------|-------|
| 2024 Q4 | 9 | Weekly | Stable pattern |
| 2025 Q1 | 6 | Biweekly | Decline begins |
| 2025 Q2 | 2 | Monthly | Sharp drop |
| 2025 Q3 | 1 | Quarterly | All-time low |
| 2025 Q4 | 2 | Every 1.5 months | Still sluggish |

Up through 2024 Q4, a new release came out every week. But starting in 2025 Q2, the cadence dropped to roughly once a month, and in Q3 there was only a single release for the entire quarter. Starburst Enterprise releases were similarly scarce.

The numbers look alarming, but this doesn't signal the "decline" of Trino. **It's a strategic choice by Starburst.**

### Why the Drop

The answer becomes clear when you look at Starburst's contribution to Trino open source. In 2024, the Starburst team accounted for **84%** of all Trino commits. 138 contributors, 2,822 commits, and over 50 companies participated — but the real development engine was Starburst. And that engine started redirecting its engineering resources elsewhere.

That "elsewhere" is AI.

---

## Starburst's AI Pivot

### The Shift in Positioning

Tracking the evolution of Starburst's official messaging reveals just how deliberate this pivot was.

| Period | Positioning | Core Message |
|--------|-----------|-------------|
| ~2023 | Open Data Lakehouse Company | Distributed query engine built on Trino |
| 2024 | Data Lake Analytics Platform | Federated queries + Iceberg |
| 2025 | **Data Platform for Apps and AI** | AI Agent + Agentic Workforce |

From "Open Data Lakehouse" to "Data Platform for Apps and AI." This wasn't just a marketing rebrand — the entire product roadmap was realigned in this direction.

### 2025 Key Announcements Timeline

**May 2025 — Launch Point**

Starburst officially announced AI Agent and AI Workflows.

- **AI Agent**: A natural-language interface for querying data. A question like "What were our sales in Europe last quarter?" is automatically converted to SQL. Air-gapped environments (finance, healthcare, government) are explicitly supported, along with Google's Agent2Agent protocol and Anthropic's Model Context Protocol (MCP).
- **AI Workflows**: A pipeline for storing vector embeddings in Iceberg tables and leveraging structured, semi-structured, and unstructured data for AI training. RAG (Retrieval-Augmented Generation) is natively supported.
- **Other**: Starburst Data Catalog (replacing Hive Metastore), Automated Table Maintenance (automated file cleanup and compaction), Native ODBC Driver, Role-based Query Routing

**October 2025 — AI & Datanova 2025**

Here, Starburst went a step further by announcing the **Agentic Workforce** platform and **Lakeside AI Architecture**.

The core concept is **Model-to-Data architecture**. Whereas the traditional approach collects data into a centralized warehouse and then runs AI models against it, Starburst proposes sending the AI model to where the data lives.

```
Traditional:   Data → Centralized Warehouse → AI Model
Starburst:     AI Model → Federated Data (where it lives)
```

The logic is that by not moving data, you can maintain data sovereignty (GDPR, Schrems II) while still enabling unified analytics. Global financial institutions like Citi and HSBC are reportedly using this approach to unify data across 165 countries.

### The Shift in the Tech Stack

Layering Starburst's tech stack makes it clear what was added in 2025.

```
┌─────────────────────────────────────┐
│  AI Agent & Agent2Agent Protocol    │  ← 2025 NEW
├─────────────────────────────────────┤
│  AI Workflows (Vector Store)        │  ← 2025 NEW
├─────────────────────────────────────┤
│  Starburst Data Catalog             │  ← 2025 NEW
├─────────────────────────────────────┤
│  Lakehouse (Trino + Iceberg)        │
├─────────────────────────────────────┤
│  Federated Data Sources (50+)       │
└─────────────────────────────────────┘
```

Trino still sits in the foundational layer, but all the innovation is happening above it. This is precisely why Trino open-source releases have slowed. **Engineering resources have shifted to the upper layers.**

---

## Vector Store on Iceberg: A Technical Innovation Worth Watching

Among Starburst's 2025 announcements, the most technically interesting is the approach of **storing vector embeddings directly in Apache Iceberg tables**.

Here's why this matters:

- **No separate vector DB required.** You no longer need to operate dedicated vector databases like Pinecone, Weaviate, or Milvus.
- **Leverages existing data engineering skills.** You can manage vector data the same way you already manage Iceberg tables.
- **Iceberg's advantages extend to vector data.** Time travel, ACID transactions, schema evolution, partitioning — all applicable to vector data.
- **Governance policies apply consistently.** The same access controls, audit logs, and data masking policies can be applied to both structured data and vector data.
- **Open format means no vendor lock-in.**

Assuming a future where AI workloads become a core requirement of data platforms, this approach is remarkably pragmatic. There may be trade-offs in search performance compared to dedicated vector DBs, but the reduction in operational complexity and unified governance make this attractive for enterprise environments.

---

## Business Results: Is the Strategy Working in the Market?

The business metrics prove that the AI pivot isn't just marketing.

**FY25 Results (announced February 2025):**

| Metric | Result |
|--------|--------|
| New customers | 20% YoY increase |
| Galaxy (SaaS) customers | 76% YoY increase |
| Galaxy usage | 94% YoY increase |
| Largest deal | Eight-figure multi-year contract with a global financial institution |
| Partnerships | Selected as the query engine for Dell Data Lakehouse |

The 76% growth in Galaxy (SaaS) customers is particularly noteworthy. It signals an accelerating shift toward cloud managed services — which, incidentally, is an alternative for teams currently self-managing open-source Trino.

The customer roster is equally impressive:
- **HSBC**: Data integration across 165 countries
- **Citi**: Unified analytics while maintaining global data sovereignty
- **Vectra AI**: Threat detection platform across 120 countries
- **ZoomInfo**: Multi-cloud data integration

---

## Competitive Positioning

Comparing Starburst's position against competitors in the data platform market sharpens its points of differentiation.

| Capability | Databricks | Snowflake | Dremio | Starburst |
|-----------|-----------|-----------|--------|-----------|
| AI Agent | O | O | X | O |
| Federated Query | Limited | Limited | O | **Core strength** |
| Data Sovereignty | Limited | Limited | Limited | **Core strength** |
| Open-source foundation | Spark | X | Arrow | Trino |
| Vector Store | O | O | X | O (Iceberg) |
| On-prem + Cloud | O | Limited | O | **Core strength** |

Starburst's core differentiators boil down to three things:

1. **True federation**: Real-time queries across 50+ data sources. Analyze data in place without moving it.
2. **Data sovereignty**: Unified analytics while complying with regulations across 165 countries. A decisive advantage in GDPR and Schrems II environments.
3. **Hybrid deployment**: Simultaneous support for on-premise and multi-cloud. Critical value for regulated industries where cloud migration is slow.

On the other hand, **Databricks** is an AI/ML Lakehouse centered on Spark + Delta Lake, strengthening governance through Unity Catalog. **Snowflake** added Iceberg support in 2024 and is pushing AI/ML through Snowpark, but its model still assumes data centralization. **Dremio** emphasizes Arrow Flight-based performance and a semantic layer, but still lags in enterprise features.

An interesting remark from Starburst CEO Justin Borgman: **"What they've done for Spark is what we aim to do for Presto (Trino)."** Just as Databricks built a powerful commercial platform on top of the Spark open-source project, Starburst aims to build the same structure on top of Trino.

---

## Is Trino Open Source Going to Be Okay?

### The Split Between Open-Source and Commercial Features

Here's what currently remains in Trino open source versus what has moved to Starburst-only.

**Trino Open Source:**
- Core query engine
- Standard connectors
- Fault-tolerant execution
- SQL MERGE
- Basic security features

**Starburst Only:**
- Warp Speed (up to 7x performance improvement)
- AI Agent & AI Workflows
- Starburst Data Catalog
- Advanced governance (RBAC, data masking, audit logs)
- Automated Table Maintenance
- Smart Indexing
- Materialized Views (partial)

The most notable item is **Warp Speed**. The fact that a proprietary indexing/caching layer delivering up to 7x performance improvement is commercial-only means that the performance gap between open source and the commercial product could widen for large-scale workloads.

### Reasons for Optimism

- **The Trino core engine has reached maturity.** As a distributed SQL query engine, it has most of the features it needs. Fewer releases doesn't mean lower quality.
- **The community is still active.** Trino Summit 2024 was a success with participation from Netflix, LinkedIn, Wise, and others. The Trino Community Broadcast continues to run. Slack and GitHub activity remains healthy.
- **Over 50 companies are contributing.** Even if Starburst's contributions decline, there's room for other companies to pick up the slack.

### Reasons for Concern

- **The company responsible for 84% of commits has started focusing elsewhere.** Whether other companies have sufficient incentive to fill this gap is an open question.
- **The key to performance optimization is commercial-only.** Teams running large-scale workloads without Warp Speed may find themselves at an increasing disadvantage.
- **All AI-related innovation is concentrated in the commercial product.** In a future where AI becomes essential to data platforms, competing on open source alone may become difficult.

---

## What Trino Operations Teams Should Consider

For teams running Trino in production, it helps to think about this situation across two time horizons.

### Short Term (1-2 years): No Major Concerns

Trino open source is still stable and battle-tested in production. Core features are sufficiently mature, and the baseline query performance and connector ecosystem are solid. There's no reason to switch to an alternative right now.

### Mid to Long Term (3-5 years): Strategic Preparation Is Needed

You need to account for the possibility that the feature gap between open source and the commercial product will widen. Preparation is needed in the following areas:

1. **Performance optimization**: How will you maintain performance for large-scale workloads without Warp Speed? Consider your own caching layers, indexing strategies, or expanding the role of complementary engines like StarRocks.
2. **AI integration**: When AI integration becomes an organizational requirement for the data platform, evaluate whether open-source Trino alone is sufficient. Can you implement approaches like Vector Store on Iceberg on your own, or do you need to combine other tools?
3. **Governance**: As the organization grows and regulations tighten, the need for advanced governance features (RBAC, data masking, audit logs) will increase. Can open source alone meet these requirements?
4. **Alternative evaluation**: Periodically evaluate adopting Starburst Galaxy, switching to other query engines, or hybrid approaches (Trino for batch, StarRocks for real-time).

---

## Final Thoughts

Starburst's AI pivot isn't just a marketing play. The business metrics — 76% growth in Galaxy customers, the largest contract in company history — prove that this strategy is working in the market. **The transition from a query engine company to an AI platform company is irreversible.**

Trino open source isn't dying anytime soon. But it's approaching a maintenance-mode state of being "sufficiently mature," and the center of gravity for innovation has clearly shifted to the commercial product. This is the same pattern Databricks followed with Spark.

If you're running Trino in production, **you can rest easy for now, but preparation for three years from now should start today.** The prudent approach is to lean on open source's stability while securing strategic options along two axes: the performance gap and AI integration.

> Technical debt always accumulates quietly. And the interest is always more expensive than you thought.

**References:**
- [Trino Release Notes](https://trino.io/docs/current/release.html)
- [Starburst Enterprise Release Notes](https://docs.starburst.io/latest/release.html)
- [TechTarget: Addition of new AI capabilities shows Starburst's growth](https://www.techtarget.com/searchdatamanagement/news/366618305/Addition-of-new-AI-capabilities-shows-Starbursts-growth)
- [BigDataWire: Starburst's New Platform Aims to Close AI's Biggest Gap](https://www.bigdatawire.com/2025/10/22/starbursts-new-platform-aims-to-close-ais-biggest-gap/)
