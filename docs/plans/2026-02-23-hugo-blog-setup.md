# Hugo Blog Setup Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Set up a bilingual (EN/KO) Hugo tech blog with PaperMod theme, sample content, AdSense/Analytics placeholders, and GitHub Actions deployment.

**Architecture:** Hugo static site in `/Users/otl/IdeaProjects/my-blog` with PaperMod theme (git submodule), bilingual content (EN default + KO at /ko/), GitHub Actions CI/CD deploying to GitHub Pages.

**Tech Stack:** Hugo v0.156.0 (extended), PaperMod theme, GitHub Actions, GitHub Pages

---

### Task 1: Install Hugo and Initialize Site

**Files:**
- Create: `hugo.yaml` (Hugo will scaffold but we replace it)
- Create: `.gitignore`
- Remove: `pyproject.toml`, `.venv/`

**Step 1: Install Hugo extended via Homebrew**

Run: `brew install hugo`
Expected: Hugo extended installed successfully

**Step 2: Verify Hugo installation**

Run: `hugo version`
Expected: Output contains `hugo v0.156` and `extended`

**Step 3: Clean existing files**

Run:
```bash
cd /Users/otl/IdeaProjects/my-blog
rm -f pyproject.toml
rm -rf .venv
```

**Step 4: Initialize Hugo site in current directory**

Run:
```bash
hugo new site /Users/otl/IdeaProjects/my-blog --force
```
Expected: Hugo site scaffolded (--force because directory exists)

**Step 5: Initialize git repo**

Run:
```bash
cd /Users/otl/IdeaProjects/my-blog
git init
```

**Step 6: Create .gitignore**

Write `.gitignore`:
```
public/
resources/
.hugo_build.lock
node_modules/
.DS_Store
.idea/
.venv/
```

**Step 7: Commit**

```bash
git add .gitignore hugo.toml archetypes/ content/ layouts/ static/ themes/ data/ i18n/ assets/
git commit -m "chore: initialize Hugo site"
```

---

### Task 2: Add PaperMod Theme

**Files:**
- Create: `themes/PaperMod/` (submodule)

**Step 1: Add PaperMod as git submodule**

Run:
```bash
cd /Users/otl/IdeaProjects/my-blog
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

**Step 2: Verify submodule**

Run: `ls themes/PaperMod/theme.toml`
Expected: File exists

**Step 3: Commit**

```bash
git add .gitmodules themes/PaperMod
git commit -m "chore: add PaperMod theme as submodule"
```

---

### Task 3: Write hugo.yaml Configuration

**Files:**
- Remove: `hugo.toml` (Hugo default, replaced by hugo.yaml)
- Create: `hugo.yaml`

**Step 1: Remove default hugo.toml and write hugo.yaml**

Delete `hugo.toml`. Write `hugo.yaml` with complete config including:
- baseURL: `https://NhoSW.github.io/my-tech-blog/`
- title: "OTL - Data Engineering"
- theme: PaperMod
- Bilingual setup (en default, ko at /ko/)
- PaperMod params (homeInfoParams, socialIcons, ShowReadingTime, ShowCodeCopyButtons, ShowToc, defaultTheme: auto, etc.)
- SEO (enableRobotsTXT, sitemap, OpenGraph, Twitter)
- Menus for both languages
- Taxonomies (categories, tags)
- Search output formats (JSON for Fuse.js)
- googleAnalytics: G-XXXXXXXXXX

Full content of `hugo.yaml`:

```yaml
baseURL: "https://NhoSW.github.io/my-tech-blog/"
languageCode: en-us
defaultContentLanguage: en
title: "OTL - Data Engineering"
theme: PaperMod
paginate: 10
enableRobotsTXT: true
googleAnalytics: "G-XXXXXXXXXX"

buildDrafts: false
buildFuture: false
buildExpired: false

minify:
  disableXML: true
  minifyOutput: true

languages:
  en:
    languageName: "English"
    weight: 1
    title: "OTL - Data Engineering"
    params:
      homeInfoParams:
        Title: "Welcome to OTL"
        Content: >
          Data Engineering blog covering Apache Kafka, Airflow, Trino, StarRocks,
          and modern data infrastructure. Sharing practical lessons from production.
    menu:
      main:
        - identifier: home
          name: Home
          url: /
          weight: 1
        - identifier: categories
          name: Categories
          url: /categories/
          weight: 2
        - identifier: tags
          name: Tags
          url: /tags/
          weight: 3
        - identifier: archives
          name: Archive
          url: /archives/
          weight: 4
        - identifier: search
          name: Search
          url: /search/
          weight: 5
        - identifier: about
          name: About
          url: /about/
          weight: 6

  ko:
    languageName: "한국어"
    weight: 2
    title: "OTL - 데이터 엔지니어링"
    params:
      homeInfoParams:
        Title: "OTL에 오신 것을 환영합니다"
        Content: >
          Apache Kafka, Airflow, Trino, StarRocks 등 데이터 엔지니어링과
          모던 데이터 인프라에 대한 실무 경험을 공유하는 블로그입니다.
    menu:
      main:
        - identifier: home
          name: 홈
          url: /ko/
          weight: 1
        - identifier: categories
          name: 카테고리
          url: /ko/categories/
          weight: 2
        - identifier: tags
          name: 태그
          url: /ko/tags/
          weight: 3
        - identifier: archives
          name: 아카이브
          url: /ko/archives/
          weight: 4
        - identifier: search
          name: 검색
          url: /ko/search/
          weight: 5
        - identifier: about
          name: 소개
          url: /ko/about/
          weight: 6

outputs:
  home:
    - HTML
    - RSS
    - JSON

taxonomies:
  category: categories
  tag: tags

params:
  env: production
  description: "Data Engineering blog - Apache Kafka, Airflow, Trino, StarRocks"
  author: "Seungwoo Noh"

  defaultTheme: auto
  disableThemeToggle: false

  ShowReadingTime: true
  ShowShareButtons: true
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  ShowToc: true
  TocOpen: false
  ShowRssButtonInSectionTermList: true

  socialIcons:
    - name: github
      url: "https://github.com/NhoSW"
    - name: linkedin
      url: "https://linkedin.com/in/"
    - name: email
      url: "mailto:"

  editPost:
    URL: "https://github.com/NhoSW/my-tech-blog/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true

  assets:
    disableFingerprinting: false

  fuseOpts:
    isCaseSensitive: false
    shouldSort: true
    location: 0
    distance: 1000
    threshold: 0.4
    minMatchCharLength: 0
    keys:
      - title
      - permalink
      - summary
      - content

markup:
  goldmark:
    renderer:
      unsafe: true
  highlight:
    noClasses: false
```

**Step 2: Verify config is valid**

Run: `hugo config`
Expected: No errors

**Step 3: Commit**

```bash
git add hugo.yaml
git rm hugo.toml
git commit -m "feat: add hugo.yaml with bilingual PaperMod config"
```

---

### Task 4: Create Search and Archive Pages

**Files:**
- Create: `content/search.md`
- Create: `content/archives.md`
- Create: `content/ko/search.md`
- Create: `content/ko/archives.md`

**Step 1: Write search.md (EN)**

```markdown
---
title: "Search"
layout: "search"
summary: "Search"
placeholder: "Search articles..."
---
```

**Step 2: Write archives.md (EN)**

```markdown
---
title: "Archive"
layout: "archives"
url: "/archives/"
summary: "Archive"
---
```

**Step 3: Write ko/search.md (KO)**

```markdown
---
title: "검색"
layout: "search"
summary: "검색"
placeholder: "글 검색..."
---
```

**Step 4: Write ko/archives.md (KO)**

```markdown
---
title: "아카이브"
layout: "archives"
url: "/ko/archives/"
summary: "아카이브"
---
```

**Step 5: Commit**

```bash
git add content/search.md content/archives.md content/ko/search.md content/ko/archives.md
git commit -m "feat: add search and archive pages for EN and KO"
```

---

### Task 5: Create About Pages

**Files:**
- Create: `content/about.md`
- Create: `content/ko/about.md`

**Step 1: Write about.md (EN)**

About page with: intro as data engineer, blog purpose, tech stack list (Kafka, Airflow, Trino, StarRocks, Kubernetes, Python, SQL), contact info.

**Step 2: Write ko/about.md (KO)**

Korean version of the about page.

**Step 3: Commit**

```bash
git add content/about.md content/ko/about.md
git commit -m "feat: add about pages for EN and KO"
```

---

### Task 6: Create Sample Post 1 - Airflow

**Files:**
- Create: `content/posts/building-data-pipeline-with-airflow.md`

**Step 1: Write the post**

Front matter:
```yaml
title: "Building Reliable Data Pipelines with Apache Airflow: Lessons from Production"
date: 2026-02-23
draft: false
author: "Seungwoo Noh"
categories: [Data Engineering, Apache Airflow]
tags: [airflow, pipeline, orchestration, best-practices]
ShowToc: true
TocOpen: false
summary: "Practical lessons from building and maintaining production Airflow DAGs. Covers DAG design patterns, error handling strategies, and operational best practices."
cover:
  image: ""
```

Content: 500-800 words covering DAG design best practices, production issues and solutions, with Python DAG code example.

**Step 2: Commit**

```bash
git add content/posts/building-data-pipeline-with-airflow.md
git commit -m "feat: add sample post - Airflow data pipeline best practices"
```

---

### Task 7: Create Sample Post 2 - Kafka Connect

**Files:**
- Create: `content/posts/kafka-connect-troubleshooting-guide.md`

**Step 1: Write the post**

Front matter with categories [Apache Kafka, Data Engineering], tags [kafka, kafka-connect, troubleshooting, debugging]. Content: 500-800 words on common Kafka Connect issues and debugging, with config examples.

**Step 2: Commit**

```bash
git add content/posts/kafka-connect-troubleshooting-guide.md
git commit -m "feat: add sample post - Kafka Connect troubleshooting"
```

---

### Task 8: Create Sample Post 3 - Trino

**Files:**
- Create: `content/posts/trino-query-optimization-tips.md`

**Step 1: Write the post**

Front matter with categories [Trino, Data Engineering], tags [trino, sql, optimization, performance]. Content: 500-800 words on Trino query optimization, with SQL examples.

**Step 2: Commit**

```bash
git add content/posts/trino-query-optimization-tips.md
git commit -m "feat: add sample post - Trino query optimization"
```

---

### Task 9: Create Sample Post 4 - StarRocks (Korean)

**Files:**
- Create: `content/ko/posts/starracks-compression-guide.md`

**Step 1: Write the post**

Front matter with title "StarRocks 압축 설정 가이드: 성능과 스토리지 최적화", categories [StarRocks, Data Engineering], tags [starracks, compression, optimization]. Content: 500-800 words in Korean on StarRocks compression settings.

**Step 2: Commit**

```bash
git add content/ko/posts/starracks-compression-guide.md
git commit -m "feat: add sample post - StarRocks compression guide (KO)"
```

---

### Task 10: Create AdSense and Analytics Placeholders

**Files:**
- Create: `layouts/partials/extend_head.html`
- Create: `layouts/partials/adsense-in-article.html`
- Create: `static/ads.txt`

**Step 1: Write extend_head.html**

```html
{{- /* Google AdSense verification meta tag */ -}}
{{- /* Uncomment and replace with your AdSense publisher ID after approval: */ -}}
{{- /* <meta name="google-adsense-account" content="ca-pub-XXXXXXXXXXXXXXXX"> */ -}}
{{- /* <script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-XXXXXXXXXXXXXXXX" crossorigin="anonymous"></script> */ -}}
```

**Step 2: Write adsense-in-article.html**

```html
{{- /* In-article AdSense ad unit */ -}}
{{- /* Uncomment after AdSense approval and replace with your ad unit code: */ -}}
{{- /*
<div class="adsense-in-article">
  <ins class="adsbygoogle"
       style="display:block; text-align:center;"
       data-ad-layout="in-article"
       data-ad-format="fluid"
       data-ad-client="ca-pub-XXXXXXXXXXXXXXXX"
       data-ad-slot="XXXXXXXXXX"></ins>
  <script>
       (adsbygoogle = window.adsbygoogle || []).push({});
  </script>
</div>
*/ -}}
```

**Step 3: Write static/ads.txt**

```
# Google AdSense ads.txt
# Replace with your actual AdSense publisher ID after approval:
# google.com, pub-XXXXXXXXXXXXXXXX, DIRECT, f08c47fec0942fa0
```

**Step 4: Commit**

```bash
git add layouts/partials/extend_head.html layouts/partials/adsense-in-article.html static/ads.txt
git commit -m "feat: add AdSense and Analytics placeholders"
```

---

### Task 11: Create GitHub Actions Deployment Workflow

**Files:**
- Create: `.github/workflows/deploy.yml`

**Step 1: Write deploy.yml**

Use the official GitHub Pages deployment approach:
- Trigger on push to main
- `actions/checkout@v4` with submodules
- `peaceiris/actions-hugo@v3` for Hugo setup (extended, v0.156.0)
- `peaceiris/actions-gh-pages@v4` to deploy to gh-pages branch

```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.156.0'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v4
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

**Step 2: Commit**

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add GitHub Actions workflow for Pages deployment"
```

---

### Task 12: Create README.md

**Files:**
- Create: `README.md`

**Step 1: Write README.md**

Include: blog intro, local dev instructions (`hugo server -D`), deployment info, directory structure, tech stack.

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with setup and deployment instructions"
```

---

### Task 13: Verify - Local Build and Server

**Step 1: Run Hugo build**

Run: `hugo`
Expected: Build succeeds with no errors, shows page counts

**Step 2: Start Hugo dev server**

Run: `hugo server -D`
Expected: Server starts at http://localhost:1313/my-tech-blog/

**Step 3: Verify manually**
- English homepage loads
- Korean homepage at /ko/ loads
- Language switcher works
- All 4 posts visible
- Search page works
- Dark mode toggle works
- Archive page works
