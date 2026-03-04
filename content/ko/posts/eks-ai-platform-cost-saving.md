---
title: "EKS AI 서빙 노드 비용 절감: 인스턴스 다양화, Consolidation, 스케줄 스케일링"
date: 2026-03-05
draft: false
categories: [Data Engineering]
tags: [eks, karpenter, cost-optimization, autoscaling, keda, kubernetes, consolidation]
showTableOfContents: true
summary: "AI 플랫폼팀의 서빙 API가 c6i.2xlarge와 m6i.2xlarge 두 종류의 온디맨드 인스턴스에 500개 파드를 항시 고정 배포하고 있었다. 인스턴스 타입 다양화와 Karpenter consolidation을 1차로 적용하고, KEDA cron 트리거 기반 스케줄 스케일링을 2차로 적용했다. 핵심 발견: consolidation 단독으로는 파드 수가 불변이면 효과가 제한적이고, 스케일인과 결합해야 노드 수가 줄어든다."
---

AI 플랫폼팀의 서빙 API 노드 비용을 절감해야 했다. 가장 큰 규모의 워크로드는 dispatch-travel-time-prediction API로, 500개 파드가 항시 배포되어 있었다. HPA가 CPU 메트릭 기준으로 구성되어 있지만, min replica가 500이라 실제 스케일인은 일어나지 않고 있었다.

Grafana에서 확인하니 일주일 동안 파드 수가 500으로 고정된 상태였다. 피크 시간 기준으로 산정된 리소스가 새벽 시간대에도 그대로 유지되고 있었다.

비용 절감을 두 단계로 나눠서 진행하기로 했다. 1차는 인스턴스 타입 다양화와 consolidation 활성화, 2차는 스케줄 기반 스케일링이다.

---

## 1차: 인스턴스 타입 다양화

기존 노드풀은 `c6i.2xlarge`와 `m6i.2xlarge` 두 종류만 허용하고 있었다. 서빙 API의 파드마다 요구하는 CPU와 메모리 비율이 다른데, 인스턴스 타입이 두 가지뿐이면 빈 패킹 효율이 떨어진다. 파드의 리소스 요청과 인스턴스 사양 사이의 남는 공간이 낭비된다.

Karpenter의 NodePool 설정에서 인스턴스 선택 조건을 변경했다.

```yaml
# 변경 전: 특정 인스턴스 타입 지정
requirements:
- key: node.kubernetes.io/instance-type
  operator: In
  values:
  - m6i.2xlarge
  - c6i.2xlarge

# 변경 후: 카테고리/세대/사이즈 기반 조건
requirements:
- key: karpenter.k8s.aws/instance-category
  operator: In
  values: ["r", "m", "c"]
- key: karpenter.k8s.aws/instance-generation
  operator: Gt
  values: ["5"]  # 6세대 이상
- key: karpenter.k8s.aws/instance-size
  operator: In
  values:
  - 2xlarge
  - 4xlarge
```

기존에 `node.kubernetes.io/instance-type`으로 특정 인스턴스를 지정하던 것을, `instance-category`, `instance-generation`, `instance-size`로 분리했다. r, m, c 카테고리의 6세대 이상, 2xlarge와 4xlarge 사이즈를 허용한다. Karpenter가 파드의 리소스 요청에 맞는 최적의 인스턴스를 자동으로 선택할 수 있게 된다.

---

## 1차: Consolidation 활성화

인스턴스 타입을 다양화한 것만으로는 기존 노드가 자동으로 교체되지 않는다. Karpenter의 consolidation을 활성화해야 노드를 더 효율적인 인스턴스로 재배치한다.

기존 서빙 API들에 영향을 주지 않기 위해, consolidation이 활성화된 별도 노드풀을 신규 생성하고 dispatch-travel-time-prediction에 먼저 적용하는 방식을 택했다.

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: ds-eks-prod-mlops-serving-with-consolidation
spec:
  weight: 50
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 480h0m0s  # 20일
    budgets:
    - nodes: "2"
    - nodes: "1"              # 일과 시간대에는 동시 중단 1개로 제한
      schedule: "0 1 * * *"   # UTC
      duration: 14h           # KST 10:00 ~ 24:00
```

새 노드풀에 전용 taint를 추가하고, dispatch-travel-time-prediction 디플로이먼트에 해당 toleration을 추가해서 점진적으로 이관했다.

### Disruption Budget 설정

처음에는 일과 시간대에 consolidation으로 인한 노드 중단을 완전히 차단했다 (`nodes: "0"`). 하지만 이렇게 하면 일과 시간 중 재배포 시 비어있는 노드가 내려가지 않는 문제가 있었다.

해당 서비스에는 PDB(PodDisruptionBudget)가 이미 적용되어 있고, 각 파드에 `sleep 30` preStop 훅이 구성되어 있다. 온디맨드만 사용하고 리소스 request가 명시된 구조에서는 일과 시간 중 consolidation이 거의 트리거되지 않는다. budget을 `nodes: "1"`로 완화했다.

### Consolidation 단독의 한계

여기서 중요한 발견이 있었다. 파드 수가 500개로 불변이면, consolidation을 적용해도 비용 절감 효과가 거의 없다.

Consolidation은 노드에 남는 공간이 있을 때 파드를 더 적은 노드로 재배치하는 것이다. 하지만 파드 수가 변하지 않으면 필요한 총 리소스 양도 변하지 않는다. 인스턴스 타입 다양화로 인한 약간의 빈 패킹 개선은 있지만, 근본적인 비용 절감은 파드 수를 줄여야 가능하다.

---

## 2차: KEDA Cron 트리거 기반 스케줄 스케일링

500개 파드가 항시 필요한 것이 아니었다. 주말 저녁 피크 시간(3000+ RPS) 기준으로 산정된 수치가 새벽 시간대에도 유지되고 있었다. CPU 사용률 20%대에서 이미 타임아웃이 발생하는 특성 때문에 HPA만으로는 적절한 스케일인이 어려웠다.

KEDA의 cron 트리거를 사용해서 시간대별로 파드 수를 조절하기로 했다.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: dispatch-travel-time-prediction-scheduled
spec:
  scaleTargetRef:
    kind: Deployment
    name: dispatch-travel-time-prediction-scheduled
  minReplicaCount: 200
  maxReplicaCount: 400
  triggers:
  # 평일 오전 (08:30~10:30)
  - type: cron
    metadata:
      timezone: Asia/Seoul
      desiredReplicas: "250"
      start: 30 8 * * 0-6
      end: 0 1 * * 0-6
  # 평일 점심 (10:30~16:00)
  - type: cron
    metadata:
      timezone: Asia/Seoul
      desiredReplicas: "300"
      start: 30 10 * * 1-5
      end: 30 14 * * 1-5
  # 평일 저녁 피크 (16:00~23:30)
  - type: cron
    metadata:
      timezone: Asia/Seoul
      desiredReplicas: "350"
      start: 0 16 * * 1-5
      end: 30 23 * * 1-5
  # 주말 저녁 피크 (16:00~23:30) — 주문량이 더 많음
  - type: cron
    metadata:
      timezone: Asia/Seoul
      desiredReplicas: "400"
      start: 0 16 * * 0,6
      end: 30 23 * * 0,6
```

새벽 시간대(01:00~08:30)에는 min replica인 200개만 유지되고, 트래픽 패턴에 따라 시간대별로 250 → 300 → 350(평일) / 400(주말) 으로 증가한다. 기존 500개 고정 대비 새벽 시간대에 300개 파드가 줄어든다.

### 기존 HPA와의 공존

기존 CPU 기반 HPA는 제거하지 않고 그대로 두었다. KEDA ScaledObject가 관리하는 스케줄 기반 디플로이먼트와, 기존 HPA가 관리하는 디플로이먼트를 분리했다. 두 디플로이먼트의 파드를 구분하기 위해 `autoscaling/scheduled: "true"` / `"false"` 라벨을 추가했다.

기존 HPA 연동 디플로이먼트의 min replica는 500에서 100으로 줄이고, max replica는 1000에서 500으로 조정했다. 스케줄 스케일링이 대부분의 용량을 담당하고, HPA는 예상치 못한 부하에 대한 안전장치로 남겨두었다.

### 라벨 셀렉터 핫픽스

배포 직후 이슈가 하나 있었다. 스케줄 기반 디플로이먼트의 selector에 `autoscaling/scheduled` 라벨이 누락되어, 기존 디플로이먼트의 파드까지 선택하는 문제가 있었다. 핫픽스로 라벨 셀렉터를 추가해서 해결했다.

---

## 결과: Consolidation + 스케일인의 시너지

스케줄 스케일링 적용 후, 의도한 대로 consolidation과의 시너지가 나타났다.

파드 스케일인 시점에 consolidation이 트리거되면서, 남은 파드들이 더 적은 노드로 재배치되었다. 파드 수의 증감에 따라 노드 수도 함께 증감하는 것을 확인했다.

Consolidation에 의한 파드 재배치 과정에서 배포된 파드 수에 다소간의 증감이 발생했지만, 노드 레벨의 disruption budget과 파드 레벨의 PDB, 그리고 `sleep 30` preStop 훅 덕분에 서비스 장애는 발생하지 않았다.

---

## 스팟 인스턴스는 적용하지 않은 이유

비용 절감 논의 중에 스팟 인스턴스 혼용도 검토했다. Karpenter에서 topologySpreadConstraint로 스팟/온디맨드 비율 조정이 가능하고, 스팟 disruption 이벤트 시 다른 노드에 예비 파드가 준비되므로 장애로 이어지지 않을 것으로 보였다.

하지만 회사 지침상 API 서비스에는 스팟을 적용하지 않는 것이 원칙이었다. 3000+ RPS를 처리하는 배달 예상 시간 예측 API가 스팟 회수 시 레플리카 수 보장이 안 되는 리스크를 감수하기에는 서비스 영향도가 너무 컸다. 스팟 적용 없이 인스턴스 다양화 + consolidation + 스케줄 스케일링의 조합으로 진행했다.

---

## 배운 것

**Consolidation 단독으로는 효과가 제한적이다.** 파드 수가 변하지 않으면 필요한 총 리소스 양도 변하지 않는다. Consolidation이 빈 패킹을 최적화하더라도, 근본적인 절감은 파드 수를 줄여야 가능하다. 인스턴스 타입 다양화 → consolidation → 스케줄 스케일링 순서로 적용하되, 핵심 효과는 스케일인에서 나온다.

**변경은 하나씩, 순서대로.** 인스턴스 타입 다양화, consolidation, 스케줄 스케일링을 한꺼번에 적용하면 문제 발생 시 원인 파악이 어렵다. 각 단계를 분리해서 적용하고 효과를 확인한 후 다음 단계로 넘어가는 것이 서비스 영향을 최소화하는 방법이다.

**비용 절감은 기술팀만의 일이 아니다.** min replica 500이라는 설정은 피크 시간대 레이턴시 요구사항에서 나온 것이다. CPU 사용률 20%에서 타임아웃이 발생하는 API 특성, 스케줄 기반 스케일링 일정, 스팟 적용 여부 모두 서비스 팀과의 논의가 필요했다. 인프라 설정 변경만으로는 해결되지 않는다.

**안전장치는 겹겹이 쌓아라.** Consolidation에 의한 노드 재배치는 파드의 일시적 중단을 수반한다. PDB, preStop 훅의 `sleep 30`, disruption budget의 동시 중단 노드 수 제한 — 이 세 가지가 함께 동작해서 서비스 무중단을 보장했다. 어느 하나만 빠져도 재배치 과정에서 트래픽 유실이 발생할 수 있다.
