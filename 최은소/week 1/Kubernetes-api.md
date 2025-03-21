# [쿠버네티스 API](https://kubernetes.io/ko/docs/concepts/overview/kubernetes-api/)

쿠버네티스 API는 클러스터 리소스를 질의(Query)하고 조작(Control)할 수 있는 HTTP 기반 인터페이스를 제공한다. 쿠버네티스의 컨트롤 플레인 중심에는 API 서버가 있으며, 이는 모든 사용자 및 내부/외부 컴포넌트와의 통신 허브 역할을 한다.

---

## 주요 개념

- **API 서버**: 모든 통신의 중심이며, HTTP API를 통해 쿠버네티스 오브젝트 상태를 조회/조작
- **지원 오브젝트**: Pod, Namespace, ConfigMap, Event 등
- **사용 방법**:
  - kubectl CLI 또는 kubeadm 같은 도구
  - 직접 REST 호출
  - 클라이언트 라이브러리 사용 권장

---

## OpenAPI 명세

### OpenAPI v2
- 엔드포인트: `/openapi/v2`
- 응답 형식: JSON 또는 Protobuf
- 내부 통신에 Protobuf 직렬화 사용 가능

### OpenAPI v3 (Beta, v1.24+)
- 엔드포인트: `/openapi/v3`
- 제공 형식: JSON
- 상대 URL + 해시 기반으로 불변 데이터 제공
- HTTP 캐싱 활용 (Expires 1년, Cache-Control: immutable)

---

## 데이터 저장

- 쿠버네티스 오브젝트의 직렬화된 상태는 `etcd`에 저장됨

---

## API 그룹과 버전

- API 경로 예시:
  - Core: `/api/v1`
  - Group: `/apis/rbac.authorization.k8s.io/v1alpha1`
- API는 **그룹 / 리소스 / 네임스페이스 / 이름** 조합으로 식별됨
- API 버전 간 변환은 API 서버가 자동 처리

#### 예시:
- `v1beta1`로 생성된 리소스를 `v1`으로도 접근 가능
- `v1beta1`이 deprecated 되더라도 `v1`으로 연속 접근 가능

---

## API 변경 정책

- 새로운 필드/리소스는 자유롭게 추가 가능
- 삭제 시에는 **API Deprecation Policy** 따름
- `v1`(GA)에 도달한 API는 호환성 유지가 강력하게 보장됨
- 베타 API 사용 시 안정화된 버전으로 마이그레이션 권장

---

## API 확장 방법

1. **Custom Resource (CRD)**  
   - 선언적으로 새로운 API 리소스 생성 가능

2. **Aggregation Layer**  
   - API 서버를 통해 외부 API를 통합 확장
