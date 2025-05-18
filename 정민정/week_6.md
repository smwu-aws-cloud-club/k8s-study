## 1. Namespace

### 1.1 개요

- Namespace는 Kubernetes 리소스를 논리적으로 분리하기 위한 방법
- 여러 팀, 프로젝트, 환경(dev/stage/prod 등)을 하나의 클러스터에서 관리할 수 있도록 지원
- 기본적으로 Kubernetes는 default, kube-system, kube-public, kube-node-lease 네 개의 네임스페이스를 포함

### 1.2 주요 특징

- 이름이 충돌하지 않도록 동일한 리소스 이름을 다른 네임스페이스에서 사용 가능
- RBAC 및 리소스 쿼터와 결합해 멀티테넌시 환경 구성에 유리함
- 리소스를 네임스페이스 단위로 그룹핑하여 관리 및 모니터링 효율성을 높임

### 1.3 사용 방법

- 네임스페이스 생성  
  `kubectl create namespace <name>`
  
- 네임스페이스 확인  
  `kubectl get namespaces`
  
- 특정 네임스페이스에 리소스 배포  
  `kubectl apply -f pod.yaml -n <namespace>`
  
- 기본 네임스페이스 설정  
  `kubectl config set-context --current --namespace=<namespace>`

### 1.4 리소스 쿼터

네임스페이스 단위로 CPU, 메모리, 스토리지 등을 제한할 수 있다.

```
yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

---

## 2. Horizontal Pod Autoscaler (HPA)

### 2.1 개요

- 애플리케이션의 부하에 따라 Pod 수를 자동으로 조절하는 Kubernetes 컨트롤러
- CPU 사용률 또는 사용자 정의 메트릭을 기반으로 스케일링

### 2.2 작동 방식

- Metrics Server를 통해 리소스 사용량 수집
- 기본 메트릭은 CPU 사용률이며, 필요 시 사용자 정의 메트릭도 사용 가능
- 목표값과 현재 측정값을 비교해 replica 수 자동 조정

### 2.3 사전 조건

- 클러스터에 metrics-server가 설치되어 있어야 작동함
- Deployment나 ReplicaSet 등의 scale 가능한 리소스를 대상으로 함

### 2.4 HPA 리소스 예시

```
yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### 2.5 커맨드로 생성

```
bash
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=1 --max=5
```

### 2.6 상태 확인

```
bash
kubectl get hpa
kubectl describe hpa nginx-hpa
```

### 2.7 유의 사항

- 스케일링 주기는 기본 15초로 설정되어 있음
- 과도한 스케일링 방지를 위해 히스테리시스와 쿨다운 설정을 고려해야 함
- 최소 및 최대 replica 수는 과도하게 좁게 설정하지 않는 것이 좋음

---

## 3. 요약 및 비교

| 항목 | Namespace | Horizontal Pod Autoscaler |
|------|-----------|----------------------------|
| 목적 | 리소스 논리적 분리 | 부하에 따른 Pod 수 자동 조절 |
| 주요 기능 | 리소스 이름 충돌 방지, 격리, 쿼터 적용 | CPU 또는 사용자 정의 메트릭 기반 스케일링 |
| 적용 대상 | 클러스터 내 전체 리소스 | Deployment, ReplicaSet 등 스케일 가능 리소스 |
| 대표 커맨드 | `kubectl create namespace` | `kubectl autoscale` |
