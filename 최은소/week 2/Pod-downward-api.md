# [다운워드(Downward) API](https://kubernetes.io/ko/docs/concepts/workloads/pods/downward-api/)

다운워드 API는 **실행 중인 컨테이너에 파드 및 컨테이너 정보를 노출**하는 두 가지 방법을 제공한다:

- 환경 변수 (Environment Variable)
- 볼륨 파일 (Volume File)

이를 통해 **쿠버네티스 API 또는 클라이언트에 의존하지 않고도 컨테이너가 자신에 대한 정보를 알 수 있게 한다.**

---

## 목적

- 쿠버네티스에 종속되지 않으면서도 컨테이너가 자신 및 클러스터에 대한 정보 접근 가능
- 애플리케이션의 결합도를 낮추고 유지보수를 용이하게 함

---

## 노출 방식

### 1. 환경 변수
컨테이너 시작 시 설정된 값으로 사용됨.

### 2. Downward API 볼륨
파일로 마운트되어 실행 중에도 값을 읽을 수 있음.

---

## fieldRef를 통해 접근 가능한 정보 (파드 필드)

| 필드 | 설명 |
|------|------|
| `metadata.name` | 파드 이름 |
| `metadata.namespace` | 파드 네임스페이스 |
| `metadata.uid` | 파드 UID |
| `metadata.annotations['<KEY>']` | 특정 어노테이션 값 |
| `metadata.labels['<KEY>']` | 특정 레이블 값 |
| `spec.serviceAccountName` | 서비스 어카운트 이름 |
| `spec.nodeName` | 노드 이름 |
| `status.hostIP` | 노드의 IP 주소 |
| `status.podIP` | 파드의 IP 주소 |

📁 *Downward API 볼륨에서만 접근 가능:*

- `metadata.labels` (모든 레이블 key=value 형식)
- `metadata.annotations` (모든 어노테이션 key=value 형식)

---

## resourceFieldRef를 통해 접근 가능한 정보 (컨테이너 리소스)

| 리소스 키 | 설명 |
|-----------|------|
| `limits.cpu` | CPU 제한 |
| `requests.cpu` | CPU 요청 |
| `limits.memory` | 메모리 제한 |
| `requests.memory` | 메모리 요청 |
| `limits.ephemeral-storage` | 임시 스토리지 제한 |
| `requests.ephemeral-storage` | 임시 스토리지 요청 |
| `limits.hugepages-*` | HugePages 제한 (기능 게이트 필요) |
| `requests.hugepages-*` | HugePages 요청 (기능 게이트 필요) |

---

## 주의 사항

- 리소스 제한이 설정되지 않으면, 노드의 최대 할당량을 기준으로 kubelet이 기본값을 제공함
