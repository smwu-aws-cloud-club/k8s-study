# 1. 쿠버네티스란 무엇인가

## 1. 쿠버네티스 개념

- 쿠버네티스는 컨테이너화된 워크로드와 서비스를 관리하기 위한 이식할 수 있고, 확장 가능한 오픈소스 플랫폼
- 선언적 구성(deployment)와 자동화 지원
    - 선언적 구성을 위한 deployment 예시
        
        ```bash
        #example.yaml
        
        apiVersion: apps/v1 #k8s에서 사용할 API 버전
        kind: Deployment 
        metadata:
          name: example #deployment의 이름
          labels:
            app: example
        spec:
          replicas: 1 #replicaset의 개수 -> 생성할 pod의 개수
          selector:
            matchLabels:
              app: example #어떤 label을 가진 pod를 관리할지 선택
          template: #생성할 pod에 부여될 label
            metadata:
              labels:
                app: example
            spec:
              containers: 
              - name: example-container-name #컨테이너 이름
                image: example-image-name:v1 #컨테이너를 빌드할 때 사용할 이미지 이름:버전 태그
                ports:
                - name: container-port
                  containerPort: 8080 #컨테이너가 노출할 포트
                command: ["sh", "-c", "java -jar server.jar"] #컨테이너 실행 시 사용할 명령어
                envFrom: #configMap으로 환경변수 주입
                  - configMapRef:
                      name: common-env
                env: #개별 환경 변수 직접 지정
                - name: PORT
                  value: "8080"
                - name: SLACK_WEBHOOK_URL
                  value: https://hooks.slack.com/services/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
                - name: EUREKA_INSTANCE_IP_ADDRESS
                  value: example-svc
        ```
        

## 2. 배포 발전 과정

### 1. Traditional Deployment(가상화 이전 배포)

- 한 개의 물리 서버에서 하나의 os를 설치하고 여러 프로그램 실행
    - 한 대의 서버에서 여러 개의 프로그램을 실행하는 경우, 어떤 프로그램이 다른 프로그램 동작을 간섭하거나 특정 프로그램이 성능을 독점할 경우 다른 프로그램 성능이 떨어지는 문제가 있음
    - 문제 해결을 위해 여러 개의 물리 서버에서 애플리케이션을 실행하는 방법이 있지만, 리소스 낭비

### 2. Virtualized Deployment(가상머신 기반 배포)

- 가상화 → 애플리케이션 사이의 격리 보장
    - 단일 물리 서버의 CPU에서 여러 가상 시스템 실행
    - 가상화를 사용해 VM간의 애플리케이션 격리
    - 각 VM은 가상화된 하드웨어 상에서 os를 포함한 모든 구성요소를 실행함
- Virtualized Deployment의 구조
    - 하이퍼바이저: 하나의 시스템 상에서 가상 컴퓨터를 여러 개 구동할 수 있도록 하는 중간계층
    - VM
        - 가상 머신에는 cpu, 메모리, 저장장치를 개별적으로 할당함
        - VM의 구조
            - OS: 가상 머신의 운영체제
            - APP: 실행 프로그램
            - Bin/Library: 프로그램 실행 환경과 관련된 파일
- 여러 프로그램들을 각각의 가상 머신에서 실행하면 하나의 컴퓨터에서 여러 개의 가상화된 컴퓨터 동작 →격리 → 프로그램 간 서로 간섭 x
- 시스템 자원 상황에 따라 가상머신 개수 조정해서 다중화, 분산처리 o
- 하지만 가상머신에 일일히 os 설치 → 리소스 낭비

### 3. Container Deployment(컨테이너 배포)

- VM과 유사하지만 격리 속성을 완화하여 애플리케이션 간 kernel을 통해 host os 공유(컨테이너는 자신만의 운영체제를 가지고 있지 x)
- 컨테이너는 자체 파일 시스템, cpu 점유율, 메모리, 프로세스 공간 등을 가짐
- 컨테이너는 하나의 os(host os)에서 구동되지만 서로 격리되어서 구동됨 → 컨테이너에서 실행되는 a라는 프로그램이 다른 컨테이너에서 실행되는 b라는 프로그램에 영향을 주지 x
- 컨테이너 사용 이유
    - VM 이미지를 사용하는 것 보다 컨테이너 이미지를 생성하는 것이 쉽고 효율적임
    - 이미지의 불변성
        - 컨테이너 이미지로 컨테이너 빌드 → 컨테이너 실행 중에 이미지가 변경되어도 실행 중인 컨테이너의 상태는 변경되지 않음 → 만약 이미지의 변경 사항을 컨테이너에 반영하려면, 변경된 이미지를 가지고 다시 컨테이너를 빌드해야 함
        - 이미지의 불변성 덕분에 안정적, 주기적으로 컨테이너 이미지를 빌드해서 배포할 수 있고 빠르고 효율적으로 롤백 가능
    - 개발, 테스팅 및 운영 환경에 걸친 일관성
    - 클라우드 및 os 배포판 간 이식성
        - os, 클라우드의 종류에 관계없이 어디에서든 구동 가능 → 컨테이너가 Host os를 공유하고, 컨테이너 이미지가 어플리케이션 실행 환경을 모두 포함하고 있기 때문
    - 애플리케이션 중심 관리
    - 자유로운 마이크로서비스

## 3. 쿠버네티스의 기능

- 서비스 디스커버리와 로드 밸런싱
    - DNS 이름(도메인 주소)을 사용하거나 자체 ip주소를 사용하여 컨테이너 노출 가능
- 스토리지 오케스트레이션
- 자동화된 롤아웃, 롤백
    - 쿠버네티스를 자동화해서 배포용 새 컨테이너를 만들고, 기존 컨테이너를 제공하고, 모든 리소스를 새 컨테이너에 적용 가능
- 자동화된 빈 패킹
    - 클러스터 노드 제공
        - 클러스터 노드는 컨테이너를 실행하는데 사용되는 리소스 제공자
        - 클러스터 노드는 컨테이너가 필요로 하는 cpu, ram 제공 → 클러스터 노드의 관리는 control plane이 담당
- 자동화된 복구
- 시크릿, configuration

# 2. 쿠버네티스 컴포넌트

- 쿠버네티스 클러스터 = 노드 컴포넌트 + 컨트롤 플레인 컴포넌트
- 쿠버네티스 배포 → 쿠버네티스 클러스터를 얻음 → 모든 클러스터는 최소 한 개의 워커 노드를 가짐
- 쿠버네티스 클러스터는 노드(워커 머신)의 집합 → 워커 노드는 pod(애플리케이션 구성 요소) 호스팅
- 컨트롤플레인 → 워커노드와 클러스터 내 pod 관리

## 1. 컨트롤 플레인 컴포넌트

- 클러스터에 관한 전반적인 결정과 클러스터 이벤트를 감지하고 반응
    - 스케줄링 수행
    - deployment의 replicas field에 대한 요구 조건 충족되지 않을 시 새로운 pods 구동

### 1. kube-apiserver

- 쿠버네티스 api를 노출하는 컨트롤 플레인 컴포넌트
- 쿠버네티스의 작업(pods, servercis, replication controllers,…)은 Api호출을 통해 이루어짐 → 이 api 요청을 받고 처리하는 컴포넌트가 api-server
    - kube-api server에게 pod 목록 달라고 요청하는 명령어
    
    ```bash
    kubectl get pods
    ```
    
- 수평 확장 가능
    - 앞단에 로드 밸런서, 뒷단에 여러 개의 kube-apiserver을 둬서 부하 분산

### 2. etcd

- 모든 클러스터 데이터를 담는 쿠버네티스 뒷단의 저장소
- 일관성, 고가용성 키-값 저장

### 3. kube-scheduler

- 노드가 배정되지 않은 새로 생성된 파드 감지
- 해당 파드가 실행될 노드 선택

### 4. kube-controller-manager

- 컨트롤러 프로세스를 실행 → 자동 감지 + 자동 실행 역할을 맡은 컴포넌트를 모아놓은 것
- 노드 컨트롤러 → 노드가 응답이 없다면 감지 + 알림, pod 재배치
- 잡 컨트롤러 → 일회성 작업을 나타내는 잡 오브젝트 감시, 해당 작업을 완료할 때까지 동작하는 파드 생성
- 엔드포인트 슬라이스 컨트롤러 → service - pod를 연결하는 endpointslice 오브젝트를 만들어서 트래픽 관리
- 서비스 어카운트 컨트롤러 → 새로운 namespace가 만들어지면 자동으로 기본 serviceAccount 생성 → 이를 통해 해당 namespace들의 pod들이 쿠버네티스 api에 접근할 수 있음

### 5. cloud-controller-manager

- 클라우드별 컨트롤 로직 포함
- 클라우드 컨트롤러 매니저를 통해 클러스터를 클라우드 공급자의 api에 연결 → 해당 클라우드 플랫폼과 상호 작용하는 컴포넌트와 클러스터와만 상호 작용하는 컴포넌트를 구분할 수 있게 해줌

## 2. 노드 컴포넌트

- 동작 중인 파드 유지, 쿠버네티스 런타임 환경 제공
- 모든 노드 상에서 동작

### 1. kubelet

- 파드에서 컨테이너가 동작하도록 관리
- 쿠버네티스를 통해 생성된 컨테이너만 관리

### 2. kube-proxy

- 클러스터의 각 노드에서 실행되는 네트워크 프록시 → 쿠버네티스 서비스 개념의 구현부
- 노드의 네트워크 규칙을 유지, 관리 → 내부 네트워크 세션이나 클러스터 바깥에서 파드로 네트워크 통신을 할 수 있도록 해줌(NodePort로 들어온 외부 요청 → 내부 pods로 라우팅)

### 3. 컨테이너 런타임

- 컨테이너 실행을 담당하는 소프트웨어
- containerd, CRI-O와 같은 컨테이너 런타임 및 모든 k8s CRI 구현체 지원

## 3. 애드온

- 쿠버네티스 리소스를 이용하여 클러스터 기능 구현 → 클러스터 단위로 기능 제공
- DNS, 대시보드, 컨테이너 리소스 모니터링, 클러스터-레벨 로깅 등 제공

# 3. 쿠버네티스 API

- 쿠버네티스 오브젝트의 상태들을 조작 가능
- 쿠버네티스 컨트롤 플레인의 핵심→ API 서버
- API 서버는 사용자, 클러스터의 다른 부분, 외부 컴포넌트가 통신할 수 있도록 HTTP API 제공
- 쿠버네티스 API로 API 오브젝트(pod, namespace, configmap, event)를 질의(query)하고 조작 가능
- kubectl 커맨드 라인 인터페이스, kubdeam 같은 커맨드 라인 도구, REST 호출로 실행 가능

# 4. 쿠버네티스 명령줄 도구

## 1. kubectl

- 쿠버네티스 API를 사용하여 쿠버네티스 클러스터의 컨트롤 플레인과 통신하기 위한 커맨드라인 툴
- 자신이 파드 안에서 실행되는지(클러스터 안에 있는지) 판별
    - `KUBERNETES_SERVICE_HOST`
    - `KUBERNETES_SERVICE_PORT` 환경 변수
    - 서비스 어카운트 토큰 파일이 `/var/run/secrets/kubernetes.io/serviceaccount/token` 에 있는지 확인
    - 세 가지가 모두 감지되면 클러스터 내 인증이 적용
- POD_NAMESPACE 환경변수
    - 네임스페이스에 속하는 자원에 대한 작업은 환경 변수에 설정된 네임스페이스를 기본값으로 사용

```bash
kubectl [command] [TYPE] [NAME] [flags]
```

- command: 하나 이상의 리소스에서 수행하려는 동작 지정 → create, get, describe, delete
- TYPE: 리소스 타입 지정
- NAME: 리소스 이름 지정 → 이름 생략 시 모든 리소스에 대한 세부 사항 표시
- flags: 선택적 플래그 지정

## 2. kubeadm

- 실행 가능한 최소 클러스터를 시작하고 실행하는 데 필요한 작업 수행 → 쿠버네티스의 핵심 구성만 빠르게 설치해 주는 도구
- 부가 설정(로깅 등)은 사용자가 직접 추가
- kubeadm은 kubeadm init, kubeadm join 같은 명령어로 Control Plane과 Worker Node를 쉽게 설치/연결할 수 있는 도구
- kubeadm은 쿠버네티스를 시작하는 데 딱 필요한 최소한의 설정(API 서버, etcd, 컨트롤러 매니저 같은 핵심 컴포넌트들)만 해줌
- 

## 3. kubelet

- 각 노드에서 실행되는 노드 에이전트 → 노드를 API 서버에 등록
- kubelet은 다양한 방식(주로 API 서버를 통해)으로 전달받은 PodSpec들을 가져와서 그 안에 정의된 컨테이너들이 실제로 실행 중이고 건강한지 보장
- Pod 정의(PodSpec)를 API 서버나 파일/HTTP로 받아서 컨테이너 상태를 유지해주는 역할
