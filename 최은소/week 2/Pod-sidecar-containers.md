# [Sidecar Container](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)

## 개요
사이드카 컨테이너는 메인 애플리케이션 컨테이너와 동일한 파드 내에서 실행되는 보조 컨테이너이다. 로깅, 모니터링, 보안, 데이터 동기화 등의 기능을 제공하며, 메인 애플리케이션 코드를 변경하지 않고도 기능을 확장하거나 지원할 수 있다.

## 주요 특징
- **Kubernetes v1.29부터 베타 기능으로 활성화됨**
- `initContainers` 필드에 정의되며 `restartPolicy: Always`로 설정하여 전체 파드 수명 동안 실행 가능
- 메인 컨테이너와 독립적으로 시작, 중지, 재시작 가능
- 같은 네트워크 및 저장소 네임스페이스를 공유
- 종료 시에는 메인 컨테이너 종료 후 반대 순서로 종료됨

## 동작 방식
- 사이드카 컨테이너는 일반 `init` 컨테이너와는 다르게 파드 수명 내내 실행됨
- 파드 종료 시, 메인 컨테이너가 먼저 종료된 후 사이드카 컨테이너가 종료됨
- `readinessProbe`를 통해 파드의 준비 상태에 영향을 줄 수 있음
- 이미지 변경 시 파드는 재시작되지 않지만 해당 컨테이너는 재시작됨

## Job에서의 사이드카 사용
- Job 리소스에서도 사이드카를 사용할 수 있음
- 메인 컨테이너 종료 후에도 사이드카가 Job 완료를 방해하지 않음

## 리소스 및 QoS
- 파드의 유효 요청/제한값은 사이드카, 앱, 초기화 컨테이너 중 가장 높은 값을 기준으로 계산됨
- 스케줄링 및 리소스 할당은 이 유효 값에 기반함
- QoS 계층도 모든 컨테이너(초기화, 앱, 사이드카 포함)의 설정을 반영하여 결정됨

## init 컨테이너와의 차이
| 항목 | Init 컨테이너 | Sidecar 컨테이너 |
|------|----------------|------------------|
| 실행 시점 | 앱 컨테이너 이전 | 앱 컨테이너와 병렬 |
| 수명 | 앱 실행 전 종료됨 | 파드 종료 시까지 실행됨 |
| Probe 지원 | 미지원 | 지원됨 |
| 통신 가능성 | 앱과 직접 통신 불가 | 앱과 직접 통신 가능 |

## 종료 시 주의 사항
- 사이드카는 종료 시 SIGTERM → SIGKILL 순으로 시그널을 받지만, 종료 준비 시간이 부족할 수 있음
- 종료 코드가 0이 아니더라도 무시해도 무방함

## 예시
```yaml
initContainers:
  - name: logshipper
    image: alpine:latest
    restartPolicy: Always
    command: ['sh', '-c', 'tail -F /opt/logs.txt']
    volumeMounts:
      - name: data
        mountPath: /opt
```

```yaml
containers:
  - name: myapp
    image: alpine:latest
    command: ['sh', '-c', 'echo "logging" >> /opt/logs.txt; sleep 1']
    volumeMounts:
      - name: data
        mountPath: /opt
```
