# [파드 QoS 클래스](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

## QoS 클래스란?
Kubernetes는 파드(Pod)에 자원 제한 및 요청에 따라 QoS(Quality of Service) 클래스를 할당하여 노드에 리소스 부족 상황이 발생할 경우 어떤 파드를 축출(evict)할지 결정하는 기준으로 삼는다.

할당 가능한 QoS 클래스:
- **Guaranteed**
- **Burstable**
- **BestEffort**

---

## 1. Guaranteed 클래스
### 조건:
- 파드 내 모든 컨테이너가 `memory limit`, `memory request`, `CPU limit`, `CPU request`를 모두 명시해야 함.
- 각 컨테이너에서 `limit == request`여야 함.

### 특징:
- 가장 우선순위가 높으며, 축출될 가능성이 가장 낮음.
- 명시된 제한을 초과하면만 종료됨.
- static CPU 관리 정책을 통해 전용 CPU 사용 가능.

---

## 2. Burstable 클래스
### 조건:
- Guaranteed 조건을 만족하지 않음.
- 최소한 하나의 컨테이너가 `memory` 또는 `CPU` 요청 혹은 제한을 가짐.

### 특징:
- `BestEffort` 파드 다음으로 축출 대상이 됨.
- 사용 가능한 리소스가 있을 경우 유연하게 자원 사용 가능.
- 제한이 없으면 노드의 용량이 기본값으로 간주됨.

---

## 3. BestEffort 클래스
### 조건:
- 파드 내 어떤 컨테이너도 `memory`나 `CPU` 요청 및 제한을 가지지 않음.

### 특징:
- 가장 낮은 우선순위를 가지며, 가장 먼저 축출됨.
- 다른 QoS 클래스에 할당되지 않은 자원 사용 가능.

---

## Memory QoS (cgroup v2)
- 기능 상태: Kubernetes v1.22 [alpha]
- `memory.min` = memory request → 커널이 reclaim하지 않음.
- `memory.high` = memory limit → 리미트에 근접 시 쓰로틀링 발생.
- QoS 클래스와 독립적으로 동작하지만 QoS를 기준으로 설정 적용.

---

## QoS 클래스와 무관한 동작
- 컨테이너가 자원 제한을 초과하면 종료 및 재시작됨.
- 자원 요청을 초과하고 노드에 압박이 생기면 파드는 축출 대상이 됨.
- 파드의 전체 요청/제한 = 파드 내 컨테이너들의 요청/제한의 합.
- `kube-scheduler`는 QoS 클래스를 기준으로 선점(preemption)하지 않음.
