---
title: "2026 Data Engineering Trends and Where Large-Scale Platforms Stand"
date: 2026-02-23
draft: false
categories: [Data Engineering]
tags: [data-engineering, trends, iceberg, airflow, starrocks, trino, ai, lakehouse]
showTableOfContents: true
summary: "Analyzing 2026 data engineering trends from Joe Reis's 1,101-respondent survey, contrasted with our team's current architecture at a large-scale platform. An honest look at what we're doing well and what we need to work on."
---

[Joe Reis](https://joereis.substack.com/p/where-data-engineering-is-heading) published his 2026 data engineering trends based on a survey of 1,101 data practitioners. As someone leading a data engineering team at a large-scale platform, I wanted to contrast these trends against our team's current architecture — taking an honest look at what we're already doing well and what we still need to tackle.

## Our Architecture in a Nutshell

We run a hybrid architecture centered on S3 as the data lake, with Apache Iceberg as the table format, Trino for batch/ad-hoc analytics, and StarRocks for real-time OLAP. Data ingestion relies on Kafka + Debezium CDC and Flink streaming, while orchestration runs on a heavily customized Airflow setup.

```
[Services] → Kafka + Debezium CDC → Flink → S3 (Iceberg)
                                         ↓
                                   ┌─────┴─────┐
                                   │            │
                                 Trino      StarRocks
                              (Batch/Ad-hoc) (Real-time OLAP)
                                   │            │
                                   └─────┬─────┘
                                         ↓
                                     Dashboard
```

---

## 1. AI Adoption — What We're Already Doing and the Walls We Need to Climb

### Trend Summary

82% of survey respondents use AI daily, yet 64% are still stuck at the experimentation stage or using it only for simple tasks. Joe Reis predicts that by the end of 2026, the qualifier "AI-assisted" will disappear from job descriptions entirely.

### What We're Doing

AI coding tools are already part of our daily pipeline development workflow. We actively use LLMs for SQL optimization, code review, and troubleshooting, and we're experimenting with natural-language-based data exploration linked to our data catalog.

### What We Need to Do

The challenge is moving beyond individual AI usage to **embedding AI into the team's entire workflow**.

- Data pipeline anomaly detection
- Automated schema evolution handling
- Auto-generation of data quality rules

To be counted among what Joe Reis calls the "10% of AI-mature teams," we need a strategy that integrates AI not as a simple helper tool but as a **core component of the platform**.

---

## 2. The Data Modeling Crisis and the Semantic Layer — Our Biggest Challenge

### Trend Summary

89% of respondents reported struggling with data modeling, and only 5% of teams are using semantic models. Joe Reis expects the semantic layer to become mainstream first, followed by an evolution toward LLMs interpreting schemas on the fly.

### What We're Doing

We manage lineage and metadata through a data catalog and have defined a table layer hierarchy (L1/L2/L3) to manage data quality at multiple levels. We also run automated data validation through custom Airflow operators.

### What We Need to Do

We evaluated adopting dbt, but that effort stalled because it got caught up in our next-generation data platform migration strategy. Rather than migrating current pipelines to dbt, we're considering moving them directly to the new platform. In the meantime, however, **the standardization and modularization of data transformations remains a gap.** This maps exactly to the "89% pain" that Joe Reis described.

The semantic layer is also uncharted territory for us. Business metric definitions vary across teams, and different teams write different SQL for the same KPI. Just as the survey showed 19% demand for semantic model training, the work of raising data literacy across the entire organization is urgent.

> Especially if we're preparing for a future where AI agents autonomously leverage data, a well-defined semantic layer isn't optional — it's essential. Even if the platform migration is delayed, modeling standards and semantic definitions can — and should — proceed independently.

---

## 3. Orchestration Consolidation — The Future of Airflow

### Trend Summary

Airflow remains dominant, but Dagster has grown from the bottom up, capturing 12% share among small companies. It's also surprising that 20% of teams across all company sizes have no orchestration at all.

### What We're Doing

We run a deeply customized Airflow setup. We've developed our own Provider packages and built platform-specific capabilities including automated data validation operators and custom transfer operators. We're currently working on the **Airflow 3.x major version upgrade**, with a systematic plan covering the Python version upgrade and breaking change migration.

### What We Need to Do

Our deep investment in Airflow is a strength, but it's also technical debt.

- Maintenance burden of custom Providers
- Compatibility issues during version upgrades
- Preparing for the new paradigm of AI agent orchestration

As Joe Reis predicted, we should also keep an eye on the trend of orchestration being absorbed into platforms. Considering alignment with our next-generation data platform, **establishing a mid-to-long-term orchestration strategy roadmap is urgent.**

---

## 4. Lakehouse vs. Warehouse — An Area Where We've Already Found Our Answer

### Trend Summary

In the survey, 44% use a Warehouse, 27% use a Lakehouse, and 12% use a Hybrid approach. As Snowflake and Databricks converge in functionality, the debate itself is becoming moot. Joe Reis predicts that by the end of 2026, the "warehouse vs. lakehouse" debate will feel outdated.

### What We're Doing

On this trend, our team is already close to the right answer. We adopted Iceberg as the standard open table format on S3 and selectively use Trino and StarRocks depending on the use case. This architecture is neither a Warehouse nor a Lakehouse — it **takes the best of both worlds**. Through CDC pipelines, we ingest real-time data into Iceberg tables and run both batch and real-time analytics on the same underlying data.

### What We Need to Do

To take full advantage of new Iceberg v3 features like Deletion Vectors and Row Lineage, we need to ensure compatibility across our query engines. Since Trino and StarRocks currently have limited Iceberg v3 support, **our engine upgrade roadmap and Iceberg version strategy need to be aligned**. We also need to strengthen the governance framework for our open table format architecture — catalog integration, access control, and data quality assurance.

---

## 5. Leadership as a Bottleneck — The Hardest and Most Important Challenge

### Trend Summary

22% of data engineers cited "lack of leadership direction" as a major issue — nearly on par with legacy technical debt at 26%. Joe Reis warns that in 2026, more data teams will be dissolved or merged into engineering organizations.

### What We're Doing

Our data platform team exists as an independent organization, owning end-to-end responsibility from infrastructure to ingestion, transformation, and analytics environments. We maintain direct communication channels with business teams and gather data requirements firsthand.

### What We Need to Do

Technical capability alone can't prove a team's value. As Joe Reis emphasized, **"Only teams that prove business value will survive."**

- A framework for **quantitatively measuring and communicating the ROI** of the data platform
- A **vision** for what role the data platform should play in the AI era
- **Reducing data downtime** through data observability
- Presenting **concrete business impact** such as pipeline development productivity metrics

---

## Summary: What We're Doing Well vs. What We Need to Do

| Area | Doing Well | Need to Do |
|------|-----------|------------|
| AI Adoption | Active use of AI coding tools at the individual level | Embed AI into team workflows, operational automation |
| Data Modeling | Catalog-based metadata management, layer hierarchy defined | Adopt semantic layer, standardize data transformations |
| Orchestration | Deep Airflow customization, 3.x upgrade in progress | Long-term orchestration strategy, AI agent readiness |
| Lakehouse/Warehouse | Iceberg-based hybrid architecture in place | Iceberg v3 compatibility, strengthen governance |
| Leadership | End-to-end platform team operation | Quantify business impact, data observability |

---

## Final Thoughts

The most striking sentence from Joe Reis's survey was this:

> "In 2026, data engineering is less about picking the right tools and more about building the organizational muscle to use them well."

Our team is ahead of the curve when it comes to the tech stack. An Iceberg-based open data lake, a hybrid architecture spanning real-time and batch, deep Airflow customization — these are levels that many organizations haven't reached yet.

But technical advantage alone isn't enough. **Standardized data transformations, a semantic layer, data observability, AI-native workflows, and above all, leadership that proves business value** — this is where we need to focus in 2026.

The debts of the past are accruing interest, and payday is approaching.

**Original source**: [Where Data Engineering Is Heading in 2026 — Joe Reis](https://joereis.substack.com/p/where-data-engineering-is-heading)
