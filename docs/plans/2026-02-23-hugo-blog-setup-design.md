# Design: OTL - Hugo Blog Setup

## Overview
Set up a Hugo + GitHub Pages tech blog with PaperMod theme, bilingual support (EN/KO), sample content, AdSense/Analytics placeholders, and GitHub Actions deployment.

## Key Decisions
- **Blog name**: "OTL - Data Engineering" (EN) / "OTL - 데이터 엔지니어링" (KO)
- **GitHub**: NhoSW/my-tech-blog → https://NhoSW.github.io/my-tech-blog/
- **Directory**: /Users/otl/IdeaProjects/my-blog (in-place, replacing existing pyproject.toml)
- **Theme**: PaperMod via git submodule
- **Config format**: YAML (hugo.yaml)
- **Languages**: English (default, no prefix) + Korean (/ko/ prefix)

## Architecture
- Hugo extended + PaperMod theme (submodule)
- Bilingual: EN default, KO at /ko/
- GitHub Actions deploys on push to main
- GA4 + AdSense placeholders ready

## Content Structure
```
content/
├── posts/                          # EN posts
│   ├── building-data-pipeline-with-airflow.md
│   ├── kafka-connect-troubleshooting-guide.md
│   └── trino-query-optimization-tips.md
├── ko/posts/                       # KO posts
│   └── starracks-compression-guide.md
├── about.md                        # EN about
├── ko/about.md                     # KO about
├── search.md                       # Search page
└── archives.md                     # Archive page
```

## Categories
Data Engineering, Apache Kafka, Apache Airflow, Trino, StarRocks, Data Quality, Infrastructure

## Approved
- Date: 2026-02-23
- Status: Approved by user
