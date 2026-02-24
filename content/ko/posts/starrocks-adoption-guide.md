---
title: "StarRocks 도입기: 실시간 OLAP으로 데이터 파이프라인을 뜯어고친 이야기"
date: 2026-02-23
draft: false
categories: [StarRocks, Data Engineering]
tags: [starrocks, olap, real-time, kafka, performance-tuning, data-pipeline]
showTableOfContents: true
summary: "기존 Trino + Airflow 파이프라인의 5분 지연을 서브초 레이턴시로 개선한 StarRocks 도입 과정을 정리했습니다. 테이블 모델 선택, 데이터 수집, 성능 튜닝, 운영 노하우까지 실무에서 얻은 교훈을 공유합니다."
---

## 도입 배경

데이터 파이프라인을 운영하다보면 한 가지 고민에 반드시 부딪힌다. **실시간 대시보드를 어떻게 만들 것인가?**

우리 팀도 똑같았다. 기존 파이프라인 구조는 이랬다.

```
Service → Kafka → Iceberg → S3 → Trino → Airflow(5분) → Dashboard
```

겉보기엔 잘 돌아갔지만 실무에서 체감하는 문제는 분명했다.

- **최소 5분 지연**. Airflow 스케줄 주기가 병목이었다
- **파이프라인 복잡도**. Kafka → Flink → Redis → API → Dashboard까지 컴포넌트 5개 이상을 관리해야 했다
- **반복되는 I/O**. Trino가 매 쿼리마다 S3를 풀스캔했다
- **높은 개발 비용**. 실시간 대시보드 하나 새로 만드는 데 약 2주 걸렸다

StarRocks를 도입한 뒤 아키텍처가 이렇게 바뀌었다.

```
Service → Kafka → StarRocks → Dashboard (서브초 레이턴시)
```

중간 컴포넌트가 빠지면서 파이프라인이 크게 단순해졌다. Kafka에서 StarRocks로 바로 넣으니 실시간성도 갖출 수 있었다.

## 도입 효과

약 3개월간 PoC를 거치고 6개월에 걸쳐 단계적으로 도입한 결과다.

| 항목 | Before | After | 개선폭 |
|------|--------|-------|--------|
| 대시보드 지연 | 5분 | < 1초 | ~300배 |
| 대시보드 개발 기간 | ~2주 | ~1주 | 50% 단축 |
| 파이프라인 컴포넌트 | 5개 이상 | 2개 | 60% 감소 |
| 쿼리 응답 시간 | 30~50초 | 5~10초 | 5~10배 |
| 하드웨어 비용 | 128GB × 18노드 | 64GB × 3노드 | ~75% 절감 |

> Trino는 절대적인 쿼리 시간에서는 빠르지만 Airflow 스케줄 지연까지 포함한 **end-to-end 레이턴시**와 하드웨어 비용 면에서 StarRocks가 실시간 워크로드에 더 맞았다.

## 테이블 모델 선택 가이드

StarRocks를 처음 도입할 때 가장 신경 써야 할 결정이 테이블 모델이다. 잘못 고르면 나중에 테이블을 다시 만들어야 한다.

### 의사결정 흐름

```
┌─────────────────────────────┐
│  어떤 데이터를 저장하는가?    │
└──────────────┬──────────────┘
               │
       ┌───────▼────────┐
       │  UPDATE 필요?   │
       └───────┬────────┘
               │
      ┌────────┴────────┐
      │                 │
    [아니오]            [예]
      │                 │
┌─────▼─────┐    ┌─────▼──────┐
│  집계 필요? │    │ Primary Key │
└─────┬─────┘    │ (빈번한     │
      │          │  UPDATE)    │
  [아니오]  [예]  └────────────┘
      │      │
┌─────▼───┐ ┌▼──────────┐
│Duplicate│ │Aggregate   │
│(원본 저장)│ │(자동 집계) ★│
└─────────┘ └────────────┘
```

### 모델 비교

| 모델 | 중복 허용 | UPDATE | 자동 집계 | 적합한 용도 |
|------|----------|--------|----------|------------|
| Duplicate Key | O | X | X | 로그, 원본 이벤트 |
| Aggregate Key | X | 자동 | O | 실시간 통계 ★ |
| Primary Key | X | O (고속) | X | 빈번한 UPDATE |

### Duplicate Key. 원본 데이터 저장

클릭 로그나 API 이벤트, 센서 데이터처럼 원본을 그대로 보관해야 할 때 쓴다.

```sql
CREATE TABLE order_events (
    event_id BIGINT,
    event_time DATETIME,
    order_id VARCHAR(50),
    user_id BIGINT,
    event_type VARCHAR(20),
    amount DECIMAL(10, 2)
)
DUPLICATE KEY(event_id, event_time)
PARTITION BY date_trunc('day', event_time)
DISTRIBUTED BY HASH(event_id) BUCKETS 10;
```

### Aggregate Key. 실시간 통계 ★

데이터가 수집되는 시점에 **자동으로 집계**가 일어난다. StarRocks 도입에서 가장 값어치있었던 모델이다.

```sql
CREATE TABLE order_stats (
    stat_time DATETIME NOT NULL COMMENT '5분 간격',
    region VARCHAR(20) NOT NULL,
    delivery_type VARCHAR(20) NOT NULL,
    -- 집계 컬럼: 수집 시 자동으로 집계 함수 적용
    order_count BIGINT SUM DEFAULT "0",
    total_amount DECIMAL(15, 2) SUM DEFAULT "0",
    user_bitmap BITMAP BITMAP_UNION,
    max_amount DECIMAL(10, 2) MAX
)
AGGREGATE KEY(stat_time, region, delivery_type)
PARTITION BY date_trunc('day', stat_time)
DISTRIBUTED BY HASH(stat_time) BUCKETS 10;
```

사용 가능한 집계 함수.

| 함수 | 용도 | 예시 |
|------|------|------|
| `SUM` | 합계 | 주문 건수, 매출 합계 |
| `MAX` / `MIN` | 최대/최소값 | 최고가, 최저가 |
| `REPLACE` | 최신 값 덮어쓰기 | 최종 상태 |
| `BITMAP_UNION` | 정확한 유니크 카운트 | 순 이용자 수 |
| `HLL_UNION` | 근사 유니크 카운트 | 대규모 카디널리티 |

> `BITMAP_UNION`은 HyperLogLog와 달리 **정확한** 유니크 카운트를 제공한다. 비즈니스 KPI 대시보드처럼 정확도가 중요하면 반드시 이걸 쓰자.

### Primary Key. 빈번한 UPDATE

주문 상태를 추적하거나 재고를 관리하는 것처럼 같은 키 데이터가 자주 갱신되는 경우에 맞다.

```sql
CREATE TABLE orders (
    order_id VARCHAR(50) NOT NULL,
    status VARCHAR(20),
    amount DECIMAL(10, 2),
    updated_at DATETIME
)
PRIMARY KEY(order_id)
DISTRIBUTED BY HASH(order_id) BUCKETS 10
PROPERTIES (
    "enable_persistent_index" = "true"
);
```

> `enable_persistent_index`를 켜면 UPDATE 성능이 크게 좋아진다.

## 데이터 수집

### Routine Load. Kafka 실시간 연동

Kafka 토픽에서 데이터를 연속으로 가져오는 방식이다. 실시간 파이프라인 대부분이 이걸 쓴다.

```sql
CREATE ROUTINE LOAD order_load ON orders
COLUMNS(
    order_id,
    user_id,
    timestamp_ms,
    amount,
    order_date = FROM_UNIXTIME(timestamp_ms / 1000)
)
PROPERTIES (
    "format" = "json",
    "jsonpaths" = "[\"$.orderId\",\"$.userId\",\"$.timestamp\",\"$.amount\"]"
)
FROM KAFKA (
    "kafka_broker_list" = "kafka-broker:9092",
    "kafka_topic" = "orders"
);
```

Aggregate Key 테이블과 결합하면 **수집 시점에 변환과 집계를 한번에** 처리할 수 있다.

```sql
CREATE ROUTINE LOAD order_stats_load ON order_stats
COLUMNS(
    timestamp_ms,
    region,
    amount,
    user_id,
    -- 5분 간격으로 라운딩
    stat_time = FROM_UNIXTIME(FLOOR(timestamp_ms / 1000 / 300) * 300),
    order_count = 1,
    total_amount = amount,
    user_bitmap = BITMAP_HASH(user_id)
)
WHERE amount > 0
PROPERTIES ("format" = "json")
FROM KAFKA (
    "kafka_broker_list" = "kafka-broker:9092",
    "kafka_topic" = "orders"
);
```

이 패턴 하나로 기존에 Flink로 처리하던 집계 로직을 SQL만으로 대체했다.

### Stream Load. 벌크 데이터 로딩

파일이나 API로 한번에 대량 로딩할 때 쓴다.

```bash
# CSV 파일 로딩
curl --location-trusted \
  -u user:password \
  -H "label:load_$(date +%Y%m%d%H%M%S)" \
  -H "column_separator:," \
  -T data.csv \
  http://starrocks-fe:8030/api/mydb/mytable/_stream_load
```

## 성능 튜닝 실전 팁

### Thread Pool 설정

동시 접속이 500 RPS 이상인 고부하 환경에서는 기본 Thread Pool 크기가 부족하다.

```properties
# be.conf
pipeline_scan_thread_pool_thread_num = 32   # 기본값: 24
pipeline_exec_thread_pool_thread_num = 32   # 기본값: 24
```

### Bucket Count 가이드라인

| 데이터 크기 | 권장 Bucket 수 |
|-----------|---------------|
| < 10 GB | 10 |
| 10~50 GB | 20 |
| 50~100 GB | 30 |
| > 100 GB | 50+ |

산정 공식. `buckets = max(1, 데이터_크기_GB / 10)`

### 파티셔닝 전략

파티션 컬럼에 함수를 쓰면 파티션 프루닝이 안 된다. 생각보다 자주 실수하는 부분이다.

```sql
-- ✅ 올바른 사용: 파티션 프루닝 동작
WHERE event_time >= NOW() - INTERVAL 3 DAY

-- ❌ 잘못된 사용: 파티션 프루닝 불가
WHERE DATE(event_time) >= CURRENT_DATE - 3
```

### TTL 설정

오래된 파티션을 자동 삭제하려면 TTL을 건다.

```sql
PROPERTIES (
    "partition_live_number" = "3"  -- 최근 3개 파티션만 유지
)
```

## 운영 노하우

### Materialized View 관리

ASYNC 리프레시가 예고 없이 멈출 때가 있다. 정기적으로 상태를 확인하고 문제가 생기면 수동으로 복구해야 한다.

```sql
-- 상태 확인
SHOW MATERIALIZED VIEWS;

-- 강제 동기 리프레시
REFRESH MATERIALIZED VIEW db.mv_name WITH SYNC MODE;

-- 비활성화된 MV 재활성화
ALTER MATERIALIZED VIEW db.mv_name ACTIVE;
```

### Routine Load 모니터링

상태가 `PAUSED`로 바뀌는 경우가 잦다. Kafka offset이 꼬이거나 비정상 메시지가 들어올 때 그렇다.

```sql
-- 상태 확인
SHOW ROUTINE LOAD FOR db.load_job;

-- 재개
RESUME ROUTINE LOAD FOR db.load_job;
```

### Scale-in 주의사항

노드를 축소할 때는 **반드시 Decommission을 먼저** 해야 한다. 이 절차를 빼먹고 노드를 줄이면 데이터가 유실된다.

```sql
-- 1. 현재 노드 확인
SHOW PROC '/backends';

-- 2. 디커미션 시작
ALTER SYSTEM DECOMMISSION BACKEND "<BE_IP>:<HEARTBEAT_PORT>";

-- 3. TabletNum이 0이 될 때까지 대기 후 제거
ALTER SYSTEM DROP BACKEND "<BE_IP>:<HEARTBEAT_PORT>";
```

## 도입 시 알아둘 점

### 알려진 제약 사항

| 이슈 | 설명 | 대안 |
|------|------|------|
| Routine Load | 비정상 메시지 처리 한계 | Kafka 단에서 사전 검증 |
| datetime 파티션 | Iceberg datetime 파티션 호환 이슈 | 대체 파티션 전략 사용 |
| 버전 업그레이드 | 4.x 대에서 버그 경험 | 스테이징 환경 필수 테스트 |

> 버전 업그레이드는 반드시 스테이징에서 충분히 검증하고 프로덕션에 적용하자. 실제로 여러 차례 업그레이드와 다운그레이드를 반복한 적이 있다. 롤백 계획은 항상 준비해두자.

### 도입 체크리스트

**배포 전**
- [ ] 유스케이스와 요구사항 정의
- [ ] 데이터 볼륨 및 증가량 추정
- [ ] 테이블 모델 선택
- [ ] 파티션 전략 설계

**배포 후**
- [ ] Routine Load 작업 생성 및 검증
- [ ] 사용자 권한 설정
- [ ] 데이터 보관 정책(TTL) 설정
- [ ] Scale-in/out 절차 문서화
- [ ] 모니터링 대시보드 구성

## 마치며

StarRocks를 도입하면서 배운 것을 정리한다.

1. **Aggregate Key 모델이 제일 쓸모있다.** 수집 시점에 자동 집계되니까 스토리지와 쿼리 성능을 동시에 잡을 수 있다.
2. **BITMAP_UNION으로 정확한 유니크 카운트를 확보하자.** 비즈니스 KPI에는 근사치가 아니라 정확한 수치가 필요하다.
3. **Routine Load + Aggregate Key 조합이 Flink를 대체한다.** SQL만으로 실시간 집계 파이프라인을 구축할 수 있다.
4. **운영 자동화에 투자하자.** Materialized View와 Routine Load 모니터링은 빠뜨리면 안 된다.

실시간 분석 워크로드에서 StarRocks는 파이프라인 복잡도를 크게 낮춰준다. 다만 버전 업그레이드나 운영 안정성 쪽은 아직 무르익는 단계라서 충분히 PoC하고 스테이징에서 검증한 뒤 도입하길 권한다.

**참고 자료**: [StarRocks 공식 문서](https://docs.starrocks.io)
