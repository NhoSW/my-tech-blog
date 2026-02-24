---
title: "NVIDIA MIG 설정 가이드. A100 GPU 하나를 여러 개로 쪼개 쓰기"
date: 2026-02-23
draft: false
categories: [Data Engineering]
tags: [nvidia, gpu, mig, kubernetes, infrastructure]
showTableOfContents: true
summary: "A100 GPU 4장으로 최대 28개의 독립 GPU 인스턴스를 만들 수 있다. MIG 개념부터 mig-parted 설치, 슬라이스 구성, Kubernetes 연동까지 실전 경험을 정리했다."
---

GPU가 비싸다. A100 한 장이 수천만 원인데 대부분의 워크로드는 80GB 메모리를 다 쓰지 않는다. Jupyter 노트북에서 간단한 실험 돌리는 데 GPU 한 장을 통째로 할당하면 나머지 70GB는 그냥 놀게 된다.

NVIDIA MIG(Multi-Instance GPU)는 이 문제를 풀어준다. 물리 GPU 하나를 최대 7개 독립 인스턴스로 쪼개서 여러 워크로드가 동시에 돌아간다. 이 글은 A100 GPU 4장 환경에서 MIG를 설정하고 Kubernetes까지 연동한 과정을 기록한 것이다.

---

## MIG란 무엇인가

MIG는 하나의 물리 GPU를 여러 독립 GPU 인스턴스로 분할하는 기술이다. 각 인스턴스가 자체 메모리와 컴퓨트 유닛을 갖고있어서 서로 간섭하지 않는다. 한 인스턴스에서 OOM이 나도 다른 인스턴스에는 영향이 없다.

A100 80GB 기준 분할 옵션은 이렇다.

| 프로필 | 메모리 | SM 수 | GPU당 최대 인스턴스 |
|--------|--------|-------|-------------------|
| 1g.10gb | ~10GB | 14 | 7 |
| 2g.20gb | ~20GB | 28 | 3 |
| 3g.40gb | ~40GB | 42 | 2 |
| 7g.80gb | ~80GB | 98 | 1 |

7분할(1g.10gb)을 하면 GPU 4장에서 인스턴스 28개가 나온다. 단순 계산으로 GPU 활용률이 4배 이상 올라간다.

### 지원 GPU 확인

**MIG는 A100, H100 등 특정 GPU에서만 쓸 수 있다.** A40이나 V100은 MIG를 지원하지 않는다. 우리도 처음에 A40 서버에서 시도했다가 안 되는 걸 확인하고 A100 서버로 전환했다.

`nvidia-smi` 출력에서 `MIG M.` 컬럼을 확인하면 된다.

```
# MIG 미지원 GPU (A40)
| MIG M. |
|  N/A   |   ← 지원하지 않음

# MIG 지원 GPU (A100)
| MIG M.   |
| Disabled |   ← 지원하지만 비활성화 상태
| Enabled  |   ← 활성화됨
```

지원 GPU 목록은 [NVIDIA 공식 문서](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html#supported-gpus)에서 확인할 수 있다.

---

## MIG 설정하기

### 사전 준비. nvidia-mig-manager 비활성화

MIG를 켜기 전에 `nvidia-mig-manager.service`를 반드시 비활성화해야 한다. 이 데몬이 부팅할 때마다 MIG를 disabled로 되돌린다.

```bash
sudo systemctl disable nvidia-mig-manager.service
```

### mig-parted 설치

`nvidia-mig-parted`는 NVIDIA에서 제공하는 MIG 설정 도구다. 선언적으로 config 파일을 작성하면 원하는 분할 구성을 한번에 적용해준다.

```bash
# deb 패키지 설치 (Ubuntu/Debian)
curl -LO https://github.com/NVIDIA/mig-parted/releases/download/v0.5.2/nvidia-mig-manager_0.5.2-1_amd64.deb
sudo dpkg -i nvidia-mig-manager_0.5.2-1_amd64.deb
```

### MIG 모드 활성화

```bash
sudo nvidia-smi -mig 1
```

이 명령을 실행하면 MIG 모드가 pending 상태가 된다. 적용하려면 **리부트가 필요하다.**

```bash
sudo reboot
```

> **VM 환경 주의사항.** GPU passthrough로 사용하는 가상머신에서는 `nvidia-smi --gpu-reset`을 지원하지 않는다. 리부트 없이 MIG를 활성화할 수 없으므로 베어메탈 서버를 쓰는 게 좋다. 우리도 이 문제 때문에 VM에서 베어메탈로 옮겼다.

### config.yaml 작성

mig-parted는 YAML 파일로 분할 구성을 선언한다. 한 파일에 여러 프리셋을 정의해놓고 필요할 때 골라 쓸 수 있다.

```yaml
version: v1
mig-configs:
  # MIG 비활성화
  all-disabled:
    - devices: all
      mig-enabled: false

  # 전체 GPU를 7분할 (1g.10gb × 7)
  all-1g.10gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "1g.10gb": 7

  # 전체 GPU를 3분할 (2g.20gb × 3)
  all-2g.20gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3

  # 전체 GPU를 2분할 (3g.40gb × 2)
  all-3g.40gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "3g.40gb": 2

  # GPU를 통째로 사용 (7g.80gb × 1)
  all-7g.80gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "7g.80gb": 1

  # GPU별로 다른 분할 구성
  custom-config:
    - devices: [0]
      mig-enabled: true
      mig-devices:
        "3g.40gb": 2
    - devices: [1]
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3
        "1g.10gb": 1
    - devices: [2]
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3
        "1g.10gb": 1
    - devices: [3]
      mig-enabled: true
      mig-devices:
        "1g.10gb": 7
```

마지막 `custom-config`가 포인트다. GPU마다 다른 프로필을 섞어서 구성할 수 있다. 왜 이렇게 해야하는지는 뒤에서 설명한다.

### 설정 적용

```bash
# 균일한 7분할 적용
sudo nvidia-mig-parted apply -f ./config.yaml -c all-1g.10gb

# 또는 커스텀 구성 적용
sudo nvidia-mig-parted apply -f ./config.yaml -c custom-config
```

`-d` 플래그를 추가하면 디버그 로그를 볼 수 있다. 적용이 끝나면 `nvidia-smi -L`로 인스턴스 목록을 확인한다.

```
GPU 0: NVIDIA A100-SXM4-80GB (UUID: GPU-xxxx)
  MIG 3g.40gb     Device  0: (UUID: MIG-xxxx)
  MIG 3g.40gb     Device  1: (UUID: MIG-xxxx)
GPU 1: NVIDIA A100-SXM4-80GB (UUID: GPU-xxxx)
  MIG 2g.20gb     Device  0: (UUID: MIG-xxxx)
  MIG 2g.20gb     Device  1: (UUID: MIG-xxxx)
  MIG 2g.20gb     Device  2: (UUID: MIG-xxxx)
  MIG 1g.10gb     Device  3: (UUID: MIG-xxxx)
...
```

### 재부팅 시 자동 적용

MIG 설정은 재부팅하면 날아간다. systemd 서비스로 등록해서 부팅할 때 자동으로 적용되게 만든다.

```ini
# /etc/systemd/system/nvidia-mig-config.service
[Unit]
Description=Apply NVIDIA MIG Configuration
After=nvidia-persistenced.service

[Service]
Type=oneshot
ExecStart=/usr/bin/nvidia-mig-parted apply -f /home/user/config.yaml -c custom-config
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable nvidia-mig-config.service
```

---

## 실전에서 배운 것. 균일 분할로는 부족하다

처음에는 단순하게 생각했다. GPU 4장을 전부 1g.10gb × 7로 쪼개면 인스턴스 28개가 나온다. 가장 많은 사용자가 동시에 쓸 수 있으니 좋지 않을까?

### 큰 모델은 MIG에서 안 돌아간다

현실은 달랐다. 추천 모델 학습하는 팀에서 문제가 터졌다. 모델이 GPU 메모리를 30GB 넘게 쓰는데 MIG 인스턴스 하나는 10GB밖에 없으니 CUDA OOM으로 로드 자체가 안 됐다.

LLM도 마찬가지였다. Llama 2 7B 모델은 4bit 양자화를 해도 MIG 디바이스에서 로드가 안 됐다. 13B은 말할것도 없다.

결론은 명확하다. **워크로드에 따라 분할 전략을 다르게 가져가야 한다.**

### GPU별 커스텀 분할

팀 내 워크로드를 분석해보니 이런 구성이 됐다.

| GPU | 프로필 | 용도 |
|-----|--------|------|
| GPU 0 | 3g.40gb × 2 | 중형 모델 학습 (메모리 40GB급) |
| GPU 1 | 2g.20gb × 3 + 1g.10gb × 1 | 중소형 실험 |
| GPU 2 | 2g.20gb × 3 + 1g.10gb × 1 | 중소형 실험 |
| GPU 3 | 1g.10gb × 7 | Jupyter 노트북, 가벼운 배치 작업 |

대형 모델 학습이 필요하면 MIG를 꺼둔 GPU를 별도로 남겨두는 방법도 있다. 우리는 LLM 실험 수요가 늘면서 일부 GPU의 MIG를 끄기도 했다.

> **주의.** MIG 슬라이스를 섞어 구성할 때 물리적 제약이 있다. GPU 내부 메모리 슬라이스는 왼쪽에서 오른쪽으로 할당되며 수직으로 공유할 수 없다. NVIDIA 공식 문서에서 지원하는 조합을 꼭 확인해야 한다.

---

## Kubernetes 연동

MIG 인스턴스를 Kubernetes에서 쓰려면 NVIDIA device plugin 설정을 바꿔야 한다.

### MIG Strategy 이해하기

device plugin에는 세 가지 MIG 전략이 있다.

| 전략 | 리소스 이름 | 설명 |
|------|------------|------|
| none | `nvidia.com/gpu` | MIG를 인식하지 않음. 물리 GPU 단위 할당 |
| single | `nvidia.com/gpu` | MIG 인스턴스를 자동 할당. 기존 워크로드 호환 |
| mixed | `nvidia.com/mig-{profile}` | MIG 프로필별로 별도 리소스 노출 |

`single` 전략은 기존 워크로드가 `nvidia.com/gpu: 1`로 요청하던 걸 수정 없이 MIG 인스턴스로 연결해준다. 편하긴 한데 어떤 크기의 인스턴스를 받을지 제어할 수가 없다.

`mixed` 전략은 `nvidia.com/mig-3g.40gb: 1`처럼 프로필을 명시해서 요청한다. GPU별로 다른 분할을 쓴다면 mixed가 맞다.

### device plugin 설정 변경

```yaml
# values.yaml
migStrategy: mixed
```

```bash
helm upgrade -i nvdp nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --create-namespace \
  --version 0.12.3 \
  -f ./values.yaml
```

적용 후 nvidia-device-plugin Pod를 재시작해야 노드 리소스에 반영된다. `kubectl describe node`로 MIG 리소스가 보이는지 확인한다.

```
Allocatable:
  nvidia.com/mig-1g.10gb:  9
  nvidia.com/mig-2g.20gb:  6
  nvidia.com/mig-3g.40gb:  2
```

### JupyterHub에서 MIG 사용

JupyterHub 프로필에서 리소스 요청을 MIG 타입으로 바꾼다.

```yaml
# 변경 전 (물리 GPU 할당)
extra_resource_limits:
  nvidia.com/gpu: "1"

# 변경 후 (MIG 인스턴스 할당)
extra_resource_limits:
  nvidia.com/mig-1g.10gb: "1"
```

mixed 전략에서 `nvidia.com/gpu: 1`을 요청하면 물리 GPU 전체가 할당되니까 주의해야 한다. MIG 인스턴스를 쓰려면 반드시 프로필 이름을 명시해야 한다.

### Airflow KubernetesPodOperator에서 MIG 사용

Airflow에서 GPU 작업 실행할 때도 리소스 요청을 MIG 타입으로 바꾼다.

```python
resources = {
    "cpu": "4",
    "memory": "16Gi",
    "nvidia.com/mig-2g.20gb": "1",
}

KubernetesPodOperator(
    ...
    container_resources=k8s.V1ResourceRequirements(
        requests=resources,
        limits=resources,
    ),
    ...
)
```

---

## 운영 팁

### MIG 설정 변경 타이밍

MIG 구성을 바꾸려면 그 GPU의 모든 프로세스를 종료해야 한다. 운영 환경에서는 배치 작업이 없는 시간대를 골라서 작업한다. 우리는 배치 스케줄이 새벽 4시 이후에 몰려 있어서 밤 9시 이후에 MIG 재설정을 진행했다.

### 서버 전체 GPU에 MIG를 걸어야 한다

한 서버에서 일부 GPU만 MIG를 켜고 나머지는 끄는 구성은 동작하지 않는다. **서버 안의 모든 GPU가 MIG enabled이거나 전부 disabled여야 한다.** 대형 모델 학습용으로 MIG 없는 GPU가 필요하면 별도 서버를 쓰거나 MIG config에서 `7g.80gb: 1`로 설정해서 GPU 전체를 하나의 인스턴스로 노출하면 된다.

### GPU Capacity 변화

MIG 적용 전후로 Kubernetes 클러스터의 GPU capacity가 크게 달라진다. 우리 환경에서는 GPU 4장(capacity 4)이 MIG 적용 후 인스턴스 기준으로 20개 이상이 됐다. 모니터링이랑 알럿 기준을 같이 조정해야 한다.

---

## 마치며

MIG는 GPU 활용률을 높이는 괜찮은 방법이다. 만능은 아니다. 실전에서 배운걸 정리하면 이렇다.

1. **지원 GPU를 먼저 확인하라.** A100, H100 같은 특정 모델만 MIG를 지원한다. A40이나 V100은 안 된다.
2. **VM 환경은 피하라.** GPU passthrough에서는 GPU reset이 안 돼서 MIG 활성화가 까다롭다. 베어메탈이 편하다.
3. **균일 분할로 시작하되 커스텀 분할로 넘어가라.** 워크로드를 분석해서 GPU별로 프로필을 섞는 게 실용적이다.
4. **대형 모델은 MIG로 안 된다.** LLM이나 대규모 추천 모델은 GPU 전체 메모리가 필요하다. MIG 없는 GPU를 따로 확보해야 한다.

GPU는 비싸지만 잘 쪼개면 기존 인프라에서 더 많은걸 해낼 수 있다.

**참고 자료.**
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html)
- [nvidia-mig-parted (GitHub)](https://github.com/NVIDIA/mig-parted)
- [NVIDIA Device Plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin)
- [Getting the Most Out of the A100 GPU with MIG](https://developer.nvidia.com/blog/getting-the-most-out-of-the-a100-gpu-with-multi-instance-gpu/)
