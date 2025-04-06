# Kubernetes Deployment 정리

## 개요

- Deployment는 파드(Pod)와 레플리카셋(ReplicaSet)에 대한 선언적 업데이트를 제공한다.
- 사용자는 의도한 상태를 기술하고, Deployment 컨트롤러는 현재 상태에서 의도한 상태로 비율을 조정하며 변경한다.
- 새 ReplicaSet을 생성하거나 기존 Deployment를 제거하고, 모든 리소스를 새 Deployment에 적용할 수 있다.
- Deployment가 소유하는 ReplicaSet은 직접 관리하지 않아야 한다.

## Deployment 유스케이스

- ReplicaSet을 롤아웃할 Deployment 생성
- PodTemplateSpec을 업데이트해 새로운 상태 선언
- 현재 상태가 불안정한 경우 이전 버전으로 롤백
- 스케일 업/다운
- 롤아웃 일시 중지 및 재개
- 상태 확인
- 불필요한 이전 ReplicaSet 정리

## Deployment 예시

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

- `.metadata.name`: nginx-deployment 이름의 Deployment 생성
- `.spec.replicas`: 3개의 Replica를 생성할 ReplicaSet 생성
- `.spec.selector`: 관리할 파드를 선택하는 조건
- `template`: 파드의 템플릿 정의

## Deployment 생성 및 확인

```bash
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
kubectl get deployments
```

출력 예시:

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/3     0            0           1s
```

## 롤아웃 상태 확인

```bash
kubectl rollout status deployment/nginx-deployment
```

## Deployment 업데이트

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
kubectl edit deployment/nginx-deployment
```

## 업데이트 후 상태 확인

- `kubectl get deployments`, `kubectl get rs`, `kubectl get pods`

## Rolling Update 전략

- `maxUnavailable`: 기본값 25%
- `maxSurge`: 기본값 25%
- 예: 최소 3개 파드 사용 보장, 최대 4개 파드까지 실행 가능

## Rollout 일시 중지 및 재개

```bash
kubectl rollout pause deployment/nginx-deployment
kubectl set image ...
kubectl set resources ...
kubectl rollout resume deployment/nginx-deployment
```

## 롤백

```bash
kubectl rollout undo deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

## Rollout 기록 확인

```bash
kubectl rollout history deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment --revision=2
```

## Proportional Scaling

- 롤아웃 중 자동 스케일링 시 레플리카셋 비율에 따라 분산

## Deployment 상태

- **진행 중**: 새로운 ReplicaSet 생성, 스케일 업/다운 중
- **완료**: 모든 레플리카가 최신 상태로 업데이트됨
- **실패**: 이미지 풀 에러, 자원 부족 등

## 실패 감지: progressDeadlineSeconds

```bash
kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
```

## 디버깅: describe 확인

```bash
kubectl describe deployment nginx-deployment
```

## 정책 및 전략

- `.spec.revisionHistoryLimit`: 롤백 가능한 수정 기록 수 (기본값 10)
- `.spec.strategy.type`: Recreate 또는 RollingUpdate
- `.spec.minReadySeconds`: 새 파드가 준비 상태가 되어야 하는 최소 시간
- `.spec.paused`: 롤아웃 일시 중지 여부

## 카나리 배포

- 각 버전을 별도의 Deployment로 관리하여 점진적으로 릴리스

## 기타 주의 사항

- 파드 템플릿 레이블과 셀렉터는 반드시 일치해야 함
- 셀렉터는 변경 불가 (apps/v1)
- 충돌 방지를 위해 컨트롤러 간 레이블 셀렉터 겹침 주의
