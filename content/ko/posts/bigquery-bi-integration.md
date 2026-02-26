---
title: "Superset에서 BigQuery가 무한 블로킹되는 이유: gevent와 gRPC의 호환성 문제"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [superset, bigquery, redash, gunicorn, gevent, grpc, troubleshooting]
showTableOfContents: true
summary: "사용자 권한별로 BigQuery를 BI에서 사용할 수 있도록 Redash와 Superset에 연동했다. Redash는 스키마 조회 불가라는 알려진 한계가 있었고, Superset은 더 심각한 문제를 만났다. SqlLab에서는 정상인데 차트 화면에서 쿼리가 무한 블로킹됐다. 원인은 gunicorn의 gevent 워커가 Python 코어 라이브러리를 멍키패칭하면서 BigQuery SDK의 gRPC 호출을 교착시키는 문제였다. 워커 타입을 gthread로 교체해서 해결했고, 업스트림에 문서 픽스 PR을 기여했다."
---

사용자 권한에 따라 BigQuery 데이터를 BI 도구에서 조회할 수 있도록 지원해달라는 요청이 들어왔다. 기존에 운영하고 있는 Redash와 Superset 양쪽에 BigQuery 연동을 진행했다.

Redash는 비교적 수월했다. 스키마 조회가 안 되는 알려진 한계가 있었지만 쿼리 자체는 동작했다. Superset은 달랐다. 쿼리가 BigQuery에 정상 제출되는데도 결과가 돌아오지 않고 무한 블로킹되는 현상을 만났다. 원인을 추적해보니 Python의 동시성 모델 깊은 곳에서 충돌이 일어나고 있었다.

---

## Redash BigQuery 연동

Redash에서 BigQuery 연동 자체는 간단하다. Data Source에 BigQuery를 추가하고 서비스 계정 JSON 키를 등록하면 된다.

쿼리 실행은 정상 동작한다. 하지만 하나 문제가 있다. 웹 UI에서 데이터셋과 테이블의 스키마를 조회할 수 없다. 사이드바에 테이블 목록이 나오지 않는다.

이 이슈는 Redash 커뮤니티의 알려진 문제(known issue)다. BigQuery의 메타데이터 API가 반환하는 구조가 Redash의 스키마 파싱 로직과 맞지 않는 것으로 보이는데, 커뮤니티에서 수년간 보고됐지만 수정되지 않았다. Redash 프로젝트 자체의 활발도가 낮아진 상황이라 앞으로도 수정될 가능성은 높지 않다.

실제 스키마 확인은 BigQuery 웹 콘솔에서 직접 해야 한다. 쿼리 작성에 불편함이 있지만, Superset에서 BigQuery를 전사 BI로 활용할 계획이었기 때문에 Redash 쪽은 이 상태로 두기로 했다.

---

## Superset BigQuery 연동

### 디펜던시 설치

Superset에 BigQuery를 연동하려면 `sqlalchemy-bigquery`와 `pybigquery` 디펜던시를 추가해야 한다. 우리는 Superset을 커스텀 Docker 이미지로 운영하고 있어서 이미지 빌드 단계에서 디펜던시를 추가했다.

BigQuery 관련 패키지가 기존 Superset 디펜던시와 버전 충돌을 일으키는 부분이 있었다. `google-cloud-bigquery`, `google-auth`, `grpcio` 등의 버전을 맞추는 작업이 필요했다. 디펜던시 컨플릭트를 해결하고 이미지를 빌드해서 배포했다.

Redash와 달리 Superset은 BigQuery 데이터셋과 테이블의 스키마가 웹 UI에 정상적으로 표시됐다. 이것만으로도 Superset을 BigQuery 전용 BI로 활용할 이유가 충분했다.

### 쿼리가 블로킹된다

디펜던시 설치와 커넥션 설정을 마치고 테스트를 시작했는데, 이상한 현상이 발생했다.

SqlLab에서 쿼리를 실행하면 정상적으로 결과가 반환된다. 하지만 차트 개발 화면에서 쿼리를 실행하면 응답이 오지 않는다. 약 60초 후에 Gateway Timeout이 발생한다.

GCP BigQuery 콘솔에서 확인해보면, 블로킹된 쿼리도 BigQuery 쪽에서는 정상적으로 제출되어 완료됐다. 쿼리는 실행됐는데, 결과가 Superset으로 돌아오지 않는 것이다.

이 시점에서 Superset GitHub Issues를 검색했다. 동일한 증상을 보고한 이슈가 이미 있었다. 하지만 이슈 스레드에서 소개된 해결 방법들을 시도해봐도 해결되지 않았다.

---

## SqlLab은 되고 차트는 안 되는 이유

직접 디버깅에 나섰다. SqlLab과 차트 개발 화면 사이에 무엇이 다른지를 추적했다.

### 비동기 모드의 동작 차이

Superset의 비동기 모드(`ASYNC_QUERIES_REDIS`)를 활성화하면, SqlLab에서 실행된 쿼리는 Celery 워커에 위임된다. Celery 워커는 별도의 프로세스에서 쿼리를 실행하고 결과를 Redis에 저장한다. 웹서버는 결과를 폴링해서 사용자에게 반환할 뿐이다.

하지만 대시보드와 차트 개발 환경에서는 비동기 모드가 켜져 있어도 쿼리 제출 로직이 웹서버 프로세스(gunicorn)에서 직접 실행된다. 파드 로그를 확인해서 이 차이를 확인했다.

`GLOBAL_ASYNC_QUERIES` 피처 플래그를 활성화하면 일부 UI에서 추가로 Celery 워커를 사용하지만, 강제 리프레시(캐시 무시) 같은 동작에서는 여전히 웹서버에서 쿼리가 제출된다.

정리하면:

| 실행 환경 | 쿼리 실행 위치 | 결과 |
|----------|-------------|------|
| SqlLab | Celery 워커 | 정상 |
| 차트 개발 화면 | gunicorn 웹서버 | 블로킹 |
| 강제 리프레시 | gunicorn 웹서버 | 블로킹 |

**Celery 워커에서 실행되면 정상, gunicorn 웹서버에서 실행되면 블로킹.** 문제는 gunicorn에 있다.

### 코드 레벨 블로킹 지점

코드를 추적해서 블로킹이 발생하는 정확한 지점을 확인했다. Superset의 `db_engine_specs/bigquery.py`에서 `python-bigquery` SDK의 메서드를 호출하는 부분이었다. Celery 워커에서는 이 호출이 정상적으로 반환되지만, gunicorn 웹서버에서는 반환되지 않고 멈춘다.

`python-bigquery` 라이브러리와 gunicorn 서버 사이에 호환성 문제가 있다는 확신이 들었다.

---

## 원인: gevent의 멍키패칭이 gRPC를 교착시킨다

### gunicorn의 워커 타입

gunicorn은 여러 워커 타입을 지원한다. 우리가 사용하던 워커 타입은 `gevent`였다.

- **sync**: 워커 하나가 요청 하나를 처리. 단순하지만 동시성이 낮다.
- **gthread**: 워커당 스레드 풀. Python의 네이티브 스레딩 사용.
- **gevent**: greenlet 기반 코루틴. 적은 리소스로 수천 개의 동시 커넥션을 처리할 수 있다.

gevent는 높은 동시성을 제공하기 위해 특별한 방법을 쓴다. Python 프로세스가 시작될 때 표준 라이브러리의 I/O, 소켓, 스레딩 관련 모듈을 자체 구현으로 동적 교체한다. 이를 **멍키패칭(monkey patching)**이라고 한다. `import socket`을 하면 실제로는 gevent의 소켓이 로드되는 식이다.

### 멍키패칭과 gRPC의 충돌

BigQuery Python SDK(`google-cloud-bigquery`)는 내부적으로 gRPC를 사용한다. gRPC는 C 코어 위에 Python 바인딩이 올라간 구조로, 네이티브 스레드를 사용한다.

문제는 gevent가 Python의 `concurrent.futures.ThreadPoolExecutor`를 멍키패칭할 때 발생한다. BigQuery SDK가 gRPC 호출 결과를 기다리면서 `ThreadPoolExecutor`를 사용하는데, gevent가 이 executor를 패칭한 버전은 gRPC의 네이티브 스레드와 호환되지 않는다. 결과적으로 gRPC 호출이 완료됐는데도 Python 쪽에서 결과를 받지 못하고 무한 대기한다.

이 문제가 새로운 것은 아니었다. `python-bigquery` 레포에는 과거에 `to_dataframe()` 메서드가 gevent 환경에서 블로킹되는 이슈가 보고됐었다. 해당 이슈는 해당 메서드에 한해 픽스됐지만, BigQuery SDK의 다른 gRPC 호출 경로에서도 동일한 문제가 발생한 것이다.

gevent의 GitHub에도, gRPC의 GitHub에도 관련 이슈가 열려 있다. gevent와 gRPC의 조합은 구조적으로 호환되기 어렵다.

### 왜 Celery 워커는 괜찮은가

Celery 워커는 gunicorn과 별개의 프로세스로 실행된다. 기본 설정에서 Celery 워커는 `prefork` 모드를 사용하는데, 이 모드에서는 gevent 멍키패칭이 적용되지 않는다. Python의 표준 스레딩이 그대로 동작하므로 gRPC와의 호환성 문제가 없다.

gunicorn 웹서버 프로세스에서만 gevent 멍키패칭이 적용되고, 그래서 웹서버에서만 블로킹이 발생했다.

---

## 해결: gevent에서 gthread로

### 워커 타입 교체

gunicorn의 워커 타입을 `gevent`에서 `gthread`로 교체했다.

`gthread`는 Python의 네이티브 스레딩을 사용하는 워커 타입이다. 멍키패칭을 하지 않기 때문에 `concurrent.futures.ThreadPoolExecutor`와 gRPC가 정상적으로 동작한다.

교체 후 테스트 결과, 웹서버에서도 BigQuery 쿼리 로직이 블로킹 없이 정상 수행됐다. SqlLab, 차트 개발 화면, 강제 리프레시 모두 정상 동작을 확인했다.

### 동시성 영향

gevent에서 gthread로 바꾸면 동시성 특성이 달라진다. gevent는 greenlet 기반이라 적은 메모리로 수천 개의 동시 커넥션을 처리할 수 있지만, gthread는 OS 스레드 기반이라 동시 커넥션 수가 스레드 풀 크기에 제한된다.

우리 환경에서는 Superset 사용자 수가 동시 수천 커넥션 수준이 아니라서 gthread로도 충분했다. 워커당 스레드 수(`--threads`)를 적절히 설정하면 실질적인 동시성 차이는 크지 않다.

### 업스트림 기여

이 경험을 바탕으로 Superset 공식 문서에 BigQuery 데이터소스 사용 시 gevent 워커를 사용하지 말라는 경고를 추가하는 PR을 제출해서 머지됐다. 동일한 문제로 시간을 낭비하는 사람이 줄어들면 좋겠다는 생각이었다.

커뮤니티 이슈 스레드에도 async 모드에서의 우회 방법과 gthread 교체를 통한 근본 해결을 공유했다.

---

## Impersonation 관련 참고

BigQuery 연동에서 추가로 검토한 부분이 사용자별 권한 제어(impersonation)다. Superset에서 BigQuery에 접근할 때 서비스 계정 하나로 모든 쿼리를 실행하면 세밀한 권한 제어가 어렵다. 개별 사용자의 GCP 계정으로 쿼리를 실행하려면 impersonation 기능이 필요하다.

Superset 커뮤니티에서 BigQuery impersonation에 대한 논의와 OAuth2 기반 데이터베이스 인증 제안(SIP-85)이 진행되고 있다. 아직 완전한 형태로 구현되지는 않았지만, 향후 이 기능이 안정화되면 사용자별 BigQuery 권한 제어가 가능해진다.

---

## 배운 것

**동일한 쿼리인데 실행 경로에 따라 결과가 다르면, 실행 환경의 차이를 의심하자.** SqlLab은 Celery 워커, 차트는 gunicorn 웹서버에서 실행된다는 차이가 핵심이었다. "같은 쿼리니까 같은 결과여야 한다"는 가정이 디버깅의 출발점을 잘못 잡게 만들 수 있다.

**gevent의 멍키패칭은 양날의 검이다.** gevent가 높은 동시성을 제공하는 핵심 메커니즘이 멍키패칭인데, 이것이 gRPC처럼 네이티브 스레드에 의존하는 라이브러리와 충돌을 일으킨다. Python 생태계에서 gevent를 사용할 때는 디펜던시 중에 gRPC나 네이티브 스레딩에 의존하는 라이브러리가 있는지 확인해야 한다.

**워커 타입 선택은 디펜던시에 영향을 받는다.** gunicorn 워커 타입을 선택할 때 동시성 요구사항만 보는 게 아니라, 어플리케이션이 사용하는 라이브러리의 스레딩 모델과 호환되는지도 확인해야 한다. BigQuery SDK + gRPC + gevent는 구조적으로 호환되지 않는다.

**커뮤니티 이슈 스레드를 끝까지 읽으면 실마리가 나온다.** Superset 이슈 스레드에서 "gunicorn 대신 waitress WSGI로 바꾸니 해결됐다"는 코멘트가 gevent 문제라는 방향을 잡는 데 결정적이었다. 이슈 스레드의 최신 코멘트까지 읽는 습관이 중요하다.

**문제를 해결했으면 업스트림에 기여하자.** 문서 한 줄 추가하는 PR이라도 같은 문제를 겪을 사람의 시간을 절약해준다. 기여하는 데 드는 비용 대비 커뮤니티에 주는 가치가 크다.

**참고 자료:**
- [Superset: Unable to query BigQuery data](https://github.com/apache/superset/issues/23979)
- [python-bigquery: to_dataframe blocked with gevent](https://github.com/googleapis/python-bigquery/issues/697)
- [gevent: BigQuery to_dataframe blocking issue](https://github.com/gevent/gevent/issues/1797)
- [gRPC: gevent compatibility](https://github.com/grpc/grpc/issues/4629)
- [Superset docs: add notice not to use gevent with BigQuery](https://github.com/apache/superset/pull/24564)
- [Superset: BigQuery impersonation discussion](https://github.com/apache/superset/discussions/18269)
- [Superset: OAuth2 for databases (SIP-85)](https://github.com/apache/superset/issues/20300)
- [Redash: BigQuery does not show schema](https://discuss.redash.io/t/big-query-does-not-show-schema/10649)
- [Gunicorn: Design / Server Model](https://gunicorn.org/design/)
