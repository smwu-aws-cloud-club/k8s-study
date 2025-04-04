# Kubernetes ReplicaSet 정리

## 개요

- ReplicaSet의 목적은 지정된 수의 파드 레플리카가 항상 실행되도록 안정성을 보장하는 것.
- 일반적으로 파드 가용성을 보장하기 위해 사용됨.
- Deployment가 ReplicaSet을 관리하기 때문에 대부분의 경우 직접 사용하는 대신 Deployment 사용이 권장됨.

## 작동 방식

- 주요 구성 요소:
  - 셀렉터: 파드를 식별
  - 레플리카 수: 유지해야 하는 파드 수
  - 파드 템플릿: 새 파드 생성 시 사용
- 생성된 파드는 `ownerReferences` 필드를 통해 레플리카셋과 연결됨.

## 사용 시기

- Custom 업데이트 오케스트레이션 필요 시 직접 사용
- 일반적 환경에서는 Deployment 사용을 권장

## 예시: frontend ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

## 주요 명령어

```bash
kubectl apply -f https://kubernetes.io/examples/controllers/frontend.yaml
kubectl get rs
kubectl describe rs/frontend
kubectl get pods
kubectl get pods frontend-b2zdv -o yaml
```

## 주의 사항

- 단독 Pod가 ReplicaSet의 셀렉터와 일치하면 자동으로 소유될 수 있음.
- 템플릿을 사용하지 않는 파드도 소유될 수 있음 → 격리 필요

## 매니페스트 작성

- `apiVersion`, `kind`, `metadata` 필수
- `.spec.template` 은 파드 정의 포함
- `.spec.selector` 는 `.spec.template.metadata.labels` 와 일치해야 함
- `.spec.replicas`: 기본값은 1

## 삭제 정책

- 파드와 함께 삭제: `propagationPolicy: Foreground`
- 파드는 유지하고 ReplicaSet만 삭제: `--cascade=orphan`

## 파드 격리

- 레이블 변경으로 ReplicaSet 소유권 제거 가능 → 디버깅/복구에 유용

## 스케일링

- `.spec.replicas` 수정
- 스케일 다운 우선순위 기준:
  - Pending 상태
  - 낮은 `pod-deletion-cost`
  - 레플리카 수 많은 노드
  - 생성 시점 최신
  - 그 외 임의

## 파드 삭제 비용

- 어노테이션: `controller.kubernetes.io/pod-deletion-cost`
- 범위: -2147483647 ~ 2147483647
- 낮은 값이 먼저 삭제됨
- default는 0

## HPA 연동

```yaml
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

```bash
kubectl autoscale rs frontend --max=10 --min=3 --cpu-percent=50
```

## 대안 오브젝트

- **Deployment**: 선언적 업데이트 + 롤링 업데이트 지원 → 권장
- **Pod**: 수동 관리 필요, 실패 복구 없음
- **Job**: 일시적 작업에 적합
- **DaemonSet**: 머신 레벨 서비스에 적합
- **ReplicationController**: 과거 방식, 현재는 ReplicaSet 선호
