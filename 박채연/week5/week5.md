# 컨피그맵 ~ 시크릿

> 한줄평 : 
컨피그맵과 시크릿을 사용하면 , 애플리케이션 코드 배포를 새로 할 필요 없이 ,
 pod 중단 없이 , 언제든지 수정사항을 적용시킬 수 있다. 완전 good
> 

# Kubernetes

하위 문서 정리

https://kubernetes.io/ko/docs/concepts/configuration/configmap/

https://kubernetes.io/ko/docs/concepts/configuration/secret/

https://kubernetes.io/ko/docs/tasks/inject-data-application/distribute-credentials-secure/

배경 

쿠버네티스에서는 설정 값을 하드코딩하지 말고 외부에서 주입하는 것을 권장한다. 

- 설정만 바꾸고 싶을때 , 컨테이너 이미지를 교체할 필요없고, 민감한 정보는 따로 안전하게 관리할 수 있다.
- 일반 설정값은 `ConfigMap`, 민감한 값은 `Secret`이렇게 나눠서 사용

# Configmap

> 키-값 쌍으로 기밀이 아닌 데이터를 저장하는데  사용하는 API 오브젝트 
파드는 볼륨에서 환경 변수, 커맨드-라인 인수 또는 구성파일로 컨피그맵을 사용할 수 있다.
> 

장점 : 컨테이너 이미지에서 환경별 구성을 분리하여, 애플리케이션을 쉽게 이식할 수 있다. 

단점 : 컨피그맵은 보안/ 암호화를 제공하지 않으므로, 기밀데이터를 저장하려는 경우 컨피그 맵 대신 시크릿(Secret)을 사용해라!

## 사용 동기

애플리케이션 코드는 그대로 두고, 실행환경에 따라 설정만 바꿔서 유연하게 배포하기 위함

[예시]

1. 코드에서는 단순히 System.getenv(”DataBase_HOST”) , process.env.DATABASE_HOST 등으로 읽고만 있음. 
2. 로컬에서는 localhost, 클라우드에서는 my-db-service로 설정
3. 코드는 그대로→ 환경만 바꿔도 잘 동작

- 한계 : 1Mib 제한
    - ConfigMap의 최대 저장용량은 1MiB이다
    - why? : etcd(쿠버네티스 내부 저장소 성능을 보호하기 위해 제한한다.
    - 더 많은 양을 저장해야하는 경우 볼륨을 마운트 하는 것을 고려하기

<aside>
💡

12-Factor App 원칙
중 하나인 ~ 

[Store config in the environment](https://12factor.net/config)

“**설정 정보는 환경 변수에 저장한다.” 를 따르는 좋은 예시이다.** 

</aside>

## Configmap 오브젝트

> 다른 오브젝트가 사용할 구성(환경설정 파일)을 저장할 수 있는 API 오브젝트 
”시스템 전체가 공유해서 참고하는 환경 설정 파일”
> 
- `spec`이 있는 대부분의 쿠버네티스 오브젝트와 달리, 컨피그맵에는 `data`  `binaryData` 필드가 존재
- 위 필드는 키-값 쌍을 값으로 허용하며, optional임
- data : UTF -8 문자열 포함
- binaryData : base64 인코딩된 값(인증서, 키 등)

⇒ 컨피그맵의 이름은 유효한 DNS 서브도메인 이름이어야함

- 용어정리
    
    spec 필드란?
    
    - 리소스의 핵심 동작 정의가 담기는 곳 ex) Pod.spec : 어떤 컨테이너를 띄울지
    

## Configmap과 파드

컨피그맵을 참조하는 파드 `spec`을 작성하고 컨피그맵 데이터를 기반으로 해당 파드의 컨테이너를 구성할 수 있다. 

- 파드와 컨피그맵은 동일한 네임스페이스에 있어야 한다.
- ⚠️ **static 파드**의 spec은 컨피그맵 또는 다른 API 오브젝트를 참조할 수 없다.

 

> static 파드 :
> 
> 
> **kubelet이 직접 실행하는 Pod**
> 
> →**API 서버와는 상관없이. 노드의 로컬 파일 시스템에 있는 YAML 파일을 보고 실행**
> 
> 💡 일반 Pod: kubectl apply → API 서버 → 스케줄링
> 💡 Static Pod: /etc/kubernetes/manifests/ 폴더에 YAML 넣으면 kubelet이 직접 실행
> 
> ⇒ 때문에 static 파드의 설정을 변경하려면, 
> 
> - ConfigMap 못 쓰니까, **YAML에 직접 값을 적어야 하거나**
> - 혹은 **외부 환경 변수나 파일로 직접 주입**해야 한다. (Docker 방식처럼)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: game-demo
data:
# 속성과 비슷한 키; 각 키는 간단한 값으로 매핑됨
	player_initial_lives: "3"
	ui_properties_file_name: "user-interface.properties"

# 파일과 비슷한 키
	game.properties: |    
		enemy.types=aliens,monsters
		player.maximum-lives=5
	user-interface.properties: |    
		color.good=purple
		color.bad=yellow
		allow.textmode=true
```

**[**Configmap으로 **pod 컨테이너를 어떻게 구성하는가? 4가지방]**

1. 컨테이너 커맨드와 인수 내에서 사용하기 (command/args) : 실행 명령어 자체에 configMap 주
2. 환경 변수로 주입하기 (`env`) 
    - Pod Yaml 파일에 정의된 값을 주입해서 애플리케이션이 사용하게 하는 방법
    - 이때, 실제 정의는 configmap에 해두고 yaml 파일에서는 configmap을 참조하는 방식으로 사용
3. 애플리케이션이 읽을 수 있는 파일 형태로 컨테이너 볼륨에 마운트
    - Yaml 파일에 정의된 configmap 과 , 경로를 통해  파일 형태로 마운트
4. 쿠버네티스 API로 직접 읽도록 실행할 코드 작 (동적 접근)

→ 소비되는 데이터를 모델링하는 방식하는 방식에 따라 다르게 쓰임 :: 이거는 실제로 활용도가 굉장히 높을 듯 하므로 , 각 방식 별 예시도 찾아보면 좋을 듯함. 

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
		readOnly:true
	volumes:  # 여기서 사용하려는 컨피그맵 참조
  -name: foo
		configmap:
			name: myconfigmap
		
```

- 만약 파드에 여러 컨테이너가 있는 경우 , 각 컨테이너는 자체 volumeMounts 블록이 필요하다.
- 컨피그맵은 각 컨피그맵 당 하나의 .spec.volumes 만 필요하다.

**[마운트 된** Configmap **자동 업데이트 ]**

- 현재 볼륨에 마운트된 컨피그맵이 업데이트 될 시 , 프로젝션 키도 업데이트 됨
- kubelet에서 주기적으로 마운트 컨피그맵이 최신상태인지 확인한다 but, kubelet은 로컬 캐시에서 컨피그맵 현재 값을 가져온다는 사실을 알 것 ( - 완전한 실시간 반영은 아님)
    
    → 때문에 새 키가 파드에 업데이트 될때 까지 총 지연시간  : kubelet 동기화 기간 + 캐시 전파 지연
    

### 변경할 수 없는(immutable) 컨피그맵

- *변경할 수 없는 시크릿과 컨피그맵* 은 개별 시크릿과 컨피그맵을 변경 할 수 없는 것으로 설정하는 옵션 `immutable: true` 을 제공한다.
    - 애플리케이션 중단을 일으킬 수 있는 원치않는 업데이트로부터 보호
    - immutable로 표시된 컨피그맵에 대한 감시를 중단 → kube-apiserver 부하 감소 → 클러스터성능향상

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable:true
```

# 시크릿(Secret)

[모범사례 참고하기!!](https://kubernetes.io/ko/docs/concepts/security/secrets-good-practices/)

> 암호, 토큰 또는 키와 같은 소량의 중요한 데이터를 포함하는 오브젝트 
→ 기밀 데이터를 애플리케이션에 넣을 필요가 없게 한다.
> 
- 파드와 독립적으로 생성되기 때문에, 파드의 생성/확인/수정 워크플로우 동안 시크릿 노출의 위험을 경감시킨다.

<aside>
⚠️

쿠버네티스 시크릿

etcd에 암호화되지않은 상태로 저장 

→ api access 권한이 있는 or etcd에 접근할 수 있는 모든 사용자가 시크릿을 조회/수정 가능

→ 해당 네임 스페이스 파드 생성 권한이 있는 누구나 해당 접근을 사용하여  시크릿을 읽을 수 있음

따라서 안전하게 사용하려면?

1. 시크릿에 대해 저장된 데이터 암호화(Encryption at Rest) 활성화
2. 시크릿에 대한 최소한의 접근 권한을 지니도록 RBAC 규칙 활성화
3. 특정 컨테이너 에서만 시크릿에 접근하도록 한다. 
4. 외부 시크릿 저장소 제공 서비스를 사용하는 것을 고려한다. 

 

</aside>

## 시크릿의 사용

파드가 시크릿을 사용하는 방법 3가지

1. [하나 이상의 컨테이너에 **마운트된 불륨 내의 파일**로써 사용](https://www.notion.so/1edbd7ce447180808fd6fdbb3dcfe347?pvs=21)
2. [컨테이너 **환경 변수**로써 사용](https://www.notion.so/1edbd7ce447180808fd6fdbb3dcfe347?pvs=21)
3. 파드의 이미지를 가져올 때 **kubelet**에 의해 사용

**시크릿의 대체품**

- 서비스어카운트 및 토큰을 이용하여 클라이언트를 식별
- 써드파티 도구를 클러스터 내부 또는 외부에 실행
- X.509 인증서를 위한 커스텀 인증자 구현 및 CertificateSigningRequests 이용
- 장치 플러그인 사용 - 노드의 암호화 하드웨어를 특정 파드에 노출

**시크릿 이름 및 데이터에 대한 제약 사항**

- secret object의 이름은 **유효한  DNS 서브도메인 이름** 이어야한다.
- `data`는 base64 인코딩된 값을 넣어야 하고, `stringData`는 일반 문자열을 바로 넣을 수 있다.
- `stringData`는 선언 시 자동으로 `data`로 변환된다.
- 동일한 키가 둘 다에 있을 경우, `stringData`값이 우선된다.

⇒ 사실상 data는 base64 포맷이라 불편하니 stringData만 쓰면 편한거 아닌가?
 근데~ data는 자동화 스크립트나 Helm , API 통신등에서 기계적으로 쓸 때 유리하다네,,

**크기 제한**

> 개별 시크릿 크기는 1Mib 제한이 있음
> 

**시크릿 수정하기** 

- 시크릿은 불변(immutable)만 아니면 수정될 수 있다.
- 방법
    - [kubectl 사용하기](https://kubernetes.io/ko/docs/tasks/configmap-secret/managing-secret-using-kubectl/#edit-secret)
    - [설정 파일 사용하기](https://kubernetes.io/ko/docs/tasks/configmap-secret/managing-secret-using-config-file/#edit-secret)

## 시크릿 다루기

**시크릿 생성하는 3가지 방식**

- [kubectl 사용](https://kubernetes.io/ko/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- [환경 설정 파일 사용](https://kubernetes.io/ko/docs/tasks/configmap-secret/managing-secret-using-config-file/)
- [kustomize 도구 사용](https://kubernetes.io/ko/docs/tasks/configmap-secret/managing-secret-using-kustomize/)

**시크릿 사용**

- !시크릿은 의존관계인 파드보다 먼저 생성되어야한다!
    - 참조가 실제로 시크릿 오브젝트를 가리키는지 확인하기 위해 볼륨 소스의 유효성 검사가 선행됨
    - 가져올 수 없는 경우 , kubelet은 해당 파드 실행을 주기적으로 재시도 한다. & 이벤트 보고

**선택적 시크릿**

- 환경 변수로 정의하는 경우, **optional** 설정 가능 (default : 필수)
- 모든 필수 시크릿이 ‘사용가능’해지기 전까지는 파드 컨테이너는 시작될 수 없다 ( 시작과정에서 실패할 것)

**파드에서 시크릿을 파일처럼 사용하기** 

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
    -name: foo # volumes[].name과 volumeMounts[].name은 반드시 일치해야함
		mountPath: "/etc/foo" # 시크릿 data map 의 각 키는 해당 경로 하위의 파일명이 된다. 
		readOnly:true
	volumes:
  -name: foo # volumes[].name과 volumeMounts[].name은 반드시 일치해야함
	secret:
		secretName: mysecret
		optional:false # 기본값임; "mysecret" 은 반드시 존재해야 함
```

1. 시크릿 생성
2. .spec.volumes[]. 하위에 볼륨 추가( 이때 `.spec.volumes[].secret.secretName` 시크릿 오브젝트 명과 일치해야함)
3. 시크릿이 필요한 각 컨테이너에 `.spec.containers[].volumeMounts[]` 추가

**특정 경로에 대한 시크릿 키 투영하기** 

- `.spec.volumes[].secret.items` : 시크릿 키가 투영되는 볼륨 내 대상 경로 변경 가능
    
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
    			readOnly:true
    	volumes:
    	  -name: foo
    			secret:
    				secretName: mysecret
    				items:
    		      -key: username
    					path: my-group/my-username
    ```
    
- 시크릿 오브젝트의 password key는 투영되지 X
- 위와 같이 명시적으로 키를 나열했다면, 나열된 모든키가 시크릿에 존재해야만 함 → 그렇지 않으면 볼륨 생성되지 X

시크릿 파일 퍼미션

> 단일 시크릿에 대한 POSIX 파일접근 퍼미션 비트 설정 가능 ( default : 0644) 
ex) 전체 시크릿 볼륨에 대한 기본 모드 설정 후 , 필요 시 각 키별로 오버라이드 가능
> 

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
	volumes:
	  -name: foo
		secret:
			secretName: mysecret
			defaultMode: 0400
```

**마운트 된 시크릿의 자동 업데이트** 

> 볼륨이 시크릿 데이터를 포함하고 있는 상태에서 , 시크릿이 업데이트 되면, 쿠버네티스가 이를 추적하고 eventually-consistent 접근 방식을 사용하여 볼륨 데이터를 업데이트 한다.
> 
- update 방식 2가지 : TTL 캐시 기반의 API watch 메커니즘으로 전파 or  kubelet 동기화 시 api 서버의 폴링

**시크릿을 환경 변수 형태로 사용하기**

1. 시크릿 생성 (여러 파드가 동일 시크릿 참조 가능)
2. `env[].valueFrom.secretKeyRef` : 시크릿의 name 과 key 정의
3. 일치하는 환경 변수에서 값을 찾도록 프로그램을 수정

```yaml
apiVersion: v1
kind: Pod
metadata:
name: secret-env-pod
spec:
	containers:
	  -name: mycontainer
		image: redis
		**env:**
		   -name: SECRET_USERNAME
				valueFrom:
					secretKeyRef:
						name: mysecret
						key: username
						optional:false # 기본값과 동일하다
						# "mysecret"이 존재하고 "username"라는 키를 포함해야 한다
		   -name: SECRET_PASSWORD
				valueFrom:
					**secretKeyRef:**
						name: mysecret
						key: password
						optional:false # 기본값과 동일하다
						# "mysecret"이 존재하고 "password"라는 키를 포함해야 한다
restartPolicy: Never
```

**올바르지 않은 환경 변수** 

- 키의 이름이 부적절한 환경 변수 이름을 갖고 있다면 , 파드가 실행되지않고, 에러가 발생한다
    
    → 파드 시작 실패 정보에 추가된다.
    
    ```yaml
    kubectl get events
    ```
    

**환경 변수로부터 시크릿 값 사용하기** 

시크릿을 환경 변수로 사용하는 컨테이너 내부에서 , 시크릿 키는 환경 변수 형태로 보인다

```yaml
echo "$SECRET_USERNAME"
```

출력 결과 : admin 

> ⚠️ 
그러나 Secret 을  환경 변수 형태로 파드에 주입하면, 
시크릿이 나중에 변경되더라도, 컨테이너 안의 환경 변수 값은 바뀌지 않는다. 
환경 변수는 파드/컨테이너 시작 시에만 초기화 되기 때문에, 반영하려면 , 컨테이너 재시작이 필요하다
> 

**컨테이너 이미지 pull 시크릿** 

- private registry에서 컨테이너 이미지를 가져오고 싶다면, 각 노드 kubelet이 해당 저장소에 인증을 수행하는 방법을 마련해야 한다. → image pull Secret 구성 (파드 level 설정)
- `imagePullSecrets`  필드 :
    - 파드가 속한 네임스페이스에 있는 시크릿들에 대한 참조 목록
    - 이미지 저장소 접근 credential을 kubelet에 전달 → 이정보를 이용하여, kubelet이 파드 대신 private image들을 가져옴

| 항목 | 설명 |
| --- | --- |
| 🔐 `imagePullSecrets` | 프라이빗 레지스트리 인증 정보 |
| 🧾 `ServiceAccount` | 파드가 실행될 때 사용하는 신분 |
| 🪄 연결 방식 | ServiceAccount에 `imagePullSecrets`를 지정해두면, 그걸 사용하는 파드들은 자동으로 인증 정보 적용됨 |

**스태틱 파드에서의 시크릿 사용** 

> 스태틱(static) 파드에서는 컨피그맵이나 시크릿을 사용할 수 없다.
> 

## 사용사례(시크릿을 컨테이너에 주입하는 5가지 방

1. 법컨테이너 환경 변수로 사용하기(env, envFrom)
    
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
    	name: mysecret
    type: Opaque
    data:
    	USER_NAME: YWRtaW4=
    	PASSWORD: MWYyZDFlMmU2N2Rm
    ```
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    	name: secret-test-pod
    spec:
    	containers:
        -name: test-container
    		image: registry.k8s.io/busybox
    		command: [ "/bin/sh", "-c", "env" ]
    		envFrom:
          -secretRef:
    				name: mysecret
    restartPolicy: Never
    ```
    
2. SSH 키를 사용하는 파드 (volumeMount)
    - 상황 : 쿠버네티스 파드가 외부 서버(git 서버, 원격 서버) 에 SSH 로 접속해야할때 SSH 키를 안전하게 관리하고 주입할 방법이 필요!
    - 키가 파일 이름 , 값이 파일 내용
    - 환경변수보다는 파일 형태로 마운트해서 사용하는 게 일반적
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    	name: secret-test-pod
    	labels:
    		name: secret-test
    spec:
    	volumes:
    	-name: secret-volume
    	secret:
    		secretName: ssh-key-secret
    	containers:
      -name: ssh-test-container
    	image: mySshImage
    	volumeMounts:
        -name: secret-volume
    		readOnly:true
    		mountPath: "/etc/secret-volume"
    ```
    
3. [운영/ 테스트 자격 증명을 사용하는 파드](https://www.notion.so/kubernetes-1cbbd7ce447180ebb790d16db41d042a?pvs=21)
    - 운영/테스트 환경에서 서로 다른 자격 증명을 파드에 주입하는 목적
    - 파드 구성은 같고, 시크릿만 바꿔서 환경 구분 가능(prod-secret , dev-secret)
    - volueMount가 환경 변수보다 상대적 안전
4. 시크릿 볼륨의 도트 파일(dotfile)
    
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
    name: dotfile-secret
    data:
    .secret-file: dmFsdWUtMg0KDQo=
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
      -name: dotfile-test-container
    image: registry.k8s.io/busybox
    command:
        - ls
        - "-l"
        - "/etc/secret-volume"
    volumeMounts:
        -name: secret-volume
    readOnly:truemountPath: "/etc/secret-volume"
    ```
    
    - 점으로 시작하는 키를 정의 하여 데이터를 “숨김” 할 수 있다.
    - **유출 가능성은 낮지만, `ls -la` 를 사용해야 이 파일들을 볼 수 있다.**
    - 파일명이 .으로 시작하는 dotfile 형태
5. 파드의 한 컨테이너에 표시되는 시크릿
    - 파드 내 특정 컨테이너에만 시크릿 주입(다른 컨테이너에는 안보이게
    - 컨테이너 별 env 또는 volumeMount를 분리해서 구성
    - 멀티 컨테이너 파드에서 역할 별 보안경계를 만들기 좋음
    - ! 특히 사이드카와 메인 컨테이너 분리 ! 에 좋음

## 시크릿 타입

- [불투명(Opaque) 시크릿](https://kubernetes.io/ko/docs/concepts/configuration/secret/#%EB%B6%88%ED%88%AC%EB%AA%85-opaque-%EC%8B%9C%ED%81%AC%EB%A6%BF)
    - 시크릿 구성 파일에서 타입을 지정하지 않았을 경우 default 타입
    - generic 하위 커맨드를 이용해서 Opaque 시크릿 타입을 표현
- [서비스 어카운트 토큰 시크릿](https://kubernetes.io/ko/docs/concepts/configuration/secret/#%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%96%B4%EC%B9%B4%EC%9A%B4%ED%8A%B8-%ED%86%A0%ED%81%B0-%EC%8B%9C%ED%81%AC%EB%A6%BF)
    - 서비스 어카운트를 확인하는 토큰 자격 증명을 저장하기 위해서 사용
    - ex) 파드 내부 애플리케이션이 쿠버네티스 api 를 사용하도록 할때 사용
    - [TokenRequest](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/) API로 얻는 토큰을 권장함- 수명이 제한되어 안전
    - 
- [도커 컨피그 시크릿](https://kubernetes.io/ko/docs/concepts/configuration/secret/#%EB%8F%84%EC%BB%A4-%EC%BB%A8%ED%94%BC%EA%B7%B8-%EC%8B%9C%ED%81%AC%EB%A6%BF)
    - 하위 두개 중 하나를 사용 가능
    - `kubernetes.io/dockercfg`
    - `kubernetes.io/dockerconfigjson`
- [기본 인증(Basic authentication) 시크릿](https://kubernetes.io/ko/docs/concepts/configuration/secret/#%EA%B8%B0%EB%B3%B8-%EC%9D%B8%EC%A6%9D-basic-authentication-%EC%8B%9C%ED%81%AC%EB%A6%BF)
    - `username:password` 문자열을 Base64로 인코딩한 후 HTTP 헤더에 담아 보내는 인증 방식
- [SSH 인증 시크릿](https://kubernetes.io/ko/docs/concepts/configuration/secret/#ssh-%EC%9D%B8%EC%A6%9D-%EC%8B%9C%ED%81%AC%EB%A6%BF)
- [TLS 시크릿](https://kubernetes.io/ko/docs/concepts/configuration/secret/#tls-%EC%8B%9C%ED%81%AC%EB%A6%BF)
- [부트스트랩 토큰 시크릿](https://kubernetes.io/ko/docs/concepts/configuration/secret/#%EB%B6%80%ED%8A%B8%EC%8A%A4%ED%8A%B8%EB%9E%A9-%ED%86%A0%ED%81%B0-%EC%8B%9C%ED%81%AC%EB%A6%BF)

# 불변(Immutable) 시크릿

```yaml
apiVersion: v1
kind: Secret
metadata:
  ...
data:
  ...
immutable:true
```

# 시크릿을 위한 정보 보안

| 항목 | 보안 고려사항 |
| --- | --- |
| 저장 방식 | etcd에 base64로 저장됨 (기본 암호화 X) |
| 접근 제어 | RBAC으로 권한 최소화 필수 |
| 환경 변수 사용 | `ps` 명령어로 노출 위험 있음 |
| 볼륨 사용 | 노드 접근 시 파일 읽기 가능 |
| 서비스 계정 토큰 | 불필요한 자동 마운트 방지 필요 |
| 감사 및 모니터링 | audit log로 이상 접근 감시 추천 |