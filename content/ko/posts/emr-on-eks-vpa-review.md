---
title: "EMR on EKS VPA 검토기: AWS 공식 기능이 동작하지 않을 때"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [emr, eks, vpa, spark, kubernetes, autoscaling]
showTableOfContents: true
summary: "EMR on EKS 환경에서 Spark executor 리소스를 자동 최적화하려고 AWS 제공 VPA를 검토했다. 약 한 달간 PoC를 진행했지만 오퍼레이터 자체가 정상 동작하지 않았다. AWS 서포트 케이스를 여러 차례 열었고 매니페스트 번들까지 받아서 커스터마이징했지만 결국 포기했다."
---

EMR on EKS로 Spark 잡을 돌리면 executor 팟이 Kubernetes 위에 뜬다. 문제는 executor의 CPU와 메모리를 얼마나 줘야 하는지 잡마다 다르고 시간대마다 다르다는 거다. 너무 많이 주면 리소스가 낭비되고 너무 적게 주면 OOM으로 죽는다.

VPA(Vertical Pod Autoscaler)를 쓰면 과거 실행 이력을 기반으로 적절한 리소스를 추천하고 자동으로 조정해준다. AWS가 EMR on EKS 전용 VPA 연동 기능을 제공한다고 해서 검토에 들어갔다. 약 한 달간 집중적으로 PoC를 진행했지만 결국 포기했다.

이 글은 그 과정에서 배운 것을 정리한 기록이다.

---

## VPA 모드 이해

EMR on EKS VPA는 세 가지 모드를 지원한다.

| 모드 | 동작 | 영향 |
|------|-----|------|
| Off | 추천값만 계산. 실제 조정 없음 (dry-run) | 없음 |
| Initial | 잡 시작 시 추천값으로 리소스 설정 | executor 재시작 없음 |
| Auto | 실행 중인 executor를 evict하고 추천값으로 재시작 | 지연 발생 가능 |

Initial 모드가 가장 안전해 보였다. executor 재시작 오버헤드가 없으니까 운영 잡에 바로 적용할 수 있을 거라 판단했다.

계획은 이랬다. 먼저 Off 모드로 전체 Spark 잡에 켜서 추천값만 수집하고 추천값이 합리적인지 확인한 뒤에 Initial 모드로 전환한다.

---

## 잡 시그니처 키 설계

VPA가 추천값을 잡별로 관리하려면 각 잡을 식별할 키가 필요하다. EMR VPA는 잡 시그니처 키라는 개념을 쓴다. 같은 시그니처 키를 가진 잡 실행 이력이 쌓이면 그걸 기반으로 리소스 추천값을 계산한다.

단순히 잡 이름만으로는 부족하다. 같은 잡이라도 피크 시간대와 새벽 시간대의 리소스 사용 패턴이 다르다. 요일에 따라서도 다르다.

시그니처 키를 이렇게 설계했다.

```python
f"{app_name}-day-{target_date.day_of_week}-hour-{target_date.hour}-"
f"{'holiday-' if is_holiday else ''}{resource_spec}"
```

구성 요소:
- `app_name`: Spark 잡 이름
- `day_of_week`: 요일 (금요일 트래픽은 주중과 다르다)
- `hour`: 시간대 (피크 타임 구분)
- `holiday`: 공휴일 여부
- `resource_spec`: DRA 활성화 여부 및 executor 개수 스펙 (VPA는 개별 팟의 버티컬 스케일링이므로 executor 수가 중요한 변수다)

### K8s 라벨 제한 문제

시그니처 키를 Kubernetes 라벨로 저장해야 하는데 라벨에는 제한이 있다.

- 최대 63자
- 영숫자, `-`, `_`, `.`만 허용
- 영숫자로 시작하고 끝나야 함

시그니처 키 원문이 이 제한을 넘길 수 있어서 deterministic hash로 변환해서 사용했다. 원문과 해시의 매핑 정보는 Airflow XCom에 기록해서 나중에 추적할 수 있게 했다.

---

## YuniKorn과의 호환성 문제

첫 번째 벽은 스케줄러 호환성이었다.

EMR on EKS에서 YuniKorn 스케줄러를 쓰고 있었는데 YuniKorn은 VPA를 지원하지 않는다. Gang scheduling 환경에서 VPA를 쓰면 문제가 생긴다. executor가 VPA에 의해 재시작될 때 처음 제출된 리소스 요청량과 VPA가 변경한 값이 달라지면서 잡이 실패하거나 좀비 상태에 빠질 수 있다.

Auto 모드는 이 이유만으로 쓸 수 없었다. Initial 모드는 시작 시점에만 개입하니까 괜찮을 거라 판단하고 진행했다.

---

## PoC 진행

테스트 환경에 EMR VPA 오퍼레이터를 배포하고 세 가지 테스트 배치를 구성했다.

- 쿠폰마트 적재 배치 (기존 EMR-on-EC2 환경에서 운영 중이던 잡)
- 딜리버리마트 적재 배치
- DW 블로그 적재 배치 (1시간 단위 배치, 피크 기준 20분 소요 — 비교 측정에 적합)

각 배치를 Initial 모드와 Auto 모드로 돌려서 VPA 없는 기존 잡과 비교할 계획이었다.

VPA 리소스가 생성되는 것까지는 확인했다. 그런데 VPA 상태가 `NoPodsMatched`로 나왔다. executor 팟의 리소스 사용 메트릭을 수집하지 못하고 있었다.

---

## 오퍼레이터의 연쇄적 문제

트러블슈팅을 시작하자 문제가 줄줄이 나왔다.

### 1. kube-apiserver 부하

EMR VPA 오퍼레이터의 recommender가 타깃 네임스페이스를 제한하는 옵션이 없었다. 모든 네임스페이스의 모든 리소스를 주기적으로 조회했다. 심지어 kube-apiserver의 모든 엔드포인트를 찌르고 있었다.

admission controller의 타깃 네임스페이스는 OperatorGroup 리소스의 스펙을 수동으로 수정해서 제한할 수 있었지만 recommender는 방법이 없었다.

### 2. VPA 리소스 인식 실패

EMR VPA 컨트롤러(`emr-dynamic-sizing-controller-manager`)는 VPA 리소스를 정상적으로 생성했다. 그런데 같은 번들에 포함된 admission controller와 recommender가 생성된 VPA를 인식하지 못했다. 시그니처 키로 기존 VPA를 찾지 못하고 recommender도 추천값을 기록하지 못했다.

### 3. 블랙박스 오퍼레이터

AWS가 번들링한 오퍼레이터 패키지를 EKS에 설치하는 구조라 코드를 직접 확인하거나 수정할 수 없었다. 로그를 보고 추측하는 수밖에 없었다.

---

## AWS 서포트 케이스

두 차례에 걸쳐 서포트 케이스를 열었다.

첫 번째는 네임스페이스 제한과 VPA 인식 실패 문제를 상세하게 정리해서 문의했다. AWS 측에서 매니페스트 번들을 통째로 공유해줬다. 직접 커스터마이징해서 재설치하라는 가이드였다.

번들을 받아서 수정하고 재설치한 뒤 다시 테스트했다. 여전히 안 됐다. 같은 이슈가 재현됐고 관련 컴포넌트 로그를 정리해서 다시 문의했다.

---

## 결론: 기능 자체가 유지보수되고 있지 않았다

한 달간의 검토 끝에 내린 결론이다.

EMR VPA 연동 기능은 **AWS 측에서 더 이상 개발/유지보수하고 있지 않은 것으로 보인다.** 근거는 두 가지다.

1. 공식 문서 외에 이 기능을 실제로 사용하고 있다는 사례를 찾을 수 없었다
2. Kubernetes 1.27에서 도입된 In-place Pod Resize 기능이 VPA의 상위 호환이다. AWS가 EMR on EKS에 이 기능을 직접 연동할 계획이라는 얘기가 서밋에서 나왔다

VPA의 가장 큰 약점은 리소스를 조정하려면 팟을 재시작해야 한다는 점이다. In-place Pod Resize는 팟을 죽이지 않고 리소스를 변경할 수 있다. Kubernetes autoscaler 쪽에도 VPA에 in-place resize를 적용하는 PR이 올라온 상태다.

---

## 배운 것

**AWS 공식 기능이라고 다 쓸 만한 건 아니다.** 공식 문서에 있다고 실제로 동작한다는 보장이 없다. 검색해서 사용 사례가 나오지 않는 기능은 의심해봐야 한다.

**블랙박스 오퍼레이터는 트러블슈팅이 고통이다.** 코드를 볼 수 없으니 로그만 보고 추측해야 한다. 서포트 케이스를 열어도 응답 주기가 길고 결국 매니페스트를 직접 고치라는 결론이 나온다.

**K8s 생태계의 방향을 읽어라.** In-place Pod Resize가 나온 시점에서 VPA의 evict-and-recreate 방식은 레거시가 될 운명이었다. 기술 선택 전에 상위 프로젝트의 로드맵을 확인하는 게 중요하다.

**Off 모드(dry-run)부터 시작하자는 판단은 맞았다.** 만약 운영 잡에 바로 적용했다면 오퍼레이터 문제로 잡 자체가 영향받을 수 있었다. Dry-run으로 시작한 덕분에 운영에는 영향을 주지 않고 문제를 파악할 수 있었다.

**참고 자료:**
- [EMR on EKS Vertical Autoscaling - AWS Documentation](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/jobruns-vas-configure.html)
- [YuniKorn VPA Support Issue](https://issues.apache.org/jira/browse/YUNIKORN-1765)
- [VPA In-place Resize PR](https://github.com/kubernetes/autoscaler/pull/6652)
