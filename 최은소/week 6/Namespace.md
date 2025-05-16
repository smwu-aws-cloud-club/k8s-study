# Kubernetes 네임스페이스 (Namespace)

쿠버네티스에서 **네임스페이스(Namespace)** 는 단일 클러스터 내에서 리소스를 격리하는 메커니즘을 제공한다.

---

## 1. 개요

- 리소스의 이름은 네임스페이스 내에서 유일하면 충분하다.
- **네임스페이스 기반 스코핑**은 네임스페이스 기반 오브젝트(예: Deployment, Service 등)에만 적용된다.
- 클러스터 범위 오브젝트(예: Node, PersistentVolume 등)에는 적용되지 않는다.

---

## 2. 사용 시점

- **필요한 경우만** 사용. 사용자 수가 적은 환경에서는 사용하지 않아도 된다.
- 여러 팀/프로젝트가 있는 환경에서 유용하다.
- 동일 네임스페이스 내 리소스 구분은 레이블을 사용.

---

## 3. 초기 네임스페이스

| 이름              | 설명 |
|------------------|------|
| `default`        | 기본 네임스페이스. 네임스페이스 없이 생성되는 리소스의 위치. |
| `kube-node-lease`| 노드 하트비트용 lease 객체 위치. |
| `kube-public`    | 인증되지 않은 클라이언트도 읽기 가능. 전체 클러스터에서 공개 리소스용. |
| `kube-system`    | 쿠버네티스 시스템 리소스 위치. |

> 참고: `kube-` 접두사 네임스페이스는 시스템용으로 예약됨

---

## 4. 네임스페이스 관리

### 생성/삭제

- 자세한 내용은 "네임스페이스 관리자 가이드" 참고.

### 조회

```bash
kubectl get namespace
```

### 네임스페이스 지정 요청

```bash
kubectl get pods --namespace=<namespace-name>
kubectl run nginx --image=nginx --namespace=<namespace-name>
```

### 기본 네임스페이스 설정

```bash
kubectl config set-context --current --namespace=<namespace-name>
kubectl config view --minify | grep namespace:
```

---

## 5. 네임스페이스와 DNS

- 서비스 DNS: `<svc-name>.<namespace>.svc.cluster.local`
- 같은 네임스페이스 내에서는 `<svc-name>`으로도 접근 가능
- 다른 네임스페이스 접근 시 FQDN 사용 필요

> **주의**: 공개 TLD와 동일한 네임스페이스 이름은 DNS 충돌 가능

---

## 6. 네임스페이스에 속하지 않는 리소스

- 예: `Node`, `PersistentVolume`, `Namespace` 자체 등
- 조회 방법:

```bash
kubectl api-resources --namespaced=true   # 네임스페이스에 속하는 리소스
kubectl api-resources --namespaced=false  # 네임스페이스에 속하지 않는 리소스
```

---

## 7. 자동 레이블링 (Kubernetes 1.21+)

- **기능 게이트**: `NamespaceDefaultLabelName` 활성화 시
- 모든 네임스페이스에 `kubernetes.io/metadata.name` 레이블 자동 부여됨 (변경 불가)

---

## 참고

- `default` 네임스페이스 대신 팀별 네임스페이스 권장
- 공개 도메인과 충돌하지 않도록 네임스페이스 이름 주의
