# Kubernetes 리소스 관리 정리

## 개요

- 쿠버네티스는 애플리케이션의 **배포, 스케일링, 업데이트, 구성, 삭제** 등을 돕는 다양한 도구를 제공한다.
- 구성 파일과 레이블은 리소스 관리를 위한 핵심 도구이다.

---

## 리소스 구성 구성하기

- 여러 리소스를 하나의 파일에 그룹화해 배포할 수 있음 (`---` 구분자 사용).
- 예시: 서비스 + 디플로이먼트 조합 (nginx-app.yaml)

```bash
kubectl apply -f nginx-app.yaml
```

- 파일 내 리소스는 선언된 순서대로 생성되므로, **Service → Deployment** 순으로 배치 권장
- 다중 `-f`, 디렉토리, URL 지원
- `--recursive` 옵션으로 서브 디렉토리도 재귀적으로 처리 가능

---

## kubectl 대량 작업 예시

```bash
kubectl apply -f dir/ --recursive
kubectl delete -f nginx-app.yaml
kubectl delete deployment,services -l app=nginx
```

---

## 효과적인 레이블 사용

- 여러 개의 레이블을 통해 리소스 필터링 및 분류 가능
- 예시: `app`, `tier`, `role`, `track` 등의 조합

```bash
kubectl get pods -Lapp -Ltier -Lrole
kubectl get pods -l app=guestbook,role=slave
```

---

## 카나리 디플로이먼트

- 동일한 애플리케이션의 **stable**과 **canary** 릴리스를 track 레이블로 구분
- 트래픽 비율 조절 → 검증 완료 후 stable로 승격 가능

---

## 레이블 및 어노테이션 업데이트

```bash
kubectl label pods -l app=nginx tier=fe
kubectl annotate pods my-nginx description='my frontend running nginx'
```

---

## 애플리케이션 스케일링

```bash
kubectl scale deployment/my-nginx --replicas=1
kubectl autoscale deployment/my-nginx --min=1 --max=3
```

---

## 리소스 인플레이스 업데이트

- `kubectl apply`: 구성 파일 기반 업데이트 (3-way diff)
- `kubectl edit`: 현재 구성 편집 후 적용
- `kubectl patch`: JSON, merge, 전략적 병합 패치 지원

---

## 파괴적 업데이트

- 수정 불가능한 필드 변경, 손상된 리소스 복구 시 사용

```bash
kubectl replace -f nginx-deployment.yaml --force
```

---

## 서비스 중단 없는 애플리케이션 업데이트

- `kubectl edit deployment/my-nginx` 를 사용하여 이미지 태그 등 변경
- Deployment는 롤링 업데이트 방식으로 파드를 순차적으로 교체
