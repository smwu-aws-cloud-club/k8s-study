# Kubernetes Service

쿠버네티스의 Service는 클러스터 내 파드 집합을 외부에 노출하는 네트워크 추상화 개념으로, 파드의 동적인 생성/삭제에도 안정적인 접속 지점을 제공한다.

---

## 1. 개념

- **Service란?** 파드 집합과 접근 정책을 정의하는 추상 객체.
- 파드는 비영구적이며 IP가 변경되므로, 지속적인 접근을 위해 서비스가 필요하다.
- 쿠버네티스는 파드에 고유 IP와 DNS 이름을 부여하며, 로드 밸런싱을 수행한다.

---

## 2. Service 정의 및 구성

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

- `selector`: 파드 선택 레이블.
- `port`: 서비스가 노출하는 포트.
- `targetPort`: 파드 내부 포트.

### 포트 이름 사용 예시

```yaml
ports:
  - port: 80
    targetPort: http-web-svc
```

### 다중 포트 서비스

```yaml
ports:
  - name: http
    port: 80
    targetPort: 9376
  - name: https
    port: 443
    targetPort: 9377
```

---

## 3. 셀렉터 없는 서비스

- 클러스터 외부 시스템을 참조하거나 수동으로 엔드포인트슬라이스를 설정할 경우 사용.
- 엔드포인트를 수동으로 명시한다.

```yaml
kind: EndpointSlice
metadata:
  name: my-service-1
  labels:
    kubernetes.io/service-name: my-service
ports:
  - protocol: TCP
    port: 9376
endpoints:
  - addresses: ["10.4.5.6", "10.1.2.3"]
```

---

## 4. 서비스 검색 방식

- **환경 변수**: 파드 내에서 자동으로 주입되는 `*_SERVICE_HOST`, `*_SERVICE_PORT`.
- **DNS**: CoreDNS가 `my-service.my-namespace.svc.cluster.local` 형태로 name resolution 제공.

---

## 5. 서비스 유형 (ServiceTypes)

| Type          | 설명 |
|---------------|------|
| ClusterIP     | 기본값, 클러스터 내부 노출 |
| NodePort      | 모든 노드에서 특정 포트로 접근 가능 |
| LoadBalancer  | 외부 클라우드 로드밸런서 노출 |
| ExternalName  | DNS 이름으로 리디렉션 |

---

## 6. Headless 서비스

- `.spec.clusterIP: None` 설정
- DNS는 각 파드 IP로 직접 해석됨
- 셀렉터 유무에 따라 동작 다름

---

## 7. AWS 예시

- 로드밸런서 SSL/TLS:
  - `service.beta.kubernetes.io/aws-load-balancer-ssl-cert`
  - `service.beta.kubernetes.io/aws-load-balancer-ssl-ports`
- 접근 로그 설정:
  - `access-log-enabled`, `emit-interval`, `s3-bucket-name`, `s3-bucket-prefix`
- 헬스체크 및 태그 지정, 보안 그룹 설정 등 다양한 어노테이션 제공

---

## 8. ExternalName

```yaml
spec:
  type: ExternalName
  externalName: my.database.example.com
```

- DNS 레벨에서 CNAME 매핑 수행
- 프록시나 포워딩 없이 동작

---

## 9. 외부 IP

- `.spec.externalIPs` 필드로 외부 IP로 접근 가능
- 클러스터 관리자가 IP 관리를 책임짐

---

## 10. 기타

- **세션 어피니티**: 클라이언트 IP 기반 고정 연결 지원
- **가상 IP 메커니즘**: 쿠버네티스는 IPVS, iptables 등을 통해 서비스 VIP 제공
