# 0402

생성자: chaeyeon park
생성 일시: 2025년 4월 2일 오전 8:01
주차: 5주차

# Todo

- 서비스 설계 (보안적인 부분 어떻게 설계할지 리서치)
- 중간 보고서 점검 및 주차별 후기 작성
- ~~회생제동 데이터의 구간변화를 주기 위해서 IPedal은 넣고 돌린 구간으로 다시 실험해보기~~
    - ~~→ aesop에서 돌려봤으나, Ipedal 을 PC에 넣는다고 해도 , 회생제동은 0-2의 값을 성공/실패구간에 상관없이 갖고있음~~

# kubernetes

## replica set

목적

지정한 수의 replica pod set의 실행이 항상 안정적으로 유지되어야한다 = 가용성 보장

### 정의

- selector : 획득 가능한 파드 식별 방법 명시
- replica 개수 : 유지해야하는 파드 개수
- 파드 템플릿 : 레플리카 수 유지를 위해 신규 파드에 대한 데이터 명시 (?)

### [metadata.ownerReferences](https://kubernetes.io/ko/docs/concepts/architecture/garbage-collection/#%EC%86%8C%EC%9C%A0%EC%9E%90-owner-%EC%99%80-%EC%A2%85%EC%86%8D-dependent) - 파드가 지닌 필드

- 레플리카 셋 연결 창구
- 오브젝트가 소유한 리소스 명시
- 소유자 정보 ( 해당 파드를 소유한 레플리카셋을 식별하는데 사용 )
- 파드에 OwnerReference가 없거나, 있는데 컨트롤러가 아니면서 레플리카셋 셀렉터와 일치한다면 레플리카 셋이 파드를 입양(adopt)한다.

### 레플리카 셋 사용

- 디플로이먼트가 레플리카셋을 관리하고 , 추가 기능(파드 업데이트 등)들을 제공하는 상위 개념
- 레플리카 셋 직접 사용 보다는 ~ **디플로이먼트를 사용하는 것을 권장**
- 레플리카셋 직접 사용은 사실상, 별도의 custom 업데이트 오케스트레이션을 적용할 것이거나, **업데이트가 전혀 필요 없는 경우**이다 . (한마디로 직접 사용 지양하라는 얘기)

### 템플릿을 사용하지 않는 파드의 획득

> 단독(bare) 파드를 만들 때, 실수로 레플리카셋의 selector와 같은 label을 붙이지 않게 주의해라
> 
> 
> 안 그러면 **그 레플리카셋이 그 파드를 ‘입양(adopt)’해서 관리하게 될 수 있다!**
> 

즉. 정리하자면 

✅**Pod이 소유자를 갖게 되는 방식은**

1️⃣**OwnerReference 필드를 직접 통해 명시적으로 설정되는 경우**와,

2️⃣**label-selector가 일치할 때 ReplicaSet이 입양(adopt)해서 소유자를 등록하는 경우**

ex) label을 지정해서 pod를 만들면 처음에는 bare pod였다가 label 과 일치하는 selector를 지닌 replicaset이 존재하면 그에게 입양되어 관리된다.

### 레플리카셋 manifest

구성 요소:   `apiVersion`, `kind` : replicaset, `metadata` 필드, [`.spec` 섹션](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)

- pod template
    - `.spec.template`
- pod selector
    - `.spec.selector`
- replica
    - `.spec.replicas` : 동시에 동작하는 파드 개수 지정 (기본값 : 1)

### 레플리카셋 작업

- 레플리카 셋 과 자식 pod 삭제
    - `kubectl delete`
        
        ```bash
        #rest api 예시
        kubectl proxy --port=8080
        curl -X DELETE  'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend'\
        > -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}'\
        > -H "Content-Type: application/json"
        ```
        
- 레플리카셋만 삭제하기
    - `kubectl delete —cascade=orphan`
        
        ```bash
        #rest api 예시
        kubectl proxy --port=8080
        curl -X DELETE  'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend'\
        > -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}'\
        > -H "Content-Type: application/json"
        ```
        
    
    → 원본 삭제 시 , 새 replicaset으로 대체할 수 있다. (`.spec.selector` 가 동일한 경우), 새 replicaset이 기존 파드를 맡지만, 새 replicaset의 pod template에 일치시키는 작업을 하지는 X, 
    
    만약에 일치시키는(업데이트) 작업을 원한다면 deployment를 사용하면 된다. replicaset은 롤링 업데이트를 지원하지 않음.
    

### 레플리카셋에서 파드 격리

- 레이블을 변경하는 방식으로 replicaset에서 파드 제거 가능 (이후 자동 교체됨)

레플리카셋의 스케일링

- `.spec.replicas` 필드 업데이트 하면 됨
- scale down 시 삭제 파드 우선순위
    - pending 상태인  pod 최우선
    - [`controller.kubernetes.io/pod-deletion-cost`](http://controller.kubernetes.io/pod-deletion-cost) 어노테이션이 설정되어있다는 가정 하에, 가장 낮은 값을 갖는 pod
    - 더 많은 replica가 있는 노드의 pod
    - 파드 생성 시간이 다를 경우, 더 최근에 생성된 pod 우선
    
    ⇒ 모든 기준이 동등하면, 랜덤 선택 ㅋ
    

### 레플리카 셋을 Horizontal Pod Autoscaler 대상 설정 (HPA)

```bash
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

### 레플리카셋의 대안

- **디플로이먼트(권장)**
- 기본 파드
- 잡
- 데몬셋
- 레플리케이션 컨트롤러(비권장)

## Deployment

Pod와 ReplicaSet에 대한 선업적 업데이트 제공

ex) 새로운 replicaset을 생성하는 디플로이먼트  정의 or 기존 디플로이먼트를 제거하고 , 모든 리소스를 새 디플로이먼트에 적용 

- 기본적으로 디플로이먼트가 소유하는 레플리카셋은 직접 관리(수정, 제어 )하지 말아야 함 ⚠️

### 유스케이스

- 레플리카셋을 롤아웃 할 디플로이먼트 생성
- 디플로이먼트의 PodTemplateSpec을 업데이트해서 파드의 새 상태를 선언한다
- 디플로이먼트의 현재 상태가 불안정하다면, 이전버전의 디플로이먼트로 롤백한다
- 디플로이먼트 스케일업
- 디플로이먼트 롤아웃 일시 중지로 PodTemplateSpec에 여러 수정 사항 적용 하고 재개하여 새 롤아웃을 시작
- 롤아웃이 막혀있는지를 나타내는 디플로이먼트 상태 이용
- 필요없는 레플리카셋 정리

*롤아웃 : 배포를 점진적으로 진행하는 과정

### 디플로이먼트 생성

```bash
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
        image: nginx:1.14.2 # 기본적으로 docker hub base
        ports:
        - containerPort: 80
```

- `.[metadata.name](http://metadata.name)` : nginx-deployment를 이름으로 가진 deployment 생성
- `.spec.replicas` : deployment는 3개의 replica pod을 생성하는 replicaset 생성
- `.spec.selector` : 생성된 replicaset이 관리할 파드를 찾아내는 방법 정의 pod template 상에 정의된 label이 `app:nginx` 라면 해당 파드를 관리,
- `template` :
    - `.metadata.labels` : app:nginx라는 레이블 부여
    - `.template.spec` : 도커 허브의 nginx 1.14.2 이미지를 실행하는 컨테이너 1개를 실행한다고 정의
    - `.spec.template.spec.containers[0].name` : 컨테이너 1개를 생성하고 , nginx 이름을 부여

### 명령어

```bash
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

```bash
kubectl get deployments
```

아직 생성 중일 경우, 다음과 같이 출력

![UP-TO-DATE : 의도한 상태를 얻기 위해 업데이트된 replica 수
AVAILABLE  :  사용가능한 replica 수](image.png)

UP-TO-DATE : 의도한 상태를 얻기 위해 업데이트된 replica 수
AVAILABLE  :  사용가능한 replica 수

```bash
kubectl rollout status deployment/nginx-deployment
```

몇초 후 ..

![3개의 replica 생성완료, 모든 replica 최신 상태, 사용 가능 상태](image%201.png)

3개의 replica 생성완료, 모든 replica 최신 상태, 사용 가능 상태

```bash
kubectl get rs
```

![image.png](image%202.png)

```bash
kubectl get pods --show-labels
```

![image.png](image%203.png)

❗label과 selector는 다른 컨트롤러와 겹쳐서는 안된다!!. 쿠버네티스가 겹치는 걸 방지해주지 않기때문에 , 알아서 조심해라~

**Pod-template-hash 레이블**

> ⚠️
**이 레이블은 변경하면 안 된다.**
> 
- 디플로이먼트의 자식 replicaset이 겹치지 않도록 보장하는 기능

### **디플로이먼트 업데이트**

- deployment.yaml의 파드 템플릿(.spec.template)이 변경된 경우에만 롤아웃이 트리거됨

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```

```bash
kubectl edit deployment/nginx-deployment
```

→ 위와같이 , `nginx:1.14.2` 이미지 대신 `nginx:1.16.1` 이미지를 사용하도록, 변경하는 두가지 방식이 존재

```bash
kubectl rollout status deployment/nginx-deployment
```

```bash
kubectl get rs
```

![위가 새로운 replicaset, 아래는 이전 replicaset이므로 0개로 scale down 되었음](image%204.png)

위가 새로운 replicaset, 아래는 이전 replicaset이므로 0개로 scale down 되었음

### 롤오버(인플라이트 다중 업데이트)

디플로이먼트 컨트롤러 : 새로운 디플로이먼트에서 레플리카셋이 의도한 파드를 생성하고 띄우는 것을 주시한다.

디플로이먼트가 업데이트되면,

 .spec.selector레이블과 일치하는 파드를 컨트롤

.spec.template이 템플릿과 불일치하면, 스케일 다운

.spec.replicas 로 새 레플리카셋이 스케일 됨

⇒ 롤아웃 진행 중 , 새로 디플로이먼트를 업데이트 하면, 기존에 스케일업하던 중인 레플리카셋에 롤오버한다. ex) 기존 룰아웃중이던 레플리카셋 5개중 3개가 생성되었다면, 이때 업데이트 할 경우 3개를 바로 죽이고, 새로업데이트한 버전으로 파드를 생성한다 ( 기존 진행 중이던 롤아웃 절차가 완료되는 것을 기다려주지 않음)

### 레이블 셀렉터 업데이트

- 일반적으로 업데이트를 권장하지 않음 → 애초에 셀렉터를 미리 계획하는것을 권장
- api 버전 apps/v1에서는 생성 이후에 변경이 아예 안됨
- 냅다 변경해버리면, 기존 셀렉터로 만든 레플리카셋과 그 파드가 고아가 되버림, 그리고 new selector에 일치하는 새로운 레플리카셋을 생성해버림.
- 셀렉터 삭제는 디플로이먼트 셀렉터의 기존 키를 삭제함 . 파드템플릿 레이블의 변경도 필요없고, 기존 레플리카셋도 고아가 되지 않지만, 제거된 레이블이 기존파드와 레플리카셋에 이미 존재한다는 점을 알기.

### 디플로이먼트 롤백

- 디플로이먼트가 지속적인 충돌로 안정적이지 않은 경우 → 롤백 가능
- ✅ 디플로이먼트의 "수정 버전(Revision)"이란?
    - 오직 파드 템플릿 `.spec.template` 내용이 바뀔 때만, 새로운 버전(Revision)이 생성
    
    ### 디플로이먼트의 롤아웃 기록 확인
    
    1. 디플로이먼트의 수정사항 확인
    
    ```bash
    kubectl rollout history deployment/nginx-deployment
    ```
    
    ex)
    
    ![image.png](image%205.png)
    
    1. 각 수정 버전의 세부 정보 확인
    
    ```bash
    kubectl rollout history deployment/nginx-deployment --revision=2
    ```
    
    ### 이전 수정 버전으로 롤백
    
    이전 버전 2로 롤백해보자
    
    1. 현재 롤아웃 실행 취소 및 이전 수정 버전으로 롤백 
    
    ```bash
    kubectl rollout undo deployment/nginx-deployment
    ```
    
    *특정 버전으로 롤백하고 싶다면
    
    ```bash
    kubectl rollout undo deployment/nginx-deployment —to-revision=2
    ```
    

### 디플로이먼트 스케일링

```bash
kubectl scale deployment/nginx-deployment --replicas=10
```

```bash
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

### 비례적 스케일링(proportional Scaling)

- 디플로이먼트 롤링업데이트는 여러버전의 애플리케이션을 동시실행 할 수 있도록 지원한다
- 롤아웃 중에 있는 디플로이먼트 롤링 업데이트를 스케일링 하는 경우, 위험을 줄이기 위해서 기존 활성화 레플리카셋과 추가레플리카셋의 균형을 조절한다.

ex) 기존 10개에서 (구버전 4개, 신버전 6개) → hpa로 15개까지 증가시키면 , 신버전만 11개가 되는게 아니라 , 40/60 으로 비율을 따져서 배분해서 증가시킨다는겨

### 디플로이먼트 롤아웃 일시중지와 재개

```bash
kubectl rollout pause deployment/nginx-deployment
```

- 디플로이먼트를 **pause한 상태에서 템플릿(spec.template)** 을 바꾸면, **롤아웃은 보류되고 있다가 resume할 때 적용된다**
- 중간에 여러 설정을 바꿔야할때, good!!! → pause 후 수정하고, resume하면 안전하게 새 버전으로 rollout됨

### 디플로이먼트 상태

- Progressing
- complete
- Failed

### 정책 초기화

- .spec.revisionHistoryLimit 필드 설정으로 유지해야하는 이전 레플리카셋 수를 명시할 수 있다. (default =10)

### 카나리 디플로이먼트

https://kubernetes.io/ko/docs/concepts/cluster-administration/manage-deployment/#%EC%B9%B4%EB%82%98%EB%A6%AC-canary-%EB%94%94%ED%94%8C%EB%A1%9C%EC%9D%B4%EB%A8%BC%ED%8A%B8 참고

### 디플로이먼트 사양

*디플로이먼트 오브젝트의 이름은 유효한 [DNS 서브도메인 이름](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/names/#dns-%EC%84%9C%EB%B8%8C%EB%8F%84%EB%A9%94%EC%9D%B8-%EC%9D%B4%EB%A6%84)이어야 함!

`.apiVersion`

`.kind`

`.metadata`

[`.spec`](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)

Pod Template

`.spec.template` , `.spec.selector` : 필수

`apiVersion` 과 `kind` 는 없음

[`.spec.template.spec.restartPolicy`](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/#%EC%9E%AC%EC%8B%9C%EC%9E%91-%EC%A0%95%EC%B1%85) Always만 허용

Replica

`.spec.replicas` : 필요한 파드의 수를 지정하는 선택적 필드

⚠️ hpa가 deployment 크기를 관리한다면, `.spec.replicas`를 설정해서는 안됨!!

Selector

`.spec.selector` : 필수 

`.spec.selector` 는 `.spec.template.metadata.labels` 과 일치해야 하며, 그렇지 않으면 API에 의해 거부
⚠️ 다른 컨트롤러를 통해 직접 레이블과 셀렉터가 일치하는 파드를 생성해서는 안됨. → 기존 디플로이먼트가 다른 파드를 만들었다고 생각하게되어 컨트롤러들이 서로 싸우게되므로, 올바른 작동이 불가함. 

Strategy

`.spec.strategy`: 이전 파드를 새로운 파드로 대체하는 전략

`.spec.strategy.type==Recreat` : 새 파드 생성전에 기존 파드 전부 죽음

`.spec.strategy.type==RollingUpdate`

진행기한 시간

`.spec.progressDeadlineSeconds`

[진행 실패](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/#%EB%94%94%ED%94%8C%EB%A1%9C%EC%9D%B4%EB%A8%BC%ED%8A%B8-%EC%8B%A4%ED%8C%A8)를 보고하기 전에 디플로이먼트가 진행되는 것을 대기시키는 시간(초)을 명시

최소 대기 시간

`.spec.minReadySeconds` 새롭게 생성된 파드의 컨테이너가 충돌하지 않고 사용할 수 있도록 준비 하는 최소 시간(초)

**수정 버전 기록 제한**

`.spec.revisionHistoryLimit` : 롤백을 허용하기 위해 보존할 이전 레플리카셋의 수 지정

**일시 정지**

`.spec.paused`

### 리소스 관리

```bash
# application/nginx-app.yaml
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

- 서비스를 먼저 지정하는 걸 권장
- kubectl apply -f {디렉터리(—recursive를 함께 사용)/여러개의 파일/ url (깃허브 등)/ }방식으로 사용하는 것도 가능

### kubectl 대량작업

작성 파일을 통해 동일 리소스를 삭제하는 방식도 가능 

```bash
kubectl delete -f https://k8s.io/examples/application/nginx-app.yaml
```

-l 이나 —selector 를 통해 지정 셀레터로 필터링해서 삭제도 가능

```bash
kubectl delete deployment,services -l app=nginx
```

### 효과적인 레이블 사용

- 세트를 서로 구별하기 위해 레이블을 ‘여러 개 사용해야하는 시나리오’ 가 존재한다.

```bash
# 프론트엔드     
     labels:
        app: guestbook
        tier: frontend
```

```bash
#Redis master
     labels:
        app: guestbook
        tier: backend
        role: master
```

```bash
#Redis master     
     labels:
        app: guestbook
        tier: backend
        role: slave
```

이 경우 다음과 같이 적용됨

![image.png](image%206.png)

```bash
kubectl get pods -lapp=guestbook,role=slave
```

```
NAME                          READY     STATUS    RESTARTS   AGE
guestbook-redis-slave-2q2yf   1/1       Running   0          3m
guestbook-redis-slave-qgazl   1/1       Running   0          3m
```

### 카나리 디플로이먼트

- `labels.track` 레이블을 이용하여 릴리스를 구별
    
    ```yaml
    name: frontend
    replicas: 3
         ...
    labels:
    app: guestbook
    tier: frontend
    track: stable
         ...
    image: gb-frontend:v3
    ```
    
    ```yaml
    name: frontend-canary
    replicas: 1
         ...
    labels:
    app: guestbook
    tier: frontend
    track: canary
         ...
    image: gb-frontend:v4
    ```
    

### 레이블 업데이트

```bash
kubectl label pods -l app=nginx tier=fe
```

### 어노테이션 업데이트

```bash
kubectl annotate pods my-nginx-v4-9gw19 description='my frontend running nginx'
```

### 애플리케이션 스케일링

```bash
kubectl scale deployment/my-nginx --replicas=1
```

### **리소스 인플레이스(in-place) 업데이트**

- 위에서 말한 케이스와 달리 ‘**중단없이 업데이트 해야할 때**’
- kubectl apply :변경한 내용만 적용
- kubectl edit : 직접 열어서 수정 (vi 등)
- kubectl patch :  지정된 필드만 직접 수정

### 파괴적 업데이트

- 한번 초기화 하면 업데이트 할 수 없는 리소스 필드를 업데이트 해야할 때..
- 손상된 파드를 고치는 재귀적 변경을 해야할 때..
- replace --force : 리소스를 삭제하고 다시 만드는 개념

```bash
kubectl replace -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml --force
```

### 무중단 업데이트

✅디폴트 쿠버네티스 Deployment 전략인 롤링 업데이트(RollingUpdate) 자체가 이미 “무중단” 지향한다는 것을 알기!!

create→ scale → edit 하면 알아서 무준단으로 점차적 업데이트함.

- 추가 정리
    - Controller 란? pod은 각자의 controller를 보통 가지고 있음
    
    ![image.png](image%207.png)
    
    - “롤아웃이 막혀있는 상태”
    
    쿠버네티스가 디플로이먼트를 **정상적으로 진행하지 못하고 멈춘 상태**
    
    이 상태는**Deployment의 상태(Conditions)**를 통해 감지 가능