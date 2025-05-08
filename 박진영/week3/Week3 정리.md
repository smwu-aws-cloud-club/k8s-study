# Week3

# 워크로드 리소스 - 레플리카셋, 디플로이먼트

## 레플리카셋

- 목적 : 레플리카 파드 집합의 실행을 항상 안정적으로 유지
- 보통 명시된 동일 파드 개수에 대한 가용성을 보증하는데 사용

### 작동 방식

- 파드의 [metadata.ownerReferences](https://kubernetes.io/ko/docs/concepts/architecture/garbage-collection/#%EC%86%8C%EC%9C%A0%EC%9E%90-owner-%EC%99%80-%EC%A2%85%EC%86%8D-dependent) 필드를 통해 파드에 연결됨
    - 이는 현재 오브젝트가 소유한 리소스를 명시
    - 해당 파드를 소유한 레플리카셋을 식별하기 위한 소유자 정보를 가짐

### 사용 시기

- 지정된 수의 파드 레플리카가 항상 실행되도록 보장
- 디플로이먼트는 레플리카셋을 관리하고 다른 유용한 기능과 함께 파드에 대한 선언적 업데이트를 제공하는 상위 개념
    - 별도의 사용자 정의(custom) 업데이트 오케스트레이션이 필요한 경우 또는 업데이트가 전혀 필요 없는 경우가 아니라면, 디플로이먼트 권장

### 레플리카셋 매니페스트 작성

- 모든 쿠버네티스 API 오브젝트와 마찬가지로 `apiVersion`, `kind`, `metadata` 필드가 필요
    - `kind` 필드의 값은 항상 레플리카셋
- 오브젝트의 이름은 유효한 DNS 서브도메인 이름이어야 함
- `.spec` 섹션 필요
- 파드 템플릿 : `.spec.template` . 재시작 정책 필드인 `.spec.template.spec.restartPolicy`는 기본값인 `Always`만 허용
- 파드셀렉터
    - `.spec.selector` 필드는 레이블 셀렉터
    - `.spec.template.metadata.labels`는 `spec.selector`과 일치해야 하며 그렇지 않으면 API에 의해 거부됨
- 레플리카
    - `.spec.replicas`를 설정해서 동시에 동작하는 파드의 수를 지정
    - 기본값 1

### 레플리카셋 작업

- 해당 파드 삭제 시 `kubectl delete`
- 레플리카셋만 삭제 시 `kubectl delete`에 `--cascade=orphan` 옵션을 사용하여 연관 파드에 영향을 주지 X

### 레플리카셋에서 파드 격리

- 레이블을 변경하면 레플리카셋에서 파드를 제거할 수 있음
- 디버깅과 데이터 복구 등을 위해 서비스에서 파드를 제거하는 데 사용할 수 있음
- 제거된 파드는 자동 교체됨

### 레플리카셋의 스케일링

- `.spec.replicas` 필드를 업데이트해서 scale-up/down

### 레플리카셋을 Horizontal Pod Autoscaler 대상으로 설정

- yaml 내부에서 `kind:HorizontalPodAutoscaler` 로 지정하거나
- `kubectl autoscale` 실행

### 레플리카셋의 대안

1. 디플로이먼트(권장)
2. 기본 파드
3. 잡
4. 데몬셋
5. 레플리케이션 컨트롤러

## 디플로이먼트(Deployment)

: 파드와 레플리카셋에 대한 선언적 업데이트 제공

- 디플로이먼트가 소유하는 레플리카셋은 관리하지 말아야 함.

### 디플로이먼트 생성

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment #라는 디플로이먼트가 생성
  labels:
    app: nginx
spec:
  replicas: 3 #디플로이먼트는 3개의 레플리카 파드를 생성하는 레플리카셋을 생성
  selector: #생레플리카셋이 관리할 파드를 찾아내는 방법을 정의
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2 
        #파드가 docker hub에 nginx 이미지를 실행하는 컨테이너 1개를 실행
        ports:
        - containerPort: 80

```

### 디플로이먼트 업데이트

```bash
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
#또는
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1

#롤아웃 상태 확인 시
kubectl rollout status deployment/nginx-deployment

#업데이트 상태 확인 가능
kubectl get rs
#새 파드만 표시
kubectl get pods
#디플로이먼트 세부 정보
kubectl describe deployments
```

### 디플로이먼트 롤백

```bash
#롤아웃 기록 확인
kubectl rollout history deployment/nginx-deployment
#각 수정 버전의 세부 정보
kubectl rollout history deployment/nginx-deployment --revision=2

#현재 롤아웃의 실행 취소 및 이전 수정 버전으로 롤백
kubectl rollout undo deployment/nginx-deployment
#특정 수정 버전으로 롤백 시
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

### 디플로이먼트 스케일링

```bash
kubectl scale deployment/nginx-deployment --replicas=10
```

### 정책 초기화

- `.spec.revisionHistoryLimit` 필드를 설정해서 디플로이먼트에서 유지해야 하는 이전 레플리카셋의 수를 명시(기본 10)

# 클러스터 관리-리소스 관리

애플리케이션 배포 및 서비스로 노출 후,

확장&업데이트를 할 때 필요한 리소스 관리!

## 리소스 구성하기

- 많은 applications는 deployment 및 service 같은 여러 resource를 필요로 함
- 동일 파일에 그룹화하여 리소스 관리를 단순화할 수 있음
    - yaml에서는 `---`로 구분
- 파일에 표시된 순서대로 생성되는 리소스
    - 스케줄러가 디플로이먼트 같이 컨트롤러에서 생성한 서비스와 관련된 파드를 분산시킬 수 있음 → 서비스를 먼저 지정하는 게 가장 좋음
- `kubectl` 은 접미사가 `.yaml`, `.yml` 또는 `.json` 인 파일을 읽음
- 동일한 마이크로서비스 또는 애플리케이션 티어(tier)와 관련된 리소스를 동일한 파일에 배치하고, 애플리케이션과 연관된 모든 파일을 동일한 디렉터리에 그룹화하는 것이 좋음.
    - **애플리케이션의 티어가 DNS를 사용하여 서로 바인딩되면, 스택의 모든 컴포넌트를 함께 배포**할 수 있음
- URL을 구성 소스로 지정할 수 있음.(github에 checkin된 파일에서 직접 배포하는 데 편리함)

## kubectl에서의 대량 작업

- 리소스가 많을 경우, `-l` 또는 `--selector` 를 사용하여 지정된 셀렉터(레이블 쿼리)를 지정하여 레이블별로 리소스를 필터링하는 것이 더 쉬움
    
    `kubectl delete deployment,services -l app=nginx`
    
- `kubectl` 은 입력을 받아들이는 것과 동일한 구문으로 리소스 이름을 출력하므로, `$()` 또는 `xargs` 를 사용하여 작업을 연결할 수 있음
    
    ```
    kubectl get$(kubectl create -f docs/concepts/cluster-administration/nginx/ -o name | grep service)
    kubectl create -f docs/concepts/cluster-administration/nginx/ -o name | grep service | xargs -i kubectl get {}
    ```
    

## 효과적인 레이블 사용

- 레이블은 레이블로 지정된 차원에 따라 리소스를 분할하고 사용할 수 있게 함

## 카나리(canary) 디플로이먼트

- 여러 레이블이 필요한 또 다른 시나리오는 동일한 컴포넌트의 다른 릴리스 또는 구성의 디플로이먼트를 구별하는 것
- 새 릴리스가 완전히 롤아웃되기 전에 실제 운영 트래픽을 수신할 수 있도록 새로운 애플리케이션 릴리스(파드 템플리트의 이미지 태그를 통해 지정됨)의 *카나리* 를 이전 릴리스와 나란히 배포하는 것이 일반적
- 안정 및 카나리 릴리스의 레플리카 수를 조정하여 실제 운영 트래픽을 수신할 각 릴리스의 비율을 결정
- 확신이 들면, 안정 릴리스의 track을 새로운 애플리케이션 릴리스로 업데이트하고 카나리를 제거할 수 있음

## 레이블 업데이트

- `kubectl label` 로 수행
    
    `kubectl label pods -l app=nginx tier=fe`
    

## 어노테이션 업데이트

- 어노테이션을 리소스에 첨부
    - 어노테이션 : 도구, 라이브러리 등과 같은 API 클라이언트가 검색할 수 있는 임의의 비-식별 메타데이터
- `kubectl annotate`

## 어플리케이션 스케일링

- 애플리케이션의 로드가 증가하거나 축소되면, `kubectl` 을 사용하여 애플리케이션을 스케일링
    
    `kubectl scale deployment/my-nginx --replicas=1`
    
    `kubectl autoscale deployment/my-nginx --min=1 --max=3`
    
    `horizontalpodautoscaler.autoscaling/my-nginx autoscaled`
    

## 리소스 인플레이스(in-place) 업데이트

- `kubectl apply`
    
    푸시하려는 구성의 버전을 이전 버전과 비교하고 지정하지 않은 속성에 대한 자동 변경 사항을 덮어쓰지 않은 채 수정한 변경 사항을 적용
    
- `kubectl edit`
    
    리소스 업데이트
    
- `kebctl patch`
    
    API 오브젝트를 인플레이스 업데이트 가능하게 함
    
    JSON 패치, JSON 병합 패치, 전략적 병합 패치를 지원함
    

## 파괴적(disruptive) 업데이트

- 한 번 초기화하면 업데이트할 수 없는 리소스 필드를 업데이트해야 하거나, 디플로이먼트에서 생성된 손상된 파드를 고치는 등의 재귀적 변경을 즉시 원할 수도 있음
- 이런 필드 변경 시 `replace --force` 를 사용하여 리소스를 삭제하고 다시 만듦
- 원 구성 파일 수정 가능

`kubectl replace -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml --force`

## 서비스 중단 없이 애플리케이션 업데이트

- 일반적으로 새 이미지 또는 이미지 태그를 지정하여, 배포된 애플리케이션을 업데이트해야 함
→ `kubectl`로 deployment를 사용해 애플리케이션 생성 및 업데이트하기