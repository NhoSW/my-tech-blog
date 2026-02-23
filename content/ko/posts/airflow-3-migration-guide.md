---
title: "Airflow 3.0 마이그레이션 가이드: 대규모 DAG 환경에서의 실전 경험"
date: 2026-02-23
draft: false
categories: [Data Engineering]
tags: [airflow, migration, orchestration, python, data-pipeline]
showTableOfContents: true
summary: "Airflow 2.x EOL을 앞두고 3.x로 마이그레이션한 실전 경험을 공유한다. 주요 Breaking Changes, 단계적 업그레이드 전략, DAG 호환성 확보 방법, 그리고 수백 개의 DAG을 운영하는 환경에서 배운 교훈을 정리했다."
---

Airflow 2.x의 End of Life가 2026년 4월 22일로 다가오고 있다. 우리 팀은 수백 개의 DAG을 운영하는 프로덕션 환경에서 Airflow 3.x 마이그레이션을 진행했다. 이 글은 그 과정에서 마주친 Breaking Changes, 단계적 업그레이드 전략, 그리고 대규모 DAG 환경에서의 실전 교훈을 정리한 기록이다.

---

## 왜 지금 마이그레이션해야 하는가

### Airflow 3.x의 핵심 개선사항

Airflow 3.x는 단순한 메이저 버전 업데이트가 아니다. 아키텍처 수준에서 근본적인 변화가 일어났다.

- **DAG 버전 관리**: `dag_id`에 버전 서픽스를 붙이거나, 스케줄 변경 시 스케줄링이 꼬이는 문제에서 해방된다.
- **네이티브 백필**: CLI나 커스텀 플러그인에 의존하던 백필이 웹 UI에서 직접 지원된다.
- **이벤트/애셋 기반 트리거**: 단순 cron 표현식을 넘어 다양한 스케줄링 방식을 제공한다.
- **React 기반 웹 UI**: Flask App Builder 기반에서 React로 전면 개편되어 사용성이 크게 향상되었다.

### 아키텍처 변화: API Server의 등장

3.x에서 가장 큰 아키텍처 변화는 **API Server가 메타 DB에 접근하는 유일한 관문**이 되었다는 점이다.

```
Airflow 2.x:
  Webserver ─── MetaDB
  Worker ────── MetaDB
  Scheduler ─── MetaDB
  DAG Code ──── MetaDB (직접 접근 가능)

Airflow 3.x:
  API Server ── MetaDB (유일한 접근 경로)
  Webserver ─── API Server
  Worker ────── API Server
  Scheduler ─── API Server
  DAG Code ──── API Server (직접 접근 불가)
```

이 변화로 인해 **DAG 최상위 코드에서 메타 DB에 직접 접근하던 모든 패턴이 깨진다.** 이것이 마이그레이션에서 가장 큰 영향을 미치는 변경사항이다.

---

## 단계적 업그레이드 전략

한 번에 최신 버전으로 올리는 것은 위험하다. 우리는 4단계 전략을 수립했다.

### 1단계: 2.x 최신 버전(2.11)으로 업데이트 (선택)

3.x로 직접 올리는 과정에서 이슈가 발생할 경우를 대비한 안전장치다. 2.11에서는 3.x에서 제거될 기능들에 대한 deprecation 경고가 표시되므로, 어떤 코드를 수정해야 하는지 사전에 파악할 수 있다.

### 2단계: 3.0.x로 업데이트

Python 3.9 환경에서는 최신 3.1.x가 아닌 **3.0.x까지만 지원**된다. Python 버전 업그레이드 전에 Airflow 메이저 버전을 먼저 올린다.

### 3단계: Python 버전 업그레이드 (3.9 → 3.12+)

Airflow 3.1.x는 Python 3.9를 지원하지 않는다. Python 3.12 이상을 목표로 하되, 의존성 호환 이슈가 있으면 3.10이나 3.11로 타협한다.

### 4단계: 3.1.x로 업데이트

최종적으로 최신 stable 릴리스로 올린다.

### 환경별 순차 적용

```
DEV → BETA & 개인환경 → STAGE → PROD
```

각 환경에서 충분한 검증을 거친 후 다음 환경으로 진행한다. 우리는 DEV 환경에서 약 2주, BETA에서 1주의 검증 기간을 가졌다.

---

## 주요 Breaking Changes와 대응 방법

### 1. `schedule_interval` → `schedule`

가장 흔하게 마주치는 변경사항이다. 기존 `schedule_interval`에 전달하던 cron 표현식을 그대로 `schedule`에 전달하면 된다.

```python
# Before (Airflow 2.x)
DAG(
    dag_id="my_dag",
    schedule_interval="5 2 * * *",
)

# After (Airflow 3.x)
DAG(
    dag_id="my_dag",
    schedule="5 2 * * *",
)
```

단순 치환이지만, DAG 수가 수백 개라면 누락 없이 전부 바꿔야 한다. CI에서 자동 검증하는 방법은 뒤에서 다룬다.

### 2. 존재하지 않는 오퍼레이터 인자 전달 불가

Airflow 3.x에서는 개별 태스크가 메타 DB상에 시리얼라이즈된 DAG을 받아 실행하도록 변경되었다. 이로 인해 `allow_illegal_arguments` 설정이 제거되고, **오퍼레이터에 정의되지 않은 인자를 전달하면 DAG 임포트 자체가 실패**한다.

```python
# 이런 코드가 2.x에서는 경고 없이 동작했지만, 3.x에서는 에러가 발생한다
MyOperator(
    task_id="my_task",
    num_partition=10,  # 실제 인자명은 num_partitions (복수형)
)
```

```
TypeError: Invalid arguments were passed to MyOperator (task_id: my_task).
Invalid arguments were:
**kwargs: {'num_partition': 10}
```

이 변경은 오히려 **잠재적 버그를 발견하는 계기**가 된다. 오랫동안 오타가 있는 인자가 무시되고 있었다면, 이번 마이그레이션에서 바로잡을 수 있다.

### 3. Deprecated 컨텍스트/템플릿 변수 제거

2.x에서 deprecated 경고만 뜨던 변수들이 3.x에서는 완전히 제거되었다. 가장 영향이 큰 것은 `execution_date`다.

| Deprecated 변수 | 대체 변수 |
|----------------|----------|
| `{{ execution_date }}` | `{{ logical_date }}` 또는 `{{ data_interval_start }}` |
| `{{ next_execution_date }}` | `{{ data_interval_end }}` |
| `{{ prev_execution_date_success }}` | `{{ prev_data_interval_start_success }}` |

Jinja 템플릿과 Python 코드 양쪽 모두 수정해야 한다.

```python
# Jinja 템플릿
# Before
"SELECT * FROM table WHERE dt = '{{ execution_date }}'"
# After
"SELECT * FROM table WHERE dt = '{{ logical_date }}'"

# Python context
# Before
execution_date = context["execution_date"]
# After
logical_date = context["logical_date"]
```

### 4. DB별 Operator 통합 → `SQLExecuteQueryOperator`

MySQL, PostgreSQL, Trino 등 DB별로 개별 존재하던 Operator가 하나의 `SQLExecuteQueryOperator`로 통합되었다. 내부적으로 커넥션 타입에 따라 적절한 Hook을 자동으로 선택한다.

```python
# Before (Airflow 2.x)
from airflow.providers.mysql.operators.mysql import MySqlOperator
MySqlOperator(
    task_id="task",
    mysql_conn_id="my_conn",
    sql="SELECT 1"
)

# After (Airflow 3.x)
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
SQLExecuteQueryOperator(
    task_id="task",
    conn_id="my_conn",  # DB별 conn_id → 통합 conn_id
    sql="SELECT 1"
)
```

### 5. `DummyOperator` → `EmptyOperator`

2.x와 3.x 양쪽에서 모두 동작하는 임포트 경로를 사용해야 한다.

```python
# v2에서만 동작 (3.x에서 에러)
from airflow.operators.dummy import DummyOperator

# v3에서만 동작
from airflow.providers.standard.operators.empty import EmptyOperator

# v2 & v3 모두 호환 (권장)
from airflow.operators.empty import EmptyOperator
```

### 6. `SimpleHttpOperator` → `HttpOperator`

```python
# Before
from airflow.providers.http.operators.http import SimpleHttpOperator
# After
from airflow.providers.http.operators.http import HttpOperator
```

### 7. Connection getter 메서드 → 속성 직접 참조

Connection 클래스의 인터페이스가 더 Pythonic하게 변경되었다.

```python
# Before
conn = BaseHook.get_connection("my_conn")
password = conn.get_password()
host = conn.get_host()

# After
conn = BaseHook.get_connection("my_conn")
password = conn.password
host = conn.host
```

### 8. 기타 패키지 경로 변경

```python
# cached_property
# Before: from airflow.compat.functools import cached_property
# After: from functools import cached_property (Python 내장)

# KubernetesPodOperator
# Before: from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import ...
# After: from airflow.providers.cncf.kubernetes.operators.pod import ...
```

---

## 대규모 DAG 환경에서의 마이그레이션 전략

### CI 파이프라인에 v3 호환성 검증 추가

수백 개의 DAG을 수동으로 검증하는 것은 불가능하다. MR(Merge Request) 단계에서 v3 호환성을 자동으로 검증하는 CI 잡을 추가했다.

```yaml
# .gitlab-ci.yml 예시
airflow-v3-compat-check:
  stage: test
  image: apache/airflow:3.0.6-python3.12
  script:
    - pip install -r requirements.txt
    - python -m py_compile dags/**/*.py
    - airflow dags list --output table
  allow_failure: true  # 초기에는 경고만, 이후 필수로 전환
```

처음에는 `allow_failure: true`로 시작해서 현황을 파악하고, 마이그레이션 기한이 다가오면 필수 검증으로 전환한다.

### 이관 허들을 의도적으로 높여라

이것이 우리가 얻은 가장 중요한 교훈이다.

모든 DAG 코드에 일괄 호환성 패치를 적용할 수도 있었다. 하지만 우리는 **의도적으로 이관 난이도를 유지**하기로 결정했다. 이유는 명확하다.

> 관성적으로 운영되고 있지만 실제로는 사용하지 않는 DAG이 상당수 존재한다.

마이그레이션을 계기로 DAG 소유자들이 **"이 DAG이 정말 필요한가?"**를 스스로 검토하도록 유도한 것이다. 결과적으로 상당수의 불필요한 DAG이 정리되었고, 이는 운영 부담 감소로 이어졌다.

구체적인 프로세스:

1. 비활성 DAG 목록을 취합하여 공유 시트에 정리
2. DAG 소유자/소속 부서에 유지 필요 여부를 기한 내 확인하도록 안내
3. 기한 내 응답 없으면 비활성화
4. v3 호환성 패치는 소유자가 직접 수행

### 커스텀 Provider 패키지 선제 대응

사내 커스텀 오퍼레이터나 유틸리티를 Provider 패키지로 제공하고 있다면, **Airflow 코어의 Breaking Changes를 흡수하는 호환 레이어**를 먼저 준비하는 것이 핵심이다.

우리는 커스텀 Provider 패키지를 4차례에 걸쳐 점진적으로 업데이트했다.

- v3.0.0: 기본 호환성 확보
- v3.0.1: 오퍼레이터 인자 검증 대응
- v3.0.2: deprecated 컨텍스트 변수 호환 레이어 추가
- v3.0.3: 문서 및 마이너 버그 수정

사용자 코드의 변경은 최소화하되, Provider 패키지 내부에서 v2/v3 분기 처리를 하는 방식으로 접근했다.

### Helm Chart 업데이트

Kubernetes 환경에서 Airflow를 운영한다면 Helm Chart도 함께 업데이트해야 한다. 3.x에서 도입된 DAG Processor 컴포넌트와 API Server 분리를 반영해야 하기 때문이다.

우선 기존 차트 버전에서 호환성을 확인한 후, 안정화되면 최신 stable 버전으로 업데이트하는 2단계 접근이 안전하다.

---

## FAB Auth Manager 이슈

3.x에서 React 기반으로 웹이 전면 개편되면서, **기존 Flask App Builder(FAB) 기반 Auth Manager가 기본 패키지에서 제거**되었다. 커스텀 Security Manager를 사용하고 있다면 추가 설치와 코드 수정이 필요하다.

```
Failed to import WoowaSecurityManager, using default security manager
```

이런 에러가 발생하면 FAB Auth Manager 패키지를 명시적으로 설치하고, 임포트 경로를 업데이트해야 한다.

---



## 마치며

Airflow 3.x 마이그레이션은 단순한 버전 업그레이드가 아니다. 아키텍처 변화(API Server 중심), 코드 호환성 변경, 인프라 업데이트까지 광범위한 작업이 필요하다.

핵심 교훈을 정리하면 다음과 같다.

1. **단계적으로 올려라** — 한 번에 최신 버전으로 뛰지 말고, 2.11 → 3.0.x → Python 업그레이드 → 3.1.x 순서로 진행하라.
2. **CI에서 자동 검증하라** — 수백 개 DAG의 호환성을 사람이 확인하는 건 불가능하다.
3. **마이그레이션을 정리의 기회로 삼아라** — 이관 허들을 의도적으로 유지해서 불필요한 DAG을 자연스럽게 걸러내라.
4. **커스텀 Provider를 선제 업데이트하라** — 사용자 코드의 변경을 최소화하는 호환 레이어가 핵심이다.

Airflow 2.x EOL까지 아직 시간이 있다고 안심하지 말자. 대규모 환경에서의 마이그레이션은 예상보다 오래 걸린다. 지금 시작해도 늦지 않다.

**참고 자료:**
- [Upgrading to Airflow 3 - Apache Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/howto/upgrading-to-3.html)
- [Apache Airflow 3 is Generally Available!](https://airflow.apache.org/blog/airflow-three-point-zero-is-here/)
- [Airflow 3.x Release Notes](https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html)
