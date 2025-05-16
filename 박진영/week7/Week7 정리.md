# Week7

# 관련 공식문서

[https://kubernetes.io/ko/docs/concepts/cluster-administration/logging/](https://kubernetes.io/ko/docs/concepts/cluster-administration/logging/)

[https://kubernetes.io/ko/docs/tasks/debug/debug-cluster/resource-usage-monitoring/](https://kubernetes.io/ko/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)

# 로깅 아키텍처

- 문제를 디버깅하고 클러스터 활동을 모니터링하는 데 유용
- 컨테이너화된 애플리케이션에 가장 쉽고 가장 널리 사용되는 로깅 방법은 표준 출력과 표준 에러 스트림에 작성하는 것
  - but 이 방법은 완전하지 않음
  → 클러스터 레벨 로깅이 요구됨
  - 클러스터에서 로그는 노드, 파드 또는 컨테이너와는 독립적으로 별도의 스토리지와 라이프사이클을 가짐
  - 로그를 저장, 분석, 쿼리하기 위해서는 별도의 백엔드가 필요
- 쿠버네티스에 통합될 수 있는 기존의 로깅 솔루션으로 로그 처리/관리

## 파드와 컨테이너 로그

- 실행중인 파드의 컨테이너에서 출력하는 로그를 감시
- `kubectl logs --previous` 를 사용해서 컨테이너의 이전 인스턴스에 대한 로그를 검색
  - 파드에 여러 컨테이너가 있는 경우, 명령에 `-c` 플래그와 컨테이너 이름을 추가
    `kubectl logs counter -c count`

## 노드가 컨테이너 로그를 처리하는 방법

![](https://kubernetes.io/images/docs/user-guide/logging/logging-node-level.png)

- 컨테이너화된 애플리케이션의 `stdout(표준 출력)` 및 `stderr(표준 에러)` 스트림에 의해 생성된 모든 출력은 컨테이너 런타임이 처리하고 리디렉션 시킴
- kubelet과의 호환성은 *CRI 로깅 포맷* 으로 표준화됨
- 기본적으로 컨테이너가 재시작하는 경우, kubelet은 종료된 컨테이너 하나를 로그와 함께 유지
- 파드가 노드에서 축출되면, 해당하는 모든 컨테이너와 로그가 함께 축출됨

## 클러스터-레벨 로깅 아키텍처

### 노드 로깅 에이전트

![](https://kubernetes.io/images/docs/user-guide/logging/logging-with-node-agent.png)

- 모든 노드에서 실행
- 노드별 하나의 에이전트만 생성하며, 노드에서 실행되는 애플리케이션에 대한 변경은 필요로 하지 않음
- 사이드카 컨테이너 함께 사용 가능
  - my-pod 안에 streaming container를 두는 방식으로.

### 애플리케이션 파드에 로깅을 위한 전용 사이드카 컨테이너를 포함

![](https://kubernetes.io/images/docs/user-guide/logging/logging-with-sidecar-agent.png)

- 상당한 리소스 소비로 이어질 수 있음
- kubelet으로 제어되지 않아서 kubectl logs로 접근 불가

### 애플리케이션 내에서 로그를 백엔드로 직접 푸시

![](https://kubernetes.io/images/docs/user-guide/logging/logging-from-application.png)

- 쿠버네티스 범위를 벗어남

# 모니터링

## 리소스 모니터링 도구

- 쿠버네티스는 각 레벨에서 애플리케이션의 리소스 사용량에 대한 상세 정보를 제공
- 이 정보는 애플리케이션의 성능을 평가하고 병목 현상을 제거하여 전체 성능을 향상케 함
- 애플리케이션 모니터링은 단일 모니터링 솔루션에 의존하지 않음
- 리소스 메트릭 또는 완전한 메트릭 파이프라인으로 모니터링 통계를 수집 가능

## 리소스 메트릭 파이프라인

- Horizontal Pod Autoscaler 컨트롤러와 같은 클러스터 구성요소나 kubectl top 유틸리티에 관련되어 있는 메트릭들로 제한된 집합을 제공
- 경량의 단기 인메모리 저장소인 metrics-server에 의해서 수집되며 `metrics.k8s.io` API를 통해 노출
- metrics-server : 클러스터 상의 모든 노드를 발견하고 각 노드의 kubelet에 CPU와 메모리 사용량을 질의
  - Kubelet은 쿠버네티스 마스터와 노드 간의 다리 역할을 하면서 머신에서 구동되는 파드와 컨테이너를 관리

## 완전한 메트릭 파이프라인

- 클러스터의 현재 상태를 기반으로 자동으로 스케일링하거나 클러스터를 조정 가능
- 모니터링 파이프라인은 kubelet에서 메트릭을 가져와서 쿠버네티스에 `custom.metrics.k8s.io`와 `external.metrics.k8s.io` API를 구현한 어댑터를 통해 노출
