# Week4

# 서비스, 로드밸런싱, 네트워킹

## 서비스

- 외부와 접하는 단일 엔드포인트 뒤에 있는 클러스터에서 실행되는 애플리케이션을 노출시킴
  (워크로드가 여러 백엔드로 나뉘어 있는 경우에도 동일)
- 파드 집합에서 실행중인 애플리케이션을 네트워크 서비스로 노출하는 추상화 방법
  - 파드 : 클러스터에서 실행 중인 컨테이너의 집합. 비영구적 리소스.
  - 서비스가 대상으로 하는 파드 집합은 일반적으로 **셀렉터**가 결정
- 파드의 논리적 집합과 그것들에 접근할 수 있는 정책을 정의하는 추상적 개념(**마이크로 서비스 패턴**)
- 쿠버네티스는 파드에게 고유한 IP 주소와 파드 집합에 대한 단일 DNS 명을 부여하고, 그것들 간에 로드-밸런스를 수행
- cloud-native : 쿠버네티스 API 사용 시 매치되는 endpoint slice를 API server에 질의 가능
  non-native application : 쿠버네티스가 애플리케이션-백엔드 파드 간에 네트워크 포트 또는 로드 밸런서를 배치할 수 있는 방법 제공

### 서비스 정의

- 파드와 비슷한 REST object
  - API server에 POST해서 새 인스턴스를 생성할 수 있음
  - 명세 예시

    ```bash
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service #라는 service object 생성
    spec:
      selector:
        app.kubernetes.io/name: MyApp#<-이 레이블을 갖는 파드 세트
      ports: #TCP 포트에서 9376에서 수신
        - protocol: TCP
          port: 80
          targetPort: 9376
    ```

    - 서비스 셀렉터와 컨트롤러는 셀렉터와 일치하는 파드를 지속적으로 검색&”my-service”라는 엔드포인트 오브젝트에 대한 모든 업데이트를 POST
    - [참고] 서비스는 **모든** 수신 port를 targetPort에 매핑할 수 있고, 기본적으로(편의상) targetPort는 port 필드와 동일하게 설정함.

  - 서비스 기본 프로토콜 : TCP(다른 프로토콜도 지원 가능)
  - 서비스가 대부분 하나 의상의 포트를 노출해야 함
    → 쿠버네티스가 service object에서 다중 port 정의를 지원함.
  - 각 port는 같거나 다른 프로토콜로 정의 가능
- 파드의 포트 정의에 이름이 있으니, targetPort에서 이 이름을 참조할 수 있음
  - 서비스의 targetPort를 파드 포트에 바인딩하는 예시
  ```bash
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels:
      app.kubernetes.io/name: proxy
  spec:
    containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80
          name: http-web-svc #파드의 포트 정의 이름

  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-service
  spec:
    selector:
      app.kubernetes.io/name: proxy
    ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80
      targetPort: http-web-svc #참조한 부분을 확인 가능
  ```

### 셀렉터가 없는 서비스

- 셀렉터 대신 매칭되는(corresponding) 엔드포인트슬라이스 object와 함께 사용되면 다른 종류의 백엔드도 추상화 가능.
  (클러스터 외부에서 실행되는 것도 포함됨) - 엔드포인트슬라이스(endpoint slices): 서비스 셀렉터와 매칭되는 파드의 IP주소를 추적한다.
- **셀렉터 유무와 상관없이 동일하게 서비스 접근**
- [예시 1] 파드 셀렉터 없이 서비스 정의
  - 셀렉터가 없음 → 매칭되는 엔드포인트 슬라이스(및 레거시 엔드포인트) 오브젝트가 자동으로 생성되지 않음.
  ```bash
  apiVersion: v1
  kind: Service
  metadata:
    name: my-service
  spec:
    ports:
      - protocol: TCP
        port: 80
        targetPort: 9376

  ```
- [예시 2] 엔드포인트 슬라이스 오브젝트를 수동으로 추가하여, 서비스를 실행 중인 네트워크 주소 및 포트에 서비스를 수동으로 매핑
  ```bash
  apiVersion: discovery.k8s.io/v1
  kind: EndpointSlice
  metadata:
    name: my-service-1 # 관행적으로, 서비스의 이름을
                       # 엔드포인트슬라이스 이름의 접두어로 사용한다.
    labels:
      # "kubernetes.io/service-name" 레이블을 설정해야 한다.
      # 이 레이블의 값은 서비스의 이름과 일치하도록 지정한다.
      kubernetes.io/service-name: my-service
  addressType: IPv4
  ports:
    - name: '' # 9376 포트는 (IANA에 의해) 잘 알려진 포트로 할당되어 있지 않으므로
               # 이 칸은 비워 둔다.
      appProtocol: http
      protocol: TCP
      port: 9376
  endpoints:
    - addresses:
        - "10.4.5.6" # 이 목록에 IP 주소를 기재할 때 순서는 상관하지 않는다.
        - "10.1.2.3"
  ```

### 커스텀 엔드포인트슬라이스

- 엔드포인트 슬라이스에 `kubernetes.io/service-name` 레이블을 설정
- 엔드포인트 IP 주소는 다른 쿠버네티스 서비스의 클러스터 IP일 수 없음

kube-proxy(클러스터 각 노드에서 실행되는 네트워크 프록시)는 가상 IP를 목적지로 지원하지 않음.

- controller 같은 예약어 사용 금지
- 소문자, 하이픈(-) 사용

에

### 엔드포인트(Endpoints)(권장 X)

- 네트워크 엔드포인트 목록 in 쿠버네티스 api
- 트래픽이 어떤 파드에 보내질 수 있는지를 정의하기 위해 서비스가 엔드포인트를 참조함.
- **엔드포인트슬라이스 API 사용 권장**
  - 하나의 엔드포인트(Endpoints) 객체가 1000개 이상의 엔드포인트(endpoints)를 갖도록 수동으로 업데이트할 수는 없음

### 애플리케이션 프로토콜

- `appProtocol`field : 각 서비스 port에 대한 애플리케이션 프로토콜 지정 방법을 제공
- 이 필드 값은 상응하는 엔드포인트와 엔드포인트슬라이스 오브젝트에 의해 미러링됨

### 멀티-포트 서비스

- 모든 포트 이름을 명확히 지정하기
- 포트 이름 : 소문자 영문자와 숫자, -만 포함. 영문자와 숫자로 시작하고 끝나야 함.

### 자신의 IP 주소 선택

- `.spec.clusterIP` field : 서비스 생성 요청 시 고유한 클러스터 IP 주소 지정 가능
  - 재사용하려는 기존 DNS 항목
  - 특정 IP 주소로 구성되어 재구성이 어려운 legacy system
- 선택한 IP 주소는 API 서버에 대해 구성된 `service-cluster-ip-range` CIDR 범위 내의 유효한 IPv4 또는 IPv6 주소
  → 유효하지 않을 경우, API server가 `422 HTTP status code`가 return

### 서비스 탐색(discovery)은 1)환경 변수 2)DNS로 가능

1. 환경 변수
   - 파드가 노드에서 실행될 때
     - **kubelet** : 각 활성화된 서비스에 대해 **환경 변수 세트를 추가**함
     - 하이픈(-) 대신 언더스코어(\_)로 변환, 소문자 대신 대문자로 변환해서 서비스 이름 작성
   - 환경 변수에만 적용됨
     - 클라이언트 파드가 생성되기 **\*전**에\* 서비스를 만들어야 한다. 그렇지 않으면, 해당 클라이언트 파드는 환경 변수를 생성할 수 없다.
2. DNS(대개 필수로 사용)
   - 클러스터-인식 DNS 서버
     - 새 서비스를 위해 쿠버네티스 API를 감시
     - 각각에 대한 DNS 레코드 세트를 생성함
     - 클러스터 전체에서 DNS 활성화 시, 모든 파드는 DNS 이름으로 서비스를 자동으로 확인 가능
   - `ExternalName` 서비스에 접근할 수 있는 유일한 방법

### headless service

- 로드-밸런싱과 단일 서비스 IP는 필요치 않은 경우에 만들 수 있음
- 명시적으로 `.spec.clusterIP=None`을 지정
  - 클러스터 IP가 할당되지 않고, kube-proxy가 서비스를 처리하지 않음
  - 프록시 안 함
- 쿠버네티스 구현에 묶이지 않고 다른 service discovery mechanism과 interface 가능
- 셀렉터 있는 경우
  쿠버네티스 컨트롤 플레인이 엔드포인트슬라이스 오브젝트 생성
  → DNS 구성 변경
- 셀렉터 없는 경우
  쿠버네티스 컨트롤 플레인이 엔드포인트슬라이스 오브젝트 생성X
  → DNS 시스템은 다음 중 하나를 탐색한 뒤 구성함
  - [`type: ExternalName`](https://kubernetes.io/ko/docs/concepts/services-networking/service/#externalname) 서비스에 대한 DNS CNAME 레코드
  - `ExternalName` 이외의 모든 서비스 타입에 대해, 서비스의 활성(ready) 엔드포인트의 모든 IP 주소에 대한 DNS A / AAAA 레코드
    - IPv4 → A, IPv6 → AAAA

## 서비스 퍼블리싱(ServiceTypes)

클러스터 밖에 위치한 외부 IP 주소에 노출하고자 하는 경우

- `ServiceTypes` : 원하는 서비스 종류를 지정
  - `ClusterIP`: 서비스를 클러스터-내부 IP에 노출시킴. 클러스터 내에서만 서비스 도달 가능. 기본값.
  - NodePort : 고정 포트(NodePort)로 각 노드의 IP에 서비스를 노출 시킴. `type: ClusterIP`인 서비스를 요청했을 때와 마찬가지로 클러스터 IP 주소를 구성
    - `--service-node-port-range` 플래그로 지정된 범위에서 포트를 할당(30000~32767 기본값)
    - 각 노드는 해당 포트(모든 노드에서 동일 포트)로 서비스 프록시
    - 할당된 포트는 `.spec.ports[*].nodePort` 필드에서 확인 가능
    - 자유롭게 자체 로드 밸런싱 솔루션을 설정하거나,
      쿠버네티스가 완벽하게 지원하지 않는 환경을 구성하거나,
      하나 이상의 노드 IP를 직접 노출시킬 수 있음
    - 포트 직접 선택
      ```bash
      apiVersion: v1
      kind: Service
      metadata:
        name: my-service
      spec:
        type: NodePort
        selector:
          app.kubernetes.io/name: MyApp
        ports:
            # 기본적으로 그리고 편의상 `targetPort` 는 `port` 필드와 동일한 값으로 설정된다.
          - port: 80
            targetPort: 80
            # 선택적 필드
            # 기본적으로 그리고 편의상 쿠버네티스 컨트롤 플레인은 포트 범위에서 할당한다(기본값: 30000-32767)
            nodePort: 30007
      ```
    - 커스텀 IP 주소
      - kube-proxy에 대한 `--nodeport-addresses` 플래그
      - kube-proxy 구성 파일의 `nodePortAddresses` 필드
        를 특정 IP 블록으로 설정 가능
  - **LoadBalancer** : 클라우드 공급자의 로드 밸런서로 서비스를 외부에 노출시킴
    - `spec.loadBalancerClass` 필드
    - 서비스에 대한 로드 밸런서를 프로비저닝
    - 실제 생성은 비동기적으로 수행
    ```bash
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      selector:
        app.kubernetes.io/name: MyApp
      ports:
        - protocol: TCP
          port: 80
          targetPort: 9376
      clusterIP: 10.0.171.239
      type: LoadBalancer
    status:
      loadBalancer: #프로비저닝된 밸런서에 대한 정보
        ingress:
        - ip: 192.0.2.127
    ```
    - 외부 로드 밸런서의 트래픽은 백엔드 파드로 전달됨
    - `loadBalancerIP` 필드가 지정되지 않으면, 임시 IP 주소로 loadBalancer가 설정됨
      - 클라우드 공급자 따라서 지정 후 반영 여부가 결정됨
    - 쿠버네티스 v1.24 기준으로 `MixedProtocolLBService` 기능 게이트는 둘 이사의 포트가 정의되면,
      로드밸런서 타입 서비스에 대해 서로 다른 프로토콜을 사용 가능하게 해줌 - 클라우드 제공자에 따라서 혼합프로토콜 지원 안 할 수도 있음
    - ExternalName: CNAME 레코드 리턴. 서비스를 이 필드 내용에 매핑함. 어떤 프록시도 설정되지 않음
    - 내부 로드 밸런서 : 클라우드 서비스 공급자에 따라 어노테이션 붙여야 함
      ```bash
      #e.g. AWS
      [...]
      metadata:
          name: my-service
          annotations:
              service.beta.kubernetes.io/aws-load-balancer-internal: "true"
      [...]
      ```
    - AWS에서는 부분적으로 TLS/SSL 지원을 위해 세 가지 어노테이션 추가 가능
      1. 사용할 인증서의 ARN 지정(IAM/Certificate Manager에서 생성된 인증서)

         ```bash
         metadata:
           name: my-service
           annotations:
             service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
         ```

      2. 파드가 알려주는 프로토콜 지정

         - HTTPS&SSL
           - ELB는 인증서를 사용해 암호화된 연결을 통해 파드가 스스로 인증할 것으로 예상함
           - 7계층 프록시 선택
             - ELB는 요청을 전달할 때 사용자와의 연결을 종료하고,
             - 헤더를 파싱하고 사용자의 IP 주소로 `X-Forwarded-For` 헤더를 삽입
         - TPC&SSL : 4계층 프록시. ELB는 헤더 수정 X

         ```bash
         metadata:
           name: my-service
           annotations:
             service.beta.kubernetes.io/aws-load-balancer-backend-protocol: (https|http|ssl|tcp)

             #service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
             #service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443,8443"
             #80은 proxy하는 HTTP
             #443, 8443은 SSL 인증서 사용

             #정책 사용
             #service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"

             #클러스터에 대한 프록시 프로토콜
             #service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
         ```

         ```bash

         aws elb describe-load-balancer-policies --query 'PolicyDescriptions[].PolicyName'
         ```
    - AWS ELB 접근 로그 어노테이션
      - `service.beta.kubernetes.io/aws-load-balancer-access-log-enabled` : 접근 로그의 활성화 여부 제어
      - …/`aws-load-balancer-access-log-emit-interval` : 접근 로그 게시하는 간격을 분 단위로 제어(5분/60분 간격)
      - …/`aws-load-balancer-access-log-s3-bucket-name` : 로그가 저장되는 S3 버킷명 제어
      - …/`aws-load-balancer-access-log-s3-bucket-prefix` : S3 버킷 생성한 논리적 계층 지정
    - AWS 연결 Draining
      - `service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled="true"` : 설정하여 관리
      - `service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout` : 인스턴스 해제 전, 기존 연결을 열어두는 목적으로 최대 시간을 지정함.(초 단위)
    - AWS 네트워크 로드 밸런서 사용하게 되면 `aws-load-balancer-type` 를 “nlb”로 지정
- type은 중첩 기능으로 설계됨.
- **인그레스(Ingress)**로 서비스 노출 가능
  - 서비스 유형은 아니지만 클러스터 진입점 역할을 함
  - 동일 IP 주소로 여러 서비스 노출 가능 → 라우팅 규칙을 단일 리소스로 통합 가능

## 인그레스(Ingress)

### 정의

- 클러스터 외부에서 클러스터 내부 [서비스](https://kubernetes.io/ko/docs/concepts/services-networking/service/)로 HTTP와 HTTPS 경로를 노출함
  - 트래픽 라우팅은 인그레스 리소스에 정의된 규칙에 의해 컨트롤됨
  ![](https://kubernetes.io/ko/docs/images/ingress.svg)
- 쿠버네티스 API를 통해 정의한 규칙에 기반하여 트래픽을 다른 백엔드에 매핑할 수 있게 함
- 클러스터 내의 서비스에 대한 외부 접근을 관리하는 API 오브젝트이며, 일반적으로 HTTP를 관리
- 부하 분산, SSL 종료, 명칭 기반의 가상 호스팅을 제공 가능

### 전제 조건 : 인그레스 컨트롤러(e.g. ingress-nginx)

### 인그레스 리소스

- [예시] service/networking/minimal-ingress.yaml
  ```bash
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: minimal-ingress #유효한 DNS 서브도메인 이름이어야 함
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /
  spec:
    ingressClassName: nginx-example
    #ingressClassName을 생략하려면
    #기본 ingress class가 정의되어 있어야 함
    #e.g. --watch-ingress-without-class 플래그
    rules:
    - http:
        paths:
        - path: /testpath
          pathType: Prefix
          backend:
            service:
              name: test
              port:
                number: 80
  ```
- 인그레스 규칙
  - 각 HTTP 규칙에는 다음 정보가 포함됨
    - 선택적 호스트
    - 경로 목록에 적힌 service.name, service.portname, service.port.number 같은 백엔드
      - 호스트와 경로가 모두 수신 요청의 내용과 일치해야 함
    - 백엔드는 서비스와 포트 이름의 조합. 규칙의 호스트와 경로가 일치한 ingress에 대한 HTTP/HTTPS 요청은 백엔드 목록으로 전송됨.
- default backend
  - 종종 사양의 경로와 일치하지 않는 서비스에 대한 모든 요청을 처리하도록 구성됨
  - 규칙이 없는 인그레스는 모든 트래픽을 단일 기본 백엔드로 전송하며, `.spec.defaultBackend`는 이와 같은 경우에 요청을 처리할 백엔드를 지정함

### Resource 백엔드

- 인그레스 오브젝트와 동일한 네임스페이스 내에 있는 다른 쿠버네티스 리소스에 대한 ObjectRef
- 서비스와 상호 배타적인 설정. 둘 다 지정하면 유효성 검사 실패!
- 정적 자산이 있는 오브젝트 스토리지 백엔드로 데이터를 수신함
- [예시]
  ```bash
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-resource-backend
  spec:
    defaultBackend:
      resource:
        apiGroup: k8s.example.com
        kind: StorageBucket
        name: static-assets
    rules:
      - http:
          paths:
            - path: /icons
              pathType: ImplementationSpecific
              backend:
                resource:
                  apiGroup: k8s.example.com
                  kind: StorageBucket
                  name: icon-assets

  ```
  - `kubectl describe ingress {ingress metadata name}`
  - pathType
    - ImplementationSpecific : 경로 유형의 일치 여부는 IngressClass에 따라 달라짐.
    - Exact : URL 경로의 대소문자를 엄격하게 일치
    - Prefix : URL 경로의 접두사를 `/` 를 기준으로 분리한 값과 일치시킴.

### 호스트네임 와일드카드

- [예시]
  ```bash
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-wildcard-host
  spec:
    rules:
    - host: "foo.bar.com"
      http:
        paths:
        - pathType: Prefix
          path: "/bar"
          backend:
            service:
              name: service1
              port:
                number: 80
    - host: "*.foo.com"
      http:
        paths:
        - pathType: Prefix
          path: "/foo"
          backend:
            service:
              name: service2
              port:
                number: 80

  ```
- [표]
  | 호스트      | 호스트 헤더       | 일치 여부                                            |
  | ----------- | ----------------- | ---------------------------------------------------- |
  | `*.foo.com` | `bar.foo.com`     | 공유 접미사를 기반으로 일치함                        |
  | `*.foo.com` | `baz.bar.foo.com` | 일치하지 않음, 와일드카드는 단일 DNS 레이블만 포함함 |
  | `*.foo.com` | `foo.com`         | 일치하지 않음, 와일드카드는 단일 DNS 레이블만 포함함 |

### 인그레스 클래스

- `.spec.parameters` 필드를 사용하여 해당 인그레스클래스와 연관있는 환경 설정을 제공하는 다른 리소스를 참조
- 범위
  - 클러스터 범위(기본)
    - `.spec.parameters` 필드만 설정하고 `.spec.parameters.scope` 필드는 지정하지 않거나
    - `.spec.parameters.scope` 필드를 `Cluster`로 지정
      → 클러스터 범위의 리소스를 참조
    - 파라미터의 `kind`(+`apiGroup`)는 클러스터 범위의 API (커스텀 리소스일 수도 있음) 를 참조
  - 네임스페이스 범위
    - `.spec.parameters` 필드를 설정하고 `.spec.parameters.scope` 필드를 `Namespace`로 지정하면, 인그레스클래스는 네임스페이스 범위의 리소스를 참조

### 인그레스 유형

1. 단일 서비스로 지원되는 인그레스

   ```bash
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: test-ingress
   spec:
     defaultBackend:
       service:
         name: test
         port:
           number: 80
   ```

   - kubectl apply -f로 생성 시,
     kubectl get ingress test-ingress로 상태 확인 가능

2. 간단한 fanout
   - HTTP URI에서 요청된 것을 기반으로 단일 IP 주소에서 1개 이상의 서비스로 트래픽을 라우팅
   - 인그레스를 사용하면 로드 밸런서의 수를 최소로 유지 가능
     ![](https://kubernetes.io/ko/docs/images/ingressFanOut.svg)
3. 이름 기반의 가상 호스팅

   - 동일한 IP 주소에서 여러 호스트 이름으로 HTTP 트래픽을 라우팅하는 것을 지원

   ![](https://kubernetes.io/ko/docs/images/ingressNameBased.svg)

4. TLS
   - 개인 키 및 인증서가 포함된 시크릿(Secret)을 지정해서 인그레스를 보호함
   - 인그레스 리소스는 단일 TLS 포트인 443만 지원하고 인그레스 지점에서 TLS 종료를 가정함
   - 가능한 모든 하위 도메인에 대해 인증서가 발급되어야 하기 때문에 TLS는 기본 규칙에서 작동하지 않는다.
     → tls 섹션의 hosts는 rules섹션의 host와 명시적으로 일치해야 한다.
   - [예시]
     ```bash
     apiVersion: v1
     kind: Secret
     metadata:
       name: testsecret-tls
       namespace: default
     data:
       tls.crt: base64 encoded cert
       tls.key: base64 encoded key
     type: kubernetes.io/tls
     ```
5. 로드밸런싱
   - 인그레스 컨트롤러는 로드 밸런싱 알고리즘, 백엔드 가중치 구성표 등 모든 인그레스에 적용되는 일부 로드 밸런싱 정책 설정으로 부트스트랩됨

### 인그레스 업데이트

- kubectl describe ingress {ingress name}
- kubectl edit ingress {ingress name}

### 인그레스 대신 서비스 노출하는 방식

- Service.Type=LoadBalancer
- Service.Type=NodePort

# 클러스터 내 어플리케이션 접근

## NGINX 인그레스(Ingress) 컨트롤러로 Minikube에서 인그레스 설정하기

HTTP URI에 따라 요청을 Service web 또는 web2로 라우팅하는 간단한 인그레스를 설정하는 방법

### Minikube 클러스터 생성(로컬에서 생성)

- 아직 클러스터를 가지고 있지 않다면, minikube를 사용해서 생성하거나 Killercoda, KodeKloud, Play with Kubernetes 같은 쿠버네티스 플레이그라운드 중 하나를 사용함

### 인그레스 컨트롤러 활성화

1. minikube addons enable ingress
2. kubectl get pods -n ingress-nginx
3. kubectl create deployment web --image=[gcr.io/google-samples/hello-app:1.0](http://gcr.io/google-samples/hello-app:1.0)
4. deployment 노출
   kubectl expose deployment web --type=NodePort --port=8080
5. 서비스 생성 여부와 노드 포트에서 사용 가능한지 확인

   kubectl get service web

6. 노드포트(NodePort)로 서비스에 접속

   minikube service web --url

### 인그레스 생성

```bash
apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example-ingress
  spec:
    ingressClassName: nginx
    rules:
      - host: hello-world.info#로 서비스로 트래픽 보내는 인그레스 정의
        http:
          paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: web
                  port:
                    number: 8080
```

1. 인그레스 오브젝트 생성
   kubectl apply -f [https://k8s.io/examples/service/networking/example-ingress.yaml](https://k8s.io/examples/service/networking/example-ingress.yaml)
2. IP 주소 설정 확인
   kubectl get ingress
3. 호스트 컴퓨터의 /etc/hosts 파일 맨 하단에 다음 행 추가(관리자 권한 필요)
   `172.17.0.15 [hello-world.info](http://hello-world.info)`
   → 웹브라우저가 hello-world.info URL에 대한 요청을 Minikube로 전송함

### 두번째 deployment 생성

kubectl create deployment web2 --image=[gcr.io/google-samples/hello-app:2.0](http://gcr.io/google-samples/hello-app:2.0)

kubectl expose deployment web2 --port=8080 --type=NodePort

### 기존 인그레스 수정

1. example-ingress.yaml 하단에 다음 줄 추가

   ```bash
             - path: /v2
               pathType: Prefix
               backend:
                 service:
                   name: web2
                   port:
                     number: 8080
   ```

2. 변경 사항 적용

   kubectl apply -f example-ingress.yaml

### 인그레스 테스트하기

curl [hello-world.info](http://hello-world.info/)

curl [hello-world.info/v2](http://hello-world.info/v2)

+)Minikube를 로컬에서 실행하는 경우 브라우저에서 [hello-world.info](http://hello-world.info/) 및 [hello-world.info/v2에](http://hello-world.info/v2%EC%97%90) 접속할 수 있음.
