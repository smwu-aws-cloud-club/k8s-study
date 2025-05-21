### 1. 로깅 개요

Kubernetes 클러스터에서 로깅은 디버깅과 모니터링을 위한 중요한 구성 요소

클러스터의 로그는 다음 세 가지 수준에서 분류됨
- 애플리케이션 로그  
- 노드 로그  
- 클러스터 로그

### 1.1 애플리케이션 로그

- 컨테이너 내에서 실행 중인 애플리케이션이 출력하는 표준 출력(stdout)과 표준 에러(stderr)를 의미
- 컨테이너 런타임은 로그 출력을 수집하고 노드의 특정 위치에 저장
  - ex) Docker는 `/var/log/containers/`에 로그 파일 생성

### 1.2 노드 로그

- Kubelet, 컨테이너 런타임, OS 레벨 로그 등을 포함
- 로그 위치는 운영체제와 런타임 설정에 따라 다름
- 예: `journalctl -u kubelet` 또는 `/var/log/syslog`

### 1.3 클러스터 로그

- 클러스터 전체에서 발생하는 이벤트, 에러, 상태 변경 사항 등을 기록
- `kube-apiserver`, `controller-manager`, `scheduler`, `etcd` 등의 로그가 해당됨
- 로깅은 주로 컨트롤 플레인 구성 요소가 관리

### 1.4 로그 수집 방법

Kubernetes는 기본적으로 로그 저장 또는 중앙 수집 기능을 제공하지 않음

사용자는 로그를 저장하고 검색하기 위한 별도 솔루션을 구축해야 한다.

### 1.5 로그 수집 아키텍처

- **노드 수준 로그 수집기**: fluentd, logstash, filebeat 등을 DaemonSet으로 배포하여 로그 수집
- **사이드카 컨테이너**: 애플리케이션 컨테이너 옆에 로그 수집용 컨테이너를 두고 공유 볼륨으로 로그 전달
- **애플리케이션 직접 전송**: 로그를 직접 외부 로그 서버로 전송 (ex. syslog, log aggregation API)

### 1.6 로그 저장소 예시

- Elasticsearch
- Cloud Logging (GKE)
- Amazon CloudWatch Logs
- Loki (Grafana)

### 1.7 kubectl로 로그 확인

```
kubectl logs <pod-name>
```

### 1.8 다중 컨테이너 로그

```
kubectl logs <pod-name> -c <container-name>
```

### 1.9 이전 로그 보기 (재시작된 경우)

```
kubectl logs <pod-name> --previous
```

### 1.10 파드별 로그 스트리밍

```
kubectl logs -f <pod-name>
```

### 1.11 로그 관련 고려 사항

- 로그는 노드 장애 시 손실될 수 있으므로 중앙 저장 필수
- 사이드카 패턴은 로그 처리와 앱을 분리하는 장점이 있지만 리소스 증가 요인
- 보안, 보존 기간, 접근 제어 등을 고려해야 함

---

### 2. 클러스터 리소스 사용량 모니터링

### 2.1 개요

클러스터의 성능을 모니터링하려면 CPU, 메모리, 디스크, 네트워크 등의 리소스 사용량을 확인해야 함
기본적인 리소스 모니터링은 `Metrics Server`와 `kubectl top` 커맨드를 통해 제공됨

### 2.2 Metrics Server

- 클러스터 내 리소스 사용량을 수집하는 경량화된 메트릭 수집기
- kubelet으로부터 주기적으로 메트릭 수집
- HPA, VPA, kubectl top 등의 기능이 Metrics Server에 의존함

### 2.3 Metrics Server 설치

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 2.4 설치 확인

```
kubectl get deployment metrics-server -n kube-system
```

### 2.5 Pod 리소스 사용량 확인

```
kubectl top pod
kubectl top pod -n <namespace>
```

### 2.6 노드 리소스 사용량 확인

```
kubectl top node
```

### 2.7 리소스 부족 디버깅 시나리오

- 특정 Pod가 CPU나 메모리 부족으로 OOMKilled 되는 경우 확인 가능  
- `kubectl describe pod <pod-name>`로 상태 및 이벤트 확인  
- `top` 명령으로 다른 Pod이나 노드에 과도한 리소스 사용이 있는지 진단 가능

### 2.8 사용량 기준 HPA 트리거

Metrics Server가 제공하는 메트릭은 HPA의 트리거로 사용됨
CPU 사용률이 일정 이상일 때 Pod이 자동으로 증가

### 2.9 kubectl describe hpa로 스케일링 상황 확인

```
kubectl describe hpa <hpa-name>
```

### 2.10 기타 모니터링 도구

- Prometheus + Grafana: 커스텀 메트릭 수집과 시각화
- Kube-state-metrics: 리소스 상태를 메트릭화
- cAdvisor: 컨테이너 리소스 사용량 수집
- Cloud Provider 모니터링 (CloudWatch, Stackdriver 등)

### 2.11 리소스 모니터링 구성 시 고려사항

- Metrics Server는 장기 저장이 안 되므로 로그/메트릭 저장소 연동 필요
- 수집 간격 조정, 보안 설정, 네트워크 부하 등을 고려해야 함
- Prometheus 연동 시 수집 주기, retention 정책, 알림 설정도 중요
