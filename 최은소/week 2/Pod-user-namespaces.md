# [사용자 네임스페이스 (User Namespaces)](https://kubernetes.io/ko/docs/concepts/workloads/pods/user-namespaces/)

## 개요
사용자 네임스페이스는 컨테이너 내 사용자와 호스트의 사용자를 분리할 수 있는 기능으로, 보안을 강화하고 격리성을 높이기 위한 리눅스 커널 기능이다. 쿠버네티스에서는 `hostUsers: false` 설정을 통해 사용자 네임스페이스를 활성화할 수 있으며, 파드가 루트로 동작하더라도 호스트에서는 비루트 사용자로 매핑되어 제한된 권한만 가진다.

---

## 주요 특징

- **루트 권한 격리**: 컨테이너 내부에서는 루트로 실행되더라도, 실제 호스트에서는 권한이 없는 사용자로 실행됨.
- **보안 강화**: 높은 심각도의 취약점(CVE)으로부터 보호 가능.
- **네임스페이스 외부 권한 없음**: `CAP_SYS_MODULE`, `CAP_SYS_ADMIN` 같은 권한도 네임스페이스 외부에서는 무효화됨.
- **UID/GID 매핑**: 컨테이너 내 UID/GID는 호스트에서 다른 ID로 매핑되며, 파드 간 ID가 중복되지 않도록 보장됨.

---

## 설정 방법

```yaml
spec:
  hostUsers: false
```

---

## 제한 사항

- `hostNetwork`, `hostPID`, `hostIPC`: 사용 불가
- 지원되는 볼륨 타입: `configMap`, `secret`, `projected`, `downwardAPI`, `emptyDir`
- `.spec.securityContext.fsGroup` 설정 필요 (기본값: 0)
- 사용 가능한 UID/GID: 0~65535

---

## 호환성

- 리눅스 전용 기능
- 컨테이너 런타임 지원 필요 (예: CRI-O 지원됨, containerd는 1.7 이상에서 예정)

---

## 보안 권장 사항

- 호스트의 UID/GID 범위를 0~65535로 제한할 것
- 컨테이너 브레이크아웃 방지
- CVE-2021-25741 같은 취약점 방지에 도움
