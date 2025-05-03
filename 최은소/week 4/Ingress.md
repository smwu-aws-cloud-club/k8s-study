# Kubernetes Ingress

## 개요
- **Ingress**는 클러스터 외부에서 내부 서비스로의 HTTP(S) 트래픽을 관리하는 Kubernetes API 오브젝트입니다.
- URI, 호스트 이름, 경로 등을 기반으로 트래픽을 백엔드 서비스로 라우팅할 수 있습니다.
- SSL 종료, 이름 기반 가상 호스팅, 로드 밸런싱을 지원합니다.

## 구성 요소 정의
- **Node**: 클러스터의 워커 노드
- **Cluster**: Kubernetes에 의해 관리되는 노드 집합
- **Edge Router**: 클러스터 경계에서 방화벽 정책 적용
- **Cluster Network**: 노드 간 네트워크
- **Service**: 파드 집합을 추상화한 네트워크 서비스

## 인그레스 리소스 구조
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /test
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 80
```

## 주요 기능
- **DefaultBackend**: 지정된 경로가 없을 경우 요청 처리
- **Resource Backend**: 외부 저장소와 같은 리소스를 백엔드로 사용
- **PathType**
  - `Prefix`: 접두사 기준 경로 매칭
  - `Exact`: 정확한 경로 매칭
  - `ImplementationSpecific`: 컨트롤러 구현에 따라 결정

## 경로 일치 예시
| 유형 | 경로 | 요청 경로 | 일치 여부 |
|------|------|-----------|-----------|
| Prefix | `/foo` | `/foo/bar` | ✅ |
| Exact | `/foo` | `/foo/` | ❌ |

## 호스트 네임 와일드카드
- `*.foo.com`은 `bar.foo.com`에는 일치하지만 `baz.bar.foo.com`에는 일치하지 않음

## TLS 구성 예시
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

## 인그레스 클래스
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

## 인그레스 유형
- **단일 백엔드**
- **팬아웃(Fanout)**: 하나의 IP에서 여러 경로로 서비스 분기
- **이름 기반 가상 호스팅**
- **TLS 지원 인그레스**
- **로드 밸런싱 지원**

## 참고
- 인그레스는 인그레스 컨트롤러가 있어야 동작
- 다양한 컨트롤러별 설정 및 어노테이션 존재
- 로드 밸런서 및 health check는 컨트롤러에 따라 다르게 작동

## 대안
- `Service.type: LoadBalancer`
- `Service.type: NodePort`
