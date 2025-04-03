## 레플리카셋

- 레플리카셋의 목적은 레플리카 파드 집합의 실행을 항상 안정적으로 유지하는 것
- 보통 명시된 동일 파드 개수에 대한 가용성을 보증하는데 사용
- 레플리카셋은 지정된 수의 파드 레플리카가 항상 실행되도록 보장
  디플로이먼트는 레플리카셋을 관리하고 다른 유용한 기능과 함께 파드에 대한 선언적 업데이트를 제공하는 상위 개념이다. 따라서 별도의 사용자 정의(custom) 업데이트 오케스트레이션이 필요한 경우 또는 업데이트가 전혀 필요 없는 경우가 아니라면, 레플리카셋을 직접 사용하기보다는 디플로이먼트를 사용하는 것을 권장한다.

### manifest 작성

레플리카셋은 모든 쿠버네티스 API 오브젝트와 마찬가지로 apiVersion, kind, metadata 필드가 필요하다.
-> 레플리카셋에 대한 **kind 필드의 값은 항상 레플리카셋**
-> 오브젝트 이름은 유효한 DNS 서브도메인 이름
-> 레플리카셋도 .spec 섹션 필요

### 삭제

```
kubectl delete
```

kubectl delete에 --cascade=orphan 옵션을 사용하여 연관 파드에 영향을 주지 않고 레플리카셋을 삭제할 수 있다.

원본이 삭제되면 새 레플리카셋을 생성해서 대체할 수 있다. 기존 .spec.selector와 신규 .spec.selector가 같으면 **새 레플리카셋은 기존 파드를 선택**한다. 하지만 신규 레플리카셋은 기존 파드를 신규 레플리카셋의 새롭고 다른 파드 템플릿에 일치시키는 작업을 수행하지는 않는다. **컨트롤 방식으로 파드를 새로운 사양으로 업데이트 하기 위해서는 디플로이먼트를 이용하면 된다. 이는 레플리카셋이 롤링 업데이트를 직접적으로 지원하지 않기 때문이다.**

### 레플리카셋 스케일링

스케일 업 또는 다운하는 방법 -> **.spec.replicas 필드 업데이트**
-> 레플리카셋 컨트롤러는 일치하는 레이블 셀렉터가 있는 파드가 의도한 수만큼 가용하고 운영 가능하도록 보장한다.

### HPA(Horizontal Pod Autoscaler)

레플리카셋은 Horizontal Pod Autoscalers (HPA)의 대상이 될 수 있다. 즉, 레플리카셋은 HPA에 의해 오토스케일될 수 있다.

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

이 매니페스트를 hpa-rs.yaml로 저장한 다음 쿠버네티스 클러스터에 적용하면 CPU 사용량에 따라 파드가 복제되는 오토스케일 레플리카셋 HPA가 생성된다.

### 디플로이먼트

- 레플리카셋을 소유하거나 업데이트를 하고, 파드의 선언적인 업데이트와 서버측 롤링 업데이트를 할 수 있는 오브젝트
- 단독으로 사용할 수 있지만, 주로 디플로이먼트로 파드의 생성과 삭제, 업데이트를 오케스트레이션하는 메커니즘으로 사용

디플로이먼트를 이용해서 배포할 때 생성되는 레플리카셋을 관리하는 것에 대해 걱정하지 않아도 된다. 디플로이먼트는 레플리카셋을 소유하거나 관리한다. 따라서 레플리카셋을 원한다면 디플로이먼트를 사용하는 것을 권장한다.

---

## 디플로이먼트

디플로이먼트(Deployment)는 파드와 레플리카셋(ReplicaSet)에 대한 선언적 업데이트를 제공한다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
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
        ports:
        - containerPort: 80
```

디플로이먼트 생성 예시다. 3개의 nginx 파드를 불러오기 위한 레플리카셋을 생성한다.

- .metadata.name 필드에 따라 nginx-deployment 이름을 가진 디플로이먼트가 생성
- .spec.replicas 필드에 따라 디플로이먼트는 3개의 레플리카 파드를 생성하는 레플리카셋을 생성
- .spec.selector 필드는 생성된 레플리카셋이 관리할 파드를 찾아내는 방법

```
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

디플로이먼트 생성

```
kubectl get deployments
```

디플로이먼트 생성 확인

- NAME은 네임스페이스에 있는 디플로이먼트 이름의 목록
- READY는 사용자가 사용할 수 있는 애플리케이션의 레플리카의 수 표시 (ready/desired 패턴을 따른다.)
- UP-TO-DATE는 의도한 상태를 얻기 위해 업데이트된 레플리카의 수 표시
- AVAILABLE은 사용자가 사용할 수 있는 애플리케이션 레플리카의 수 표시
- AGE는 애플리케이션의 실행된 시간 표시

```
kubectl rollout status deployment/nginx-deployment
```

디플로이먼트의 롤아웃 상태 확인

```
kubectl get rs
```

디플로이먼트로 생성된 레플리카셋 확인(rs)

```
kubectl get pods --show-labels
```

각 파드에 자동으로 생성된 레이블 확인

### 디플로이먼트 업데이트

```
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
```

nginx:1.14.2 이미지 대신 nginx:1.16.1 이미지를 사용하도록 nginx 파드 업데이트

```
kubectl rollout status deployment/nginx-deployment
```

롤아웃 상태 확인

```
kubectl describe deployments
```

디플로이먼트 세부 정보 확인

### 디플로이먼트 롤백

디플로이먼트가 지속적인 충돌로 안정적이지 않은 경우에 롤백을 할 수 있다. 기본적으로 모든 디플로이먼트의 롤아웃 기록은 시스템에 남아있어 언제든지 원할 때 롤백이 가능하다.

```
kubectl rollout undo deployment/nginx-deployment
```

### 디플로이먼트 스케일링

```
kubectl scale deployment/nginx-deployment --replicas=10
```

클러스터에서 horizontal Pod autoscaling를 설정 한 경우 디플로이먼트에 대한 오토스케일러를 설정할 수 있다. 그리고 기존 파드의 CPU 사용률을 기준으로 실행할 최소 파드 및 최대 파드의 수를 선택할 수 있다.

```
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

### 디플로이먼트 상태

디플로이먼트는 라이프사이클 동안 다양한 상태로 전환된다. 새 레플리카셋을 롤아웃하는 동안 진행 중이 될 수 있고, 완료이거나 진행 실패일 수 있다.

#### 디플로이먼트 진행 중

롤아웃이 "진행 중" 상태가 되면, 디플로이먼트 컨트롤러는 디플로이먼트의 .status.conditions에 다음 속성을 포함하는 컨디션을 추가한다.

- type: Progressing
- status: "True"
- reason: NewReplicaSetCreated | reason: FoundNewReplicaSet | reason: ReplicaSetUpdated

kubectl rollout status 를 사용해서 디플로이먼트의 진행 상황을 모니터할 수 있다.

#### 디플로이먼트 완료

- 디플로이먼트과 관련된 모든 레플리카가 지정된 최신 버전으로 업데이트 되었을 때. 즉, 요청한 모든 업데이트가 완료되었을 때
- 디플로이먼트와 관련한 모든 레플리카를 사용할 수 있을 때
- 디플로이먼트에 대해 이전 복제본이 실행되고 있지 않을 때

위와 같은 특성을 가지게 되면 디플로이먼트가 완료로 표시된다.

#### 디플로이먼트 실패

디플로이먼트시 새 레플리카셋인 완료되지 않은 상태에서는 배포를 시도하면 고착될 수 있다. 이 문제는 다음 몇 가지 요인으로 인해 발생한다.

- 할당량 부족
- 준비성 프로브(readiness probe)의 실패
- 이미지 풀 에러
- 권한 부족
- 범위 제한
- 애플리케이션 런타임의 잘못된 구성

---

## 리소스 관리

많은 애플리케이션들은 디플로이먼트 및 서비스와 같은 여러 리소스를 필요로 한다. 여러 리소스의 관리는 동일한 파일에 그룹화하여 단순화할 수 있다(YAML에서 --- 로 구분)

```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
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
        ports:
        - containerPort: 80
```

예시다. 단일 리소스와 동일한 방식으로 여러 리소스 생성이 가능하다.

### 카나리(canary) 디플로이먼트

여러 레이블이 필요한 또 다른 시나리오는 동일한 컴포넌트의 다른 릴리스 또는 구성의 디플로이먼트를 구별하는 것이다. 새 릴리스가 완전히 롤아웃되기 전에 실제 운영 트래픽을 수신할 수 있도록 **새로운 애플리케이션 릴리스(파드 템플리트의 이미지 태그를 통해 지정됨)의 카나리를 이전 릴리스와 나란히 배포**하는 것이 일반적이다.

### 레이블 업데이트

```
kubectl label

kubectl label pods -l app=nginx tier=fe
```

새로운 리소스를 만들기 전에 기존 파드 및 기타 리소스의 레이블을 다시 지정해야 하는 경우가 있다. 위의 명령어로 실행할 수 있다.

### 어노테이션 업데이트

어노테이션을 리소스에 첨부하려고 할 수도 있다. 어노테이션은 도구, 라이브러리 등과 같은 API 클라이언트가 검색할 수 있는 임의의 비-식별 메타데이터이다.

### 애플리케이션 스케일링

애플리케이션의 로드가 증가하거나 축소되면, kubectl을 사용하여 애플리케이션을 스케일링한다.

```
kubectl scale deployment/my-nginx --replicas=1
```

위의 명령어를 예시로 보면, nginx 레플리카 수를 3에서 1로 줄일 수 있다.
