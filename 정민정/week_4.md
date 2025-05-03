## 1. Service (서비스)

### 1.1 서비스
Kubernetes에서 **Pod들의 집합에 네트워크 접점을 제공하는 추상 객체**이다. Pod는 언제든지 IP 주소가 바뀔 수 있기 때문에, 안정적인 네트워크 액세스를 보장하기 위해 서비스가 필요하다.

서비스는 내부 혹은 외부의 클라이언트가 **Pod에 안정적으로 접근할 수 있는 방식**을 제공한다.

### 1.2 서비스가 필요한 이유
- Pod는 생성과 삭제가 빈번하므로 IP가 바뀌기 쉽다.
- 애플리케이션은 고정된 주소를 통해 백엔드를 호출해야 한다.
- Pod를 직접 바라보지 않고, **Service를 통해 로드 밸런싱된 접근**이 가능하다.

### 1.3 서비스 작동 원리
- Selector를 통해 **label이 지정된 Pod들**을 선택
- 선택된 Pod에 대해 고정 IP(DNS)와 포트를 노출함
- kube-proxy가 iptables 또는 IPVS를 사용해 트래픽을 라우팅함

### 1.4 서비스 유형

#### ClusterIP (기본값)
- 클러스터 내부에서만 접근 가능한 가상 IP 생성
- 외부에서는 직접 접근할 수 없음

#### NodePort
- 클러스터 외부에서 접근할 수 있도록 각 노드에 고정 포트를 개방
- 외부에서 `<NodeIP>:<NodePort>`로 접근 가능

#### LoadBalancer
- 클라우드 환경에서 **외부 로드 밸런서**를 생성해 서비스 노출
- GCP, AWS, Azure와 같은 퍼블릭 클라우드에서 사용됨

#### ExternalName
- 외부 DNS 이름(CNAME)을 서비스 이름으로 매핑
- 실제로는 외부 서비스를 참조함 (e.g., 외부 데이터베이스)

### 1.5 서비스 예시

```
yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  type: ClusterIP
```

### 1.6 서비스 검색 (Service Discovery)
- CoreDNS를 통해 자동으로 서비스 이름으로 접근 가능
- 형식: `<서비스명>.<네임스페이스>.svc.cluster.local`
- 예: `my-service.default.svc.cluster.local`

---

## 2. Ingress (인그레스)

### 2.1 인그레스
Ingress는 **HTTP(S) 트래픽을 클러스터 외부에서 내부 서비스로 라우팅하기 위한 규칙 집합**이다.

Ingress는 클러스터 외부에서 특정 도메인 및 경로로 들어온 요청을 내부의 서비스로 연결해주는 역할을 한다.

### 2.2 인그레스 구성 요소
- **Ingress 객체**: 라우팅 규칙이 정의된 리소스
- **Ingress Controller**: 실제로 네트워크 트래픽을 처리하고 라우팅하는 컴포넌트 (예: nginx, traefik 등)

### 2.3 인그레스의 장점
- 여러 서비스를 단일 도메인 아래 경로로 구분 가능
- SSL 종료 지점으로 사용 가능
- 다양한 인증/인가 플러그인과 통합 가능

### 2.4 인그레스 예시
```
yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: foo-service
            port:
              number: 8080
```

### 2.5 인그레스 동작 방식
- DNS 요청이 ingress controller가 설치된 Node로 전달됨
- Controller는 정의된 규칙에 따라 특정 서비스로 트래픽을 포워딩함
- 내부적으로는 NodePort 또는 LoadBalancer로 접근할 수 있음

## 3. 정리
- Service는 Kubernetes에서 Pod를 안정적으로 접근하기 위한 핵심 네트워크 기능이다.
- Ingress는 HTTP(S) 기반의 트래픽 라우팅을 위한 고수준 기능이다.
- Minikube에서는 Ingress 실습이 가능하며, 실제 배포 환경에서는 다양한 Controller와 함께 사용된다.
- Ingress와 Service는 **서로 보완적인 관계**이며, 다양한 네트워크 구조 설계에 사용된다.