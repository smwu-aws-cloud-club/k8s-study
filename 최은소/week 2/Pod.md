# [파드(Pod)](https://kubernetes.io/ko/docs/concepts/workloads/pods/)

## 개요
- **파드(Pod)**는 쿠버네티스에서 생성 및 관리할 수 있는 가장 작은 배포 단위.
- 하나 이상의 **컨테이너**로 구성되며, **스토리지 및 네트워크**를 공유함.
- 애플리케이션의 "논리 호스트" 역할을 수행.

---

## 파드 구조 및 사용
- **공유 콘텍스트**: 네임스페이스, cgroup, 공유 볼륨 등.
- **단일 컨테이너 파드**: 가장 일반적인 유형.
- **다중 컨테이너 파드**: 밀접하게 결합된 작업을 수행할 경우에 사용.
- **초기화 컨테이너**, **임시 컨테이너**를 포함할 수 있음.

---

## 파드 예시
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

---

## 파드 생성
```bash
kubectl apply -f https://k8s.io/examples/pods/simple-pod.yaml
```

---

## 파드와 워크로드 리소스
- **Deployment**, **Job**, **StatefulSet** 등으로 파드 자동 생성 및 관리.
- **파드 템플릿(PodTemplate)**을 기반으로 생성됨.
- 템플릿 변경 시 기존 파드를 갱신하지 않고 **새 파드 생성**.

---

## 파드 스케줄링과 OS
- `.spec.os.name` 필드를 통해 OS 지정 가능 (`linux` 또는 `windows`)
- kubelet은 해당 노드의 OS와 일치하지 않으면 실행을 거부함

---

## 파드 내 리소스 공유
- **공유 스토리지**: 모든 컨테이너가 접근 가능한 볼륨
- **공유 네트워크**: 동일한 IP 및 포트 공간 공유
- 컨테이너 간 **localhost 통신 가능**

---

## 특권 컨테이너
- 리눅스의 `privileged` 플래그로 특권 모드 설정 가능
- 윈도우는 HostProcess 설정을 통해 사용 가능

---

## 정적 파드 (Static Pods)
- kubelet이 직접 관리하는 파드
- API 서버에 **미러 파드**로 표시됨
- **컨트롤 플레인 구성요소 실행**에 적합

---

## 컨테이너 프로브
- kubelet이 컨테이너 상태 점검
  - `ExecAction`, `TCPSocketAction`, `HTTPGetAction` 등 사용
- 자세한 정보는 [파드 라이프사이클 문서](/week%202/Pod-lifecycle.md) 참조
