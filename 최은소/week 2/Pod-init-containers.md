# [초기화 컨테이너 (Init Containers)](https://kubernetes.io/ko/docs/concepts/workloads/pods/init-containers/)

초기화 컨테이너는 파드의 **앱 컨테이너들이 실행되기 전**에 실행되는 특수한 컨테이너이다. 주로 앱 이미지에는 없는 유틸리티, 설정 스크립트를 실행하는 데 사용된다.

---

## 주요 특징

- 파드의 `.spec.initContainers` 필드에 명시
- **순차 실행**: 다음 초기화 컨테이너는 이전 컨테이너가 성공적으로 완료된 후 실행됨
- **앱 컨테이너가 실행되기 전까지** 반드시 완료되어야 함
- 실패 시 `restartPolicy`에 따라 재시도

---

## 일반 컨테이너와의 차이점

| 항목 | 일반 컨테이너 | 초기화 컨테이너 |
|------|----------------|------------------|
| 목적 | 앱 실행 | 사전 작업 수행 |
| 실행 시점 | 파드 시작 후 병렬 실행 가능 | 앱 컨테이너 실행 전 순차 실행 |
| 프로브 지원 | O | X (`livenessProbe`, `readinessProbe`, `startupProbe` 미지원) |
| 완료 조건 | 항상 실행 | 완료 필요 (종료 코드 0) |
| 리소스 처리 | 앱 컨테이너 리소스 합산 기준 | 가장 큰 값 기준 |

---

## 장점

- 앱 컨테이너와 별개로 유틸리티 또는 셋업 스크립트 실행 가능
- 보안성 분리: 민감한 작업은 앱 이미지와 분리된 초기화 컨테이너에서 수행
- 명확한 선행 조건 충족 시점 확보
- 앱 컨테이너 시작 지연 가능

---

## 예시

```yaml
initContainers:
- name: init-myservice
  image: busybox:1.28
  command: ['sh', '-c', 'until nslookup myservice.namespace.svc.cluster.local; do echo waiting; sleep 2; done']
- name: init-mydb
  image: busybox:1.28
  command: ['sh', '-c', 'until nslookup mydb.namespace.svc.cluster.local; do echo waiting; sleep 2; done']
```

---

## 상태 확인

```bash
kubectl logs myapp-pod -c init-myservice
kubectl logs myapp-pod -c init-mydb
```

---

## 리소스 계산

- 초기화 컨테이너 리소스는 앱 컨테이너보다 **우선 계산**됨
- 스케줄링은 초기화 컨테이너 포함 리소스 요청 기준
- 파드의 QoS 등급은 앱 + 초기화 컨테이너 기준

---

## 주의 사항

- 이름 중복 불가 (앱 컨테이너와 초기화 컨테이너 이름은 모두 고유해야 함)
- 멱등성 필요: 초기화 컨테이너는 재시작될 수 있음
- `activeDeadlineSeconds` 설정 시 초기화 컨테이너도 영향 받음
- `.status.initContainerStatuses`로 상태 확인 가능

---

## 언제 사용하나?

- 앱 실행 전에 외부 서비스 확인 필요할 때
- ConfigMap/Secret 기반 설정 생성
- 초기 데이터 세팅 또는 레지스트리 등록 시
- 앱 시작 지연, 백엔드 의존 조건 충족 등