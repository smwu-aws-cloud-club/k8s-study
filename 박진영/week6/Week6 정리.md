# Week6

# 관련 공식문서

[https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/namespaces/](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/namespaces/)

[https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale/](https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale/)

# 네임스페이스

- 단일 클러스터 내에서의 리소스 그룹 격리 메커니즘을 제공
- 네임스페이스 기반 오브젝트 *(예: 디플로이먼트, 서비스 등)* 에만 적용 가능
- 클러스터 범위의 오브젝트 *(예: 스토리지클래스, 노드, 퍼시스턴트볼륨 등)* 에는 적용 불가능
- 이름의 범위를 제공
- 클러스터 자원을 (리소스 쿼터를 통해) 여러 사용자 사이에서 나누는 방법
- 프로덕션 클러스터의 경우, default 네임스페이스를 사용하지 않는 걸 추천. 다른 네임스페이스를 만들어 사용하길 권고.

## 여러 개의 네임스페이스를 사용하는 경우

- 리소스의 이름은 네임스페이스 내에서 유일해야하지만, 네임스페이스를 통틀어서 유일할 필요는 없음
- 네임스페이스는 서로 중첩될 수 없으며, 각 쿠버네티스 리소스는 하나의 네임스페이스에만 있을 수 있음
- 동일한 소프트웨어의 다른 버전과 같이 약간 다른 리소스를 분리하기 위해 여러 네임스페이스를 사용할 필요는 없음

## 초기 네임스페이스

- **default**
- **kube-node-lease**
    - 각 노드와 연관된 리스 오브젝트를 가짐.
    - 노드 리스는 kubelet이 하트비트를 보내서 컨트롤 플레인이 노드의 장애를 탐지할 수 있게 함.
- **kube-public**
    - 모든 클라이언트(인증되지 않은 클라이언트 포함)가 읽기 권한으로 접근 가능
- **kube-system**
    - 생성한 오브젝트를 위한

## 네임스페이스 다루기

- `kube-` 접두사로 시작하는 네임스페이스는 쿠버네티스 시스템용으로 예약되어 있으므로, 사용자는 이러한 네임스페이스를 생성하지 않음
- 네임스페이스 조회 `kubectl get namespace`
- 요청에 네임스페이스 설정 시, `--namespace` 플래그를 사용
    
    ```bash
    kubectl run nginx --image=nginx --namespace=<insert-namespace-name-here>
    kubectl get pods --namespace=<insert-namespace-name-here>
    ```
    
- 선호하는 네임스페이스 설정
    - 모든 kubectl 명령에서 쓰는 네임스페이스를, 컨텍스트에 영구 저장 가능함
        
        ```
        kubectl config set-context --current --namespace=<insert-namespace-name-here>
        # 확인하기
        kubectl config view --minify | grep namespace:
        ```
        

## 네임스페이스와 DNS

- 서비스를 생성하면 해당 DNS 엔트리가 생성됨
    - 엔트리의 형식 : `<서비스-이름>.<네임스페이스-이름>.svc.cluster.local`
    - 이 경우, 컨테이너가 <서비스-이름>만 사용하는 경우, 네임스페이스 내에 국한된 서비스로 연결됨
- 네임스페이스를 넘어서 접근하기 위해서는, 전체 주소 도메인 이름(FQDN)을 사용

## 모든 오브젝트가 네임스페이스에 속하지는 않음

- 대부분의 쿠버네티스 리소스 : 네임스페이스 소속
- 네임스페이스 리소스 자체는 네임스페이스 소속이 X
- 노드, 퍼시스턴스 볼륨 같은 저수준 리소스 : 네임스페이스 소속이 X

```
# 네임스페이스에 속하는 리소스
kubectl api-resources --namespaced=true

# 네임스페이스에 속하지 않는 리소스
kubectl api-resources --namespaced=false
```

## 자동 레이블링

- 트롤 플레인은 `NamespaceDefaultLabelName` 기능 게이트가 활성화된 경우 모든 네임스페이스에 변경할 수 없는(immutable) 레이블 `kubernetes.io / metadata.name` 을 설정함
- 레이블 값은 네임스페이스 이름

# Horizontal Pod Autoscaling

- 워크로드 리소스(e.g. Deployment, StatefulSets)를 자동 업데이트
- 워크로드 크기를 수요에 맞게 자동으로 스케일링 하는 게 목표
- 부하 증가에 대해 파드를 더 배치하는 것
    - ↔ 수직 스케일링
     : 워크로드를 위해 이미 실행 중인 파드에 더 많은 자원을 할당
- 크기 조절이 불가능한 오브젝트(e.g. 데몬셋)에는 적용 불가
- API 자원 및 컨트롤러 형태로 구현됨

## HorizontalPodAutoscaler는 어떻게 작동하는가?

![image.png](image.png)

- 디플로이먼트 및 디플로이먼트의 레플리카셋의 크기를 조정
- 실행 주기는 `kube-controller-manager`의 `--horizontal-pod-autoscaler-sync-period` 파라미터에 의해 설정
- 컨트롤러 매니저는 `scaleTargetRef`에 의해 정의된 타겟 리소스를 찾고 나서, 타겟 리소스의 `.spec.selector` 레이블을 보고 파드를 선택하며, 리소스 메트릭 API(파드 단위 리소스 메트릭 용) 또는 커스텀 메트릭 API(그 외 모든 메트릭 용)로부터 메트릭을 수집
- HorizontalPodAutoscaler를 사용하는 일반적인 방법

## 알고리즘 세부 정보

- 가장 기본적인 관점에서, HorizontalPodAutoscaler 컨트롤러
    
    → 원하는(desired) 메트릭 값과 현재(current) 메트릭 값 사이의 비율로 작동
    `원하는 레플리카 수 = ceil[현재 레플리카 수 * ( 현재 메트릭 값 / 원하는 메트릭 값 )]`
    
- `targetAverageValue` 또는 `targetAverageUtilization`가 지정되면, `currentMetricValue`는 HorizontalPodAutoscaler의 스케일 목표 안에 있는 모든 파드에서 주어진 메트릭의 평균을 취하여 계산
- `--horizontal-pod-autoscaler-initial-readiness-delay` 플래그: 기본값 30초
- `--horizontal-pod-autoscaler-cpu-initialization-period` 플래그: 기본값 5분

## 리소스 메트릭 지원

- 모든 HPA 대상은 스케일링 대상에서 파드의 리소스 사용량을 기준으로 스케일링 가능
- 파드의 명세를 정의할 때는 `cpu` 및 `memory` 와 같은 리소스 요청을 지정해야 함
    
    ```yaml
    type: Resource
    resource:
    name: cpu
    target:
    type: Utilization
    averageUtilization: 60
    ```
    
- 모든 컨테이너의 리소스 사용량이 합산되므로 총 파드 사용량이 개별 컨테이너 리소스 사용량을 정확하게 나타내지 못할 수 있음
- 단일 컨테이너가 높은 사용량으로 실행될 수 있고 전체 파드 사용량이 여전히 허용 가능한 한도 내에 있기 때문에 HPA가 스케일링되지 않는 상황이 발생할 수 있음

## **구성가능한 스케일링 동작**

- `behavior` 필드(API 레퍼런스 참고)를 사용하여 스케일업 동작과 스케일다운 동작을 별도로 구성 가능
- 각 방향에 대한 동작은 `behavior` 필드 아래의 `scaleUp` / `scaleDown`를 설정하여 지정 가능
- *안정화 윈도우* 를 명시하여 스케일링 목적물의 레플리카 수 흔들림을 방지
- 스케일링 정책을 이용하여 스케일링 시 레플리카 수 변화 속도를 조절 가능
- 스케일링 정책
    - 스펙의 `behavior` 섹션에 하나 이상의 스케일링 폴리시를 지정
    - 여러 개 지정된 경우 가장 많은 양의 변경을 허용하는 정책이 기본적으로 선택
- 안정화 윈도우
    - 스케일링에 사용되는 메트릭이 계속 변동할 때 레플리카 수의 흔들림을 제한하기 위해 사용
    - 오토스케일링 알고리즘은 이전의 목표 상태를 추론하고 워크로드 수의 원치 않는 변화를 방지하기 위해 이 안정화 윈도우를 활용