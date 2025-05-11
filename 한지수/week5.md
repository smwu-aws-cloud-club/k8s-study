# 컨피그맵

> 컨피그맵은 키-값 쌍으로 기밀이 아닌 데이터를 저장하는 데 사용하는 API 오브젝트이다. 파드는 볼륨에서 환경 변수, 커맨드-라인 인수 또는 구성 파일로 컨피그맵을 사용할 수 있다.
>
>
> 컨피그맵을 사용하면 컨테이너 이미지에서 환경별 구성을 분리하여, 애플리케이션을 쉽게 이식할 수 있다.
>

## 사용 동기

컨피그맵(ConfigMap)은 애플리케이션의 환경 설정 데이터를 코드와 별도로 관리하는 Kubernetes 리소스다. 로컬 개발 환경과 클라우드 환경에서 동일한 애플리케이션을 실행할 때, 각 환경에 맞는 설정을 환경 변수로 설정할 수 있다.

예를 들어, `DATABASE_HOST`는 로컬에서는 `localhost`, 클라우드에서는 Kubernetes 서비스를 참조하도록 설정할 수 있다.

컨피그맵의 데이터 용량은 1MiB를 초과할 수 없으며, 더 큰 데이터는 볼륨을 마운트하거나 별도의 데이터베이스를 사용하는 방식으로 처리해야 한다.

## 컨피그맵 오브젝트

컨피그맵(ConfigMap)은 다른 Kubernetes 오브젝트가 사용할 설정 데이터를 저장하는 API 오브젝트다. 대부분의 쿠버네티스 오브젝트와 달리, 컨피그맵은 `data`와 `binaryData` 필드를 갖는다. `data`는 UTF-8 문자열을 포함하고, `binaryData`는 바이너리 데이터를 base64로 인코딩한 문자열을 포함한다.

- 유효한 DNS 서브도메인 이름이어야 한다.
- 각 키는 영숫자, , `_`, `.` 문자로 구성되며, `data`와 `binaryData` 필드의 키는 겹치지 않아야 한다.

## 컨피그맵과 파드

컨피그맵을 참조하는 파드의 `spec`을 작성하여 파드 내부의 컨테이너를 구성할 수 있다. 파드와 컨피그맵은 동일한 네임스페이스에 있어야 한다. 스태틱 파드는 컨피그맵이나 다른 API 오브젝트를 참조할 수 없다.

- 컨피그맵을 사용하여 파드 내부에서 데이터를 처리하는 방법
    - 컨테이너 커맨드와 인수 내에서 사용
    - 컨테이너에 대한 환경 변수로 사용
    - 애플리케이션이 읽을 수 있도록 읽기 전용 볼륨에 파일 추가
    - 쿠버네티스 API를 사용하여 컨피그맵을 읽는 코드 작성

첫 세 가지 방법은 kubelet이 컨테이너 시작 시 컨피그맵의 데이터를 사용한다. 네 번째 방법은 애플리케이션이 컨피그맵 변경 시 업데이트를 받기 위해 쿠버네티스 API를 구독하는 방식이다.

다음은 `game-demo` 컨피그맵을 사용하여 파드를 구성하는 예시이다:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        - name: PLAYER_INITIAL_LIVES
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: player_initial_lives
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
        - name: config
          mountPath: "/config"
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: game-demo
        items:
          - key: "game.properties"
            path: "game.properties"
          - key: "user-interface.properties"
            path: "user-interface.properties"
```

## 컨피그맵 사용하기

### **파드에서 컨피그맵을 파일로 사용하기**

1. 새로운 컨피그맵을 생성하거나 기존의 컨피그맵을 사용
2. 파드의 `.spec.volumes[]` 아래에 컨피그맵을 참조하는 볼륨을 추가
3. 각 컨테이너에서 `.spec.containers[].volumeMounts[]`를 설정하고, `mountPath`를 지정하여 컨피그맵이 마운트될 디렉터리를 설정한다. `readOnly: true`를 설정한다.
4. 프로그램이 컨피그맵 파일을 찾을 수 있도록 이미지 또는 커맨드 라인을 수정한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: mypod
      image: redis
      volumeMounts:
        - name: foo
          mountPath: "/etc/foo"
          readOnly: true
  volumes:
    - name: foo
      configMap:
        name: myconfigmap
```

### **마운트된 컨피그맵의 자동 업데이트**

- 컨피그맵이 업데이트되면 마운트된 파일도 자동으로 업데이트된다. kubelet은 주기적으로 컨피그맵의 최신 상태를 확인한다.
- 환경 변수로 사용된 컨피그맵은 자동으로 업데이트되지 않으며, 파드를 다시 시작해야 한다.
- **subPath** 볼륨 마운트에서 사용되는 컨피그맵은 업데이트를 받지 않는다.

## 변경할 수 없는 컨피그

- 우발적인 업데이트로부터 애플리케이션 중단을 방지할 수 있고, 변경 감시를 중단하여 부하를 줄일 수 있다.
- `immutable`로 설정된 컨피그맵은 `data` 또는 `binaryData` 필드 내용을 변경할 수 없다.
- `immutable` 필드를 `true`로 설정하여 변경할 수 없는 컨피그맵 생성

    ```yaml
    apiVersion: v1
    kind: ConfigMap
 
    immutable: true
    ```
  
<hr>

# 시크릿
> 시크릿(Secret)은 암호, 토큰, 키와 같은 소량의 중요한 기밀 데이터를 포함하는 Kubernetes 오브젝트이다. 시크릿을 사용하면 중요한 정보를 파드 명세나 컨테이너 이미지에 직접 포함하지 않아도 되므로 보안성이 향상된다.
> 

시크릿은 애플리케이션 코드에 기밀 데이터를 하드코딩할 필요가 없게 해주며, 파드와 독립적으로 생성될 수 있어 파드를 생성, 수정, 확인하는 과정에서 시크릿이 노출될 위험을 줄일 수 있다.

Kubernetes와 클러스터에서 실행되는 애플리케이션은 시크릿을 비휘발성 저장소에 기록하지 않도록 설정하는 등 추가적인 보안 조치를 취할 수 있다. 시크릿은 ConfigMap과 유사한 구조를 가지고 있지만, 일반적인 설정 정보가 아닌 민감한 정보를 안전하게 보관하기 위해 설계된 리소스이다.

<aside>
💡

주의해야 할 점은 Kubernetes 시크릿이 기본적으로 API 서버의 데이터 저장소인 etcd에 **암호화되지 않은 상태로 저장된다는 점**이다. 따라서 API 접근 권한이 있는 사용자나 etcd에 접근 가능한 사람은 누구나 시크릿을 조회하거나 수정할 수 있다. 또한, 네임스페이스에서 파드를 생성할 권한이 있는 사람은 간접적인 방법을 통해 해당 네임스페이스의 모든 시크릿에 접근할 수 있다.

</aside>

시크릿을 보다 안전하게 사용하려면 다음과 같은 보안 조치를 따른다.

1. 저장된 데이터를 암호화하는 Encryption at Rest 기능을 활성화해야 한다.
2.  RBAC 권한 설정을 통해 최소한의 접근 권한만 부여하도록 구성해야 한다.
3. 특정 컨테이너에서만 시크릿에 접근하도록 제한해야 한다.
4. 외부 시크릿 저장소 서비스의 사용도 고려할 수 있다.

# 시크릿의 사용

파드가 시크릿을 사용하는 주요한 방법으로 다음의 세 가지가 있다.

- 하나 이상의 컨테이너에 마운트된 볼륨 내의 파일로써 사용
- 컨테이너 환경 변수로써 사용
- 파드의 이미지를 가져올 때 Kubelet에 의해 사용

## 시크릿의 대체품

기밀 데이터를 보호하기 위해 시키릿 대산 다음의 대안 중 하나를 고를 수 있다.

- 클러스터 내 애플리케이션 간 인증에는 ServiceAccount 및 토큰을 사용하여 클라이언트를 식별할 수 있음
- 비밀 정보 관리 도구를 클러스터 내부 또는 외부에 배포하여, 인증된 클라이언트에만 비밀 정보를 제공하고 HTTPS로만 접근을 제한할 수 있음
- *X.509 인증서 발급을 위해 커스텀 인증자(signer)**를 구현하고, CertificateSigningRequests를 통해 필요한 파드에 인증서를 제공할 수 있음
- 장치 플러그인을 통해 노드의 TPM 등 암호화 하드웨어를 특정 파드에 노출시키고, 해당 하드웨어가 구성된 노드에 신뢰할 수 있는 파드만 스케줄링할 수 있음

---

# 시크릿 다루기

## 시크릿 생성하기

시크릿 생성에는 다음과 같은 방법이 있다.

- kubectl 사용
- 환경 설정 파일 사용
- kustomize 도구 사용

## 시크릿 이름 및 데이터에 대한 제약 사항

- 시크릿 오브젝트의 이름은 유효한 DNS 서브도메인 형식이어야 함
- 시크릿 정의 시, `data`와 `stringData` 필드를 선택적으로 사용할 수 있음
- `data` 필드의 값은 반드시 Base64로 인코딩된 문자열이어야 함
- Base64 인코딩이 번거롭다면, 일반 문자열을 입력할 수 있는 `stringData` 필드를 사용할 수 있음
- 두 필드의 키는 영숫자, 하이픈(-), 밑줄(_), 점(.)만 사용 가능
- `stringData`의 키-값 쌍은 자동으로 `data` 필드로 변환됨
- 동일한 키가 두 필드에 모두 존재하면, `stringData`의 값이 우선 적용됨

## 크기 제한

- 개별 시크릿의 최대 크기는 1 MiB로 제한되어 있음

  → 이는 너무 큰 시크릿이 API 서버나 kubelet의 메모리를 고갈시키는 것을 방지하기 위함

- 하지만, 작은 시크릿을 너무 많이 생성해도 메모리 부족 문제가 발생할 수 있음

  → 이를 방지하기 위해, 리소스 쿼터(ResourceQuota)를 사용하여 네임스페이스별 시크릿 수를 제한할 수 있음


## 시크릿 수정하기

시크릿은 불변(immutable) 설정이 아닌 경우 수정이 가능하다.

- 시크릿 수정 방법
    - `kubectl` 명령어 사용
    - 설정 파일로 수정
    - Kustomize 사용 시, 기존 시크릿이 아닌 새로운 시크릿 오브젝트가 생성됨

시크릿 수정 내용이 파드에 자동 반영되는지는 사용 방식에 따라 다르다.

→ 시크릿을 마운트한 방식에 따라 파드에 자동으로 업데이트될 수 있다.

## 시크릿 사용하기

시크릿을 파드에 볼륨 형태로 마운트하여 사용할 수 있다.

시크릿을 파드의 환경 변수로 설정하여 파드 내에서 사용할 수 있다.

시크릿은 파드에 직접 연결되지 않더라도, 외부 시스템과 상호작용하는 시스템의 다른 구성 요소에서 필요한 자격 증명으로 사용할 수 있다.

- 시크릿을 참조하는 파드가 있을 경우
    - Kubernetes는 해당 시크릿이 유효한 시크릿을 가리키는지 검사한다.
- 시크릿 생성 순서
    - 시크릿은 이를 사용하는 파드보다 먼저 생성되어야 한다.
- 시크릿을 가져오지 못하는 경우
    - 파드 실행을 주기적으로 재시도하고, 관련된 에러 이벤트를 기록

### 선택적 시크릿

시크릿을 기반으로 컨테이너 환경 변수를 정의할 때, 환경 변수는 선택 사항으로 설정할 수 있다. → 기본적으로 시크릿은 필수 사항이다.

모든 필수 시크릿이 사용 가능해지기 전에는 파드의 컨테이너가 시작되지 않는다.

만약 파드가 시크릿의 특정 키를 참조하고, 해당 시크릿이 존재하지만 그 키가 존재하지 않을 때 → 파드는 시작 과정에서 실패한다.

## 파드에서 시크릿을 파일처럼 사용하기

시크릿을 생성하거나 기존 시크릿을 사용한다. 여러 파드가 동일한 시크릿을 참조할 수 있다.

- 파드 정의 수정
    - 볼륨 이름을 지정하고, `.spec.volumes[].secret.secretName` 필드에 시크릿 오브젝트의 이름을 지정한다.
- 컨테이너에 추가
    - 시크릿을 필요로 하는 각 컨테이너에 `.spec.containers[].volumeMounts[]`를 추가한다.
    - `.spec.containers[].volumeMounts[].readOnly = true`로 설정하고, `.spec.containers[].volumeMounts[].mountPath`를 지정한다.
    - 프로그램이 해당 디렉터리에서 파일을 찾도록 이미지나 커맨드 라인을 수정한다.
    - 시크릿의 각 키는 `mountPath` 아래의 파일명으로 표시된다.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
      - name: mypod
        image: redis
        volumeMounts:
        - name: foo
          mountPath: "/etc/foo"
          readOnly: true
      volumes:
      - name: foo
        secret:
          secretName: mysecret #시크릿 오브젝트 필드
          optional: false #"mysecret"은 반드시 존재해야 한다
    
    ```

- 파드에 여러 컨테이너가 있을 때
    - 각 컨테이너는 자체 `volumeMounts` 블록이 필요하지만, 시크릿에 대해서는 시크릿당 하나의 `.spec.volumes[]`만 필요하다.
- 서비스 어카운트 수동 생성
    - 수동생성보다 TokenRequest API를 사용하는 것이 권장된다.
    - `kubectl create token` 명령어를 통해 토큰을 얻을 수 있다.

## 특정 경로에 대한 시크릿 키 투영하기

`.spec.volumes[].secret.items` 필드를 사용하여 각 키의 대상 경로를 변경→ 시크릿의 특정 키만 지정된 경로에 투영되도록 설정할 수 있다.

## 시크릿 파일 퍼미션

- 시크릿 볼륨에 마운트된 파일의 기본 퍼미션은 **0644**이다.
- 전체 볼륨의 기본 퍼미션을 변경하려면 `.spec.volumes[].secret.defaultMode`에 원하는 모드를 지정한다.

    ```yaml
    volumes:
    - name: foo
      secret:
        secretName: mysecret
        defaultMode: 0400
    ```


→ `/etc/foo` 아래 모든 시크릿 파일이 `0400` 퍼미션으로 생성된다.

- `defaultMode`에 10진수 값을 써야 한다.
- 필요하다면 `.spec.volumes[].secret.items`와 함께 각 키별로 개별 퍼미션을 오버라이드할 수도 있다.

## 볼륨으로부터 시크릿 값 사용하기

시크릿 볼륨이 마운트되면, 시크릿 키는 파일로 나타난다. 각 파일에는 base64로 디코딩된 실제 시크릿 값이 저장된다.

```bash
ls /etc/foo/
# username
# password

cat /etc/foo/username
# admin

cat /etc/foo/password
# 1f2d1e2e67df
```

애플리케이션은 이 파일들을 읽어 시크릿 데이터를 사용할 수 있다.

## 마운트된 시크릿의 자동 업데이트

- 시크릿이 변경되면, 쿠버네티스는 일관성 있는 방식(eventually-consistent)으로 시크릿 볼륨 데이터를 자동 업데이트한다.

  → `subPath`로 마운트된 시크릿은 자동으로 갱신되지 않는다.

- kubelet은 파드가 사용하는 시크릿 키/값을 캐시에 저장하고, 변경을 감지하여 업데이트한다.

  → 감지 전략은 kubelet 설정의 `configMapAndSecretChangeDetectionStrategy`로 지정되며, 기본값은 `Watch`이다.

  → 업데이트 방식은 Watch + TTL 기반 캐시(기본), kubelet이 주기적으로 API 서버에 폴링

- 시크릿 값이 실제로 파드에 반영되기까지는 kubelet 동기화 주기 + 캐시 전파 지연 만큼 시간이 걸릴 수 있다.
    - 감지 방식에 따라 지연 시간은 다르다:
        - Watch 방식: 감시 지연
        - TTL 캐시: TTL 시간만큼 지연
        - Polling: 지연 없음 (즉시 반영)

## 시크릿을 환경 변수 형태로 사용하기

- 파드에서 시크릿 키 값을 환경 변수로 사용하려면, 컨테이너의 `env[].valueFrom.secretKeyRef`를 설정한다.
    - 하나의 시크릿을 여러 파드에서 참조할 수 있다.
- 시크릿의 각 키를 환경 변수로 지정하려면, 해당 컨테이너에 각각 명시해야 한다.
    - 시크릿 이름과 키는 `name`, `key` 필드에 지정한다.
    - `optional` 필드로 해당 키가 없어도 파드 생성이 가능하도록 설정할 수 있다 (`false`가 기본값).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef: # 시크릿 키 값을 환경변수로 사용
            name: mysecret #시크릿 이름
            key: username # 시크릿 키
            optional: false # 기본
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef: 
            name: mysecret  
            key: password
            optional: false  
  restartPolicy: Never
```

## 올바르지 않은 환경 변수

`envFrom`으로 시크릿을 참조할 때, 유효하지 않은 환경 변수 이름을 가진 키는 무시된다. 이 경우 파드는 시작되지만, 해당 키들은 환경 변수로 설정되지 않는다.

- `kubectl get events` 명령으로 확인

→ 이벤트 `REASON`이 `InvalidVariableNames`로 표시되며 건너뛴 키 목록이 메시지에 포함된다.

## 환경 변수로부터 시크릿 값 사용하기

시크릿은 base64 디코드된 값으로 환경 변수에 설정되고, 컨테이너 내부에서 일반 환경 변수처럼 접근할 수 있다.

```bash
echo "$SECRET_USERNAME"  # 출력 예: admin
echo "$SECRET_PASSWORD"  # 출력 예: 1f2d1e2e67df
```

- 환경 변수는 파드 생성 시점에만 설정되므로, 시크릿이 변경되어도 컨테이너 재시작 없이는 반영되지 않는다.
    - 시크릿 변경을 감지해 컨테이너를 재시작하는 써드파티 솔루션을 사용할 수 있다.

## 컨테이너 이미지 풀 시크릿

비공개 저장소에서 컨테이너 이미지를 가져오고 싶다면, 이미지 풀 시크릿 을 구성할 수 있다. 이러한 시크릿은 파드 수준에 설정된다.

- imagePullSecrets를 사용해 kubelet이 인증할 수 있도록 설정한다.
- kubelet은 imagePullSecrets를 이용해 비공개 이미지 저장소에서 컨테이너 이미지를 가져온다.
- 수동 지정

  파드 정의에 직접 `imagePullSecrets`를 지정.

- 서비스어카운트에 연결하여 자동화
    - 시크릿을 생성한 뒤, 서비스어카운트에 연결.
    - 해당 서비스어카운트를 사용하는 파드는 자동으로 `imagePullSecrets`를 상속받음.

# 사용 사례

### 1. **컨테이너 환경 변수로 시크릿 사용**

- 시크릿을 정의

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  USER_NAME: YWRtaW4=       # base64 인코딩된 'admin'
  PASSWORD: MWYyZDFlMmU2N2Rm # base64 인코딩된 '1f2d1e2e67df'

```

- 시크릿을 생성

```bash
kubectl apply -f mysecret.yaml
```

- `envFrom` 사용 시, 모든 키를 환경 변수로 노출. 시크릿 키는 파드의 환경변수 이름이 됨

```yaml
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:       #모든 키를 환경변수로 노출
      - secretRef: 
          name: mysecret
```

### 2. **SSH 키를 사용하는 파드**

- SSH 키 파일로 시크릿 생성:

```bash
kubectl create secret generic ssh-key-secret \
  --from-file=ssh-privatekey=/path/to/id_rsa \
  --from-file=ssh-publickey=/path/to/id_rsa.pub
```

- 파드에서 시크릿을 마운트하여 SSH 키 사용:

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: ssh-key-secret

volumeMounts:
- name: secret-volume
  mountPath: "/etc/secret-volume"
  readOnly: true
```

- 키 부분 사용 가능 경로
    - `/etc/secret-volume/ssh-privatekey`
    - `/etc/secret-volume/ssh-publickey`

### 3. **운영/테스트 자격 증명 분리**

- 시크릿 생성:

```bash
kubectl create secret generic prod-db-secret \
  --from-literal=username=produser \
  --from-literal=password='Y4nys7f11'

kubectl create secret generic test-db-secret \
  --from-literal=username=testuser \
  --from-literal=password='iluvtests'
```

- 파드 2개에서 각기 다른 시크릿 참조:

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: prod-db-secret 또는 test-db-secret
```

- 동일한 파드 템플릿에서 시크릿만 다르게 적용 가능
- 서비스어카운트로 파드 명세 단순화:

```yaml
spec:
  serviceAccount: prod-db-client
```

### 4. **도트 파일(.dotfile) 형태의 시크릿 사용**

점으로 시작하는 키를 정의하여 데이터를 "숨김"으로 만들 수 있다. 이 키는 도트 파일 또는 "숨겨진" 파일을 나타낸다.

- `/etc/secret-volume/.secret-file`
- `ls -la`로 확인 가능

```yaml

data:
  .secret-file: dmFsdWUtMg0KDQo= # base64 인코딩된 값
```

### 5. **시크릿을 한 컨테이너에만 노출**

- 보안을 위해 시크릿을 분리된 signer 컨테이너에만 노출

  → 애플리케이션 취약점이 있어도 키 유출 위험 감소

- 프론트엔드는 비즈니스 로직만 수행, signer는 HMAC 요청만 처리하는 경

# 시크릿 타입

### 시크릿 타입 종류

쿠버네티스에서 시크릿 타입은 기밀 데이터를 처리할 때 사용된다. 기본적으로 여러 빌트인 타입이 있으며, 각 타입은 특정 용도에 맞게 구성된다.

### 1. 빌트인 시크릿 타입

- **Opaque**: 임의의 사용자 정의 데이터
- **kubernetes.io/service-account-token**: 서비스 어카운트 토큰
- **kubernetes.io/dockercfg**: 직렬화된 `~/.dockercfg` 파일
- **kubernetes.io/dockerconfigjson**: 직렬화된 `~/.docker/config.json` 파일
- **kubernetes.io/basic-auth**: 기본 인증을 위한 자격 증명
- **kubernetes.io/ssh-auth**: SSH 인증 자격 증명
- **kubernetes.io/tls**: TLS 인증서와 키
- **bootstrap.kubernetes.io/token**: 부트스트랩 토큰

### 2. Opaque 시크릿

- 기본 시크릿 타입
- `kubectl create secret generic` 명령어로 생성
- 데이터 없이 빈 시크릿을 만들 수 있음.

```bash
kubectl create secret generic empty-secret
kubectl get secret empty-secret
```

### 3. 서비스 어카운트 토큰 시크릿

- `kubernetes.io/service-account-token`
- 서비스 어카운트 토큰 저장
- TokenRequest API를 사용해 안전한 토큰을 얻는 것이 추천됨.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-sa-sample
  annotations:
    kubernetes.io/service-account.name: "sa-name"
type: kubernetes.io/service-account-token
data:
  # 사용자는 불투명 시크릿을 사용하므로 추가적인 키 값 쌍을 포함할 수 있다.
  extra: YmFyCg==
```

### 4. 도커 컨피그 시크릿

- `kubernetes.io/dockercfg` / `kubernetes.io/dockerconfigjson`
- 도커 레지스트리 인증 정보 저장

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-dockercfg
type: kubernetes.io/dockercfg
data:
  .dockercfg: |
    "<base64 encoded ~/.dockercfg file>"    
```

- `kubectl` 을 사용하고 싶은 경우

```yaml
kubectl create secret docker-registry secret-tiger-docker \
  --docker-email=tiger@acme.example \
  --docker-username=tiger \
  --docker-password=pass1234 \
  --docker-server=my-registry.example:5000
```

### 5. 기본 인증 시크릿

- `kubernetes.io/basic-auth`
- 기본 인증 자격 증명 저장

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: "admin"
  password: "t0p-Secret"
```

### 6. SSH 인증 시크릿

- `kubernetes.io/ssh-auth`
- SSH 인증 데이터 저장

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-ssh-auth
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: MIIEpQIBAAKCAQEAulqb/Y ...
```

### 7. TLS 시크릿

- `kubernetes.io/tls`
- TLS 인증서 및 키 저장

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  tls.crt: MIIC2DCCAcCgAwIBAgIBATANBgkqh ...
  tls.key: MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ ...
```

### 6. 부트스트랩 토큰 시크릿

- `bootstrap.kubernetes.io/token`
- 노드 부트스트랩 과정에서 인증 및 서명을 위한 토큰 저장에 사용 `kubeadm`을 사용하여 클러스터에 노드를 조인할 때 활용

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-5emitj        # <token-id>가 포함된 이름
  namespace: kube-system              # 일반적으로 kube-system 네임스페이스에 생성
type: bootstrap.kubernetes.io/token
stringData:
  token-id: "5emitj"                             # 6자리 문자열, 이름에 사용됨
  token-secret: "kq4gihvszzgn1p0r"               # 16자리 시크릿 문자열
  expiration: "2020-09-13T04:39:10Z"             # RFC3339 포맷의 만료 시각 (UTC)
  usage-bootstrap-authentication: "true"         # 인증 사용 여부
  usage-bootstrap-signing: "true"                # 서명 사용 여부
  auth-extra-groups: "system:bootstrappers:kubeadm:default-node-token"  # 추가 인증 그룹

```

# 불변 시크릿

`Secret` 또는 `ConfigMap` 리소스에 `immutable: true` 필드를 설정하여 수정이 불가능한 상태로 만드는 기능이다.

- 한 번 immutable로 설정하면 `data` 변경 불가
- 새로운 시크릿 생성 시 `immutable: true` 설정

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: dXNlcg==  # base64 인코딩된 값
  password: cGFzc3dvcmQ=
immutable: true
```

# 시크릿을 위한 정보 보안

- Kubernetes의 Secret은 ConfigMap과 비슷하지만 민감한 정보를 다루기 때문에 보안이 강화되어 있다. 서비스 어카운트 토큰이나 외부 시스템 자격 증명 등 중요한 데이터를 저장하는 데 사용된다.
- Secret은 해당 노드에서 실행 중인 파드가 요청할 때에만 노드로 전송된다. kubelet은 Secret 데이터를 디스크에 저장하지 않기 위해 tmpfs와 같은 메모리 기반 파일 시스템에 데이터를 유지한다. 파드가 삭제되면 kubelet은 로컬에 복사된 Secret 데이터를 제거한다.
- 기본적으로 파드 안의 컨테이너는 서비스 어카운트와 연관된 Secret만 사용할 수 있다. 다른 Secret을 사용하려면 환경 변수로 지정하거나 볼륨으로 마운트해야 한다.
- Secret은 요청한 파드에만 전달되며, 다른 파드에서 해당 Secret에 접근할 수 없다.

  → 단, 노드에서 privileged: true로 실행 중인 컨테이너는 해당 노드에 존재하는 모든 Secret에 접근할 수 있으므로 보안상 주의가 필요하다.

<hr>

# Horizontal Pod Autoscaler(HPA)

> Horizontal Pod Autoscaler(HPA)는 쿠버네티스에서 워크로드(디플로이먼트, 스테이트풀셋 등)의 파드 수를 자동으로 조절해주는 기능이다. 주로 CPU, 메모리 사용률, 커스텀 메트릭 등 다양한 지표를 기준으로 파드 수를 늘리거나 줄인다.
>

# HorizontalPodAutoscaler는 어떻게 작동하는가?

## 컨트롤 루프 기반 동작

- HPA는 간헐적으로 실행되는 컨트롤 루프로 동작하며, 실행 주기는 `kube-controller-manager`의 `-horizontal-pod-autoscaler-sync-period` 파라미터(기본 15초)로 설정된다.
- 매 주기마다 HPA 컨트롤러는 클러스터 내 모든 HPA 오브젝트에 대해 동작을 반복한다.
- **`scaleTargetRef`** 필드로 지정된 타겟 리소스의 **`.spec.selector`** 레이블을 통해 관리 중인 파드들을 선택한다.
    - **리소스 메트릭(CPU/메모리)**: 각 파드에 대해 리소스 메트릭 API(metrics.k8s.io 등)에서 값을 수집한다.
    - **커스텀/외부 메트릭**: custom.metrics.k8s.io, external.metrics.k8s.io 등에서 값을 수집한다.

## 스케일링 알고리즘

```
원하는 레플리카 수 = ceil[현재 레플리카 수 * ( 현재 메트릭 값 / 원하는 메트릭 값 )]
```

- 각 파드의 컨테이너 자원 요청이 적절히 설정되어 있지 않으면 해당 파드는 계산에서 제외된다.
- 여러 메트릭을 지정한 경우, 각 메트릭별로 계산된 원하는 레플리카 수 중 가장 큰 값을 최종적으로 선택한다.
- 계산된 비율이 10% 이내 차이면 스케일링을 건너뛴다.
- 삭제 중인 파드, 실패한 파드, 아직 Ready 상태가 아닌 파드는 계산에서 제외하거나 보수적으로 처리한다. 메트릭이 누락된 파드가 있을 경우, 스케일 다운 시 해당 파드는 100% 사용 중, 스케일 업 시 0% 사용 중으로 가정해 평균을 산출한다.
- 최근 일정 시간(기본 5분) 내의 권장 레플리카 수 중 가장 큰 값을 선택해 점진적으로 파드 수를 줄인다.

## API 오브젝트

HPA는 쿠버네티스의 Autoscaling API 리소스이며 메모리와 커스텀 메트릭을 기반으로 스케일링을 지원한다.

HPA 오브젝트 생성 시 지정된 이름은 유효한 DNS 서브도메인 이름이어야 한다

## 워크로드 스케일링의 안정성

HPA를 사용하여 레플리카 그룹의 크기를 관리할 때, 메트릭의 동적 특성 때문에 레플리카 수가 자주 요동칠 수 있다.

→ 시스템이 변화를 안정적으로 반영하지 못하고 계속해서 변화하는 상황을 초래

## 롤링 업데이트 중 오토스케일링

쿠버네티스에서 디플로이먼트는 롤링 업데이트를 지원하며, HPA를 사용해 `replicas` 필드를 관리한다.

- HPA는 디플로이먼트의 레플리카 수를 조정하며, 디플로이먼트 컨트롤러는 이를 기반으로 롤아웃을 처리한다.
- 스테이트풀셋은 오토스케일링 시 레플리카셋 없이 파드 수를 직접 관리한다.

##  리소스 메트릭 지원

- HPA는 파드의 리소스 사용량을 기준으로 스케일링을 수행하며, 이를 위해 파드의 `cpu` 및 `memory`와 같은 리소스 요청을 정의해야 한다.
- HPA는 개별 컨테이너의 리소스 사용량을 기준으로 스케일링할 수 있는데, 특정 컨테이너의 리소스를 추적하려면 `ContainerResource` 메트릭 소스를 사용한다.
- 컨테이너 이름을 변경할 경우, HPA의 사양을 업데이트하여 변경된 이름을 추적하고, 롤아웃 후 이전 이름을 제거하여 정리해야 한다.

##  사용자 정의 메트릭을 이용하는 스케일링

HPA 컨트롤러는 쿠버네티스 API로부터 커스텀 메트릭을 조회하여 스케일링 결정을 내린다.

##  복수의 메트릭을 이용하는 스케일링

HPA 컨트롤러는 각 메트릭을 확인하고 새로운 스케일링 크기를 제안하며, 가장 큰 값을 선택하여 워크로드의 크기를 조정한다.

단, 이 값은 설정된 '총 최대값'을 초과하지 않는다.

## 메트릭 API를 위한 지원

HPA)를 사용하려면, 다음과 같은 기능 필요

- API 애그리게이션 레이어 활성화
- API 등록
    - **리소스 메트릭**: `metrics.k8s.io` API, 일반적으로 메트릭 서버가 제공하며 클러스터 애드온으로 실행된다.
    - **사용자 정의 메트릭**: `custom.metrics.k8s.io` API, 메트릭 솔루션 공급업체가 제공하는 "어댑터" API 서버에서 제공된다. 이를 위해 메트릭 파이프라인을 확인해야 한다.
    - **외부 메트릭**: `external.metrics.k8s.io` API, 사용하려는 어댑터에서 제공될 수 있다.

## 구성가능한 스케일링 동작

`behavior` 필드를 통해 스케일 업과 스케일 다운 동작을 별도로 구성할 수 있다.

### 스케일링 정책

- 파드 수(`type: Pods`)나 현재 레플리카 수의 비율(`type: Percent`)로 설정
- 여러 개의 정책이 정의된 경우 기본적으로 가장 큰 변화를 허용하는 정책이 적용된다.

  `selectPolicy` 필드를 통해 이러한 정책 선택 방식을 변경할 수 있는데, 가장 적은 변경을 선택하도록 하거나(`Min`), 해당 방향의 스케일링을 아예 비활성화(`Disabled`)할 수도 있다.


### 안정화 윈도우

- 메트릭이 자주 변동할 때 레플리카 수가 불안정하게 계속 바뀌는 현상을 막기 위해 사용
- 설정된 시간 동안 계산된 목표 중 가장 높은 값을 참조함으로써, 스케일 다운 시 불필요한 파드 삭제와 재생성을 방지
- 스케일 업에는 기본적으로 안정화 윈도우가 적용되지 않기 때문에, 메트릭이 필요하다고 판단되면 즉시 스케일 업이 이루어진다.

## kubectl의 HorizontalPodAutoscaler 지원

HPA는 다른 Kubernetes 리소스들과 마찬가지로 `kubectl` 명령어로 관리할 수 있다.

- `kubectl create` : 새 HPA를 생성
- `kubectl get hpa` : 목록을 조회
- `kubectl describe hpa` : 상세 정보를 확인
- `kubectl delete hpa` : 삭제
- `kubectl autoscale` : HPA를 보다 간편하게 생성할 수 있는 전용 명령어

##  암시적 점검 모드(maintenance-mode) 비활성화

HPA를 직접 수정하지 않고도 대상 리소스를 암시적으로 비활성화할 수 있다.

- 대상의 `replicas`가 0으로 설정되고, HPA의 `minReplicas`가 0보다 크면, HPA는 자동으로 동작을 멈추고 `ScalingActive=false`로 설정된다.
- 다시 활성화하려면 `replicas` 또는 HPA 설정을 수동으로 조정해야 한다.

### 디플로이먼트/스테이트풀셋에 HPA 적용 시 주의사항

- HPA를 사용하는 경우, 매니페스트 파일에서 `spec.replicas` 값을 삭제

  →  제거하지 않으면 `kubectl apply` 실행 시 해당 값으로 파드 수가 덮어씌워져 HPA 동작에 문제가 생길 수 있다.

- `spec.replicas`를 삭제하면 기본값 1이 적용되어 일시적으로 파드 수가 1개로 줄어들 수 있다.
