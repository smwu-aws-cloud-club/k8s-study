## 1. 쿠버네티스란?

- 컨테이너화된 워크로드와 서비스를 관리하기 위한 이식성이 있고, 확장 가능한 오픈소스 플랫폼
- 선언적 구성과 자동화를 모두 지원
- 크고 빠르게 성장하는 생태계를 가지고 있음
- 쿠버네티스 서비스, 지원 그리고 도구들은 광범위하게 제공됨

![](https://velog.velcdn.com/images/icandooooo/post/1bbe383a-8241-425c-a579-5d69f1e49c89/image.png)

- 전통적 배포 시대
  초기에는 애플리케이션을 물리 서버에서 실행했다. 한 물리 서버에서 여러 애플리케이션을 실행하므로 **리소스 할당의 문제가 발생**했다. 예를 들어 물리 서버 하나에서 여러 애플리케이션을 실행하면, 리소스 전부를 차지하는 애플리케이션 인스턴스가 있을 수 있고, 결과적으로는 다른 애플리케이션의 성능이 저하될 수 있었다.

  - 서로 다른 여러 물리 서버에서각 애플리케이션을 실행할 수 있었다.
    -> 하지만 리소스가 충분히 활용되지 않아 확장 가능 불가, 많은 물리 서버 유지하는 데에 높은 비용

- 가상화 배포 시대
  해결책으로 가상화가 도입됐다. **단일 물리 서버의 CPU에서 여러 가상 시스템(VM)을 실행**할 수 있다. 가상화를 사용하면 VM간에 애플리케이션을 격리하고 애플리케이션의 정보를 다른 애플리케이션에서 자유롭게 액세스할 수 없으므로, 일정 수준의 보안성을 제공할 수 있다. **물리 서버에서 리소스를 보다 효율적으로 활용**할 수 있으며, 쉽게 애플리케이션을 추가하거나 업데이트할 수 있고 하드웨어 비용을 절감할 수 있어 **더 나은 확장성을 제공**한다. 가상화를 통해 일련의 물리 리소스를 폐기 가능한(disposable) 가상 머신으로 구성된 클러스터로 만들 수 있다.

  - 각 VM은 가상화된 하드웨어 상에서 자체 운영체제를 포함한 모든 구성 요소를 실행하는 하나의 완전한 머신이다.

- 컨테이너 개발 시대
  컨테이너는 VM과 유사하지만 격리 속성을 완화하여 **애플리케이션 간에 운영체제(OS)를 공유**한다. 그러므로 컨테이너는 가볍다고 여겨진다. VM과 마찬가지로 컨테이너에는 자체 파일 시스템, CPU 점유율, 메모리, 프로세스 공간 등이 있다. 기본 인프라와의 종속성을 끊었기 때문에, 클라우드나 OS 배포본에 모두 이식할 수 있다.

  - VM 이미지를 사용하는 것에 비해 컨테이너 이미지 생성이 보다 쉽고 효율적
  - 안정적이고 주기적으로 컨테이너 이미지를 빌드해서 배포할 수 있고 (이미지의 불변성 덕에) 빠르고 효율적으로 롤백 가능
  - 배포 시점이 아닌 빌드/릴리스 시점에 애플리케이션 컨테이너 이미지를 만들기 때문에, 애플리케이션이 인프라스트럭처에서 분리됨
  - 가시성(observability) : OS 수준의 정보와 메트릭에 머무르지 않고, 애플리케이션의 헬스와 그 밖의 시그널을 볼 수 있음
  - 개발, 테스팅 및 운영 환경에 걸친 일관성 : 랩탑에서도 클라우드에서와 동일하게 구동된다.
  - 애플리케이션 중심 관리: 가상 하드웨어 상에서 OS를 실행하는 수준에서 논리적인 리소스를 사용하는 OS 상에서 애플리케이션을 실행하는 수준으로 추상화 수준이 높아짐
  - 느슨하게 커플되고, 분산되고, 유연하며, 자유로운 마이크로서비스: 애플리케이션은 단일 목적의 머신에서 모놀리식 스택으로 구동되지 않고 보다 작고 독립적인 단위로 쪼개져서 동적으로 배포되고 관리될 수 있음

### 쿠버네티스의 필요성

쿠버네티스는 분산 시스템을 탄력적으로 실행하기 위한 프레임 워크를 제공한다. 애플리케이션의 확장과 장애 조치를 처리하고, 배포 패턴 등을 제공한다.(ex. 카나리아 배포 쉽게 관리 가능)

쿠버네티스는 다음을 제공한다.

- 서비스 디스커버리와 로드 밸런싱
- 스토리지 오케스트레이션
- 자동화된 롤아웃과 롤백
- 자동화된 빈 패킹(bin packing)
- 자동화된 복구(self-healing)
- 시크릿과 구성 관리

### 쿠버네티스가 아닌 것

쿠버네티스는 전통적인, 모든 것이 포함된 Platform as a Service(PaaS)가 아니다. 쿠버네티스는 하드웨어 수준보다는 **컨테이너 수준에서 운영**되기 때문에, PaaS가 일반적으로 제공하는 배포, 스케일링, 로드 밸런싱과 같은 기능을 제공하며, 사용자가 로깅, 모니터링 및 알림 솔루션을 통합할 수 있다. 하지만, 쿠버네티스는 **모놀리식(monolithic)이 아니어서,** 이런 기본 솔루션이 선택적이며 추가나 제거가 용이하다. 쿠버네티스는 개발자 플랫폼을 만드는 구성 요소를 제공하지만, 필요한 경우 사용자의 선택권과 유연성을 지켜준다.

---

## 2. 쿠버네티스 컴포넌트

쿠버네티스 **클러스터는 컨테이너화된 애플리케이션을 실행하는 노드라고 하는 워커 머신의 집합**. **모든 클러스터는 최소 한 개의 워커 노드를 가진다**.

워커 노드는 애플리케이션의 구성요소인 파드를 호스트한다. 컨트롤 플레인은 워커 노드와 클러스터 내 파드를 관리한다. 프로덕션 환경에서는 일반적으로 컨트롤 플레인이 여러 컴퓨터에 걸쳐 실행되고, 클러스터는 일반적으로 여러 노드를 실행하므로 내결함성과 고가용성이 제공된다.

- 파드 : 클러스터에서 실행 중인 컨테이너의 집합

![](https://velog.velcdn.com/images/icandooooo/post/46670679-bf08-49bc-8e50-72b36ac9ccda/image.png)

쿠버네티스 클러스터 구성 요소

### 컨트롤 플레인 컴포넌트

컨트롤 플레인 컴포넌트는 클러스터에 관한 전반적인 결정(ex. 스케줄링)을 수행하고 클러스터 이벤트(ex. 디플로이먼트의 replicas 필드에 대한 요구 조건이 충족되지 않을 경우 새로운 파드 구동)를 감지하고 반응한다.

#### kube-apiserver

- API 서버는 쿠버네티스 API를 노출하는 쿠버네티스 컨트롤 플레인 컴포넌트
- API 서버는 쿠버네티스 컨트롤 플레인의 프론트 엔드

#### etcd

모든 클러스터 데이터를 담는 쿠버네티스 뒷단의 저장소로 사용되는 일관성·고가용성 키-값 저장소

#### kube-scheduler

노드가 배정되지 않은 새로 생성된 파드를 감지하고, 실행할 노드를 선택하는 컨트롤 플레인 컴포넌트

#### kube-controller-manager

컨트롤러 프로세스를 실행하는 컨트롤 플레인 컴포넌트

#### cloud-controller-manager

클라우드별 컨트롤 로직을 포함하는 쿠버네티스 컨트롤 플레인 컴포넌트

### 노드 컴포넌트

노드 컴포넌트는 동작 중인 파드를 유지시키고 쿠버네티스 런타임 환경을 제공하며, 모든 노드 상에서 동작한다.

#### kubelet

클러스터의 각 노드에서 실행되는 에이전트. Kubelet은 파드에서 컨테이너가 확실하게 동작하도록 관리함

#### kube-proxy

kube-proxy는 클러스터의 각 노드에서 실행되는 네트워크 프록시로, 쿠버네티스의 서비스 개념의 구현부

#### 컨테이너 런타임

컨테이너 실행을 담당하는 소프트웨어

### 애드온

쿠버네티스 리소스(데몬셋, 디플로이먼트 등)를 이용하여 클러스터 기능을 구현한다. 이들은 클러스터 단위의 기능을 제공하기 때문에 애드온에 대한 네임스페이스 리소스는 kube-system 네임스페이스에 속한다.

---

## 3. 쿠버네티스 API

**쿠버네티스 컨트롤 플레인의 핵심은 API 서버**다. API 서버는 최종 사용자, 클러스터의 다른 부분 그리고 외부 컴포넌트가 서로 통신할 수 있도록 HTTP API를 제공한다.

쿠버네티스 API를 사용하면 쿠버네티스의 API 오브젝트(ex. 파드(Pod), 네임스페이스(Namespace), 컨피그맵(ConfigMap) 그리고 이벤트(Event))를 질의(query)하고 조작할 수 있다.

대부분의 작업은 kubectl 커맨드 라인 인터페이스 또는 API를 사용하는 kubeadm과 같은 다른 커맨드 라인 도구를 통해 수행할 수 있다. 그러나, REST 호출을 사용하여 API에 직접 접근할 수도 있다.

---

## 4. 명령줄 도구(kubectl)

### kubectl

쿠버네티스 API를 사용하여 쿠버네티스 클러스터의 컨트롤 플레인과 통신하기 위한 커맨드라인 툴이다

```
**kubectl [command] [TYPE] [NAME] [flags]**
```

- command

  - 하나 이상의 리소스에서 수행하려는 동작을 지정
  - ex) create, get, describe, delete

- TYPE

  - 리소스 타입 지정
  - 대소문자를 구분하지 않으며 단수형, 복수형 또는 약어 형식 지정 가능
  - ex) kubectl get pod pod1 / kubectl get pods pod1 / kubectl get po pod1 모두 같은 표현이다.

- NAME

  - 리소스 이름 지정
  - 이름은 대소문자를 구분한다. 이름을 생략하면, 모든 리소스에 대한 세부 사항이 표시된다.
  - ex) kubectl get pods

- flags
  - 선택적 플래그 지정
  - ex) -s 또는 --server 플래그를 사용하여 쿠버네티스 API 서버의 주소와 포트 지정 가능

```
kubectl help
```

위 명령어를 통해 도움을 받을 수 있다.

#### POD_NAMESPACE 환경 변수

POD_NAMESPACE 환경 변수가 설정되어 있으면, 네임스페이스에 속하는 자원에 대한 CLI 작업은 환경 변수에 설정된 네임스페이스를 기본값으로 사용

- ex) 환경 변수가 seattle로 설정되어 있으면, kubectl get pods 명령은 seattle 네임스페이스에 있는 파드 목록을 반환
  - **파드가 네임스페이스에 속하는 자원**이며, 명령어에 네임스페이스를 특정하지 않았기 때문
  - **kubectl api-resources** 명령을 실행하고 결과를 확인하여 **특정 자원이 네임스페이스에 속하는 자원인지 판별**

```
--namespace <value>
```

명시적으로 인자를 사용하면 위와 같은 동작을 오버라이드한다.

```
kubectl config set-context --current --namespace=<namespace-name>
```

kubectl이 클러스터 외부에서 실행되었으며 네임스페이스가 명시되지 않은 경우 kubectl 명령어는 클라이언트 구성에서 현재 컨텍스트(current context)에 설정된 네임스페이스에 대해 동작한다. **kubectl이 동작하는 기본 네임스페이스를 변경하려면 위의 명령어를 실행**하면 된다.

### 출력 옵션

```
kubectl [command] [TYPE] [NAME] -o <output_format>
```

kubectl 명령의 기본 출력 형식은 사람이 읽을 수 있는 일반 텍스트 형식이다. 특정 형식으로 터미널 창에 세부 정보를 출력하려면, 지원되는 kubectl 명령에 -o 또는 --output 플래그를 추가할 수 있다.

- -o wide
  - 추가 정보가 포함된 일반 텍스트 형식으로 출력된다. 파드의 경우, 노드 이름이 포함된다.
- -o json
  - JSON 형식의 API 오브젝트를 출력한다.
- -o yaml
  - YAML 형식의 API 오브젝트를 출력한다.

#### 사용자 정의 열

사용자 정의 열을 정의하고 원하는 세부 정보만 테이블에 출력하려면, custom-columns 옵션을 사용할 수 있다.

```
kubectl get pods <pod-name> -o custom-columns=NAME:.metadata.name,RSRC:.metadata.resourceVersion

kubectl get pods <pod-name> -o custom-columns-file=template.txt
```

#### 오브젝트 목록 정렬

터미널 창에서 정렬된 목록으로 오브젝트를 출력하기 위해, 지원되는 kubectl 명령에 --sort-by 플래그를 추가할 수 있다. --sort-by 플래그와 함께 숫자나 문자열 필드를 지정하여 오브젝트를 정렬한다.

```
kubectl [command] [TYPE] [NAME] --sort-by=<jsonpath_exp>

kubectl get pods --sort-by=.metadata.name
```

#### kubectl apply

```
# example-service.yaml의 정의를 사용하여 서비스를 생성한다.
kubectl apply -f example-service.yaml

# example-controller.yaml의 정의를 사용하여 레플리케이션 컨트롤러를 생성한다.
kubectl apply -f example-controller.yaml

# <directory> 디렉터리 내의 .yaml, .yml 또는 .json 파일에 정의된 오브젝트를 생성한다.
kubectl apply -f <directory>
```

파일 또는 표준입력에서 리소스를 적용하거나 업데이트한다.'

#### kubectl get

```
# 모든 파드를 일반 텍스트 출력 형식으로 나열한다.
kubectl get pods

# 모든 파드를 일반 텍스트 출력 형식으로 나열하고 추가 정보(예: 노드 이름)를 포함한다.
kubectl get pods -o wide

# 지정된 이름의 레플리케이션 컨트롤러를 일반 텍스트 출력 형식으로 나열한다. 팁: 'replicationcontroller' 리소스 타입을 'rc'로 짧게 바꿔쓸 수 있다.
kubectl get replicationcontroller <rc-name>

# 모든 레플리케이션 컨트롤러와 서비스를 일반 텍스트 출력 형식으로 함께 나열한다.
kubectl get rc,services

# 모든 데몬 셋을 일반 텍스트 출력 형식으로 나열한다.
kubectl get ds

# 노드 server01에서 실행 중인 모든 파드를 나열한다.
kubectl get pods --field-selector=spec.nodeName=server01
```

하나 이상의 리소스를 나열한다.

#### kubectl describe

```
# 노드 이름이 <node-name>인 노드의 세부 사항을 표시한다.
kubectl describe nodes <node-name>

# 파드 이름이 <pod-name> 인 파드의 세부 정보를 표시한다.
kubectl describe pods/<pod-name>

# 이름이 <rc-name>인 레플리케이션 컨트롤러가 관리하는 모든 파드의 세부 정보를 표시한다.
# 기억하기: 레플리케이션 컨트롤러에서 생성된 모든 파드에는 레플리케이션 컨트롤러 이름이 접두사로 붙는다.
kubectl describe pods <rc-name>

# 모든 파드의 정보를 출력한다.
kubectl describe pods
```

초기화되지 않은 리소스를 포함하여 하나 이상의 리소스의 기본 상태를 디폴트로 표시한다.

- kubectl get 명령은 일반적으로 **동일한 리소스 타입의 하나 이상의 리소스를 검색하는 데 사용**된다. 예를 들어, -o 또는 --output 플래그를 사용하여 출력 형식을 사용자 정의할 수 있는 풍부한 플래그 세트가 있다. -w 또는 --watch 플래그를 지정하여 특정 오브젝트에 대한 업데이트 진행과정을 확인할 수 있다.
- kubectl describe 명령은 **지정된 리소스의 여러 관련 측면을 설명하는 데 더 중점**을 둔다. API 서버에 대한 여러 API 호출을 호출하여 사용자에 대한 뷰(view)를 빌드할 수 있다. 예를 들어, kubectl describe node 명령은 노드에 대한 정보뿐만 아니라, 노드에서 실행 중인 파드의 요약 정보, 노드에 대해 생성된 이벤트 등의 정보도 검색한다.

#### kubectl delete

```
# pod.yaml 파일에 지정된 타입과 이름을 사용하여 파드를 삭제한다.
kubectl delete -f pod.yaml

# '<label-key>=<label-value>' 레이블이 있는 모든 파드와 서비스를 삭제한다.
kubectl delete pods,services -l <label-key>=<label-value>

# 초기화되지 않은 파드를 포함한 모든 파드를 삭제한다.
kubectl delete pods --all
```

파일, 표준입력 또는 레이블 선택기, 이름, 리소스 선택기나 리소스를 지정하여 리소스를 삭제한다.

#### kubectl exec

```
# 파드 <pod-name>에서 'date'를 실행한 결과를 얻는다. 기본적으로, 첫 번째 컨테이너에서 출력된다.
kubectl exec <pod-name> -- date

# 파드 <pod-name>의 <container-name> 컨테이너에서 'date'를 실행하여 출력 결과를 얻는다.
kubectl exec <pod-name> -c <container-name> -- date

# 파드 <pod-name>에서 대화식 TTY를 연결해 /bin/bash를 실행한다. 기본적으로, 첫 번째 컨테이너에서 출력된다.
kubectl exec -ti <pod-name> -- /bin/bash
```

파드의 컨테이너에 대해 명령을 실행한다.

#### kubectl logs

```
# 파드 <pod-name>에서 로그의 스냅샷을 반환한다.
kubectl logs <pod-name>

# 파드 <pod-name>에서 로그 스트리밍을 시작한다. 이것은 리눅스 명령 'tail -f'와 비슷하다.
kubectl logs -f <pod-name>
```

파드의 컨테이너에 대한 로그를 출력한다.

#### kubectl diff

```
# "pod.json"에 포함된 리소스의 차이점을 출력한다.
kubectl diff -f pod.json

# 표준입력에서 파일을 읽어 차이점을 출력한다.
cat service.yaml | kubectl diff -f -
```

제안된 클러스터 업데이트의 차이점을 본다.

---

## 5. Kubeadm

Kubeadm은 쿠버네티스 클러스터 생성을 위한 "빠른 경로"의 모범 사례로 kubeadm init 및 kubeadm join 을 제공하도록 만들어진 도구다.

kubeadm은 실행 가능한 최소 클러스터를 시작하고 실행하는 데 필요한 작업을 수행한다.

- kubeadm init : 쿠버네티스 컨트롤 플레인 노드를 부트스트랩한다.
- kubeadm join : 쿠버네티스 워커(worker) 노드를 부트스트랩하고 클러스터에 조인시킨다.
- kubeadm upgrade : 쿠버네티스 클러스터를 새로운 버전으로 업그레이드한다.
- kubeadm config : kubeadm v1.7.x 이하의 버전을 사용하여 클러스터를 초기화한 경우, kubeadm upgrade 를 위해 사용자의 클러스터를 구성한다.
- kubeadm token : kubeadm join 을 위한 토큰을 관리한다.
- kubeadm reset : kubeadm init 또는 kubeadm join 에 의한 호스트의 모든 변경 사항을 되돌린다.
- kubeadm certs : 쿠버네티스 인증서를 관리한다.
- kubeadm kubeconfig : kubeconfig 파일을 관리한다.
- kubeadm version : kubeadm 버전을 출력한다.
- kubeadm alpha : 커뮤니티에서 피드백을 수집하기 위해서 기능 미리 보기를 제공한다.

---

## 6. kubelet

kubelet은 각 노드에서 실행되는 기본 "노드 에이전트"다.

```
kubelet [flags]
```
