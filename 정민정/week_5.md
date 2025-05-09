## 1. ConfigMap 개요

### 1.1 개념

- ConfigMap은 **비밀이 아닌 구성 데이터를 키-값 형태로 저장**하는 객체
- Pod에서 환경 변수, 커맨드 라인 인자, 설정 파일 형태로 데이터를 전달할 수 있음
- Kubernetes 리소스와 설정 값을 분리해 애플리케이션의 이식성과 유연성을 높임

### 1.2 주요 특징

- ConfigMap은 쿠버네티스의 표준 리소스로 API 객체임
- ConfigMap의 값은 문자열로 제한되며, 바이너리 데이터는 적합하지 않음
- 데이터는 환경 변수, 커맨드 라인 인자, 또는 파일로 마운트 가능

### 1.3 생성 방법

#### 리터럴 값으로 생성

```
bash
kubectl create configmap game-config --from-literal=player_initial_lives=3
```

#### 파일로부터 생성

```
bash
kubectl create configmap game-config --from-file=game.properties
```

#### 디렉터리 전체

```
bash
kubectl create configmap game-config --from-file=/configs/
```

#### YAML 정의

```
yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-config
data:
  player_initial_lives: "3"
  difficulty: "medium"
```

### 1.4 사용 예시

#### 환경 변수로 주입

```
yaml
env:
- name: PLAYER_LIVES
  valueFrom:
    configMapKeyRef:
      name: game-config
      key: player_initial_lives
```

#### 볼륨 마운트

```
yaml
volumes:
- name: config-volume
  configMap:
    name: game-config
volumeMounts:
- name: config-volume
  mountPath: /etc/config
```

### 1.5 제한 사항 및 주의사항

- ConfigMap 값은 **비밀 데이터용이 아님**
- Pod이 실행 중일 경우, ConfigMap이 변경돼도 자동 반영되지 않음
- 너무 많은 데이터를 넣으면 etcd 크기 제한을 초과할 수 있음

---

## 2. Secret 개요

### 2.1 개념

- Secret은 **비밀번호, OAuth 토큰, SSH 키 등 민감 정보를 저장하는 객체**
- ConfigMap과 달리 민감한 데이터에 대한 **기본 보호 기능**을 제공함
- etcd에 저장되며, 전송 시 base64 인코딩 처리됨

### 2.2 Secret 사용 이유

- 민감 정보를 컨테이너 이미지에 하드코딩하지 않기 위해
- RBAC 및 암호화 설정과 함께 보안성을 높이기 위해
- 환경 변수나 마운트된 파일로 Pod에 주입 가능

### 2.3 타입

| 타입                           | 설명                           |
| ------------------------------ | ------------------------------ |
| Opaque                         | 기본 타입. 사용자가 키-값 지정 |
| kubernetes.io/dockerconfigjson | 프라이빗 레지스트리 인증       |
| kubernetes.io/tls              | 인증서와 개인 키 저장          |

### 2.4 생성 방법

#### 리터럴로 생성

```
bash
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secret
```

#### YAML 정의

```
yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
data:
  username: YWRtaW4=
  password: c2VjcmV0
```

> base64로 인코딩된 문자열을 사용해야 함

#### 파일로부터 생성

```
bash
kubectl create secret generic app-secret --from-file=./app.conf
```

### 2.5 사용 예시

#### 환경 변수로 주입

```
yaml
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: username
```

#### 파일로 마운트

```
yaml
volumes:
- name: secret-volume
  secret:
    secretName: db-secret
volumeMounts:
- name: secret-volume
  mountPath: /etc/secret
  readOnly: true
```

### 2.6 보안 주의사항

- base64는 암호화가 아님. 실제 보안 보호를 위해선 etcd 암호화와 RBAC 설정 필요
- 환경 변수로 주입된 경우, `ps` 같은 명령어로 노출될 수 있으므로 파일 마운트 방식 권장
- Secret은 삭제되거나 변경될 수 있으므로 의존성을 줄이는 것이 좋음

---

## 3. 시크릿 데이터를 안전하게 Pod에 분배하기

### 3.1 목적

- 애플리케이션이 안전하게 비밀 정보에 접근할 수 있도록 Secret을 Pod에 주입하는 방법을 실습함
- 환경 변수와 볼륨 마운트를 이용한 주입 방식 확인

### 3.2 Secret 생성

```
bash
kubectl create secret generic db-credentials \\
  --from-literal=username=admin \\
  --from-literal=password=secret123
```

### 3.3 환경 변수로 주입하는 Pod 정의

```
yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-client
spec:
  containers:
  - name: app
    image: busybox
    command: [ \"sh\", \"-c\", \"env | grep DB_ && sleep 3600\" ]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

### 3.4 파일로 마운트하는 Pod 정의

```
yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-client-vol
spec:
  containers:
  - name: app
    image: busybox
    command: [ \"sh\", \"-c\", \"cat /etc/creds/* && sleep 3600\" ]
    volumeMounts:
    - name: creds
      mountPath: \"/etc/creds\"
      readOnly: true
  volumes:
  - name: creds
    secret:
      secretName: db-credentials
```

### 3.5 결과 확인

```
bash
kubectl exec db-client -- printenv | grep DB_
kubectl exec db-client-vol -- cat /etc/creds/username
kubectl exec db-client-vol -- cat /etc/creds/password
```

### 3.6 권장 사항

- 환경 변수보다 파일 마운트 방식이 보안에 더 안전함
- RBAC을 통해 Secret 객체 접근 권한을 엄격하게 제어
- Secret을 포함한 데이터는 최소 권한 원칙에 따라 분배되어야 함
