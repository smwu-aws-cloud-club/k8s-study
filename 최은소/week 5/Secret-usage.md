# 시크릿(Secret)을 사용하여 안전하게 자격증명 배포하기

쿠버네티스에서 시크릿(Secret)을 사용하여 민감한 데이터를 안전하게 파드에 주입해보자.

---

## 시작하기 전에
- 쿠버네티스 클러스터 필요
- `kubectl` CLI 설치 및 설정
- 2개 이상의 워커 노드 추천
- Playground: [Killercoda](https://killercoda.com), [KodeKloud](https://kodekloud.com), [Play with Kubernetes](https://labs.play-with-k8s.com/)

---

## 시크릿 데이터 base64 인코딩
```bash
echo -n 'my-app' | base64          # 사용자 이름 인코딩
echo -n '39528$vdg7Jb' | base64    # 비밀번호 인코딩
```

- 결과:
  - username → `bXktYXBw`
  - password → `Mzk1MjgkdmRnN0pi`

---

## 시크릿 생성하기 (YAML 사용)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  username: bXktYXBw
  password: Mzk1MjgkdmRnN0pi
```

```bash
kubectl apply -f https://k8s.io/examples/pods/inject/secret.yaml
kubectl describe secret test-secret
```

---

## 시크릿 생성하기 (kubectl CLI 사용)
```bash
kubectl create secret generic test-secret \
  --from-literal='username=my-app' \
  --from-literal='password=39528$vdg7Jb'
```

---

## 파드에서 볼륨으로 시크릿 사용하기
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
    - name: test-container
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secret-volume
  volumes:
    - name: secret-volume
      secret:
        secretName: test-secret
```

```bash
kubectl apply -f https://k8s.io/examples/pods/inject/secret-pod.yaml
kubectl exec -it secret-test-pod -- /bin/bash
ls /etc/secret-volume
cat /etc/secret-volume/username
cat /etc/secret-volume/password
```

---

## 환경 변수로 시크릿 사용하기

### 단일 시크릿 키
```bash
kubectl create secret generic backend-user --from-literal=backend-username='backend-admin'
```

```yaml
env:
- name: SECRET_USERNAME
  valueFrom:
    secretKeyRef:
      name: backend-user
      key: backend-username
```

```bash
kubectl create -f https://k8s.io/examples/pods/inject/pod-single-secret-env-variable.yaml
kubectl exec -it env-single-secret -- /bin/sh -c 'echo $SECRET_USERNAME'
```

### 여러 시크릿 키
```bash
kubectl create secret generic db-user --from-literal=db-username='db-admin'
```

```yaml
env:
- name: BACKEND_USERNAME
  valueFrom:
    secretKeyRef:
      name: backend-user
      key: backend-username
- name: DB_USERNAME
  valueFrom:
    secretKeyRef:
      name: db-user
      key: db-username
```

```bash
kubectl create -f https://k8s.io/examples/pods/inject/pod-multiple-secret-env-variable.yaml
kubectl exec -it envvars-multiple-secrets -- /bin/sh -c 'env | grep _USERNAME'
```

### `envFrom` 으로 전체 키-값 환경 변수로 설정
```bash
kubectl create secret generic test-secret \
  --from-literal=username='my-app' \
  --from-literal=password='39528$vdg7Jb'
```

```yaml
envFrom:
- secretRef:
    name: test-secret
```

```bash
kubectl create -f https://k8s.io/examples/pods/inject/pod-secret-envFrom.yaml
kubectl exec -it envfrom-secret -- /bin/sh -c 'echo "username: $username
password: $password"'
```
