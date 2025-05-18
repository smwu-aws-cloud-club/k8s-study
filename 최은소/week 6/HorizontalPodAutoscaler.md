# Horizontal Pod Autoscaler (HPA)

## 개요
쿠버네티스에서 `HorizontalPodAutoscaler`는 워크로드 리소스(예: Deployment, StatefulSet 등)를 자동으로 업데이트하고, 수요에 맞게 파드 수를 수평적으로 스케일링하는 오브젝트이다.

## 작동 방식
- 주기적으로 메트릭(CPU/메모리 사용률, 커스텀 메트릭 등)을 수집하고, 원하는 레플리카 수를 계산
- 대상 리소스의 `.spec.selector`로 파드 선택
- 사용률(또는 원시 값)을 기반으로 현재 메트릭 / 원하는 메트릭 비율 계산
- 허용 오차(`--horizontal-pod-autoscaler-tolerance`, 기본값 0.1) 내이면 스케일링 생략
- 스케일다운 시 안정화 윈도우(`--horizontal-pod-autoscaler-downscale-stabilization`, 기본값 5분) 고려

## 기본 공식
```
desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]
```

## 주요 구성 요소
- **메트릭 서버(metrics-server)**: `metrics.k8s.io` API 제공
- **커스텀 메트릭 어댑터**: `custom.metrics.k8s.io` API
- **외부 메트릭 어댑터**: `external.metrics.k8s.io` API

## 지원 리소스 메트릭 예시
```yaml
type: Resource
resource:
  name: cpu
  target:
    type: Utilization
    averageUtilization: 60
```

## 컨테이너 리소스 메트릭 예시
```yaml
type: ContainerResource
containerResource:
  name: cpu
  container: application
  target:
    type: Utilization
    averageUtilization: 60
```

## 스케일링 동작 구성 예시
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
    - type: Pods
      value: 5
      periodSeconds: 60
    selectPolicy: Min
```

## 명령어 예시
```bash
kubectl autoscale deployment nginx --cpu-percent=50 --min=1 --max=10
kubectl get hpa
kubectl describe hpa <name>
kubectl delete hpa <name>
```

## HPA와 Deployment 연동 시 주의사항
- `spec.replicas`는 매니페스트에서 제거 권장
- 그렇지 않으면 `kubectl apply`시 해당 값으로 다시 조정됨
- 서버 사이드 적용 시 안전하게 오너십을 넘기는 방식 사용 가능:
```bash
kubectl apply -f nginx-deployment-replicas-only.yaml --server-side --field-manager=handover-to-hpa
```

## 추가 기능 요약
- **복수 메트릭**: 가장 큰 스케일 제안 사용
- **안정화 윈도우**: 갑작스러운 변화 방지
- **스케일링 제한 정책**: `Pods`, `Percent` 유형으로 설정
- **스케일 비활성화**: `selectPolicy: Disabled` 사용
- **점검모드 비활성화**: replicas=0 설정 시 자동 중단됨

