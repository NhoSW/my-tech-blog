---
title: "Iceberg - 왜 CDC 테이블의 컴팩션이 까다로운가"
date: 2026-02-24
draft: false
categories: [Data Engineering]
tags: [iceberg, cdc, compaction, lakehouse, kafka-connect, deletion-vector, iceberg-v3]
showTableOfContents: true
summary: "Iceberg v2의 row-level delete 구현, position delete와 equality delete의 차이, 실시간 CDC 싱크와 컴팩션 간 커밋 충돌 원인을 코드 레벨에서 분석했다. v3 Deletion Vector가 이 문제를 어떻게 바꾸는지, 그리고 v2 환경에서의 운영 우회책까지 함께 정리한다."
---

Iceberg CDC 테이블에 컴팩션을 돌리면 이런 에러가 뜬다.

```
org.apache.iceberg.exceptions.ValidationException:
Cannot commit, found new position delete for replaced data file
```

append-only 테이블에서는 안 나는 에러다. CDC 테이블에서만 나온다. 왜 그런지 이해하려면 Iceberg v2의 delete 메커니즘부터 알아야 한다.

---

## Iceberg v2의 row-level delete

### 데이터를 지우는 두 가지 방법: COW vs MOR

Iceberg에서 행을 삭제하거나 업데이트하는 방법은 두 가지다.

**Copy-on-Write (COW)**: 변경이 생기면 해당 데이터 파일을 통째로 다시 쓴다. 삭제할 행을 빼고 나머지를 새 파일에 복사한다. 읽기 성능이 좋지만 쓰기 비용이 크다. 배치성 대량 갱신에 적합하다.

**Merge-on-Read (MOR)**: 데이터 파일은 건드리지 않는다. 대신 "이 행은 삭제되었다"는 정보를 별도 **delete file**에 기록한다. 쓰기가 빠르지만 읽을 때 원본과 delete file을 병합해야 하므로 읽기 비용이 올라간다. CDC/Upsert 테이블에 적합하다.

CDC 파이프라인은 업데이트가 쉴 새 없이 들어오므로 MOR이 맞다. 문제는 이 MOR 경로에서 생기는 delete file이 컴팩션과 충돌한다는 점이다.

### Delete file의 두 종류

MOR에서 사용하는 delete file에는 두 가지가 있다.

**Equality delete**: 삭제할 행의 키 값만 기록한다. "이 PK를 가진 행을 지워라"는 의미다. 여러 데이터 파일에 걸쳐 있어도 하나의 delete file로 표현 가능하다. 대신 읽기 시 키 매칭 비용이 든다.

**Position delete**: 특정 데이터 파일의 특정 위치(행 번호)를 기록한다. "A.parquet 파일의 42번째 행을 지워라"는 식이다. 읽기 시 정확히 그 위치만 건너뛰면 되므로 equality delete보다 읽기 성능이 좋다.

| 유형 | 기록 내용 | 읽기 비용 | 적합한 경우 |
|------|----------|----------|-----------|
| Equality delete | 키 값 | 높음 (키 매칭) | PK 기반 대량 삭제 |
| Position delete | 파일 경로 + 행 번호 | 낮음 (위치 점프) | 스트림 내 빈번한 업데이트 |

### 엔진마다 지원 범위가 다르다

Iceberg 스펙에는 두 종류 모두 정의되어 있지만 모든 엔진이 전부 구현한 건 아니다.

| 엔진 | Equality delete 읽기 | Position delete 읽기 | Equality delete 쓰기 | Position delete 쓰기 |
|------|---------------------|---------------------|---------------------|---------------------|
| Spark | O | O | X | O |
| Flink | O | O | O (UPSERT 모드) | 상황에 따라 |
| Trino | O | O | X | O |
| Athena | O | O | X | O |
| Hive/Impala | O | O | X | O |

Spark은 equality delete를 읽을 수는 있지만 쓰지는 못한다. Trino와 Athena도 마찬가지다. 행 삭제 시 position delete만 기록한다. Flink만 UPSERT 모드에서 equality delete 쓰기를 지원한다.

이 차이가 컴팩션 전략에 영향을 준다.

---

## Delta Writer: 같은 커밋 안에서 delete 전략이 갈린다

CDC 싱크에서 iceberg-core의 `BaseTaskWriter`가 레코드를 처리하는 로직이 흥미롭다.

```java
public void deleteKey(T key) throws IOException {
    if (!internalPosDelete(asStructLikeKey(key))) {
        eqDeleteWriter.write(key);
    }
}
```

한 커밋을 만드는 과정에서 삭제 요청이 들어오면 먼저 **이번 스트림(커밋)에 포함된 데이터인지** 확인한다.

- **이번 스트림에 있는 레코드** → position delete로 기록
- **이번 스트림에 없는 레코드** → equality delete로 기록

왜 전부 equality delete로 통일하지 않을까? 이유는 읽기 성능이다. position delete가 MOR 읽기에서 훨씬 효율적이기 때문에 가능한 한 position delete를 쓰려고 한다. iceberg-core 쓰기 로직에 읽기 성능 최적화가 녹아 있는 셈이다.

이 설계가 컴팩션에서 문제를 일으킨다.

---

## 스냅샷 타입과 커밋 충돌

### Iceberg의 네 가지 스냅샷 operation

Iceberg 스냅샷에는 operation 필드가 있다. 어떤 종류의 변경이 일어났는지 나타낸다.

| Operation | 의미 | 대표 작업 |
|-----------|------|----------|
| `append` | 데이터 파일 추가 | 배치 적재, INSERT |
| `replace` | 데이터/삭제 파일 교체 | **컴팩션** |
| `overwrite` | 논리적 덮어쓰기 | **업서트**, UPDATE, DELETE |
| `delete` | 데이터 파일 제거 또는 삭제 파일 추가 | DROP PARTITION 등 |

컴팩션은 `replace`다. 여러 개의 작은 파일을 큰 파일로 합치고 기존 파일을 교체한다. CDC 싱크는 `overwrite`다. 업서트 과정에서 delete file을 생성한다.

### 낙관적 동시성 제어

Iceberg는 낙관적 동시성(optimistic concurrency)으로 커밋 충돌을 처리한다. Git과 비슷하다.

1. 현재 스냅샷을 기준으로 새 메타데이터 트리를 만든다
2. 원자적 커밋(atomic swap)을 시도한다
3. 그 사이에 다른 커밋이 끼어들었으면 **검증(validation)**을 수행한다
4. 검증 통과하면 커밋 성공, 실패하면 재시도

핵심은 3번의 검증 규칙이다. **어떤 스냅샷 타입끼리 충돌하고 어떤 조합은 자동으로 병합되는가.**

---

## Append-only vs CDC: 왜 차이가 나는가

### Append-only 테이블: 충돌이 거의 없다

Append-only 테이블은 데이터를 추가만 한다. delete file이 없다. 컴팩션(`replace`)이 돌아가는 동안 새 데이터가 들어와도(`append`) 서로 다른 파일을 건드리므로 자동 병합된다. Git에서 서로 다른 파일을 수정한 두 브랜치가 충돌 없이 머지되는 것과 같다.

### CDC 테이블: position delete가 문제다

CDC 테이블은 다르다. 업서트가 일어나면 delete file이 생긴다. 컴팩션이 데이터 파일 A를 새 파일 B로 교체하려는 순간, CDC 싱크가 파일 A에 대한 position delete를 만들어 버리면 어떻게 될까?

```
시간 순서:
1. 컴팩션 시작: 파일 A를 읽어서 새 파일 B를 만드는 중
2. CDC 싱크: 파일 A의 42번째 행을 삭제 (position delete 생성)
3. 컴팩션 완료: 파일 A → 파일 B 교체 커밋 시도
4. 검증 실패: "파일 A에 대한 새 position delete가 생겼는데 파일 A를 교체하면 그 delete가 유실된다"
```

```
ValidationException: Cannot commit, found new position delete for replaced data file
```

컴팩션 입장에서는 파일 A를 이미 읽어서 새 파일을 만들었는데 그 사이에 파일 A에 대한 삭제가 추가된 것이다. 이 삭제를 반영하지 않고 교체하면 데이터 정합성이 깨지므로 Iceberg가 커밋을 거부한다.

### Equality delete는 왜 덜 부딪히나

Equality delete는 "이 키를 가진 행을 지워라"라는 논리적 선언이다. 특정 파일에 바인딩되지 않는다. 컴팩션이 파일을 교체하더라도 키 기반 삭제는 새 파일에도 그대로 적용 가능하다.

실제로 Iceberg에는 컴팩션 시 시작 시점의 `sequence-number`를 유지하는 메커니즘이 있다([PR #3480](https://github.com/apache/iceberg/pull/3480)). 이걸로 equality delete와의 충돌을 자동 우회한다. 기본 옵션으로 활성화되어 있다.

하지만 position delete는 특정 파일의 특정 위치를 가리킨다. 파일이 바뀌면 위치도 의미를 잃는다. 자동 우회가 불가능하다.

---

## 커밋 인터벌 딜레마

"그럼 싱크 커밋 인터벌을 조정하면 되지 않나?" 라고 생각할 수 있다. 쉽지 않다.

**인터벌을 길게 잡으면**: 한 커밋에 포함되는 레코드가 많아진다. 같은 레코드가 INSERT 후 UPDATE되거나 DELETE되는 확률이 올라간다. delta writer가 이번 스트림 내 레코드를 position delete로 처리하므로 position delete 발생이 늘어난다.

**인터벌을 짧게 잡으면**: position delete 발생은 줄어들지만 커밋이 자주 일어난다. 컴팩션이 완료되기 전에 새 커밋이 끼어들 확률이 높아진다. position delete가 한 건만 있어도 충돌이 발생한다.

어느 쪽이든 충돌 확률을 0으로 만들 수 없다.

---

## v2에서의 개선 시도 — 전부 실패했다

이 문제를 해결하려는 PR이 여러 개 올라왔지만 **전부 머지되지 못하고 닫혔다.**

| PR | 접근 방식 | 결과 |
|----|----------|------|
| [#4703](https://github.com/apache/iceberg/pull/4703) | 컴팩션 검증 시 position delete를 선택적 무시 | 리뷰어들이 "위험하다" 판단, 닫힘 |
| [#4748](https://github.com/apache/iceberg/pull/4748) | Flink upsert에서 position delete와 데이터 파일의 sequence number가 같다는 점을 이용 | 닫힘 |
| [#5760](https://github.com/apache/iceberg/pull/5760) | manifest entry에 `min-data-sequence-number` 필드를 추가해 비충돌 delete를 필터링 | stale bot이 닫음 |
| [#7249](https://github.com/apache/iceberg/pull/7249) | `position-deletes-within-commit-only` 스냅샷 프로퍼티로 같은 커밋 내 position delete 선언 | stale bot이 닫음 |

결국 v2 프레임워크 안에서는 깔끔한 해법이 나오지 않았다. 커뮤니티의 방향은 **v3에서 구조적으로 해결하는 쪽**으로 수렴했다.

---

## Iceberg v3: Deletion Vector가 바꾸는 것

v3 스펙은 2025년 초 확정되었고 Iceberg 1.8.0(2025년 2월)부터 구현이 들어가기 시작했다. 핵심 변경은 **Deletion Vector(DV)** 도입이다.

### Position delete file → Deletion Vector

v3에서 position delete file은 **신규 생성이 금지**된다. 대신 DV가 그 자리를 대신한다.

> Position delete files must not be added to v3 tables, but existing position delete files are valid.

DV는 Puffin 파일에 저장되는 **Roaring bitmap**이다. 데이터 파일 하나당 "몇 번째 행이 삭제되었는지"를 비트맵으로 표현한다.

| 항목 | v2 Position delete | v3 Deletion Vector |
|------|-------------------|-------------------|
| 저장 형식 | Parquet (파일 경로 + 행 번호 컬럼) | Puffin (Roaring bitmap) |
| 데이터 파일당 수 | 무제한 (N개 누적 가능) | **최대 1개** |
| 새 삭제 발생 시 | 별도 delete file 추가 | 기존 DV를 읽어서 **병합 후 교체** |

### 왜 컴팩션 충돌이 줄어드는가

v2에서 충돌이 나는 이유는 position delete file이 데이터 파일과 **독립적으로 존재**하기 때문이었다. 컴팩션이 데이터 파일을 교체하는 동안 CDC 싱크가 같은 파일에 대한 새 position delete file을 만들면 — 교체 후에 그 delete가 가리키는 파일이 없어진다.

DV는 구조가 다르다.

1. **데이터 파일에 종속**: DV는 해당 데이터 파일의 sidecar다. 컴팩션이 데이터 파일을 새로 쓰면 DV의 삭제분도 함께 반영되고 기존 DV는 제거된다.
2. **독립적 누적 불가**: 데이터 파일 하나에 DV는 최대 하나다. 새 삭제가 들어오면 기존 DV를 읽어서 병합한 뒤 교체한다. v2처럼 delete file이 쌓여서 "파일 A에 대한 새 position delete" 문제가 발생할 여지가 줄어든다.
3. **컴팩션 빈도 자체가 줄어든다**: DV는 compact한 비트맵이라 v2 position delete file처럼 소파일 문제가 없다. 컴팩션을 덜 돌려도 되니 충돌 윈도우도 줄어든다.

### Row Lineage와 행 수준 충돌 검출

v3는 DV 외에 **Row Lineage**도 도입한다. 모든 행에 고유 `_row_id`와 `_last_updated_sequence_number`가 부여된다. DV + Row Lineage를 결합하면 **행 수준(row-level) 충돌 검출**이 가능해진다([Issue #14613](https://github.com/apache/iceberg/issues/14613)).

v2에서는 같은 데이터 파일을 건드리면 무조건 충돌이었다. v3에서는 같은 파일이라도 **서로 다른 행을 수정했으면 자동 병합**될 수 있다. CDC 싱크가 42번째 행을 삭제하고 컴팩션이 다른 행들을 정리하는 상황이라면 충돌 없이 커밋이 가능해지는 셈이다.

### 아직 완벽하지는 않다

OCC(낙관적 동시성)는 여전히 적용된다. 두 writer가 같은 데이터 파일의 DV를 동시에 갱신하면 한쪽은 재시도해야 한다. 하지만 v2와 다른 점이 있다.

- 재시도 비용이 낮다: 비트맵 병합만 다시 하면 된다. 데이터를 다시 스캔할 필요가 없다.
- 충돌 범위가 좁다: 파일 단위가 아니라 DV 단위다.
- Row lineage 기반 행 수준 충돌 검출이 엔진에 구현되면 같은 파일 내 다른 행 수정은 충돌에서 제외된다.

### 엔진 지원 현황 (2025년 기준)

| 엔진 | v3 DV 지원 |
|------|-----------|
| Spark (Iceberg 1.8.0+) | 지원 |
| AWS EMR / Athena / Glue | 2025년 11월 발표, 지원 |
| Databricks | 지원 (row-level concurrency 포함) |
| Trino (Starburst) | 지원 추가 중 |
| Flink | 구현 진행 중 |

v3 마이그레이션은 기존 v2 테이블에서 `ALTER TABLE ... SET TBLPROPERTIES ('format-version' = '3')`로 전환할 수 있다. 기존 position delete file은 유효하게 유지되며 새 delete부터 DV로 기록된다.

---

## 현재 시점의 운영 우회책: CDC를 잠시 멈추고 컴팩션하기

현실적으로 가장 안정적인 방법은 **컴팩션 윈도우 동안 CDC 싱크를 일시 중단**하는 것이다.

```
컴팩션 파이프라인:

1. 새벽 저부하 시간대에 CDC 싱크 커넥터를 pause
2. 컴팩션 실행 (rewrite_data_files)
3. 컴팩션 완료 후 CDC 싱크 재개
```

카카오 테크 블로그에서도 비슷한 운영을 시사하고 있다. 12시간 간격으로 실시간 CDC 싱크를 중단하고 컴팩션을 돌리는 구조다.

### 주의사항

- **컴팩션 시간이 예상보다 길 수 있다.** 하루치 CDC 데이터가 쌓인 테이블을 컴팩션하면 시간이 오래 걸린다. 윈도우 길이를 실측으로 결정해야 한다.
- **파티션별 분할 실행**을 고려하라. 전체 테이블을 한 번에 컴팩션하지 말고 파티션 단위로 나눠서 실행하면 시간을 줄일 수 있다.
- **delete file 정리도 별도로 필요하다.** `rewrite_position_delete_files`로 delete 소파일을 정리하는 minor compaction도 주기적으로 돌려야 한다. 단 버전별 버그가 보고되어 있으니 호환성을 확인하라.

---

## 운영 체크리스트

1. **런타임 버전 확인**: 사용 중인 EMR/Spark/Flink/Trino 버전에서 `rewrite_data_files` 검증 규칙과 `rewrite_position_delete_files` 지원 여부를 확인한다
2. **충돌률 모니터링**: CDC 커밋 주기 대비 컴팩션 수행 시간을 실측하고 충돌 발생 빈도를 추적한다
3. **메타데이터 대시보드**: 스냅샷/매니페스트 테이블에서 파일 수, 평균 크기, delete file 누적량을 시각화한다
4. **컴팩션 윈도우 확보**: 야간 CDC pause 또는 파티션별 분할 컴팩션 전략을 수립한다
5. **커뮤니티 패치 추적**: PR #4703, #7249 같은 개선안의 반영 여부를 버전별로 점검한다

---

## 마치며

정리하면 이렇다.

- Iceberg v2의 MOR 경로에서 **position delete는 읽기 성능을 위한 최적화**다
- 하지만 position delete는 특정 파일에 바인딩되어 있어서 **컴팩션(replace)과 충돌**한다
- Equality delete는 `sequence-number` 메커니즘으로 자동 우회되지만 position delete는 안 된다
- v2 프레임워크 내에서 이 문제를 해결하려는 PR은 전부 머지되지 못했다
- **v3의 Deletion Vector가 구조적 해결책**이다. Position delete file 대신 데이터 파일당 하나의 비트맵으로 삭제를 관리하고 Row Lineage로 행 수준 충돌 검출까지 가능해진다
- v3 전환 전까지는 **컴팩션 윈도우 동안 CDC를 잠시 멈추는 것**이 현실적인 답이다

v3 스펙은 확정되었고 엔진 지원도 빠르게 확대되고 있다. v3로 전환하면 이 글에서 다룬 운영 부담 대부분이 사라진다. 아직 v2를 쓰고 있다면 v3 마이그레이션 계획을 세우는 게 장기적으로 맞다.

**참고 자료:**
- [Row-Level Changes on the Lakehouse: COW vs MOR in Apache Iceberg (Dremio)](https://www.dremio.com/blog/row-level-changes-on-the-lakehouse-copy-on-write-vs-merge-on-read-in-apache-iceberg/)
- [Apache Iceberg Spec - Row-level Deletes](https://iceberg.apache.org/spec/#row-level-deletes)
- [Apache Iceberg - Reliability (Concurrency)](https://iceberg.apache.org/docs/latest/reliability/)
- [PR #3480: Core: support rewrite data files with starting sequence number](https://github.com/apache/iceberg/pull/3480)
- [PR #4703: API: Optionally ignore position deletes in rewrite validation](https://github.com/apache/iceberg/pull/4703)
- [PR #7249: Avoid conflicts between rewrite datafiles and flink CDC writes](https://github.com/apache/iceberg/pull/7249)
- [로그 유형별 Iceberg 테이블 적재 및 운영 전략 (Kakao Tech)](https://tech.kakao.com/posts/656)
- [Improve Position Deletes in V3 (Issue #11122)](https://github.com/apache/iceberg/issues/11122)
- [Row Lineage for V3 (Issue #11129)](https://github.com/apache/iceberg/issues/11129)
- [Row-level concurrency (Issue #14613)](https://github.com/apache/iceberg/issues/14613)
- [What's New in Apache Iceberg v3 (Google Open Source Blog)](https://opensource.googleblog.com/2025/08/whats-new-in-iceberg-v3.html)
- [Iceberg V3 Deletion Vectors on Amazon EMR (AWS Blog)](https://aws.amazon.com/blogs/big-data/unlock-the-power-of-apache-iceberg-v3-deletion-vectors-on-amazon-emr/)
