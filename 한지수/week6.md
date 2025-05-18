
# 시크릿(Secret)을 사용하여 안전하게 자격증명 배포하기

## 시작하기 전

- 쿠버네티스 클러스터와 kubectl이 클러스터와 통신할 수 잇도록 설정되어 있어야 한다. 최소 2개의 노드가 있는 클러스터가 권장된다.
- 사용자 이름과 비밀번호 등 시크릿 데이터를 base-64로 변환하다.

```bash
echo -n 'my-app' | base64      # 결과: bXktYXBw
echo -n '39528$vdg7Jb' | base64 # 결과: Mzk1MjgkdmRnN0pi
```
<br>

## **시크릿 생성하기**

사용자 이름과 암호가 포함된 시크릿을 생성하는 방법이다.

- yaml 구성 파일 사용하기

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: test-secret
    data:
      username: bXktYXBw          # base64 인코딩된 'my-app'
      password: Mzk1MjgkdmRnN0pi  # base64 인코딩된 '39528$vdg7Jb'
    ```
<br>

- 시크릿 생성 및 확인

    ```bash
    kubectl apply -f https://k8s.io/examples/pods/inject/secret.yaml
    kubectl get secret test-secret
    kubectl describe secret test-secret
    ```

    - `kubectl get secret` 결과 : 시크릿 이름, 타입, 데이터 개수, 생성 시간 확인 가능
    - `kubectl describe secret` 결과 : 시크릿 상세 데이터(암호화된 크기 등) 확인 가능

  <br>
  
- kubectl 명령어로 직접 시크릿 생성하기

    ```bash
    kubectl create secret generic test-secret --from-literal='username=my-app' --from-literal='password=39528$vdg7Jb'
    ```

    - Base64 인코딩을 건너뛰고 해 명령어로 간편하게 생성 가능

<br>

## **볼륨을 통해 시크릿 데이터에 접근할 수 있는 파드 생성하기**

- 파드 구성 파일

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
<br>

- 파드 생성 및 상태 학인

    ```bash
    kubectl apply -f https://k8s.io/examples/pods/inject/secret-pod.yaml
    kubectl get pod secret-test-pod
    ```

    - 파드가 Running 상태인지 확인

    <br>
- 파드 내 컨테이너 셀 접속

    ```bash
    kubectl exec -i -t secret-test-pod -- /bin/bash
    ```

    <br>
- 시크릿 데이터 확인

    ```bash
    ls /etc/secret-volume
    # 출력: password  username
    
    cat /etc/secret-volume/username
    # 출력: my-app
    
    cat /etc/secret-volume/password
    # 출력: 39528$vdg7Jb
    ```

    - 시크릿은 `/etc/secret-volume` 경로에 파일로 마운트됨

<br>

## **시크릿 데이터를 사용하여 컨테이너 환경 변수 정의하기**

### **단일 시크릿 데이터로 컨테이너 환경 변수 정의하기**

- 시크릿 생성

    ```bash
    kubectl create secret generic backend-user --from-literal=backend-username='backend-admin'
    ```
- 파드 구성 파일

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: env-single-secret
    spec:
      containers:
      - name: envars-test-container
        image: nginx
        env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: backend-user        # 시크릿 이름
              key: backend-username     # 시크릿 키
    
    ```
- 파드 생성

    ```yaml
    kubectl create -f https://k8s.io/examples/pods/inject/pod-single-secret-env-variable.yaml
    ```

- 시크릿 값을 환경 변수로 출력

    ```yaml
    kubectl exec -i -t env-single-secret -- /bin/sh -c 'echo $SECRET_USERNAME'
    ```

- 출력 결과

    ```yaml
    backend-admin
    ```

<br>

### **여러 시크릿 데이터로 컨테이너 환경 변수 정의하기**

- 시크릿 2개 생성

    ```bash
    kubectl create secret generic backend-user --from-literal=backend-username='backend-admin'
    kubectl create secret generic db-user --from-literal=db-username='db-admin'
    ```

- 파드 구성 파일

    ```bash
    apiVersion: v1
    kind: Pod
    metadata:
      name: envvars-multiple-secrets
    spec:
      containers:
      - name: envars-test-container
        image: nginx
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

- 파드 생성

    ```bash
    kubectl create -f https://k8s.io/examples/pods/inject/pod-multiple-secret-env-variable.yaml
    ```

- 환경 변수 확인

    ```bash
    kubectl exec -i -t envvars-multiple-secrets -- /bin/sh -c 'env | grep _USERNAME'
    ```

- 출력 결과

    ```bash
    DB_USERNAME=db-admin
    BACKEND_USERNAME=backend-admin
    ```

<br>

## **시크릿의 모든 키-값 쌍을 컨테이너 환경 변수로 구성하기**

- 시크릿 생성

    ```bash
    kubectl create secret generic test-secret \
      --from-literal=username='my-app' \
      --from-literal=password='39528$vdg7Jb'
    ```
<br>

- 파드 구성 파일

    ```bash
    apiVersion: v1
    kind: Pod
    metadata:
      name: envfrom-secret
    spec:
      containers:
      - name: envars-test-container
        image: nginx
        envFrom:
        - secretRef:
            name: test-secret
    ```

    - `envFrom` : 시크릿의 모든 데이터를 컨테이너 환경 변수로 정의함. 시크릿의 키는 파드에서 환경 변수의 이름이 된다.

    <br>
  
- 파드 생성

    ```bash
    kubectl create -f https://k8s.io/examples/pods/inject/pod-secret-envFrom.yaml
    ```
    <br>

- 환경 변수 출력

    ```bash
    kubectl exec -i -t envfrom-secret -- /bin/sh -c 'echo "username: $username\npassword: $password\n"'
    ```
    <br>

- 출력 결과

    ```bash
    username: my-app
    password: 39528$vdg7Jb
    ```
<br>
<hr>

# 네임 스페이스

> 네임 스페이스는 클러스터 내 리소스를 논리적으로 격리하기 위한 메커니즘이다.
리소스 이름은 네임 스페이스 내에서만 유일하면 되며, 네임 스페이스 간에는 중복되어도 된다.
네임 스페이스 기반 스코핑은 디플로이먼트나 서비스와 같은 네임스페이스 기반 오브젝트에만 적용된다. 스토리지 클래스, 노드 , 퍼시스턴트볼륨과 같은 클러스터 범위 오브젝트에는 적용되지 않는다.
>

## **여러 개의 네임스페이스를 사용하는 경우**

네임 스페이스는  여러 팀이나 프로젝트가 함께 사용하는 환경에서 리소스를 격리하고 관리하기 위해 만들어졌다.

- 리소스 이름은 네임 스페이스 내에서만 유일하면 되며, 네임스페이스는 중첩될 수 없고 하나의 리소스는 하나의 네임 스페이스에만 속할 수 있
- 리소스 쿼터를 통해 클러스터 자원을 사용자 간에 분배 가능
- 소프트웨어의 버전 차이와 같은 단순한 구분은 네임 스페이스보다 레이블을 사용하는 것이 좋음
- 프로덕션 환경에서는 default 네임 스페이스 대신 별도의 네임 스페이스를 만들어 사용하는 것을 권장

<br>


## **초기 네임스페이스**

쿠버네티스는 기본적으로 네 개의 초기 네임 스페이스를 가진다.

- `default` : 네임 스페이스 없이 리소스를 만들 때 사용되는 기본 네임 스페이스
- `kube-node-lease` : 각 노드에 대한 리스 오브젝트를 포함하며, 노드의 상태를 감지하기 위한 하트비트 용도로 사용
- `kube-public` : 모든 클라이언트가 읽기 접근할 수 있는 네임 스페이스로, 주로 공개 리소스를 위해 예약됨
- `kube-system` : 쿠버네티스 시스템 컴포넌트가 사용하는 네임 스페이스이다.

<br>


##  **네임스페이스 다루기**

- ``kube-` 접두사로 시작하는 네임스페이스` : 쿠버네티스 시스템용으로 예약되어 있으므로, 사용자는 이러한 네임스페이스를 생성하지 않는다.

### **네임스페이스 조회**

- 클러스터의 현재 네임스페이스 나열

```bash
kubectl get namespace
```

- 예시 출력

```mathematica
NAME              STATUS   AGE
default           Active   1d
kube-node-lease   Active   1d
kube-public       Active   1d
kube-system       Active   1d
```

### **요청에 네임스페이스 설정하기**

현재 명령어에 네임스페이스 적용

```bash
kubectl run nginx --image=nginx --namespace=<namespace명>
kubectl get pods --namespace=<namespace명>
```

### **선호하는 네임스페이스 설정하기**

- 현재 컨텍스트에 네임 스페이스 영구 설정

```bash
kubectl config set-context --current --namespace=<namespace명>
```

- 설정된 네임 스페이스 확인

```bash
kubectl config view --minify | grep namespace:
```

<br>


## **네임스페이스와 DNS**

서비스를 생성하면 `<서비스-이름>.<네임스페이스-이름>.svc.cluster.local` 형식의 DNS 엔트리가 자동 생성된다.

- 같은 네임스페이스 내에서는 <서비스-이름> 만으로 접근 가능하며, 네임 스페이스를 넘어서려면 FQDN을 사용
- 모든 네임 스페이스 이름은 RFC 1123 규격의 DNS 레이블이아야 하며, 공개 최상위 도메인(TLD)과 동일한 네임 스페이스 이름은 DNS 충돌 위험이 있으므로 피함

  → 이를 방지하기 위해 신뢰된 사용자만 네임 스페이스를 생성할 수 있도록 권한을 제한하고, 필요 시 어드미션  웹훅 등으로 공개 TLD와 같은 이름의 네임스페이스 생성을 막을 수 있

<br>

## **모든 오브젝트가 네임스페이스에 속하지는 않음**

대부분의 쿠버네티스 리소스(파드, 서비스, 레필리케이션 컨트롤러 등)는 네임스페이스에 속한다.

그러나 네임스페이스 리소스 자체, 노드,. 퍼시스턴트 볼륨과 같은 일부 리소스는 네임스페이스에 속하지 않는다.

- 네임 스페이스에 속하는 리소스 조회

```bash
kubectl api-resources --namespaced=true
```

- 네임 스페이스에 속하지 않는 리소스 조회

```bash
kubectl api-resources --namespaced=false
```

<br>

## 자동 레이블링

쿠버네티스 컨트롤 플레인에서 NamespaceDefaultLabelName 기능 게이트가 활성화되면, 모든 네임스페이스에 변경 불가능한 레이블 `kuubernetes.io/metatata.name`이 네임스페이스 이름으로 자동 설정된다.