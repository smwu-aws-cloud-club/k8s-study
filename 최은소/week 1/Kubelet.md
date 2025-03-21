# [Kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)

`kubelet`은 각 노드에서 실행되며, 쿠버네티스 노드 에이전트로 작동한다. `PodSpec`에 따라 컨테이너가 실행되고 정상 상태인지 감시한다.

---

## 핵심 기능

- `PodSpec` 기반으로 파드 실행
- API 서버 또는 파일/HTTP 엔드포인트로부터 `PodSpec` 수신 가능
- 쿠버네티스 외부에서 생성된 컨테이너는 관리하지 않음

---

## Manifest 제공 방식

- **파일 기반**: CLI 플래그로 경로 전달 (`--pod-manifest-path`)
- **HTTP 기반**: CLI 플래그로 URL 전달 (`--manifest-url`)
- 기본 감시 주기: 20초 (설정 가능)

---

## 주요 CLI 플래그

> 대부분의 플래그는 `--config` 설정 파일을 통해 설정하는 것을 권장함

- `--config`: kubelet 설정 파일 경로
- `--kubeconfig`: API 서버 접속용 kubeconfig 경로
- `--hostname-override`: 노드 이름 강제 지정
- `--container-runtime-endpoint`: 컨테이너 런타임 소켓 경로
- `--cert-dir`: TLS 인증서 저장 경로
- `--register-node`: 노드를 API 서버에 등록 (기본값: true)
- `--pod-manifest-path`: static pod 파일 경로
- `--fail-swap-on`: 스왑 사용시 kubelet 실행 중단 (기본값: true)

---

## 인증/인가 관련

- `--anonymous-auth`: 익명 인증 허용 여부 (기본값: true)
- `--authentication-token-webhook`: TokenReview API 사용
- `--authorization-mode`: `AlwaysAllow` 또는 `Webhook`

---

## 리소스 관련

- `--cgroup-driver`: cgroup 드라이버 (`cgroupfs`, `systemd`)
- `--cpu-manager-policy`: `none`, `static`
- `--eviction-hard`: 강제 제거 기준
- `--max-pods`: 최대 파드 수 (기본값: 110)
- `--pods-per-core`: 코어당 파드 수
- `--kube-reserved`, `--system-reserved`: 시스템 리소스 예약

---

## 보안 관련

- `--client-ca-file`: 클라이언트 인증용 CA 파일
- `--rotate-certificates`: 인증서 자동 갱신
- `--seccomp-default`: 기본 seccomp 프로파일 사용 여부

---

## 로깅 및 디버깅

- `--v=2`: 로깅 상세 레벨 설정
- `--log-flush-frequency`: 로그 플러시 주기
- `--enable-debugging-handlers`: 디버깅용 핸들러 활성화

---

## 기능 게이트 (Feature Gates)

- `--feature-gates="FeatureA=true,FeatureB=false"` 형식으로 활성화/비활성화
- 예: `GracefulNodeShutdown`, `DynamicResourceAllocation`, `SidecarContainers` 등
