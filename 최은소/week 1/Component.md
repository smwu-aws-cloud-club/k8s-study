# [쿠버네티스 컴포넌트](https://kubernetes.io/ko/docs/concepts/overview/components/)

## 개요

![쿠버네티스 클러스터 구성 요소](https://kubernetes.io/images/docs/components-of-kubernetes.svg)<br>

쿠버네티스는 **컨트롤 플레인 컴포넌트**와 **노드 컴포넌트**로 구성된다. 또한 클러스터에서 실행되는 **애드온**이 추가적으로 기능을 확장한다.

---

## 컨트롤 플레인 컴포넌트

쿠버네티스 클러스터의 전반적인 상태를 관리한다.

### kube-apiserver
- 쿠버네티스 API를 노출하는 프론트엔드.
- 모든 REST 요청을 처리하고 클러스터 상태를 etcd와 동기화.

### etcd
- 모든 클러스터 데이터를 저장하는 키-값 저장소.
- 고가용성과 백업 필수.

### kube-scheduler
- 새로 생성된 파드를 적절한 노드에 할당.

### kube-controller-manager
- 여러 컨트롤러들을 단일 프로세스로 실행.
- 주요 컨트롤러: 노드, 레플리케이션, 엔드포인트, 서비스 계정 등.

### cloud-controller-manager
- 클라우드 제공자 API와 상호작용.
- 노드, 라우트, 로드밸런서, 볼륨 관련 작업 수행.

---

## 노드 컴포넌트

각 워커 노드에서 실행되며 컨테이너를 관리한다.

### kubelet
- 각 파드가 정상적으로 실행되도록 보장.
- 컨테이너 런타임과 통신.

### kube-proxy
- 네트워크 프록시 및 로드밸런서 역할.
- 서비스와 파드 간 통신 라우팅.

### 컨테이너 런타임
- 실제 컨테이너를 실행하는 소프트웨어.
- 예: containerd, CRI-O, Docker 등.

---

## 애드온

쿠버네티스 기능을 확장하는 컴포넌트. 일반적으로 파드와 서비스로 동작함.

### DNS
- 클러스터 내 서비스 이름 해석을 위한 핵심 애드온.
- 모든 서비스와 파드는 DNS 이름을 가짐.

### 웹 UI (대시보드)
- 웹 기반 UI로 클러스터를 시각적으로 관리 가능.

### 컨테이너 리소스 모니터링
- 컨테이너 성능 및 리소스 사용량 측정.

### 클러스터 수준 로깅
- 클러스터 내 로그 수집 및 저장을 위한 시스템.
- 일반적으로 외부 저장소 사용 (예: Elasticsearch).
