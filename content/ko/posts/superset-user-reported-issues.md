---
title: "Superset 사용자 제보 이슈 6건 대응기"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [superset, csv, encoding, postgresql, troubleshooting]
showTableOfContents: true
summary: "데이터 거버넌스팀에서 Superset 사용 중 발견한 이슈 6건을 대응했다. Slack 스크린샷 확장자 누락, 대시보드 자동 새로고침, SqlLab 데이터셋 덮어쓰기 버그, 윈도우 CSV 한글 깨짐, 쿼리 결과 행 제한, CSV 다운로드 500 에러까지. 가장 까다로웠던 건 50만 행 이상 CSV 다운로드 시 PostgreSQL idle_in_transaction_session_timeout이 메타데이터 DB 커넥션을 끊어버리는 문제였다."
---

데이터 거버넌스팀에서 Superset을 적극적으로 활용하면서 발견한 이슈 6건을 공유해주셨다. 단순한 설정 변경으로 해결되는 것부터, RDS 로그를 뒤져서 원인을 찾아야 했던 것까지 난이도가 다양했다. 각 이슈의 원인과 대응을 정리한다.

---

## 1. Slack 차트 스크린샷 확장자 누락

**제보 내용:** Superset에서 Slack으로 피딩되는 차트 스크린샷 파일을 다운로드하면 확장자가 없어서, OS가 이미지 파일로 인식하지 못한다.

**원인:** Slack API를 호출할 때 파일명에 확장자를 포함하지 않고 있었다. Slack은 파일명을 그대로 사용하기 때문에, 확장자가 없으면 다운로드한 파일에도 확장자가 붙지 않는다.

**대응:** Slack API 호출 인자에서 파일명을 `.png` 확장자와 함께 전달하도록 수정했다.

---

## 2. 대시보드 자동 새로고침 초기화

**제보 내용:** 대시보드에서 auto-refresh interval을 설정해도 재접속하면 초기화된다.

**원인:** 대시보드 우측 상단의 auto-refresh 메뉴는 현재 브라우저 세션에만 적용되는 설정이다. Superset의 설계상 이 값은 서버에 저장되지 않는다.

**대응:** 대시보드 자체에 리프레시 인터벌을 설정하는 방법을 안내했다. 대시보드 편집 모드에서 설정한 리프레시 인터벌은 대시보드 메타데이터에 저장되므로, 누가 접속하든 동일하게 적용된다.

이 케이스는 버그가 아니라 UX 혼동이었다. 같은 목적의 설정이 세션 레벨과 대시보드 레벨 두 곳에 있으면 사용자 입장에서 혼란스럽다.

---

## 3. SqlLab 데이터셋 덮어쓰기 버그

**제보 내용:** SqlLab에서 "overwrite existing dataset" 옵션을 선택하면 기존 데이터셋 목록이 제대로 표시되지 않는다.

**원인:** Superset 업스트림의 알려진 버그였다.

**대응:** Apache Superset 커뮤니티에서 해당 이슈의 픽스 커밋을 확인하고 체리피킹해서 적용했다.

---

## 4. 윈도우 환경 CSV 한글 깨짐

**제보 내용:** Superset에서 다운로드한 CSV를 윈도우의 Excel에서 열면 한글이 깨진다.

**원인:** Superset이 CSV를 표준 UTF-8로 인코딩해서 내보내고 있었다. 문제는 Microsoft Excel이다. Excel은 CSV를 열 때 BOM(Byte Order Mark)이 없으면 시스템 기본 인코딩(한국어 윈도우에서는 EUC-KR)으로 해석한다.

UTF-8로 인코딩된 파일이라도 BOM이 없으면 Excel은 UTF-8인지 알 수 없다. macOS나 Linux에서는 문제가 되지 않지만, 윈도우 Excel 사용자가 많은 환경에서는 반드시 고려해야 하는 부분이다.

**대응:** CSV 인코딩을 `utf-8`에서 `utf-8-sig`로 변경했다. `utf-8-sig`는 파일 시작 부분에 BOM(`EF BB BF`)을 삽입하는 Python의 UTF-8 변형 인코딩이다. BOM이 있으면 Excel이 파일을 UTF-8로 올바르게 인식한다.

---

## 5. 쿼리 결과 행 제한 10만 → 100만

**제보 내용:** Superset에서 쿼리 결과가 10만 행으로 제한된다. 100만 행까지 출력되면 좋겠다.

**대응:** Superset의 `ROW_LIMIT` 관련 설정값을 변경해서 최대 100만 행까지 응답하도록 조정했다.

단순한 설정 변경이지만, 행 제한을 무조건 올리는 것이 항상 좋은 것은 아니다. 행 수가 늘어나면 Superset 웹서버의 메모리 사용량과 응답 시간이 비례해서 증가한다. 100만 행이면 데이터 크기에 따라 수 GB의 메모리를 점유할 수 있다. 운영 환경의 리소스 상황을 보면서 결정해야 하는 설정이다.

---

## 6. CSV 다운로드 500 에러 — 가장 까다로웠던 이슈

**제보 내용:** Superset에서 50만 행 이상의 쿼리 결과를 CSV로 다운로드하면 약 1분 뒤에 500 에러가 발생한다. 20만 행까지는 문제없다.

이 이슈는 조건이 구체적이어서 원인 추적이 가능했다. "20만 행은 되고 50만 행은 안 된다"는 건 어딘가에 시간 기반 제한이 걸려 있다는 뜻이다.

### 에러 로그 추적

Superset 서버 로그에서 해당 시점의 에러를 확인했다.

```
psycopg2.OperationalError: SSL connection has been closed unexpectedly
```

Superset이 Redshift에서 데이터를 가져오는 과정에서 SSL 커넥션이 끊긴 것처럼 보이지만, 자세히 보면 이 에러는 Redshift 커넥션이 아니다. Superset의 메타데이터 데이터베이스(Aurora PostgreSQL) 커넥션에서 발생한 에러다.

### 원인 분석

CSV 다운로드 흐름을 추적해보면 이렇다:

1. 사용자가 CSV 다운로드 요청
2. Superset이 메타데이터 DB(Aurora PostgreSQL)에서 사용자의 다운로드 권한을 확인하는 트랜잭션을 연다
3. 권한 확인 후, Redshift에서 실제 데이터를 가져오기 시작한다
4. 50만 행 이상이면 Redshift에서 데이터를 가져오는 데 1분 이상 소요된다
5. 이 동안 메타데이터 DB 쪽 트랜잭션은 열린 채 idle 상태로 대기한다
6. Aurora RDS의 `idle_in_transaction_session_timeout` 설정이 60초(60,000ms)로 되어 있어서, idle 트랜잭션이 강제 종료된다
7. 트랜잭션이 끊기면서 Superset 서버가 500 에러를 반환한다

해당 시점의 Aurora RDS 에러 로그를 확인하니 이를 뒷받침하는 로그가 있었다.

```
master@superset:[17810]:FATAL: terminating connection due to idle-in-transaction timeout
```

### 대응

RDS 파라미터 그룹에서 `idle_in_transaction_session_timeout`을 60,000ms(1분)에서 600,000ms(10분)로 변경 요청했다. Superset 웹서버의 다른 타임아웃 설정(ALB, gunicorn 등)이 이미 10분으로 설정되어 있으므로, 이 값도 맞추는 것이 합리적이다.

더 근본적인 해결 방법도 있다. 데이터 다운로드 중에 권한 체크용 트랜잭션이 열려 있을 필요가 없도록 Superset 코드 자체를 수정하거나, 메타데이터 DB 앞에 PgBouncer 같은 커넥션 풀러를 두는 방식이다. 하지만 RDS 파라미터 변경이 가장 빠르고 안전한 대응이어서 이 방식을 선택했다.

설정 변경 후 50만 행 이상의 CSV 다운로드도 정상 동작함을 확인했다.

---

## 기타: Helm 차트 db-init Job arm 노드 이슈

Superset 베이스 Helm 차트에 포함된 db-init Job이 amd64 이미지를 사용하는데, 노드 셀렉터가 설정되어 있지 않아서 arm 노드에 스케줄링되면 실패하는 문제도 있었다. Helm 차트 버전을 v0.7.7에서 v0.8.10으로 올려서 해결했다.

---

## 배운 것

**"N만 행은 되고 M만 행은 안 된다"는 시간 기반 제한의 신호다.** 행 수 자체가 아니라 처리 시간에 걸리는 타임아웃을 의심해야 한다. 이 관점이 없으면 Redshift 커넥션만 계속 들여다보게 된다.

**에러가 발생하는 커넥션과 원인이 되는 커넥션이 다를 수 있다.** SSL 커넥션 에러가 Redshift가 아닌 메타데이터 DB에서 발생했다. 로그에서 커넥션 대상을 정확히 확인하는 것이 중요하다.

**타임아웃 설정은 시스템 전체에서 일관되게 맞춰야 한다.** ALB, gunicorn은 10분인데 DB만 1분이면, 가장 짧은 쪽에서 병목이 생긴다. 체인의 가장 약한 고리 원칙이다.

**CSV + 한글 + 윈도우 = BOM이 필수다.** `utf-8-sig`는 Python에서 BOM을 삽입하는 가장 간단한 방법이다. 윈도우 Excel 사용자가 있는 환경이라면 기본으로 적용하는 것이 좋다.

**사용자 제보는 가장 가치 있는 QA다.** 6건의 이슈 중 3건은 내부 테스트로는 발견하기 어려웠을 것이다. 윈도우 환경의 CSV 한글 깨짐이나 50만 행 이상의 다운로드 실패는 실제 사용 패턴에서만 나타난다.

**참고 자료:**
- [Apache Superset - SqlLab overwrite dataset fix](https://github.com/apache/superset/pull/25679)
- [Apache Superset - Helm chart db-init fix](https://github.com/apache/superset/pull/23416)
- [PostgreSQL: idle_in_transaction_session_timeout](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-IDLE-IN-TRANSACTION-SESSION-TIMEOUT)
- [Python: utf-8-sig encoding](https://docs.python.org/3/library/codecs.html#module-encodings.utf_8_sig)
