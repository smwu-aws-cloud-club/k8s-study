# 레플리카셋

### 레플리카셋의 작동방식

레플리카셋(ReplicaSet)은 특정 개수의 파드를 유지하도록 관리하는 쿠버네티스 오브젝트이다. 이를 위해 셀렉터를 사용하여 관리할 파드를 식별하고, 유지해야 할 파드 개수를 설정하며, 필요할 경우 생성할 신규 파드의 템플릿을 정의한다. 레플리카셋은 지정된 설정을 충족하기 위해 파드를 생성하거나 삭제하며, 새로운 파드를 만들 때는 미리 정의된 파드 템플릿을 사용한다.

또한 레플리카셋은 파드의 `metadata.ownerReferences` 필드를 통해 해당 파드와 연결되며, 이를 통해 자신이 소유한 파드를 추적하고 관리한다. 셀렉터를 활용하여 새로운 파드를 식별하며, 만약 파드에 `OwnerReference`가 없거나 컨트롤러가 아닌 상태에서 레플리카셋의 셀렉터와 일치한다면, 해당 파드는 자동으로 레플리카셋의 관리 대상이 된다.

### 레플리카셋을 사용하는 시기

레플리카셋은 지정된 수의 파드 레플리카가 항상 실행되도록 보장하지만, 디플로이먼트는 이를 관리하면서 선언적 업데이트 기능을 제공하는 상위 개념이다. 따라서 별도의 사용자 정의 업데이트가 필요하지 않다면, 레플리카셋을 직접 사용하기보다는 디플로이먼트를 사용하는 것이 권장된다.

### 예시

1. **레플리카셋 매니페스트 정의**
- `frontend.yaml` 파일을 생성하여 레플리카셋을 정의한다.
- 주요 설정:
    - `replicas: 3` → 3개의 파드를 유지
    - `selector.matchLabels` → `tier: frontend` 레이블을 가진 파드 관리
    - `template.spec.containers` → `php-redis` 컨테이너 사용

1. **레플리카셋 적용 및 확인**
- 다음 명령어를 실행하여 레플리카셋을 생성한다.

```bash
kubectl apply -f https://kubernetes.io/examples/controllers/frontend.yaml
```

- 현재 배포된 레플리카셋 확인

```bash
kubectl get rs
```

- 출력 예시

```bash
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       6s
```

1. **레플리카셋 상세 정보 확인**
- 레플리카셋의 상태 확인

```bash
kubectl describe rs/frontend
```

- 출력 내용
    - `Replicas: 3 current / 3 desired` → 3개의 파드 실행 중
    - `Pods Status: 3 Running / 0 Waiting / 0 Failed` → 모든 파드 정상 실행
    - 생성된 파드 이벤트 정보 출력

1. **파드 상태 확인**
- 실행 중인 파드 목록 조회

```bash
kubectl get pod
```

- 출력 예시

```sql
NAME             READY   STATUS    RESTARTS   AGE
frontend-b2zdv   1/1     Running   0          6m36s
frontend-vcmts   1/1     Running   0          6m36s
frontend-wtsmm   1/1     Running   0          6m36s
```

1. **파드의 소유자 확인**
- 특정 파드의 상세 정보를 확인하여 해당 파드가 레플리카셋에 속해 있는지 확인

```bash
kubectl get pods frontend-b2zdv -o yaml
```

- 출력 내용 (`ownerReferences` 필드 확인)

```yaml
ownerReferences:
- apiVersion: apps/v1
  controller: true
  kind: ReplicaSet
  name: frontend
  uid: <고유 ID>
```

→ 파드의 `ownerReferences` 필드에 `frontend` 레플리카셋이 소유자로 설정됨.

이와 같은 과정을 통해 레플리카셋을 생성하고, 파드를 관리하며 상태를 모니터링할 수 있다.

### 템플릿을 사용하지 않는 파드의 획득

1. **단독 파드와 레플리카셋의 관계**
- 단독(bare)파드를 생성하는 것은 가능하지만, 해당 파드가 레플리카셋의 셀렉터와 일치하는지는 레이블을 가지지 않도록 권장됨.
- 레플리카셋은 단순히 템플릿에서 생성한 파드뿐만 아니라, 셀렉터에 맞는 다른 파드도 자동으로 쇼유할 수 있기 때문.

1. **단독 파드 생성 및 레플리카셋의 소유권 획득 예제**

**(1) 단독 파드 생성**

- `pod-rs.yaml`을 적용하여 단독 파드(`pod1`, `pod2`)를 생성

```yaml
kubectl apply -f https://kubernetes.io/examples/pods/pod-rs.yaml
```

**(2) 기존 레플리카셋과 충돌**

- 이전에 생성된 `frontend` 레플리카셋과 `pod1`, `pod2`의 레이블(`tier: frontend`)이 일치함.
- 따라서, 레플리카셋이 해당 파드를 인식하여 자동으로 소유하고, 필요 이상이면 즉시 종료.
- 실행된 파드 확인

```bash
kubectl get pods
```

- 출력 예시 (단독 파드가 종료됨)

```sql
NAME             READY   STATUS        RESTARTS   AGE
frontend-b2zdv   1/1     Running       0          10m
frontend-vcmts   1/1     Running       0          10m
frontend-wtsmm   1/1     Running       0          10m
pod1             0/1     Terminating   0          1s
pod2             0/1     Terminating   0          1s
```

**(3) 단독 파드를 먼저 생성하고 이후 레플리카셋 생성**

- 단독 파드를 먼저 생성

```bash
kubectl apply -f https://kubernetes.io/examples/pods/pod-rs.yaml
```

- 이후, `frontend` 레플리카셋을 생성

```bash
kubectl apply -f https://kubernetes.io/examples/controllers/frontend.yaml
```

**(4) 레플리카셋이 단독 파드를 소유함**

- 레플리카셋이 `pod1`과 `pod2`를 인식하고, 추가적인 신규 파드만 생성하여 필요한 개수를 유지.
- 실행된 파드 확인

```bash
kubectl get pods
```

- 출력 예시

```sql
NAME             READY   STATUS    RESTARTS   AGE
frontend-hmmj2   1/1     Running   0          9s
pod1             1/1     Running   0          36s
pod2             1/1     Running   0          36s
```

→ 기존 단독 파드(`pod1`, `pod2`)도 레플리카셋에 의해 소유됨.

**3. 결론**

- 레플리카셋은 셀렉터와 일치하는 단독 파드를 자동으로 소유할 수 있다.
- 따라서 단독 파드를 생성할 때는 레플리카셋과 동일한 레이블을 가지지 않도록 주의해야 한다.
- 반대로 특정 파드를 레플리카셋이 자동으로 관리하도록 의도적으로 설계할 수도 있다.

### 레플리카셋의 대안

- **디플로이먼트(권장)**

디플로이먼트는 레플리카셋을 소유하고 업데이트하며, 파드의 선언적 업데이트와 롤링 업데이트를 제공하는 오브젝트이다. 레플리카셋을 단독으로 사용할 수 있지만, 일반적으로 디플로이먼트를 통해 파드의 생성, 삭제, 업데이트를 관리한다. 디플로이먼트를 사용하면 생성된 레플리카셋을 직접 관리할 필요가 없으며, 레플리카셋이 필요할 경우 디플로이먼트를 활용하는 것이 권장된다.

- **기본 파드**

레플리카셋은 노드 장애나 유지보수로 인해 종료된 파드를 자동으로 교체하여 안정성을 보장한다. 따라서 단일 파드만 필요한 경우에도 레플리카셋을 사용하는 것이 권장된다. 이는 프로세스 관리와 유사하지만, 개별 프로세스가 아닌 여러 노드에 걸친 다수의 파드를 관리하는 역할을 한다. 또한, 로컬 컨테이너의 재시작은 Kubelet과 같은 노드 내 에이전트가 처리하도록 위임한다.

- **잡**

스스로 종료되는 것이 예상되는 파드의 경우에는 레플리카셋 대신 [`잡`](https://kubernetes.io/ko/docs/concepts/workloads/controllers/job/)을 이용한다 (즉, 배치 잡).

- **데몬셋**

머신 모니터링이나 로깅처럼 머신-레벨 기능을 제공하는 파드는 레플리카셋 대신 데몬셋을 사용해야 한다. 데몬셋은 머신의 수명과 연관되며, 다른 파드보다 먼저 실행되고 머신 종료 시 안전하게 종료될 수 있도록 관리된다.

- **레플리케이션 컨트롤러**

레플리카셋은 레플리케이션 컨트롤러의 후속 버전으로, 동일한 용도와 유사한 동작 방식을 가진다. 다만, 레플리케이션 컨트롤러는 설정 기반 셀렉터를 지원하지 않기 때문에 레플리카셋이 더 선호된다.

# **디플로이먼트**

디플로이먼트는 파드와 레플리카셋에 대한 선언적 업데이트를 제공하며, 현재 상태를 원하는 상태로 조정하는 역할을 한다. 이를 통해 새 레플리카셋을 생성하거나 기존 디플로이먼트를 교체할 수 있다. 디플로이먼트가 관리하는 레플리카셋은 직접 조작하지 않는 것이 원칙이며, 특별한 경우에는 쿠버네티스 리포지터리에 이슈를 제기할 수 있다.

### 유스케이스

디플로이먼트의 주요 유스케이스는 다음과 같다.

새로운 레플리카셋을 롤아웃하여 파드를 생성하고, 롤아웃 상태를 확인한다. PodTemplateSpec을 업데이트하면 새 레플리카셋이 생성되고, 디플로이먼트는 기존 파드를 점진적으로 교체한다. 상태가 불안정하면 이전 버전으로 롤백할 수 있으며, 필요에 따라 스케일 업도 가능하다. 또한, 롤아웃을 일시 중지하여 여러 변경 사항을 적용한 후 재개할 수 있으며, 롤아웃 상태를 확인하고 불필요한 이전 레플리카셋을 정리할 수도 있다.

### 디플로이먼트 생성

3개의 `nginx` 파드를 불러오기 위한 레플리카셋을 생성한다.

```yaml
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

- `nginx-deployment`라는 이름의 디플로이먼트가 생성
- `replicas: 3` 설정으로 3개의 nginx 파드를 생성하는 레플리카셋을 관리
- `.spec.selector.matchLabels`로 `app: nginx` 레이블을 가진 파드를 선택하여 관리

**디플로이먼트 실행 과정**

1. 디플로이먼트 생성

```bash
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

1. 디플로이먼트 목록 확인

```sql
kubectl get deployments
```

1. 디플로이먼트의 롤아웃 상태 확인

```sql
kubectl rollout status deployment/nginx-deployment
```

1. 레플리카셋(ReplicaSet) 목록 확인

```sql
kubectl get rs
```

1. 실행 중인 파드 목록과 레이블 확인

```sql
kubectl get pods --show-labels
```

**생성 정리**

디플로이먼트는 레플리카셋(RS)을 생성 및 관리하여 파드의 선언적 업데이트를 수행하며, 롤아웃(Rollout)과 롤백(Rollback)을 통해 버전 관리를 할 수 있다. 또한, `PodTemplateSpec`이 변경되면 새로운 레플리카셋을 생성하여 점진적으로 업데이트를 진행한다.

디플로이먼트를 사용할 때 주의할 점으로는, `.spec.selector.matchLabels`에서 레이블이 다른 컨트롤러(StatefulSet 등)와 겹치지 않도록 설정해야 하며, `pod-template-hash` 레이블은 디플로이먼트가 자동으로 관리하는 값이므로 변경하면 안 된다.

### 디플로이먼트 업데이트

1. **디플로이먼트 업데이트**

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```

또는 다음의 명령어도 사용할 수 있다.

```bash
kubectl edit deployment/nginx-deployment
```

대안으로 디플로이먼트를 `edit` 해서 `.spec.template.spec.containers[0].image` 를 `nginx:1.14.2` 에서 `nginx:1.16.1` 로 변경한다.

```bash
kubectl edit deployment/nginx-deployment
```

1. **롤아웃 상태 확인**

```bash
kubectl rollout status deployment/nginx-deployment
```

1. **디플로이먼트 업데이트 후 확인**
- Deployment 목록 조회

```bash
kubectl get deployments
```

- ReplicaSet 생성 확인

```bash
kubectl get rs
```

- Pods 조회

```
kubectl get pods
```

1. **Deployment 상세 정보 확인**

```bash
kubectl describe deployments
```

**5. Deployment 업데이트 과정**

- 새로운 ReplicaSet 생성 후 점진적으로 파드 교체
- 기본적으로 25% 이하의 파드는 사용할 수 없고, 125% 이하까지만 추가 생성 가능
- 최소한 전체 파드 수의 75%는 항상 유지

1. **롤오버 (In-flight 다중 업데이트)**
- 업데이트 중 추가 업데이트 발생 시, 새 ReplicaSet을 생성하고 기존 업데이트를 롤오버
- 기존 ReplicaSet이 완전히 반영되지 않아도 새로운 버전으로 변경
- 예: `nginx:1.14.2` → `nginx:1.16.1` 업데이트 시, 기존 파드 일부가 실행 중이어도 즉시 새로운 버전 적용

1. **레이블 셀렉터 업데이트**
- 권장하지 않고 처음부터 유의하여 설계해야한다.
- 변경 시 유의점
    - `apps/v1` API 버전에서는 Deployment의 셀렉터 변경 불가
    - 기존 ReplicaSet이 orphan(고아) 상태가 되고 새로운 ReplicaSet 생성
    - 삭제된 레이블은 기존 파드와 ReplicaSet에 계속 존재

**업데이트 정리**

Deployment의 롤링 업데이트는 새로운 ReplicaSet을 생성하고 점진적으로 교체하는 방식으로 진행된다. 레이블 변경은 신중해야 하며, 기존 셀렉터는 변경할 수 없다.

### **롤백**

1. **롤백이 필요한 상황**

디플로이먼트 업데이트 후 지속적인 충돌이 발생하거나, 예상치 못한 오류가 발생하면 롤백이 필요할 수 있다.

- Kubernetes는 기본적으로 모든 디플로이먼트의 롤아웃 기록을 저장하여 언제든지 롤백 가능
- `kubectl rollout history`를 통해 확인할 수 있는 롤아웃 수정 버전 수는 변경 가능

1. **롤백 과정**
- **잘못된 업데이트 진행 예시**

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.161
```

이후 `kubectl rollout status` 명령어를 실행하면 롤아웃이 고착됨을 확인할 수 있다.

```bash
kubectl rollout status deployment/nginx-deployment
```

`kubectl get pods` 명령어로 상태를 확인하면, 잘못된 이미지 때문에 `ImagePullBackOff` 상태가 발생한다.

```bash
kubectl get pods
```

- **디플로이먼트 롤아웃 기록 확인**

```bash
kubectl rollout history deployment/nginx-deployment
```

각 수정 버전의 세부 정보를 보려면 아래와 같이 사용한다.

```bash
kubectl rollout history deployment/nginx-deployment --revision=2
```

- **이전 수정 버전으로 롤백**

1) 가장 최근 안정된 버전으로 롤백

```bash
kubectl rollout undo deployment/nginx-deployment
```

2) 특정 수정 버전으로 롤백

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

- **롤백 성공 확인**

```
kubectl get deployment nginx-deployment
```

또는 아래와 같이 사용할 수도 있다.

```
kubectl describe deployment nginx-deployment
```

**롤백 정리**
디플로이먼트는 기본적으로 롤아웃 기록을 유지하므로 잘못된 업데이트 발생 시 언제든지 롤백할 수 있다. `kubectl rollout history`를 사용하면 수정 기록을 확인할 수 있으며, 필요에 따라 특정 버전으로 롤백이 가능하다. 롤백 후에는 `kubectl get deployment` 및 `kubectl describe deployment` 명령어를 통해 정상적으로 동작하는지 확인해야 한다.

## 디플로이먼트 스케일링

디플로이먼트의 스케일을 조정할 수 있다.

```
kubectl scale deployment/nginx-deployment --replicas=10
```

클러스터에서 **Horizontal Pod Autoscaling**을 설정한 경우, 디플로이먼트에 대한 오토스케일러를 설정할 수 있다. 기존 파드의 CPU 사용률을 기준으로 최소 및 최대 파드 수를 지정할 수 있다.

```
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

## 비례적 스케일링 (Proportional Scaling)

디플로이먼트 롤링 업데이트 중에 스케일링을 수행할 경우, 기존 레플리카셋의 추가 레플리카를 균형 있게 조절하여 위험을 줄이는 방식을 비례적 스케일링이라 한다.

예를 들어, 10개의 레플리카를 가진 디플로이먼트에서 `maxSurge=3`, `maxUnavailable=2` 로 설정되어 있다면:

- 현재 레플리카 상태를 확인

```
kubectl get deploy
```

- 새로운 이미지로 업데이트

```
kubectl set image deployment/nginx-deployment nginx=nginx:sometag
```

- 롤아웃 상태를 확인

```
kubectl get rs
```

이후 오토스케일러가 레플리카를 15개로 증가시키면, 비례적 스케일링을 통해 새로운 5개의 레플리카가 기존 레플리카셋과 새 레플리카셋에 균등하게 분배된다.

- 롤아웃 프로세스를 확인

```
kubectl get deploy
```

- 각 레플리카셋에 추가된 상태를 확인

```
kubectl get rs
```

## 디플로이먼트 롤아웃 일시 중지와 재개

디플로이먼트를 업데이트할 때, 불필요한 롤아웃을 방지하기 위해 롤아웃을 일시 중지할 수 있다. 일시 중지 후 여러 수정 사항을 적용한 다음, 롤아웃을 재개할 수 있다.

- 디플로이먼트 상태 확인

```
kubectl get deploy
```

- 롤아웃 상태 확인

```
kubectl get rs
```

- 롤아웃을 일시 중지

```
kubectl rollout pause deployment/nginx-deployment
```

- 이미지를 업데이트

```
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```

- 일시 중지 상태이므로 롤아웃이 시작되지 않는다.  롤아웃 이력을 확인한다.

```
kubectl rollout history deployment/nginx-deployment
```

- 롤아웃을 재개

```
kubectl rollout resume deployment/nginx-deployment
```

- 롤아웃 진행 상태를 확인

```
kubectl get rs -w
```

- 최신 롤아웃 상태 확인

```
kubectl get rs
```

일시 중지된 디플로이먼트는 재개되기 전까지 롤백할 수 없다.

## 디플로이먼트 상태

디플로이먼트는 라이프사이클 동안 다양한 상태로 전환되다. 새 레플리카셋을 롤아웃하는 동안 **"진행 중", "완료", "실패"** 상태가 될 수 있다.

1. **진행 중**
- 진행 중에 해당하는 경우
    - 새 레플리카셋의 생성
    - 새로운 레플리카셋 스케일업
    - 기존 레플리카셋 스케일 다운
    - 새 파드가 준비되거나 최소 준비 시간 동안 대기

이때 `.status.conditions` 속성에는 다음이 포함된다.

```
type: Progressing
status: "True"
reason: NewReplicaSetCreated | FoundNewReplicaSet | ReplicaSetUpdated
```

진행 상태를 확인하려면:

```
kubectl rollout status deployment/nginx-deployment
```

**2. 완료**

- 완료에 해당하는 경우
    - 모든 레플리카가 최신 버전으로 업데이트
    - 모든 레플리카가 사용 가능
    - 이전 레플리카셋이 실행되지 않음

이때 `.status.conditions` 속성에는 다음이 포함된다.

```
type: Progressing
status: "True"
reason: NewReplicaSetAvailable
```

완료 여부를 확인하려면:

```
kubectl rollout status deployment/nginx-deployment
```

**3. 실패**

- 실패에 해당하는 경우
    - 할당량 부족
    - 준비성 프로브 실패
    - 이미지 풀 오류 발생
    - 권한 부족
    - 애플리케이션 구성 오류 발생

진행 데드라인(`.spec.progressDeadlineSeconds`)을 설정하여 디플로이먼트가 일정 시간 내 완료되지 않으면 실패 상태로 간주할 수 있다.

```
kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
```

데드라인을 초과하면 `.status.conditions` 속성에 다음이 추가된다.

```
type: Progressing
status: "False"
reason: ProgressDeadlineExceeded
```

진행이 실패했는지 확인하려면:

```
kubectl rollout status deployment/nginx-deployment
```

실패 시 종료 상태는 `1`이 반환된다.

```
echo $?
1
```

**실패한 디플로이먼트 조치**

- 스케일 업/다운
- 이전 버전으로 롤백
- 롤아웃 일시 중지 후 수정

상세한 상태 확인은 다음 명령어를 사용한다.

```bash
kubectl describe deployment nginx-deployment
kubectl get deployment nginx-deployment -o yaml
```

### **디플로이먼트 설정 및 전략**

**1. 정책 초기화**

- `.spec.revisionHistoryLimit` 필드를 설정하여 유지할 이전 레플리카셋 개수 지정 (기본값: 10).
- `0`으로 설정하면 모든 기록이 초기화되며 롤백 불가.

**2. 카나리 디플로이먼트**

- 특정 사용자 또는 서버 대상으로 점진적 배포.
- 각 릴리스마다 별도 디플로이먼트 생성.

**3. 디플로이먼트 사양**

- 기본 필드: `.apiVersion`, `.kind`, `.metadata`.
- `.spec.template`: 파드 템플릿이며, `.spec.selector`와 일치해야 함.
- `.spec.replicas`: 기본값 `1`, HPA 사용 시 설정하면 안 됨.

**4. 셀렉터**

- `.spec.selector`는 `.spec.template.metadata.labels`와 일치해야 함.
- 생성 후 변경 불가, 중복되는 컨트롤러 생성 금지.

**5. 배포 전략**

- `.spec.strategy.type`: `Recreate`(재생성) 또는 `RollingUpdate`(롤링 업데이트, 기본값).

  **롤링 업데이트 전략**

    - **Max Unavailable**: 사용 불가한 최대 파드 수 (기본값 `25%`).
    - **Max Surge**: 의도한 파드 수 대비 초과 생성 가능 수 (기본값 `25%`)


**6. 진행 기한 및 최소 대기 시간**

- `.spec.progressDeadlineSeconds`: 기본 `600초`, 진행 실패로 간주되는 시간.
- `.spec.minReadySeconds`: 새 파드가 최소한 준비 상태를 유지해야 하는 시간 (기본값 `0초`)

**7. 버전 기록 제한**

- `.spec.revisionHistoryLimit` 기본값 `10`.
- `0`으로 설정 시 이전 레플리카셋 삭제되며 롤백 불가.

**8. 일시 정지**

- `.spec.paused`: `true` 설정 시 롤아웃 중지.

# 리소스 관리

## **1. 리소스 구성하기**

- 여러 리소스를 하나의 YAML 파일에 정의 가능 (`--` 사용).
- 단일 명령어로 여러 리소스 배포 가능.

**예제: 다중 리소스 배포**

```yaml
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

**배포 명령어**

```bash
kubectl apply -f nginx-app.yaml  # YAML 파일로 배포
kubectl apply -f ./my-k8s-configs  # 디렉터리 내 모든 YAML 배포
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/nginx/nginx-deployment.yaml  # 원격 YAML 배포
```

## **kubectl에서의 대량 작업**

- 여러 리소스를 한 번에 삭제 또는 조회 가능.

**대량 삭제**

```
kubectl delete deployment,services -l app=nginx
```

- `app=nginx` 레이블을 가진 모든 `deployment` 및 `service` 삭제.

**동적 리소스 추출 및 조회**

```
kubectl get $(kubectl create -f nginx/ -o name | grep service)
```

- 생성된 리소스 중 `service`만 필터링하여 조회.

### **xargs 활용**

```
kubectl create -f nginx/ -o name | grep service | xargs -i kubectl get {}
```

- 여러 리소스를 한 번에 처리할 때 유용.

## **효과적인 레이블 사용**

- 여러 레이블을 활용하여 리소스를 세분화하고 관리 가능.

**예제: 방명록 애플리케이션 레이블링**

- 프론트엔드

```yaml
labels:
  app: guestbook
  tier: frontend

```

- 백엔드 (Redis 마스터)

```yaml
labels:
  app: guestbook
  tier: backend
  role: master

```

- 백엔드 (Redis 슬레이브)

```yaml
labels:
  app: guestbook
  tier: backend
  role: slave

```

### **레이블 기반 조회**

```bash
kubectl get pods -Lapp -Ltier -Lrole  # 레이블 필터링 조회
kubectl get pods -l app=guestbook,role=slave  # 특정 레이블을 가진 파드만 조회
```

## **카나리(Canary) 디플로이먼트**

- 새로운 버전의 애플리케이션을 점진적으로 배포하여 테스트.
- `track` 레이블을 활용해 안정(stable) 버전과 구분.

**예제: 카나리 배포**

- **안정(stable) 버전**

```yaml
labels:
  app: guestbook
  tier: frontend
  track: stable
image: gb-frontend:v3

```

- **카나리(canary) 버전**

```yaml
labels:
  app: guestbook
  tier: frontend
  track: canary
image: gb-frontend:v4

```

- **트래픽 분배**

```yaml
selector:
  app: guestbook
  tier: frontend

```

- 배포 후 안정 버전을 업데이트하고 카나리 버전 제거 가능.

## **레이블 및 어노테이션 업데이트**

**레이블 추가/수정**

```
kubectl label pods -l app=nginx tier=fe
```

- `app=nginx` 레이블을 가진 모든 파드에 `tier=fe` 추가.

**어노테이션 추가**

```
kubectl annotate pods my-nginx-v4-9gw19 description='my frontend running nginx'

```

- 메타데이터 추가(조회 시 선택적으로 사용 가능).

## **애플리케이션 스케일링**

**수동 스케일링**

```
kubectl scale deployment/my-nginx --replicas=1

```

- `nginx` 디플로이먼트를 1개의 파드로 축소.

**자동 스케일링 (HPA: Horizontal Pod Autoscaler)**

```
kubectl autoscale deployment/my-nginx --min=1 --max=3

```

- 최소 1개, 최대 3개의 파드로 자동 확장.

## **리소스 인플레이스(In-Place) 업데이트**

- **구성 파일을 유지하면서** 리소스를 업데이트하는 방법.
- 변경 사항을 기존 리소스와 비교 후 필요한 부분만 수정.

**사용법**

```
kubectl apply -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml

```

- 기존 리소스와 **3-way diff(이전 상태, 현재 상태, 새로운 설정 비교)** 를 수행하여 업데이트.

## **파괴적(Disruptive) 업데이트**

- **수정할 수 없는 필드**를 변경해야 할 경우, 리소스를 삭제 후 재생성.
- 강제 삭제(`-force`) 후 재적용.

**사용법**

```
kubectl replace -f nginx-deployment.yaml --force

```

- 기존 리소스를 삭제하고 다시 생성.

## **서비스 중단 없이 애플리케이션 업데이트**

- `kubectl edit`을 사용해 `image` 값을 변경하여 점진적 배포.

**예제: 버전 업데이트**

```bash
kubectl create deployment my-nginx --image=nginx:1.14.2
kubectl scale deployment my-nginx --replicas=3
kubectl edit deployment/my-nginx  # nginx:1.14.2 → nginx:1.16.1 변경
```

- 기존 파드를 단계적으로 교체하여 서비스 중단 없이 업데이트 진행.