---
title: "Starburst의 AI 피벗: Trino 오픈소스는 괜찮을까?"
date: 2026-02-23
draft: false
categories: [Data Engineering]
tags: [trino, starburst, ai, open-source, lakehouse, iceberg, vector-store]
showTableOfContents: true
summary: "Starburst가 쿼리 엔진 회사에서 AI 플랫폼 기업으로 전환하면서 Trino 오픈소스 릴리스가 63% 감소했다. Trino를 프로덕션에서 운영하는 팀의 관점에서, 이 변화가 의미하는 것과 앞으로의 전략을 정리한다."
---

Trino를 프로덕션 쿼리 엔진으로 운영하는 팀이라면 2025년 한 해 동안 느꼈을 것이다. **릴리스가 뜸해졌다.** 감각이 아니라 숫자로 드러나는 변화다. 2024년 30개였던 Trino 오픈소스 릴리스가 2025년에는 11개로 줄었다. 63% 감소.

이 글에서는 Starburst가 왜, 어떻게 AI 중심 플랫폼 기업으로 전환했는지 분석하고 Trino 오픈소스 사용자 입장에서 이 변화를 어떻게 바라봐야 하는지 정리한다.

---

## Trino 릴리스, 무슨 일이 일어났나

### 릴리스 빈도가 급격히 줄었다

Trino 오픈소스 릴리스 패턴을 분기별로 보면 감소 추세가 뚜렷하다.

| 기간 | 릴리스 수 | 평균 주기 | 비고 |
|------|----------|----------|------|
| 2024 Q4 | 9개 | 주 1회 | 안정적 패턴 |
| 2025 Q1 | 6개 | 2주 1회 | 감소 시작 |
| 2025 Q2 | 2개 | 월 1회 | 급격한 감소 |
| 2025 Q3 | 1개 | 분기 1회 | 최저점 |
| 2025 Q4 | 2개 | 1.5개월 1회 | 여전히 저조 |

2024년 Q4까지만 해도 매주 새 릴리스가 나왔다. 그런데 2025년 Q2부터 월 1회 수준으로 떨어졌고 Q3에는 분기 전체에 릴리스가 단 1개였다. Starburst Enterprise 릴리스도 마찬가지로 극소수에 그쳤다.

숫자만 놓고 보면 심각해 보이지만 이것이 Trino의 "쇠퇴"를 뜻하지는 않는다. **Starburst가 내린 전략적 선택이다.**

### 왜 줄었는가

Trino 오픈소스에 Starburst가 얼마나 기여했는지 보면 답이 나온다. 2024년 기준 Starburst 팀이 Trino 전체 커밋의 **84%**를 차지했다. 기여자 138명, 커밋 2,822개, 참여 기업 50곳 이상이었지만 실질적인 개발 동력은 Starburst였다. 그 Starburst가 엔지니어링 리소스를 다른 곳에 집중하기 시작한 것이다.

그 "다른 곳"이 바로 AI다.

---

## Starburst의 AI 피벗

### 포지셔닝이 바뀌었다

Starburst 공식 메시징 변화를 추적하면 전환이 얼마나 의도적이었는지 알 수 있다.

| 시기 | 포지셔닝 | 핵심 메시지 |
|------|---------|-----------|
| ~2023 | Open Data Lakehouse Company | Trino 기반 분산 쿼리 엔진 |
| 2024 | Data Lake Analytics Platform | 페더레이션 쿼리 + Iceberg |
| 2025 | **Data Platform for Apps and AI** | AI Agent + Agentic Workforce |

"Open Data Lakehouse"에서 "Data Platform for Apps and AI"로. 단순한 마케팅 변화가 아니라 제품 로드맵 전체가 이 방향으로 정렬되었다.

### 2025년 주요 발표 타임라인

**2025년 5월 — Launch Point**

Starburst는 AI Agent와 AI Workflows를 공식 발표했다.

- **AI Agent**: 자연어로 데이터를 쿼리하는 인터페이스. "What were our sales in Europe last quarter?"라는 질문이 자동으로 SQL로 변환된다. Air-gapped 환경(금융, 의료, 정부)을 명시적으로 지원하며 Google Agent2Agent 프로토콜과 Anthropic Model Context Protocol(MCP)도 지원한다.
- **AI Workflows**: Vector embeddings를 Iceberg 테이블에 저장하고 구조화/반구조화/비구조화 데이터를 AI 학습에 활용하는 파이프라인. RAG(Retrieval-Augmented Generation)를 네이티브로 지원한다.
- **기타**: Starburst Data Catalog(Hive Metastore 대체), Automated Table Maintenance(파일 정리 및 컴팩션 자동화), Native ODBC Driver, Role-based Query Routing

**2025년 10월 — AI & Datanova 2025**

여기서 Starburst는 **Agentic Workforce** 플랫폼과 **Lakeside AI Architecture**를 발표하며 한 단계 더 나아갔다.

핵심 개념은 **Model-to-Data 아키텍처**다. 기존 방식은 데이터를 중앙 웨어하우스로 모은 뒤 AI 모델을 돌리는 것이었다. Starburst는 반대로 AI 모델을 데이터가 있는 곳으로 보낸다.

```
기존 접근:    Data → Centralized Warehouse → AI Model
Starburst:   AI Model → Federated Data (where it lives)
```

데이터를 이동시키지 않으므로 데이터 주권(GDPR, Schrems II)을 유지하면서도 통합 분석이 가능하다는 논리다. Citi, HSBC 같은 글로벌 금융기관이 165개국에 흩어진 데이터를 이 방식으로 통합하고 있다고 한다.

### 기술 스택이 달라졌다

Starburst 기술 스택을 레이어로 표현하면 2025년에 추가된 부분이 드러난다.

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

Trino는 여전히 기반 레이어에 있지만 새로운 기능은 전부 위에서 만들어지고 있다. Trino 오픈소스 릴리스가 줄어든 이유가 여기 있다. **엔지니어링 리소스가 상위 레이어로 이동한 것이다.**

---

## Vector Store on Iceberg: 주목할 만한 기술적 시도

2025년 Starburst 발표 중 기술적으로 가장 흥미로운 것은 **Apache Iceberg 테이블에 직접 vector embeddings를 저장**하는 접근이다.

왜 의미가 있는가.

- **별도 벡터 DB가 필요 없다.** Pinecone, Weaviate, Milvus 같은 전용 벡터 데이터베이스를 운영하지 않아도 된다.
- **기존 데이터 엔지니어링 스킬을 그대로 쓴다.** Iceberg 테이블을 다루는 방식 그대로 벡터 데이터를 관리할 수 있다.
- **Iceberg가 제공하는 기능이 벡터 데이터에도 적용된다.** Time travel, ACID 트랜잭션, 스키마 진화, 파티셔닝을 벡터 데이터에서도 활용 가능하다.
- **거버넌스 정책을 일관되게 적용할 수 있다.** 구조화 데이터와 벡터 데이터에 동일한 접근 제어, 감사 로그, 데이터 마스킹 정책을 걸 수 있다.
- **오픈 포맷이므로 vendor lock-in이 없다.**

AI 워크로드가 데이터 플랫폼에서 빠질 수 없는 요소가 되는 미래를 가정하면 이 접근은 상당히 실용적이다. 전용 벡터 DB 대비 검색 성능에는 트레이드오프가 있겠지만 운영 복잡도를 줄이고 거버넌스를 통합할 수 있다는 장점은 엔터프라이즈 환경에서 매력적이다.

---

## 비즈니스 성과: 시장에서 통하고 있는가

AI 피벗이 단순한 마케팅이 아니라는 것은 비즈니스 지표가 보여준다.

**FY25 실적 (2025년 2월 발표):**

| 지표 | 성과 |
|------|------|
| 신규 고객 | 전년 대비 20% 증가 |
| Galaxy(SaaS) 고객 | 전년 대비 76% 증가 |
| Galaxy 사용량 | 전년 대비 94% 증가 |
| 최대 계약 | 글로벌 금융기관과 8자리(억 단위) 다년 계약 |
| 파트너십 | Dell Data Lakehouse의 쿼리 엔진으로 선정 |

Galaxy(SaaS) 고객이 76% 늘었다는 점은 주목할 만하다. 클라우드 매니지드 서비스로 전환이 빨라지고 있다는 뜻이고 오픈소스 Trino를 직접 운영하는 팀 입장에서는 대안이 되기도 한다.

주요 고객사도 인상적이다.
- **HSBC**: 165개국 데이터 통합
- **Citi**: 글로벌 데이터 주권을 유지하며 통합 분석
- **Vectra AI**: 120개국 위협 탐지 플랫폼
- **ZoomInfo**: 멀티클라우드 데이터 통합

---

## 경쟁 환경에서 어디에 서 있는가

데이터 플랫폼 시장에서 Starburst를 경쟁사와 비교하면 차별화 지점이 드러난다.

| 기능 | Databricks | Snowflake | Dremio | Starburst |
|------|-----------|-----------|--------|-----------|
| AI Agent | O | O | X | O |
| Federated Query | 제한적 | 제한적 | O | **핵심 강점** |
| Data Sovereignty | 제한적 | 제한적 | 제한적 | **핵심 강점** |
| 오픈소스 기반 | Spark | X | Arrow | Trino |
| Vector Store | O | O | X | O (Iceberg) |
| On-prem + Cloud | O | 제한적 | O | **핵심 강점** |

Starburst가 내세우는 차별화는 두 축이다.

하나는 **페더레이션과 데이터 주권**이다. 50개 이상 데이터 소스에 대한 실시간 쿼리를 지원하면서 데이터를 이동시키지 않는다. 165개국 규제를 준수하면서 통합 분석을 제공하는 점은 GDPR, Schrems II 환경에서 결정적 장점이 된다.

다른 하나는 **하이브리드 배포**다. On-premise와 멀티 클라우드를 동시에 지원한다. 규제 산업에서 클라우드 전환이 더딘 기업에게 강한 소구점이 된다.

**Databricks**는 Spark + Delta Lake 중심 AI/ML Lakehouse로 Unity Catalog를 통해 거버넌스를 강화하고 있다. **Snowflake**는 2024년 Iceberg 지원을 추가하고 Snowpark으로 AI/ML을 밀고 있지만 여전히 데이터 중앙화가 전제다. **Dremio**는 Arrow Flight 기반 성능과 시맨틱 레이어를 내세우지만 엔터프라이즈 기능에서는 아직 격차가 있다.

흥미로운 것은 Starburst CEO Justin Borgman의 발언이다. **"What they've done for Spark is what we aim to do for Presto(Trino)."** Databricks가 Spark 오픈소스 위에 강력한 상용 플랫폼을 구축한 것처럼 Starburst도 Trino 위에 같은 구조를 만들겠다는 것이다.

---

## Trino 오픈소스, 괜찮을 것인가

### 오픈소스와 상용 기능이 갈라지고 있다

현재 Trino 오픈소스에 남아있는 것과 Starburst 전용으로 넘어간 것을 정리하면 다음과 같다.

**Trino 오픈소스:**
- 핵심 쿼리 엔진
- 기본 커넥터
- Fault-tolerant execution
- SQL MERGE
- 기본 보안 기능

**Starburst 전용:**
- Warp Speed (최대 7배 성능 향상)
- AI Agent & AI Workflows
- Starburst Data Catalog
- 고급 거버넌스 (RBAC, 데이터 마스킹, 감사 로그)
- Automated Table Maintenance
- Smart Indexing
- Materialized Views (일부)

가장 눈에 띄는 것은 **Warp Speed**다. 최대 7배 성능 향상을 제공하는 독점 인덱싱/캐싱 레이어가 상용 전용이라는 것은 대규모 워크로드에서 오픈소스와 상용 제품 사이 성능 차이가 점점 벌어질 수 있다는 뜻이다.

### 낙관적으로 볼 수 있는 근거

- **Trino 코어 엔진은 이미 성숙 단계다.** 분산 SQL 쿼리 엔진으로서 필요한 기능은 대부분 갖추고 있다. 릴리스 빈도가 줄었다고 품질이 떨어지는 것은 아니다.
- **커뮤니티는 여전히 활발하다.** Trino Summit 2024에는 Netflix, LinkedIn, Wise 등이 참여했고 Trino Community Broadcast도 계속 운영되고 있다.
- **50곳 이상 기업이 기여하고 있다.** Starburst 기여가 줄더라도 다른 기업이 메울 여지는 있다.

### 우려할 점

- **커밋의 84%를 담당하던 회사가 다른 곳에 집중하기 시작했다.** 나머지 기업이 이 공백을 메울 동기가 충분한지는 불확실하다.
- **성능 최적화 핵심이 상용 전용이다.** Warp Speed 없이 대규모 워크로드를 운영하는 팀은 갈수록 불리해질 수 있다.
- **AI 관련 새 기능이 모두 상용 제품에 몰려 있다.** 데이터 플랫폼에 AI가 필수가 되는 미래에서 오픈소스만으로는 경쟁력을 확보하기 어려워질 수 있다.

---

## Trino 운영 팀이 고려해야 할 것

Trino를 프로덕션에서 운영하는 팀 입장에서 이 상황을 시간 축 두 개로 나눠 생각해 볼 수 있다.

### 단기 (1~2년): 큰 문제 없다

Trino 오픈소스는 여전히 안정적이고 프로덕션에서 검증된 기술이다. 핵심 기능은 충분히 성숙했고 기본 쿼리 성능과 커넥터 생태계는 탄탄하다. 당장 대안으로 갈아타야 할 이유는 없다.

### 중장기 (3~5년): 전략적 대비가 필요하다

오픈소스와 상용 제품 사이 기능 격차가 벌어질 가능성을 감안해야 한다. 특히 다음 영역에서 대비가 필요하다.

1. **성능 최적화**: Warp Speed 없이 대규모 워크로드 성능을 어떻게 확보할 것인가. 자체 캐싱 레이어나 인덱싱 전략을 검토하고 StarRocks 같은 보완 엔진 도입도 따져봐야 한다.
2. **AI 통합**: 데이터 플랫폼에 AI를 통합하는 것이 조직 요구사항이 될 때 오픈소스 Trino만으로 충분한지 평가해야 한다. Vector Store on Iceberg 같은 접근을 자체 구현할 수 있는지, 다른 도구와 조합이 필요한지 검토가 필요하다.
3. **거버넌스**: 조직이 커지고 규제가 강화될수록 고급 거버넌스 기능(RBAC, 데이터 마스킹, 감사 로그)이 더 절실해진다. 오픈소스만으로 이를 충족할 수 있는지 따져봐야 한다.
4. **대안 평가**: Starburst Galaxy 도입이나 다른 쿼리 엔진으로 전환, 하이브리드 접근(배치는 Trino, 실시간은 StarRocks) 등을 주기적으로 비교 평가해야 한다.

---

## 마치며

Starburst의 AI 피벗은 단순한 마케팅이 아니다. Galaxy 고객 76% 증가, 역대 최대 계약 등 비즈니스 지표가 이 전략이 시장에서 먹히고 있음을 보여준다. **쿼리 엔진 회사에서 AI 플랫폼 기업으로 전환하는 흐름은 되돌리기 어렵다.**

Trino 오픈소스는 당장 죽지 않는다. 하지만 "충분히 성숙한" 상태로 유지보수 모드에 가까워지고 있으며 새로운 기능 개발의 무게 중심은 분명히 상용 제품으로 옮겨갔다. Databricks가 Spark에 대해 했던 것과 같은 패턴이다.

Trino를 프로덕션에서 운영하는 팀이라면 **지금 당장은 안심해도 되지만 3년 뒤를 위한 대비는 지금 시작해야 한다.** 오픈소스가 주는 안정성에 기대면서도 성능 격차와 AI 통합이라는 두 축에서 선택지를 확보해 두는 것이 현명하다.

> 기술 부채는 늘 조용히 쌓인다. 이자는 언제나 우리가 생각했던 것보다 비싸다.

**참고 자료:**
- [Trino Release Notes](https://trino.io/docs/current/release.html)
- [Starburst Enterprise Release Notes](https://docs.starburst.io/latest/release.html)
- [TechTarget: Addition of new AI capabilities shows Starburst's growth](https://www.techtarget.com/searchdatamanagement/news/366618305/Addition-of-new-AI-capabilities-shows-Starbursts-growth)
- [BigDataWire: Starburst's New Platform Aims to Close AI's Biggest Gap](https://www.bigdatawire.com/2025/10/22/starbursts-new-platform-aims-to-close-ais-biggest-gap/)
