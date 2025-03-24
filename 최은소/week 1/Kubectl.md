# [kubectl 명령줄 도구](https://kubernetes.io/ko/docs/reference/kubectl/)

`kubectl`은 쿠버네티스 API와 통신하여 클러스터를 관리하는 커맨드라인 툴이다.

## 구성
- 기본 config 파일 경로: `$HOME/.kube/config`
- 대체 방법:
  - `KUBECONFIG` 환경 변수
  - `--kubeconfig` 플래그

## 기본 구문
```
kubectl [command] [TYPE] [NAME] [flags]
```
- **command**: 실행할 작업 (e.g. `get`, `create`, `delete`)
- **TYPE**: 리소스 타입 (`pod`, `pods`, `po` 등)
- **NAME**: 리소스 이름 (생략 시 전체 대상)
- **flags**: 옵션 지정 (`--namespace`, `-o yaml` 등)

## 리소스 지정 방법
- 단일 타입 여러 리소스: `kubectl get pod pod1 pod2`
- 여러 타입 혼합: `kubectl get pod/pod1 svc/svc1`
- 파일에서 지정: `kubectl get -f pod.yaml`

## 네임스페이스 동작
- 클러스터 내 실행 시:
  - `POD_NAMESPACE` 환경 변수로 기본 네임스페이스 오버라이드 가능
- 클러스터 외 실행 시:
  - 현재 컨텍스트의 네임스페이스 사용
- 변경 명령어:
  ```bash
  kubectl config set-context --current --namespace=<namespace>
  ```

## 주요 명령어 요약

| 명령어 | 설명 |
|--------|------|
| `get` | 리소스 조회 |
| `describe` | 리소스 상세 정보 |
| `apply -f` | 리소스 적용/업데이트 |
| `delete` | 리소스 삭제 |
| `exec` | 파드 내 명령 실행 |
| `logs` | 파드 로그 출력 |
| `diff` | 적용 전 리소스 차이 비교 |
| `plugin` | 플러그인 등록/실행 |

## 출력 형식

| 플래그 | 설명 |
|--------|------|
| `-o yaml` | YAML 출력 |
| `-o json` | JSON 출력 |
| `-o wide` | 추가 정보 포함 |
| `-o name` | 리소스 이름만 출력 |
| `--sort-by` | jsonpath 기준으로 정렬 |
| `--server-print=false` | 서버 측 열 출력 비활성화 |

## 예제

```bash
# 파드 목록 출력
kubectl get pods

# 특정 파드 상세 정보
kubectl describe pod web-pod

# yaml 포맷으로 파드 정보 출력
kubectl get pod web-pod -o yaml

# 레이블 기준 리소스 삭제
kubectl delete pods -l app=myapp

# 파드에서 bash 실행
kubectl exec -ti my-pod -- /bin/bash
```

## 플러그인

- 이름: `kubectl-<plugin-name>`
- 위치: `$PATH` 내 실행 가능 위치
- 등록: `chmod +x`, `mv`로 PATH로 이동
- 리스트 확인:
  ```bash
  kubectl plugin list
  ```