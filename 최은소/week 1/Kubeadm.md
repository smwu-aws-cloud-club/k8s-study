# [Kubeadm 요약](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

`kubeadm`은 쿠버네티스 클러스터 생성을 위한 빠른 경로를 제공하는 도구로, `kubeadm init`과 `kubeadm join` 명령어를 중심으로 작동한다.

---

## 역할 및 목적

- 최소 실행 가능한 클러스터 부트스트랩 제공
- **시스템 프로비저닝은 포함하지 않음**
- 애드온 설치(모니터링, 대시보드 등)는 kubeadm 범위 밖

---

## 사전 요구사항

- Linux 머신 (Debian, RedHat 등)
- **최소 사양**: 2GB RAM, 2 CPU
- 모든 노드 간 네트워크 연결
- 고유한 호스트 이름, MAC 주소, `product_uuid`
- 스왑 비활성화 필수
- 필수 포트 개방 필요

---

## 컨테이너 런타임

- 쿠버네티스는 CRI(컨테이너 런타임 인터페이스)를 통해 런타임과 통신
- 자동 감지 가능, 충돌 시 명시 필요
- 주요 런타임과 소켓:
  - `containerd`: `/var/run/containerd/containerd.sock`
  - `CRI-O`: `/var/run/crio/crio.sock`
  - `cri-dockerd`: `/var/run/cri-dockerd.sock` (도커 호환)

> 도커는 기본적으로 CRI 미지원 → `cri-dockerd` 필요

---

## 설치 구성 요소

| 컴포넌트 | 설명 |
|----------|------|
| `kubeadm` | 클러스터 부트스트랩 |
| `kubelet` | 파드 및 컨테이너 관리 |
| `kubectl` | 커맨드라인 관리 도구 |

- 버전 일치 필수 (`kubeadm`, `kubelet`, `kubectl`)
- `kubelet`은 `API 서버`보다 높은 버전이면 안 됨

---

## 설치 (Debian/Ubuntu 기준)

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg \
  https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] \
  https://apt.kubernetes.io/ kubernetes-xenial main" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

> `/etc/apt/keyrings` 디렉터리가 없다면 수동 생성 필요

---

## 주의 사항

- 시스템 자동 업그레이드 시 Kubernetes 패키지 제외 필요
- 버전 차이 정책 확인 필수
- `kubeadm`은 `kubelet`, `kubectl`을 관리하지 않음

---

## 향후 사용 기대

- 고수준 배포 도구가 `kubeadm` 기반으로 구축되는 것이 이상적
- 모든 배포의 기반 도구로 표준화 가능성