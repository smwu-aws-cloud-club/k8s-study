# [파드 라이프사이클](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/)

## 파드의 생명주기 개요
파드는 `Pending` 단계에서 시작하여 컨테이너가 실행되면 `Running`, 성공적으로 종료되면 `Succeeded`, 실패하면 `Failed`로 전환된다. 노드와의 통신 문제로 상태를 알 수 없으면 `Unknown`으로 표시된다.

---

## 파드의 상태 단계 (`phase`)
| 단계 | 설명 |
|------|------|
| Pending | 승인되었지만 아직 실행 준비가 되지 않은 상태. |
| Running | 노드에 바인딩되고 컨테이너가 실행 중. |
| Succeeded | 모든 컨테이너가 성공적으로 종료됨. |
| Failed | 하나 이상의 컨테이너가 실패로 종료됨. |
| Unknown | 상태를 알 수 없는 경우. |

---

## 컨테이너 상태
컨테이너는 `Waiting`, `Running`, `Terminated` 상태를 가질 수 있다. `kubectl describe pod` 명령어를 통해 확인 가능.

---

## 재시작 정책
`restartPolicy`는 `Always`, `OnFailure`, `Never` 중 하나이며 기본값은 `Always`.

---

## 파드 컨디션 (`PodConditions`)
- `PodScheduled`
- `PodHasNetwork` (알파 기능)
- `ContainersReady`
- `Initialized`
- `Ready`

---

## 준비성 게이트 (`Readiness Gates`)
파드 상태에 사용자 정의 준비 조건을 추가할 수 있으며, 모든 조건이 `True`여야 `Ready` 상태로 간주된다.

---

## 프로브 종류
- `livenessProbe`: 컨테이너가 동작 중인지 확인
- `readinessProbe`: 요청을 받을 준비가 되었는지 확인
- `startupProbe`: 애플리케이션이 시작되었는지 확인

---

## 프로브 메커니즘
- `exec`: 명령어 실행
- `httpGet`: HTTP 요청
- `tcpSocket`: TCP 연결
- `grpc`: gRPC 헬스체크 (알파)

---

## 파드 종료
파드를 삭제하면 TERM → KILL 시그널이 전송되며, `terminationGracePeriodSeconds` 동안 graceful shutdown을 시도한다.

---

## 강제 종료
`--force --grace-period=0` 플래그를 사용해 즉시 삭제 가능. 이는 비정상 종료로 이어질 수 있음.

---

## 파드 가비지 콜렉션
`PodGC`는 임계값을 초과한 종료된 파드를 정리하거나, 고아 파드 등을 자동 제거한다.
