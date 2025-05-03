# Week1

# 쿠버네티스 개념

- 쿠버네티스(k8s)란?
    - 컨테이너화된 워크로드와 서비스를 관리하기 위해
    - 이식 & 확장이 가능인 오픈소스 플랫폼
    - 선언적 구성과 자동화 모두 지원
    - 키잡이(helmsman)나 파일럿을 뜻하는 그리스어에서 유래.
- 쿠버네티스의 유용성
    
    ![](https://kubernetes.io/images/docs/Container_Evolution.svg)
    
    - 애플리케이션을 물리서버, 가상머신에서 배포하던 시기를 거쳐, “컨테이너 개발 시대”에 다다름
    - 컨테이너는
        - 가상머신과 유사하되 격리 속성을 완화하여 애플리케이션 간에 OS를 공유해서 가볍다.
        - 기본 인프라와 종속성을 끊어서 클라우드/OS 배포폰 모두 이식 가능
    - app 관련 컨테이너 이미지(설계도) 생성이 VM를 사용하는 것보다 빠르다
    - 지속적인 개발, 통합 및 배포(CI/CD)
    - 배포 시점이 아니라 빌드/release 시점에 컨테이너 이미지를 만들어서, app이 인프라에서 분리됨
    - 가시성(observability) : OS 수준의 정보/metirc만이 아니라 app의 health 및 그 외 신호 확인 가능
    - 일관성 : 랩탑에서도 클라우드에서와 동일하게 구동
    - 이식성 : OS 가 뭐든 클라우드가 뭐든 구동
    - app 중심 관리 : 추상화 수준이 높아짐
    - loosely coupled, 분산되고, 유연하며, 자유로운 마이크로 서비스 구현 가능
- 분산 시스템을 탄력적으로 실행할 수 있는 프레임워크 제공
    - 서비스 디스커버리와 로드밸런싱
        - DNS 이름을 사용하거나 자체 IP 주소를 사용하여 컨테이너를 노출 가능
        - 컨테이너에 대한 트래픽이 많으면, 쿠버네티스는 네트워크 트래픽을 로드밸런싱하고 배포하여 배포가 안정적으로 이루어질 수 있음
    - 스토리지 오케스트레이션 : 원하는 저장소 시스템을 자동으로 탑재
    - 자동화된 롤아웃과 롤백
    - 자동화된 bin packing
    - 자동화된 복구(self-healing)
    - 시크릿과 구성 관리
- Platform as a Service(PaaS)가 아님
- 모놀리식(monolithic)이 아님 → 선택적으로 솔루션 추가/제거 용이
- 지원하는 애플리케이션 유형에 제약이 없음. 컨테이너에서 애플리케이션이 구동되면 쿠버네티스에서도 잘 동작됨
- 소스코드를 배포하지 않고 애플리케이션을 빌드하지 않음.
- 미들웨어, 데이터 처리 프레임워크(e.g. spark), 데이터베이스, 캐시, 클러스터 스토리지 시스템(e.g. Ceph) 같이 ‘애플리케이션 레벨의 서비스’를 제공하지는 않음.
- 로깅, 모니터링, 경보 솔루션을 포함하진 않음
- 오케스트레이션의 필요성을 없애줌
    - 독립적이고 조합 가능한 제어 프로세스들로 구성됨
    - 지속적으로 현재 상태를 입력 받은 의도한 상태로 나아가도록 함
    - 중앙화된 제어 필요 없음

## 쿠버네티스 컴포넌트

쿠버네티스 클러스터 = **노드** 컴포넌트(컴퓨터 집합) + **컨트롤 플레인** 컴포넌트

![](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

- 노드 : containered application을 실행하는 worker machine의 집합
    - 모든 클러스터는 최소 한 개의 worker node를 가짐
    - 일반적으로 여러 노드를 실행해서 내결함성&고가용성 제공
    - worker node : 애플리케이션의 구성 요소인 파드를 호스트함
        - 파드 : 클러스터에서 실행중인 컨테이너 집합
    - 동작 중인 파드를 유지시키고 쿠버네티스 런탕미 환경을 제공하며, 모든 노드 상에서 동작함
    - **kubelet**
        - 각 노드에서 실행되는 agent
        - 파드에서 컨테이너가 확실히 동작하게 관리함
        - 파드 스펙 집합을 받아 컨테이너가 해당 파드 스펙에 따라 건강하게 동작하는 걸 확실히 함
        - 쿠버네티스로 생성되지 않는 컨테이너는 관리하지 않음
    - **kube-proxy**
        - 각 노드에서 실행되는 network proxy
        - 서비스 개념의 구현부
        - 네트워크 규칙 유지 관리 → 내부 네트워크 세션/클러스터 외부에서 파드로 네트워크 통신 가능하게 함
        - OS에 가용한 패킷 필터링 계층이 있으면 이를 사용함.
        그렇지 않으면 트래픽 자체를 forward함
    - **컨테이너 런타임**
        - 컨테이너 실행 담당 SW
        - containerd, CRI-O, k8s CRI
- 컨트롤 플레인 : worker node와 클러스터 내 파드를 관리
    - 프로덕션 환경에선 여러 컴퓨터에 걸쳐 실행되는 게 일반적
    - 클러스터에 관한 전반적인 결정(e.g. scheduling)을 수행하고 클러스터 이벤트를 감지하고 반응
    - 클러스터 내 어느 머신에서든지 동작 가능
    - 간결성을 위해 보통은 같은 머신 상에서 모든 컨트롤 플레인 컴포넌트를 구동시키고 사용자 컨테이너는 해당 머신 상에서 동작시키지 않음.
    - **kube-apiserver**
        - 쿠버네티스 API를 노출시키는 컨트롤 플레인 컴포넌트
        - 컨트롤 플레인의 frontend
        - 수평으로 확장(더 많은 인스턴스 배포하는 방식) → 트래픽 균형 조절
    - **etcd**
        - 쿠버네티스 backend에 있는 키-값 저장소
    - **kube-scheduler**
        - 노드가 배정되지 않은, 새로 생성된 파드를 감지하고
        - 실행할 노드를 선택
    - **kube-controller-manager**
        - 컨트롤러 프로세스를 실행함
            - 컨트롤러 : 서버로 클러스터 공유 상태를 감시하고, 현 상태를 원하는 상태로 이행시키는 컨트롤 루프
        - 각 컨트롤러는 분리되어 있지만 복잡성을 낮추고자 단일 바이너리로 컴파일되고 단일 프로세스 내에서 실행됨
            - 노드 컨트롤러 : 노드 다운 시 책임
            - 잡 컨트롤러 : 일회성 작업인 잡 오브젝트를 감시 및 작업 완료까지 동작하는 파드 생성
            - 엔드포인트슬라이스 컨트롤러 : 서비스-파드 간 연결고리 제공을 위한 엔드포인트슬라이스 객체를 채움
            - 서비스어카운트 컨트롤러 : 새 네임스페이스에 대한 기본 service account 생성
    - cloud-controller-manager
        - 클라우드별 컨트롤 로직을 포함함
        - 클라우드 공급자의 API에 연결하고, 플랫폼과 상호 작용하는 컴포넌트와 클러스터와만 상호작용하는 컴포넌트 구분하게 도움
        - 사내/PC 내부 학습 환경에서 쿠버네티스 실행 중이라면 클러스터에는 클라우드 컨트롤러 매니저가 없음
- 애드온
    - 쿠버네티스 리소스를 이용해 클러스터 기능 구현
    - `kube-system` 네임스페이스에 애드온 리소스가 속함
    - DNS
        - 절대적으로 요구되진 않지만, 대체로 쿠버네티스 클러스터는 클러스터 DNS를 갖춰야 함
            - 클러스터 DNS
                - 구성환경 내 다른 DNS 서버와 더불어, 
                쿠버네티스 서비스를 위한 DNS 레코드 제공까지 담당하는 
                DNS 서버
        - 쿠버네티스로 구동되는 컨테이너는 DNS 검색에서 이 DNS 서버를 자동으로 포함함
    - 웹 UI(dashboard)
    - container resource monitoring
    - cluster-level logging

## 쿠버네티스 API

- 쿠버네티스 object 상태를 쿼리/조작
- 컨트롤 플레임의 핵심 : API 서버와 그게 노출하는 HTTP API
- client와 cluster의 다른 부분적/전체 외부 컴포넌트는 API 서버로 서로 통신
- kubectl, kubeadm 같은 CLI로 보통 작업
    - rest 호출로 직접 접근 가능
- 쿠버네티스 API로 app 작성 시 client labraries 중 하나를 사용하는 게 좋음
- `/openapi/v2` endpoint
    
    
    | 헤더 | 사용할 수 있는 값 | 참고 |
    | --- | --- | --- |
    | `Accept-Encoding` | `gzip` | *이 헤더를 제공하지 않는 것도 가능* |
    | `Accept` | `application/com.github.proto-openapi.spec.v2@v1.0+protobuf` | *주로 클러스터 내부 용도로 사용* |
    |  | `application/json` | *기본값* |
    |  | `*` | `JSON으로 응답` |
- `/openapi/v3/apis/<group>/<version>?hash=<hash>` endpoint
    
    
    | 헤더 | 사용할 수 있는 값 | 참고 |
    | --- | --- | --- |
    | `Accept-Encoding` | `gzip` | *이 헤더를 제공하지 않는 것도 가능* |
    | `Accept` | `application/com.github.proto-openapi.spec.v3@v1.0+protobuf` | *주로 클러스터 내부 용도로 사용* |
    |  | `application/json` | *기본값* |
    |  | `*` | `JSON으로 응답` |
- 오브젝트 직렬화 상태를 etcd에 기록 및 저장

### API 그룹과 버전 규칙

- API 서버는 여러 API 버전을 통해 동일한 기본 데이터를 제공할 수 있음
    
    [예시] 동일한 리소스에 대해 `v1` 과 `v1beta1` 이라는 두 가지 API 버전
    →API의 `v1beta1` 버전을 사용하여 오브젝트를 만든 경우, `v1beta1` 버전이 사용 중단(deprecated)되고 제거될 때까지는 `v1beta1` 또는 `v1` API 버전을 사용하여 해당 오브젝트를 읽거나, 업데이트하거나, 삭제 가능
    → `v1` API를 사용하여 계속 오브젝트에 접근하고 수정할 수 있음
    
- API 확장 방법 2가지
    1. 커스텀 리소스를 쓰면 API 서버가 선택한 리소스 API를 제공하는 법을 선언적으로 정의 가능
    2. aggregation layer를 구현해 확장
    

# CLI

## 명령줄 도구 (kubectl)

- 쿠버네티스 API를 사용하여 쿠버네티스 클러스터의 컨트롤 플레인과 통신하기 위한 커맨드라인 툴
- `kubectl` 은 config 파일을 $HOME/.kube 에서 찾음.
    - KUBECONFIG 환경변수 설정 또는  `--kubeconfig` 플래그를 설정하여 다른 kubeconfig 파일을 지

```bash
kubectl [command] [TYPE] [NAME] [flags]
```

- command : 하나 이상의 리소스에서 수행하려는 동작 지정
    - create
    - get
    - describe
    - delete
- TYPE : 리소스 타입. 대소문자 구분 없이 단수/복수/약어 형식 지정 가능
    
    ```bash
    #동일 출력 결과
    kubectl get pod pod1
    kubectl get pods pod1
    kubectl get po pod1
    ```
    
- NAME  : 리소스 이름 지정. 대소문자 구분함. 이름 생략 시 모든 리소스에 대한 세부 사항 표시
    
    `kubectl get pods`
    
    `kubectl get pod example-pod1 example-pod2`-> 동일 타입 리소스들을 그룹화
    
    `kubectl get pod/example-pod1 replicationcontroller/example-rc1` → 리소스 여러 개를 개별적으로 타입 지정
    
    `-f file1 -f file2 -f file<#>` →하나 이상의 파일로 리소스 지정
    
    - json 대신 yaml이 구성 파일 친화적 → `kubectl get -f ./pod.yaml`
- flags : 선택적 flag 지정. `-s`나 `--server` 플래그를 써서 쿠버네티스 API 서버의 주소와 포트 지정 가능

### 클러스터 내 인증과 네임스페이스 오버라이드

1. `kubectl`은 먼저 자신이 파드 안에서 실행되고 있는지, 즉 클러스터 안에 있는지를 판별
2. `KUBERNETES_SERVICE_HOST`와 `KUBERNETES_SERVICE_PORT` 환경 변수, 그리고 서비스 어카운트 토큰 파일이 `/var/run/secrets/kubernetes.io/serviceaccount/token` 경로에 있는지를 확인
3. 세 가지가 모두 감지되면, 클러스터 내 인증이 적용됨
- **`POD_NAMESPACE` 환경 변수** :  네임스페이스에 속하는 자원에 대한 CLI 작업은 환경 변수에 설정된 네임스페이스를 기본값으로 사용함
- kubectl이 동작하는 기본 네임스페이스를 변경 시 명령어
    
    `kubectl config set-context --current --namespace=<namespace-name>`
    

### 출력 옵션

`kubectl [command] [TYPE] [NAME] -o <output_format>`

### 사용자 정의 열

- 사용자 정의 열을 정의하고 원하는 세부 정보만 테이블에 출력하려면, `custom-columns` 옵션 사용
- 인라인으로 정의하거나 템플릿 파일을 사용하도록 선택 가능
    - `-o custom-columns=<spec>` 또는 `-o custom-columns-file=<filename>`

### 서버측 열

- 서버에서 오브젝트에 대한 특정 열 정보 수신을 지원
- 클라이언트가 출력할 수 있도록, 주어진 리소스에 대해 서버가 해당 리소스와 관련된 열과 행을 반환한다는 것을 의미
- 서버가 출력의 세부 사항을 캡슐화하도록 하여, 동일한 클러스터에 대해 사용된 클라이언트에서 사람이 읽을 수 있는 일관된 출력을 허용함

### 오브젝트 목록 정렬

`kubectl [command] [TYPE] [NAME] --sort-by=<jsonpath_exp>`

### 일반적인 예제

- `kubectl apply` - 파일 또는 표준입력에서 리소스를 적용하거나 업데이트
- `kubectl get` - 하나 이상의 리소스를 나열
- `kubectl describe` - 초기화되지 않은 리소스를 포함하여 하나 이상의 리소스의 기본 상태를 디폴트로 표시
- `kubectl delete` - 파일, 표준입력 또는 레이블 선택기, 이름, 리소스 선택기나 리소스를 지정하여 리소스를 삭제
- `kubectl exec` - 파드의 컨테이너에 대해 명령을 실행
- `kubectl logs` - 파드의 컨테이너에 대한 로그를 출력
- `kubectl diff` - 제안된 클러스터 업데이트의 차이점 확인

## 설치 도구 (Kubeadm)

- 쿠버네티스 클러스터 생성을 위한 "빠른 경로"의 모범 사례로 `kubeadm init` 및 `kubeadm join` 을 제공하도록 만들어진 도구
- 실행 가능한 최소 클러스터를 시작하고 실행하는 데 필요한 작업을 수행

## Component tools (kubelet)

- 주요 node agent로, 각 노드에서 운영된다
- 각 노드를 
1)hostname 중 하나에서 사용하는 api server
2)hostname을 override하는 flag
3)클라우드 공급자를 위한 특정 로직
에 register할 수 있다.
- PodSpec 측면에서 작동함
    - PodSpec은 yaml/json 객체로, pod를 설명한다
    - 다양한 메카니즘으로 지공되는 podspec들의 집합을 구성하고, podspec 내에 설명되는 컨테이너이 작동중이고 healthy하다는 걸 ‘보증’함
    - 쿠버네티스로 안 만들어진 컨테이너는 관리하지 않는다
- api server에서의 podspec 말고도 다음 두 가지 방법으로 컨테이너 manifest가 kubelet에 제공될 수 있다.
    1. file
        - CLI로 flag로서 Path가 전달되면 해당 path 하의 파일들은 업데이트를 위해 주기적으로 모니터링된다
        - 모니터링 주기는 20초가 기본값이고, flag로 설정 가능하다
    2. http endpoint
        - CLI로 parameter를 전달한다.
        - 모니터링 주기는 20초가 기본값이고, flag로 설정 가능하다