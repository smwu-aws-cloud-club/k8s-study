# [중단(Disruption)](https://kubernetes.io/ko/docs/concepts/workloads/pods/disruptions/)

## 1. 중단의 종류

### 자발적 중단
- 사람이 의도적으로 발생시키는 중단
- 예시:
  - 파드 템플릿 업데이트로 인한 재시작
  - 디플로이먼트 삭제
  - 노드 드레이닝 (관리자의 업그레이드 작업 등)

### 비자발적 중단
- 시스템 또는 인프라 오류로 발생
- 예시:
  - 노드의 하드웨어 오류
  - VM 장애
  - 커널 패닉
  - 네트워크 파티션
  - 리소스 부족으로 인한 파드 축출

---

## 2. 중단 처리 전략

- 파드 리소스를 명시적으로 요청
- 애플리케이션 복제
- 영역 간/랙 간 분산 배치 (안티-어피니티)
- 고가용성 클러스터 구성

---

## 3. Pod Disruption Budget (PDB)

### 개념
- 자발적 중단 시 유지해야 할 최소 파드 수 정의
- Eviction API를 통해 사용
- 디플로이먼트, 스테이트풀셋 등과 함께 사용

### 예시
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

---

## 4. PDB 동작 예시

- 클러스터 노드가 3개, 파드가 3개 (pod-a, pod-b, pod-c)
- PDB: 최소 2개 파드 유지
- 하나의 노드를 드레인하면 pod-a 제거되고, pod-d 생성됨
- pod-d가 준비되기 전까지 다른 노드 드레인 불가
- 새 노드를 추가해야 다음 드레인이 가능

---

## 5. Pod 중단 조건 (v1.26+)

### 조건 종류
- `PreemptionByKubeScheduler`: 높은 우선순위 파드에 의해 선점됨
- `DeletionByTaintManager`: 테인트로 인해 삭제됨
- `EvictionByEvictionAPI`: API에 의해 축출
- `DeletionByPodGC`: 존재하지 않는 노드로 인해 삭제 예정
- `TerminationByKubelet`: 노드 압박 또는 셧다운으로 종료

---

## 6. 역할 분리

- 애플리케이션 소유자: PDB 설정
- 클러스터 관리자: Eviction API 사용
- PDB는 역할 분리를 위한 인터페이스 제공

---

## 7. 클러스터 관리자 작업

### 업그레이드 전략
- 다운타임 허용
- 장애 조치(failover) 클러스터 운영
- 무중단 전환 (전환 비용 존재)
- **PDB로 중단 견디는 애플리케이션 작성**
