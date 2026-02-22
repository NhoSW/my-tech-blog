---
title: "StarRocks 압축 설정 가이드: 성능과 스토리지 최적화"
date: 2026-02-23
draft: false
author: "Seungwoo Noh"
categories: [StarRocks, Data Engineering]
tags: [starrocks, compression, optimization]
ShowToc: true
TocOpen: false
summary: "StarRocks에서 압축 설정을 최적화하여 스토리지 비용을 절감하고 쿼리 성능을 향상시키는 방법을 실무 경험을 바탕으로 정리했습니다."
cover:
  image: ""
---

## 왜 압축 설정이 중요한가

StarRocks를 운영하다 보면 데이터가 수십 TB 규모로 늘어나는 시점이 반드시 온다. 이때 압축 설정 하나로 스토리지 비용이 30~50% 차이 나는 경우를 여러 번 경험했다. 단순히 저장 공간만의 문제가 아니다. 압축률이 높으면 디스크 I/O가 줄어들어 스캔 성능이 좋아지고, 반대로 압축/해제에 CPU를 많이 쓰면 지연 시간이 늘어난다. 결국 워크로드 특성에 맞는 압축 알고리즘을 선택하는 것이 StarRocks 운영의 핵심 튜닝 포인트 중 하나다.

## 지원되는 압축 알고리즘 비교

StarRocks는 여러 압축 알고리즘을 지원한다. 실무에서 주로 사용하는 세 가지를 비교해 보겠다.

| 알고리즘 | 압축률 | 압축 속도 | 해제 속도 | 적합한 워크로드 |
|---------|--------|----------|----------|---------------|
| **LZ4** | 보통 (2~3x) | 매우 빠름 | 매우 빠름 | 실시간 분석, 저지연 쿼리 |
| **ZSTD** | 높음 (4~6x) | 보통 | 빠름 | 배치 분석, 콜드 데이터 |
| **Snappy** | 낮음 (1.5~2x) | 빠름 | 빠름 | 범용, 레거시 호환 |
| **ZLIB** | 높음 (4~5x) | 느림 | 보통 | 아카이빙, 저빈도 접근 데이터 |

개인적으로 가장 많이 쓰는 조합은 **핫 데이터에 LZ4, 콜드 데이터에 ZSTD**다. Snappy는 Hadoop 에코시스템에서 넘어온 데이터를 다룰 때 간혹 사용하지만, 신규 테이블에는 권장하지 않는다.

## 테이블 생성 시 압축 설정 방법

테이블을 생성할 때 `PROPERTIES`에서 `compression` 속성을 지정하면 된다. 별도로 설정하지 않으면 StarRocks 기본값인 LZ4가 적용된다.

### 실시간 분석용 테이블 (LZ4)

```sql
CREATE TABLE analytics.realtime_events (
    event_id       BIGINT,
    user_id        BIGINT,
    event_type     VARCHAR(64),
    event_time     DATETIME,
    properties     JSON
)
ENGINE = OLAP
DUPLICATE KEY(event_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 32
PROPERTIES (
    "replication_num" = "3",
    "compression" = "LZ4"
);
```

LZ4는 해제 속도가 압도적으로 빠르기 때문에, 대시보드 쿼리처럼 수백 밀리초 이내 응답이 필요한 테이블에 적합하다.

### 배치 분석용 테이블 (ZSTD)

```sql
CREATE TABLE warehouse.order_history (
    order_id       BIGINT,
    customer_id    BIGINT,
    order_date     DATE,
    total_amount   DECIMAL(18, 2),
    status         VARCHAR(32),
    items          ARRAY<STRUCT<sku STRING, qty INT, price DECIMAL(10,2)>>
)
ENGINE = OLAP
DUPLICATE KEY(order_id)
PARTITION BY RANGE(order_date) (
    PARTITION p2025 VALUES LESS THAN ('2026-01-01'),
    PARTITION p2026 VALUES LESS THAN ('2027-01-01')
)
DISTRIBUTED BY HASH(customer_id) BUCKETS 16
PROPERTIES (
    "replication_num" = "2",
    "compression" = "ZSTD"
);
```

ZSTD는 압축률이 LZ4 대비 1.5~2배 높아서, 파티션 단위로 수억 건 이상 적재되는 히스토리 테이블에서 스토리지 절감 효과가 크다.

### 기존 테이블 압축 변경

이미 운영 중인 테이블의 압축 알고리즘을 변경하고 싶다면 `ALTER TABLE`을 사용할 수 있다. 단, 변경 이후 새로 적재되는 데이터부터 적용되며 기존 세그먼트는 Compaction이 수행되어야 반영된다는 점에 유의하자.

```sql
ALTER TABLE warehouse.order_history
SET ("compression" = "ZSTD");
```

## 워크로드별 권장 압축 설정

실무에서 반복적으로 검증한 결과를 기반으로 정리하면 다음과 같다.

- **실시간 대시보드 / Ad-hoc 쿼리**: LZ4를 권장한다. CPU 오버헤드가 거의 없어 P99 지연 시간에 미치는 영향이 최소화된다.
- **야간 배치 리포트 / ETL 결과 테이블**: ZSTD를 권장한다. 쿼리 빈도가 낮고 데이터 양이 많은 경우 스토리지 절감 효과가 비용에 직접 반영된다.
- **로그성 대용량 적재**: ZSTD를 사용하되 `zstd_compression_level`을 3 이하로 낮추면 압축 속도와 압축률 사이의 균형을 잡을 수 있다.

## 압축률과 성능 트레이드오프 실측 결과

약 50억 건(원본 약 800GB)의 이벤트 로그 테이블을 대상으로 압축 알고리즘별 벤치마크를 수행한 결과다.

| 지표 | LZ4 | ZSTD (level 3) | ZSTD (level 9) |
|------|-----|----------------|----------------|
| 압축 후 크기 | 320 GB | 195 GB | 170 GB |
| 압축률 | 2.5x | 4.1x | 4.7x |
| 단순 스캔 쿼리 (Avg) | 1.2초 | 1.5초 | 1.8초 |
| 집계 쿼리 (Avg) | 3.4초 | 3.8초 | 4.5초 |
| 데이터 적재 속도 | 120 MB/s | 95 MB/s | 60 MB/s |

LZ4 대비 ZSTD level 3은 스토리지를 약 39% 절감하면서도 쿼리 지연은 약 10~15%만 증가했다. 반면 ZSTD level 9는 추가 압축 이득 대비 적재 속도 저하가 커서 대부분의 환경에서는 level 3이 최적의 선택이었다.

## 운영 팁과 모니터링

마지막으로 실무에서 압축 관련 운영 시 놓치기 쉬운 포인트를 정리한다.

**Compaction 모니터링을 반드시 하자.** 압축 알고리즘을 변경한 뒤 Compaction이 완료되기 전까지 혼합 세그먼트가 존재하면 쿼리 성능이 일시적으로 불안정해질 수 있다. BE의 `compaction_score` 메트릭을 모니터링하여 Compaction 적체 여부를 확인해야 한다.

**테이블 단위로 압축 전략을 분리하라.** 하나의 클러스터에서 모든 테이블에 동일한 압축을 적용하는 것은 비효율적이다. 접근 빈도, 데이터 크기, SLA에 따라 테이블별로 다르게 설정하는 것이 올바른 접근이다.

**디스크 사용량 추이를 추적하라.** 압축 변경 후 `SHOW DATA` 명령으로 테이블별 실제 디스크 사용량을 주기적으로 확인하고, 기대한 압축률이 나오지 않는다면 데이터 특성(카디널리티, NULL 비율 등)을 재점검해야 한다.

```sql
SHOW DATA FROM warehouse.order_history;
```

압축 설정은 한 번 정하고 끝나는 것이 아니라, 데이터 특성과 워크로드가 변함에 따라 지속적으로 재검토해야 하는 영역이다. 이 글이 StarRocks 운영에서 압축 전략을 수립하는 데 실질적인 참고가 되길 바란다.
