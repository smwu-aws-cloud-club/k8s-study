# Kubernetes Secret

시크릿은 암호, 토큰 또는 키 등 민감한 데이터를 저장하기 위한 오브젝트입니다. 쿠버네티스는 컨피그맵과 비슷하지만 보안 강화를 위해 시크릿을 따로 제공한다.

## 시크릿 사용 이유
- 파드 스펙이나 이미지에 민감한 정보를 포함하지 않도록 하기 위함
- 파드와 독립적으로 생성되어 노출 위험을 줄일 수 있음
- 비휘발성 저장소 기록 방지 등 보안 조치 가능

## 보안 권장사항
- 저장소 암호화 (Encryption at Rest)
- 최소 권한 부여 (RBAC)
- 외부 시크릿 저장소 사용 고려

## 주요 사용 방식
- 볼륨 마운트로 파일처럼 사용
- 환경 변수로 사용
- 이미지 풀에 인증 정보 제공 (imagePullSecrets)

## 시크릿 생성 방법
- `kubectl create secret`
- 매니페스트 파일 사용
- Kustomize 도구 사용

## 시크릿 데이터 구성
- `data`: base64 인코딩된 문자열
- `stringData`: 평문 입력 (내부적으로 data로 병합됨)

## 크기 제한
- 최대 1MiB
- 리소스 쿼터를 통해 개수 제한 가능

## 마운트 예시

```yaml
volumes:
- name: foo
  secret:
    secretName: mysecret
containers:
- name: redis
  image: redis
  volumeMounts:
  - name: foo
    mountPath: "/etc/foo"
    readOnly: true
```

## 환경 변수 사용 예시

```yaml
env:
- name: SECRET_USERNAME
  valueFrom:
    secretKeyRef:
      name: mysecret
      key: username
```

## 시크릿 타입

| 타입 | 설명 |
|------|------|
| Opaque | 사용자 정의 데이터 |
| kubernetes.io/service-account-token | 서비스 어카운트 토큰 |
| kubernetes.io/dockercfg | 도커 레거시 config |
| kubernetes.io/dockerconfigjson | 도커 config.json |
| kubernetes.io/basic-auth | 기본 인증 (username/password) |
| kubernetes.io/ssh-auth | SSH 키 |
| kubernetes.io/tls | TLS 인증서/키 |
| bootstrap.kubernetes.io/token | 노드 부트스트랩 토큰 |

## 불변(Immutable) 시크릿

```yaml
immutable: true
```

- 데이터 변경 불가
- 삭제 후 재생성만 가능
- apiserver 부하 절감 효과 있음

## 마운트된 시크릿 자동 업데이트

- 기본적으로 `watch` 방식 사용
- `subPath` 마운트는 자동 업데이트 불가
- 환경 변수 방식은 자동 업데이트 안됨 (재시작 필요)

## 보안 참고 사항

- 노드에 필요할 때만 시크릿 전송됨
- tmpfs에 저장됨
- privileged 컨테이너는 모든 시크릿 접근 가능

## 사용 사례

- SSH 키, API 토큰, DB 인증정보 저장
- TLS 시크릿을 통한 Ingress 인증서 관리
- 환경 분리에 따른 시크릿 템플릿 구성