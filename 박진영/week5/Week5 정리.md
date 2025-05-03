# Week5

# 컨피그맵(ConfigMap)

- 키-값 쌍으로 **기밀이 아닌 데이터**를 저장하는 데 사용하는 API 오브젝트
    - **보안/암호화를 제공하지 않기** 때문에, **기밀이라면 시크릿(Secret)이나 3rd party 도구를 사용**하기 바람
- 파드는 볼륨에서 환경변수, 커맨드라인 인수나 구성파일로 컨피그맵 사용
- 컨테이너 이미지에서 환경별 구성을 분리해서 애플리케이션에 쉽게 이식하게 되움
- 많은 양의 데이터를 보유하도록 설계되지 않았음
    - 데이터 : 1MiB 초과할 수 없음
    - 초과 시, 볼륨 마운트나 별도의 DB, 파일 서비스 사용을 고려해야 함

## 컨피그맵 오브젝트

- 컨피그맵에는 `data` 및 `binaryData` 필드가 선택사항으로 있음
    - `data`는 UTF-8 문자열을 포함하도록 설계됨
    - `binaryData`는 바이너리 데이터를 base64 encoded characters로 포함하게끔 설게됨
    - 각 필드 하의 키는 영숫자 문자, -, _, .으로 구성되어야 함.
    - `data`의 키는 `binaryData` 키와 겹치면 안된다.
- 컨피그맵 이름은 유효한 DNS 서브도메인 이름이어야 함
- 컨피그맵 정의에 `immutable` 필드를 추가하면 변경 불가능한 컨피그맵이 생성된다.

## 컨피그맵과 파드

- 컨피그맵을 참조하는 파드 `spec`를 작성하고 컨피그맵 데이터 기반으로 해당 파드의 컨테이너 구성 가능
    - 파드와 컨피그맵은 동일한 네임스페이스 하에 있어야 한다.
    - static pod의 spe은 컨피그맵이나 다른 API 오브젝트를 참조할 수 없음
- 컨피그맵으로 파드 내부에 컨테이너 구성 가능한 네가지 방법
    1. 컨테이너 커맨드와 인수 내에서
    2. 컨테이너에 대한 환경 변수
    3. 애플리케이션이 읽을 수 있도록 읽기 전용 볼륨에 파일 추가
    4. 쿠버네티스 API를 사용하여 컨피그맵을 읽는 파드 내에서 실행할 코드 작성
        - 컨피그맵과 데이터를 읽기 위해 코드를 작성해야 함을 의미
- 컨피그맵은 단일 라인 속성(single line property) 값과 멀티 라인의 파일과 비슷한(multi-line file-like) 값을 구분하지 않음
- 

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
        # 환경 변수 정의
        - name: PLAYER_INITIAL_LIVES # 참고로 여기서는 컨피그맵의 키 이름과
          # 대소문자가 다르다.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 이 값의 컨피그맵.
              key: player_initial_lives # 가져올 키.
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
    # 파드 레벨에서 볼륨을 설정한 다음, 해당 파드 내의 컨테이너에 마운트한다.
    - name: config
      configMap:
        # 마운트하려는 컨피그맵의 이름을 제공한다.
        name: game-demo
        # 컨피그맵에서 파일로 생성할 키 배열
        items:
          - key: "game.properties"
            path: "game.properties"
          - key: "user-interface.properties"
            path: "user-interface.properties"

```

- 볼륨 정의 및 demo 컨테이너에 /config로 마운트하면 컨피그맵에 key가 4개가 있어도 `/config/game.properties` 와 `/config/user-interface.properties` 2개의 파일이 생성됨
    
    → 파드 정의가 `volume` 섹션에서 `items` 배열을 지정하기 때문
    → `items` 배열을 완전히 생략하면, 컨피그맵의 모든 키가 키와 이름이 같은 파일이 되고, 4개의 파일을 얻게 됨
    

## 컨피그맵 사용

- 컨피그맵은 데이터 볼륨으로 마운트할 수 있음
- 파드에 직접적으로 노출되지 않고, 시스템의 다른 부분에서도 사용할 수 있음
- 일반적인 사용 방법 : 동일 네임스페이스 파드에서 실행되는 컨테이너에 대한 설정을 구성하는 것
- 컨피그맵을 별도로 사용할 수도 있음
- 파드의 볼륨에서 컨피그맵을 마운트하는 예시
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: mypod
    spec:
    containers:
      -name: mypod
    image: redis
    volumeMounts:
        -name: foo
    mountPath: "/etc/foo"
    readOnly:truevolumes:
      -name: foo
    configmap:
    name: myconfigmap
    ```
    
    - 사용하려는 각 컨피그맵은 `.spec.volumes` 에서 참조해야 함
    - 파드에 여러 컨테이너가 있는 경우 각 컨테이너에는 자체 `volumeMounts` 블록이 필요하지만, 컨피그맵은 각 컨피그맵 당 하나의 `.spec.volumes` 만 필요

## **변경할 수 없는(immutable) 컨피그맵**

- *변경할 수 없는 시크릿과 컨피그맵* 은 개별 시크릿과 컨피그맵을 변경할 수 없는 것으로 설정하는 옵션을 설정함
    - `ImmutableEphemeralVolumes` 기능 게이트에 의해 제어
    - `immutable` 필드를 `true` 로 설정하여 변경할 수 없는 컨피그맵을 생성
- 장점
    - 애플리케이션 중단을 일으킬 수 있는 우발적(또는 원하지 않는) 업데이트로부터 보호
    - immutable로 표시된 컨피그맵에 대한 감시를 중단하여, kube-apiserver의 부하를 크게 줄임으로써 클러스터의 성능을 향상시킴
- 컨피그맵을 immutable로 표시하면, 이 변경 사항을 되돌리거나 `data` 또는 `binaryData` 필드 내용을 **변경할 수 *없음.***
컨피그맵만 삭제하고 다시 작성할 수 있음
→ 기존 파드는 삭제된 컨피그맵에 대한 마운트 지점을 유지하므로, 이러한 파드를 다시 작성하는 것을 권장

# 시크릿(Secret)

: 암호, 토큰 또는 키와 같은 소량의 중요한 데이터를 포함하는 오브젝트

- 사용하지 않을 경우 주요 정보가 파드 명세나 컨테이너 이미지에 포함될 수 있음
- 시크릿을 쓰는 파드와 독립적으로 생성될 수 있어, 파드 관련 워크플로우 동안 시크릿이 노출되는 위험을 줄인다.
- 기본적으로 API 서버의 기본 데이터 저장소(etcd)에 암호화되지 않은 상태로 저장됨.
    - API 접근(access) 권한이 있는 모든 사용자 또는 etcd에 접근할 수 있는 모든 사용자는 시크릿을 조회하거나 수정 가능
    - 네임스페이스에서 파드를 생성할 권한이 있는 사람은 누구나 해당 접근을 사용하여 해당 네임스페이스의 모든 시크릿을 읽을 수 있음
        - 간접 접근 포함(e.g. 디플로이먼트 생성 기능)
- 안전하게 사용하는 단계
    1. 저장된 데이터 암호화(Encription at Rest) 활성화
    2. 최소 접근 권한을 지니개 RBAC 규칙을 활성화 또는 구성
    3. 특정 컨테이너에서만 시크릿에 접근케 함
    4. 외부 시크릿 저장소 제공 서비르르 사용하는 걸 고려

## 파드가 시크릿을 사용하는 방법 세 가지

1. 하나 이상의 컨테이너에 마운트된 볼륨 내의 파일로써 사용
2. 컨테이너 환경 변수로써 사용
3. 파드의 이미지를 가져올 때 kubelet에 의해 사용

+) 컨트롤 플레인도 시크릿을 씀
(e.g. 부트스트랩 토큰 시크릿은 노드 등록을 자동화하는 데 도움을 주는 메커니즘)

## 시크릿의 대안(이걸 시크릿과 조합하여 쓸 수도 있음)

- 클라우드 네이티브 구성 요소가 동일 쿠버네티스 클러스터 내에 있는 다른 애플리케이션에 인증해야하는 경우, 서비스 어카운트 및 그의 토큰을 이용해 클라이언트 식별 가능
- 비밀 정보 관리 기능을 제공하는 써드파티 도구를 클러스터 내부 또는 외부에 실행 가능
- 인증을 위해, X.509 인증서를 위한 커스텀 인증자(signer)를 구현하고, CertificateSigningRequests를 사용하여 해당 커스텀 인증자가 인증서를 필요로 하는 파드에 인증서를 발급하게 함
- 장치 플러그인을 사용하여 노드에 있는 암호화 하드웨어를 특정 파드에 노출할 수 있음

## 시크릿 다루기

### 시크릿 생성

1. kubectl 사용
2. 환경설정파일 사용
3. kustomize 도구 사용

### 시크릿 수정

- 불변(immutable)만 아니면 수정 가능
- kubectl로 수정
- 설정 파일로 수정
- kustomize 도구로 수정→이 경우에는 새로운 `Secret` 오브젝트가 생성되게 됨

### 시크릿 사용

- 시크릿은 데이터 볼륨으로 마운트되거나 파드의 컨테이너에서 사용할 [환경 변수](https://kubernetes.io/ko/docs/concepts/containers/container-environment/)로 노출될 수 있음
- 파드에 직접 노출되지 않고, 시스템의 다른 부분에서도 사용할 수 있음
- 특정된 오브젝트 참조(reference)가 실제로 시크릿 유형의 오브젝트를 가리키는지 확인하기 위해, 시크릿 볼륨 소스의 유효성이 검사됨
    
    → 시크릿은 자신에 의존하는 파드보다 먼저 생성되어야 함
    
- 선택적 시크릿
    - 기본적으로 시크릿은 필수 사항
    - 시크릿으로 컨테이너 환경 변수 정의 시, 선택사항으로 표시 가능

### 파드에서 시크릿을 파일처럼 사용하기

- 파드 안에서 시크릿의 데이터에 접근하고 싶다면, 한 가지 방법은 쿠버네티스로 하여금 해당 시크릿의 값을 파드의 하나 이상의 컨테이너의 파일시스템 내에 파일 형태로 표시하도록 만드는 것
    - 구성 단계
    1. 시크릿을 생성하거나 기존 시크릿을 사용한다. 여러 파드가 동일한 시크릿을 참조할 수 있다.
    2. `.spec.volumes[].` 아래에 볼륨을 추가하려면 파드 정의를 수정한다. 볼륨의 이름을 뭐든지 지정하고, 시크릿 오브젝트의 이름과 동일한 `.spec.volumes[].secret.secretName` 필드를 생성한다.
    3. 시크릿이 필요한 각 컨테이너에 `.spec.containers[].volumeMounts[]` 를 추가한다. 시크릿을 표시하려는 사용되지 않은 디렉터리 이름에 `.spec.containers[].volumeMounts[].readOnly = true`와 `.spec.containers[].volumeMounts[].mountPath`를 지정한다.
    4. 프로그램이 해당 디렉터리에서 파일을 찾도록 이미지 또는 커맨드 라인을 수정한다. 시크릿 `data` 맵의 각 키는 `mountPath` 아래의 파일명이 된다.
    

### 특정 경로에 대한 시크릿 키 투영

`.spec.volumes[].secret.items` 를 사용하면, `items` 에 지정된 키만 투영
키를 명시적으로 나열했다면, 나열된 모든 키가 해당 시크릿에 존재해야 하며, 그렇지 않으면 볼륨이 생성되지 않음

### **시크릿 파일 퍼미션**

- 단일 시크릿 키에 대한 POSIX 파일 접근 퍼미션 비트(기본 `0644`)를 설정할 수 있음
- 전체 시크릿 볼륨에 대한 기본 모드를 설정하고 필요한 경우 키별로 오버라이드 가능
    
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
          mountPath: "/etc/foo" #시크릿이 마운트 되는 경로
      volumes:
      - name: foo
        secret:
          secretName: mysecret
          defaultMode: 0400 #시크릿 볼륨 마운트로 생성된 모든 파일 퍼미션
    ```
    
- JSON으로 파드/파드 템플릿 정의 시, 8진수 표기가 지원되지 않음.
defaultMode 값으로 10진수를 사용하기.
    - YAML은 8진수 표기 가능

### **시크릿을 환경 변수 형태로 사용**

1. 시크릿 생성 또는 기존 시크릿을 사용(여러 파드가 동일 시크릿 참조 가능)
2. 사용하려는 각 시크릿 키에 대한 환경 변수를 추가하려면 시크릿 키 값을 사용하려는 각 컨테이너에서 파드 정의를 수정함.
시크릿 키를 사용하는 환경 변수는 시크릿의 이름과 키를 `env[].valueFrom.secretKeyRef` 에 채워야 함
3. 프로그램이 지정된 환경 변수에서 값을 찾도록 이미지 및/또는 커맨드 라인을 수정
- 올바르지 않은 환경변수면 그 키는 건너뛰어지지만 파드를 시작할 수는 있음.
    - `reason`이 `InvalidVariableNames`로 설정된 이벤트와 유효하지 않은 스킵된 키 목록 메시지가 파드 시작 실패 정보에 추가됨
    - `kubectl get events` 로 확인 가능

### 컨테이너 이미지 풀 시크릿

비공개 저장소에서 컨테이너 이미지를 가져오고 싶다면, 각 노드의 kubelet이 해당 저장소에 인증을 수행하기 위해 이미지 풀 시크릿 구성 가능

- 파드 수준에서 설정됨
- 파드의 `imagePullSecrets` 필드를 사용해 이미지 저장소 접근 크리덴셜을 kubelet에 전달하고, kubelet이 파드 대신 비공개 이미지를 가져오게 된다.

## 사용 사례

### 컨테이너 환경 변수로 사용하기

- 시크릿 정의 작성 후 `kubectl apply -f {정의한 파일명}.yaml`
- 모든 시크릿 데이터를 컨테이너 환경 변수로 정의하는 데 `envFrom`을 사용함
- 시크릿 키는 파드의 환경 변수 이름이 됨

### SSH 키를 사용하는 파드

- 몇 가지 SSH 키를 포함하는 시크릿 생성
    
    `kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/path/to/.ssh/id_rsa --from-file=ssh-publickey=/path/to/.ssh/id_rsa.pub`
    
- SSH 키를 포함하는 `secretGenerator` 필드가 있는 `kustomization.yaml` 를 만들 수도 있음
    - 다른 사용자가 시크릿에 접근할 수 있기에, SSH 송신 전 신중하게 생각하기
    - 쿠버네티스 클러스터를 공유하는 모든 사용자가 접근할 수 있으면서도, 크리덴셜이 유출된 경우 폐기가 가능한 서비스 아이덴티티를 가리키는 SSH 프라이빗 키를 만드는 게 대안이 될 수 있음
- SSH 키를 가진 시크릿을 참조하고 볼륨에서 시크릿을 사용하는 파드 생성 가능
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: secret-test-pod
      labels:
        name: secret-test
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: ssh-key-secret
      containers:
      - name: ssh-test-container
        image: mySshImage
        volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"
    ```
    
- 컨테이너 명령이 실행될 때, 다음 경로에서 키 부분을 사용 가능
    
    ```bash
    /etc/secret-volume/ssh-publickey
    /etc/secret-volume/ssh-privatekey
    ```
    

### 운영/테스트 자격 증명을 사용하는 파드

운영 환경의 자격 증명이 포함된 시크릿을 사용하는 파드
vs. 테스트 환경의 자격 증명이 있는 시크릿을 사용하는 다른 파드

- `secretGenerator` 필드가 있는 `kustomization.yaml` 을 만들거나 `kubectl create secret` 을 실행
    
    `kubectl create secret generic prod-db-secret --from-literal=username=produser --from-literal=password=Y4nys7f11`
    
- 테스트 환경의 자격 증명에 대한 시크릿 생성
    
    `kubectl create secret generic test-db-secret --from-literal=username=testuser --from-literal=password=iluvtests`
    
- $, \, *, =, ! 같은 특수문자는 사용자의 Shell에 의해 해석되고 escaping이 요구됨.
→ 따옴표(’)로 묶어야 함.
**`kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb='`**
    - 파일(`--from-file`)에서는 비밀번호의 특수 문자를 이스케이프할 필요가 없음
1. 파드 생성
    
    ```bash
    cat <<EOF > pod.yaml
    apiVersion: v1
    kind: List
    items:
    - kind: Pod
      apiVersion: v1
      metadata:
        name: prod-db-client-pod
        labels:
          name: prod-db-client
      spec:
        volumes:
        - name: secret-volume
          secret:
            secretName: prod-db-secret
        containers:
        - name: db-client-container
          image: myClientImage
          volumeMounts:
          - name: secret-volume
            readOnly: true
            mountPath: "/etc/secret-volume"
    - kind: Pod
      apiVersion: v1
      metadata:
        name: test-db-client-pod
        labels:
          name: test-db-client
      spec:
        volumes:
        - name: secret-volume
          secret:
            secretName: test-db-secret
        containers:
        - name: db-client-container
          image: myClientImage
          volumeMounts:
          - name: secret-volume
            readOnly: true
            mountPath: "/etc/secret-volume"
    EOF
    ```
    
2. 동일한 `kustomization.yaml`에 파드 추가
    
    ```bash
    cat <<EOF >> kustomization.yaml
    resources:
    - pod.yaml
    EOF
    ```
    
3. API 서버에 이런 모든 오브젝트를 적용함
    
    ```bash
    kubectl apply -k .
    ```
    
    - 두 컨테이너 모두 각 컨테이너의 환경에 대한 값을 가진 파일시스템에 다음의 파일이 존재
    `/etc/secret-volume/username
     /etc/secret-volume/password`
4. 두 파드의 사양이 한 필드에서만 어떻게 다른지 확인
    1. `prod-db-secret` 을 가진 `prod-user`
    2. `test-db-secret` 을 가진 `test-user` 
5. 파드 명세 단축
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: prod-db-client-pod
    labels:
    name: prod-db-client
    spec:
    serviceAccount: prod-db-client
    containers:
      -name: db-client-container
    image: myClientImage
    ```
    

### **시크릿 볼륨의 도트 파일(dotfile)**

- ‘.’으로 시작하는 키를 정의 → 데이터를 “숨김” 처리할 수 있음

```yaml
apiVersion: v1
kind: Secret
metadata:
name: dotfile-secret
data:
.secret-file: dmFsdWUtMg0KDQo= #하나의 파일을 포함하는 볼륨
---apiVersion: v1
kind: Pod
metadata:
name: secret-dotfiles-pod
spec:
volumes:
  -name: secret-volume
secret:
secretName: dotfile-secret
containers:
  -name: dotfile-test-container#이 컨테이너는
image: registry.k8s.io/busybox
command:
    - ls
    - "-l"
    - "/etc/secret-volume"
volumeMounts:
    -name: secret-volume
readOnly:truemountPath: "/etc/secret-volume"#이 경로 하의 /.secret-file에 이 파일을 갖게 됨
```

### **파드의 한 컨테이너에 표시되는 시크릿**

- HTTP 요청을 처리 → 비즈니스 로직 수행 → HMAC가 있는 일부 메시지에 서명해야 하는 프로그램일 때 고려됨
- 두 개의 컨테이너의 두 개 프로세스로 나눌 수 있음
    - 프론트엔드 컨테이너 : 사용자 상호작용과 비즈니스 로직 처리. 개인키 확인 불가.
    - 서명자 컨테이너 : 프론트엔드의 간단한 서명 요청(localhost 네트워킹 등을 통해)에 응답
- 공격자가 공격하기 더 어렵게 만듦.

## 시크릿 타입

| 빌트인 타입 | 사용처 |
| --- | --- |
| `Opaque` | 임의의 사용자 정의 데이터(기본값) |
| `kubernetes.io/service-account-token` | 서비스 어카운트 토큰 |
| `kubernetes.io/dockercfg` | 직렬화 된(serialized) `~/.dockercfg` 파일 |
| `kubernetes.io/dockerconfigjson` | 직렬화 된 `~/.docker/config.json` 파일 |
| `kubernetes.io/basic-auth` | 기본 인증을 위한 자격 증명(credential) |
| `kubernetes.io/ssh-auth` | SSH를 위한 자격 증명 |
| `kubernetes.io/tls` | TLS 클라이언트나 서버를 위한 데이터 |
| `bootstrap.kubernetes.io/token` | 부트스트랩 토큰 데이터 |
- 공개 사용을 위한 시크릿 타입을 정의하는 경우, 규칙에 따라 `cloud-hosting.example.net/cloud-api-credentials`와 같이 시크릿 타입 이름 앞에 도메인 이름 및 `/`를 추가하여 전체 시크릿 타입 이름을 구성

### 불투명(Opaque) 시크릿

- `Opaque` : 시크릿 구성 파일에서 시크릿 타입을 지정하지 않았을 경우의 기본 시크릿 타입
- `kubectl`로 시크릿 생성 시 `generic` 하위 커맨드 사용
    
    ```
    kubectl create secret generic empty-secret
    kubectl get secret empty-secret
    ```
    

### 서비스 어카운트 토큰 시크릿

`kubernetes.io/service-account-token` 시크릿 타입은 서비스 어카운트를 확인하는 토큰 자격 증명을 저장하고자 쓰임

→ **TokenRequest API**로 토큰을 얻기를 권장(`kubectl create token`)

### 도커 컨피그 시크릿

이미지에 대한 도커 registry 접속 자격 증명을 저장하기 위한 시크릿을 생성하기 위해, 다음 type 값 중 하나 사용(둘 다 base64로 콘텐츠를 인코딩해야 함)

- `kubernetes.io/dockercfg`
- `kubernetes.io/dockerconfigjson`
    - 크릿 오브젝트의 `data` 필드가 `.dockerconfigjson` 키를 꼭 포함

+)**base64 인코딩 수행을 원하지 않는다면, 그 대신 `stringData` 필드를 사용**

### 기본 인증(Basic authentication) 시크릿

`kubernetes.io/basic-auth` 타입은 기본 인증을 위한 자격 증명을 저장하기 위해 제공됨

시크릿의 `data` 필드가 다음의 두 키 중 하나를 포함해야 함
(값은 모두 base64로 인코딩된 문자열 또는 `stringData` 사용)

- `username`: 인증을 위한 사용자 이름
- `password`: 인증을 위한 암호나 토큰

다른 사람이 이 시크릿의 목적을 이해하는 데 도움이 되며, 예상되는 키 이름에 대한 규칙이 설정됨

### SSH 인증 시크릿

빌트인 타입 `kubernetes.io/ssh-auth` 는 SSH 인증에 사용되는 데이터를 저장하기 위해서 제공됨

`ssh-privatekey` 키-값 쌍을 사용할 SSH 자격 증명으로 `data` (또는 `stringData`) 필드에 명시해야 할 것

- Opaque로 시크릿 생성도 가능하지만,
이걸 사용하면 다른 사람이 시크릿 목적을 이해하는 데 도움이 됨. 그리고 예상되는 키 일므에 대한 규칙이 설정되며, API 서버가 시크릿 정의에 필수 키가 명시되었는지 확인함.
- 오직 사용자 편의를 위해 제공됨
- 자체적으로 SSH 클라이언트와 호스트 서버 간에 신뢰할 수 있는 통신을 설정하지는 않음
→ ConfigMap에 추가된 파일 같이 “**중간자**” 공격을 완화하려면 신뢰를 설정하는 2차 수단이 요구됨

### TLS시크릿

- `kubernetes.io/tls`
- TLS 시크릿의 일반적인 용도
    - 인그레스에 대한 전송 암호화(encription in transit) 구성
- TLS 시크릿 : 다른 리소스와 함께 사용하거나 워크로드에서 직접 사용 가능
    - `tls.key` 와 `tls.crt` 키가 시크릿 구성의 `data` (또는 `stringData`) 필드에서 제공되어야 함
    - API 서버가 각 키에 대한 값이 유효한지 실제로 검증하지는 않음
- 공개/개인 키 쌍은 사전에 준비되어야 함
    
    ```bash
    kubectl create secret tls my-tls-secret\
      --cert=path/to/cert/file\ #공개 키 인증서
      --key=path/to/key/file #개인 키 인증서
    ```
    
- `kubernetes.io/tls` 시크릿은 키 및 인증서에 Base64 인코드된 DER 데이터를 보관함
    - 이 base64 데이터는 PEM 형식에서 맨 윗줄과 아랫줄을 제거한 것과 동일

### **부트스트랩 토큰 시크릿**

- `type: bootstrap.kubernetes.io/token`
- 잘 알려진 컨피그맵에 서명하는 데 사용되는 토큰을 저장
- 보통 kube-system 네임스페이스에 생성되며 <token-id> 가 해당 토큰 ID의 6개 문자의 문자열으로 구성된 bootstrap-token-<token-id> 형태로 이름이 지정됨

## 불변(immutable) 시크릿

- 잘못된(또는 원치 않은) 업데이트를 차단하여 애플리케이션 중단을 방지
- (수만 개 이상의 시크릿-파드 마운트와 같이 시크릿을 대규모로 사용하는 클러스터의 경우,) 불변 시크릿으로 전환하면 kube-apiserver의 부하를 크게 줄여 클러스터의 성능을 향상시킬 수 있음.
kubelet은 불변으로 지정된 시크릿에 대해서는 [감시(watch)]를 유지할 필요가 없기 때문.

```yaml
apiVersion: v1
kind: Secret
metadata:
  ...
data:
  ...
immutable:true
```

- 기존의 수정 가능한 시크릿을 변경하여 불변 시크릿으로 바꿀 수도 있음
- 한번 불변으로 바꾸면 이 변경을 취소할 수 없음. 재생성 추천

## 시크릿을 위한 정보 보안(Information security)

### **Secret vs ConfigMap 차이 및 보호 방식 (Kubernetes)**

1. **Secret과 ConfigMap은 유사하지만**,
    
    Secret은 **민감한 정보 보호**를 위해 **추가 보안 조치**가 적용됨.
    
2. **Secret의 특징**:
    - 서비스 어카운트 토큰 등 **보안 수준이 다양한 데이터**를 포함할 수 있음.
    - **요청된 파드가 있는 노드에만 전달**됨.
    - **tmpfs 메모리**에 저장되어 디스크에 기록되지 않음.
    - 파드가 종료되면 kubelet이 Secret의 **로컬 복사본을 자동 삭제**함.
3. **파드 내 컨테이너의 Secret 접근**:
    - 기본적으로는 **자기 서비스 어카운트에 해당하는 Secret만 접근** 가능.
    - 다른 Secret을 사용하려면 **명시적 환경변수 또는 볼륨 마운트** 설정 필요.
4. **보안 격리 보장**:
    - 같은 노드에 있는 다른 파드의 Secret은 **볼 수 없음**.
    - 하나의 파드는 **자신이 요청한 Secret만 사용 가능**.

# 시크릿(Secret)을 사용하여 안전하게 자격증명 배포하기

: 암호 및 암호화 키와 같은 민감한 데이터를 파드에 안전하게 주입하는 방법

준비

- 쿠버네티스 클러스터
- kubectl CLI 툴이 클러스터와 통신 가능한 상태
- 노드가 적어도 2개 포함된 클러스터에서 실행
    - 클러스터가 없으면 minikube로 사용/생성하거나 쿠버네티스 플레이그라운드로 진행

## 시크릿 데이터→base-64 로 변환

```bash
echo -n 'my-app' | base64
echo -n '39528$vdg7Jb' | base64
```

## 시크릿 생성

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  username: bXktYXBw
  password: Mzk1MjgkdmRnN0pi
```

1. 시크릿 생성
    
    `kubectl apply -f [https://k8s.io/examples/pods/inject/secret.yaml](https://k8s.io/examples/pods/inject/secret.yaml)`
    
2. 시크릿 정보 확인
    
    `kubectl get secret test-secret`
    
3. 상세 정보 확인
    
    `kubectl describe secret test-secret`
    

### base64 인코딩 단계를 건너뛰고자 kubectl로 직접 시크릿 생성

`kubectl create secret generic test-secret --from-literal='username=my-app' --from-literal='password=39528$vdg7Jb’`

## 볼륨으로 시크릿 데이터에 접근 가능한 파드 생성하기

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
        # name must match the volume name below
        - name: secret-volume
          mountPath: /etc/secret-volume
  # The secret data is exposed to Containers in the Pod through a Volume.
  volumes:
    - name: secret-volume
      secret:
        secretName: test-secret

```

1. 파드 생성하기
    
    `kubectl apply -f [https://k8s.io/examples/pods/inject/secret-pod.yaml](https://k8s.io/examples/pods/inject/secret-pod.yaml)`
    
2. 파드 실행 여부 확인
    
    `kubectl get pod secret-test-pod`
    
3. 파드에서 실행 중인 컨테이너 셸 가져오기
    
    `kubectl exec -i -t secret-test-pod -- /bin/bash`
    
4. 시크릿데이터는 /etc/secret-volume에 마운트된 볼륨으로 컨테이너에 노출됨.
셸에서 해당 경로의 파일 나열하기
    
    ```bash
    # 컨테이너 내부의 셸에서 실행하자
    ls /etc/secret-volume
    ```
    
5. 셸에서 `username` 및 `password` 파일의 내용을 출력
    
    ```bash
    # 컨테이너 내부의 셸에서 실행하자
    echo "$( cat /etc/secret-volume/username )"
    echo "$( cat /etc/secret-volume/password )"
    ```
    

## 시크릿 데이터를 사용해 컨테이너 환경변수 정의

### 단일 시크릿 데이터로 컨테이너 환경 변수 정의

- 환경 변수를 시크릿의 키-값 쌍으로 정의
    
    `kubectl create secret generic backend-user --from-literal=backend-username='backend-admin’`
    
- 시크릿에 정의된 `backend-username` 값을 파드 명세의 `SECRET_USERNAME` 환경 변수에 할당
    
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
                 name: backend-user
                 key: backend-username
    ```
    
- 파드 생성 및 환경변수 내용 출력
    
    `kubectl create -f [https://k8s.io/examples/pods/inject/pod-single-secret-env-variable.yaml](https://k8s.io/examples/pods/inject/pod-single-secret-env-variable.yaml)`
    
    `kubectl exec -i -t env-single-secret -- /bin/sh -c 'echo $SECRET_USERNAME’`
    

### 여러 시크릿 데이터로 컨테이너 환경 변수 정의

1. 시크릿 생성
    
    ```bash
    kubectl create secret generic backend-user --from-literal=backend-username='backend-admin'
    kubectl create secret generic db-user --from-literal=db-username='db-admin'
    ```
    
2. 파드 명세에 환경 변수 정의
    
    ```yaml
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
    
3. 파드 생성 및 컨테이너 환경변수 출력
    
    `kubectl create -f https://k8s.io/examples/pods/inject/pod-multiple-secret-env-variable.yaml`
    
    `kubectl exec -i -t envvars-multiple-secrets -- /bin/sh -c 'env | grep _USERNAME’`
    

## 시크릿의 모든 키-값 쌍을 컨테이너 환경 변수로 구성하기(k8s v1.6 이상)

1. 여러 키-값 쌍을 포함하는 시크릿 생성
    
    `kubectl create secret generic test-secret --from-literal=username='my-app' --from-literal=password='39528$vdg7Jb’`
    
2. envFrom으로 시크릿의 모든 데이터를 컨테이너 환경변수로 정의함. 시크릿 키는 파드에서 환경 변수의 이름이 됨
    
    ```yaml
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
    
3. 파드 생성 및 환경 변수 출력
    
    `kubectl create -f https://k8s.io/examples/pods/inject/pod-secret-envFrom.yaml`
    
    `kubectl exec -i -t envfrom-secret -- /bin/sh -c 'echo "username: $username\npassword: $password\n"’`