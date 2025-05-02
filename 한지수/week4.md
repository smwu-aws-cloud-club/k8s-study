# 서비스
> 클러시터 내부에서 실행되는 여러 파드 집합을 하나의 네트워크 엔드포인트로 추상화하여 내외부에서 접근할 수 있도록 해주는 기능이다.
>

<aside>
💡

쿠버네티스 파드는 생성과 삭제가 반복되며, 각 파드는 고유한 IP 주소를 갖지만 파드가 교체될 때마다 IP가 바뀔 수 있다.

</aside>

<aside>
💡

디플로이먼트를 사용하면 파드 집합이 동적으로 변하기 때문에, 프론트엔드 파드가 백엔드 파드의 IP를 직접 추적해 연결하는 것이 어렵다.

</aside>

→ 다음과 같은 문제를 해결하기 위해 서비스를 제공하여, 백엔드 파드 집합에 **고정된 DNS이름과 IP를 부여**하고, 프론트엔드는 이 서비스 이름만 사용해 항상 백엔드에 연결하게 된다.

<br>

## 1. 서비스 리소스

### 추상적 개념

쿠버네티스에서 서비스는 여러 파드를 논리적으로 묶고, 이들에 접근하는 방법을 정의하는 추상적 개념이다(=마이크로-서비스)

서비스는 보통 셀렉터를 이용해 파드 집합을 지정하며, 실제 파드가 바뀌더라도 클라이언트는 서비스만 알면 연결할 수 있다

EX) 여러개의 백엔드 파드가 있을 때, 프론트엔드는 각각의 벡엔드를 신경쓰지 않고 서비스 엔드포인트만 사용하면 된다.

→ 이렇게 서비스는 파드와 클라이언트 사이의 결합을 느슨하게 만들어준다.

### 서비스 접근 방식

서비스의 접근 방식은 두 가지로 나눠서 볼 수 있다.

- 서비스 디스커버리를 위해 쿠버네티스 API를 사용할 수 있는 경우

  → 파드가 바뀌거나 추가/삭제되어도 엔드포인트 슬라이스가 자동으로 최신 상태로 유지되고, 애플리케이션은 이를 API로 실시간 조회할 수 있다.

  → 클라우드 네이티브 서비스 디스커버리


- 쿠버네티스 API를 직접 사용할 수 없는, 즉 클라우드 네이티브가 아닌 일반 어플리케이션의 경우

  → 쿠버네티스가 대신 네트워크 포트나 로드벨런서 같은 전통적인 방식으로 연결을 제공한다. 이 경우 애플리케이션은 쿠버네티스 내부구조를 몰라도 되고, 외부에서 할당된 주소나 포트로만 접근한다.

  → 전통적 네트워크 연결방식

<br>

## 2. 서비스 정의

### 서비스 정의 구조

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp # 대상 파드 레이블
  ports:
    - protocol: TCP
      port: 80            # 서비스 노출 포트
      targetPort: 9376    # 파드의 실제 포트
```

- 기본 구조
    - selector : 레이블을 기준으로 연결할 파드 선택
    - ports: 외부 접근 포트 매핑
- Cluster IP 할당
    - 쿠버네티스는 서비스에 고유한 가상 IP를 부여한다.
    - 이 IP는 클러스터 내부에서만 접근 가능하며, 서비스 프록시가 트레픽을 파드로 전달한다.
- 동적 엔드포인트 관리
    - 서비스 컨트롤러는 selector와 일치하는 파드를 지속적으로 탐지한다.
    - 파드가 추가/삭제되면 엔트포인트 오브젝트가 자동으로 업데이트되어 최신 파드 IP를 추적한다.
- 포트 매핑 규칙
    - targetPort 생략 시 : port 값과 동일하게 설정된다

      EX: `port: 80` → `targetPort: 80`.

    - 파드정의에서 포트에 이름을 지정하면, 서비스에서 이름으로 참조 가능하다.

        ```yaml
        # 파드 정의
        ports:
          - containerPort: 80
            name: http-web
        
        # 서비스 정의
        targetPort: http-web  # 이름으로 포트 지정
        ```

    - 쿠버네티스 서비스는 여러 포트와 다양한 프로토콜을 한 번에 지원할 수 있어서, 파드의 포트가 달라도 하나의 서비스로 관리할 수 있다.

### 셀렉터가 없는 서비스

셀렉터 없이 서비스와 엔드포인트를 직접 연결하는 방식이다.

- 파드와 자동연결하지 않는다
- 대신 EndpointSlice 오브젝트를 직접 만들어서 서비스가 연결할 IP와 포트를 수동으로 지정한다
- 이 방식은 쿠버네티스 내부 파드 뿐 아니라 외부 시스템이나 점진적 이전 중인 리소스도 동일한 서비스 방식으로 연결할 수 잇어 환경구성에 큰 유연성을 제공한다.

서비스에 셀렉터를 지정하지 않으면, 매칭되는 엔드포인트 슬라이스 오브젝트가 자동으로 생성되지 않으므로 수동으로 매핑한다.

```yaml
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


### 커스텀 엔드포인트

서비스를 위한 엔드포인트슬라이스 오브젝트를 생성할 때, 엔드포인트 이름으로는 원하는 어떤 이름도 사용할 수 있다.

커스텀 엔드포인트슬라이스를 만들 때는 다음을 지킨다.

- 이름은 네임스페이스 내에서 고유해야 한다.
- 반드시 `kubernetes.io/service-name` 레이블로 어떤 서비스와 연결되는지를 명시한다
- 엔드포인트 IP는 루프백(127.0.0.1 등), 링크-로컬(169.254.x.x 등), 또는 다른 서비스의 Cluster IP를 쓰면 안 된다.
- 누가 이 엔드포인트슬라이스를 관리하는지 `endpointslice.kubernetes.io/managed-by` 레이블에 명확히 적어야 하며, 쿠버네티스 시스템이 쓰는 controller라는 예약된 값은 사용하지 않아야 한다.

### 셀렉터가 없는 서비스에 접근하기

셀렉터가 없는 서비스도 일반 서비스와 동일하게 접근할 수 있으며, 트래픽은 엔드포인트슬라이스에 정의된 IP와 포트로 전달된다.

- ExternalName 서비스

  셀렉터 대신 외부 DNS 이름을 사용해 외부 리소스에  연결하는 특수한 서비스 타입


### 엔드포인트슬라이스

쿠버네테스 서비스가 관리하는 네트워크 엔드포인트를 여러개의 작은 그룹으로 나누어 관리하는 오브젝트이다.

- 쿠버네티스 클러스터는 각 엔드포인트슬라이스가 얼마나 많은 엔드포인트를 나타내는지를 추적한다.
- 기존의 모든 엔드포인트슬라이스가 엔드포인트를 최소 100개이상 갖게되면 새 엔드포인트슬라이스를 생성한다.

### 엔드포인트

쿠버네티스 API에서 네트워크 엔드포인트의 목록을 정의하며, 일반적으로 트래픽이 아닌 어떤 파드에 보내질 수 있는지를 정의하기 위해 서비스가 참조한다.

- 엔드포인트 대신 엔드포인트 슬라이스 API를 사용하는 것을 권장한다.
- 쿠버네티스는 단일 엔드포인트 오브젝트에 최대 1000개의 엔드포인트만 저장할 수 있다. 초과 시 `endpoints.kubernetes.io/over-capacity: truncated` 어노테이션을 추가하여 초과상태를 표시한다.
- 이때 실제 트래픽은 모든 백엔드(1000개 이상)로 전송되지만 기존 엔드포인트 API를 사용하는 로드벨런서는 최대 1000개까지만 인식하여 트래픽을 분산한다.

### 애플리케이션 프로토콜

`appProtocol` 필드는 각 서비스 포트에 대한 애플리케이션 프로토콜을 지정하는 방법을 제공한다.

- 이 필드의 값은 상응하는 엔드포인트와 엔드포인트슬라이스 오브젝트에 의해 미러링된다.

<br>

## 3. 멀티-포트 서비스

### 서비스 멀티 포트 지원

쿠버네티스는 서비스 오브젝트에서 멀티 포트 정의를 구성할 수 있도록 지원한다.

멀티 포트 서비스는 각 포트에 고유한 이름을 반드시 지정해야하며, 이름은 소문자 영숫자와 하이픈(-)만 사용하고, 영숫자로 시작·끝나야 한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http                  # 첫 번째 포트이름 
      protocol: TCP               # 프로토콜 
      port: 80                    # 서비스가 노출하는 포트 
      targetPort: 9376            # 실제 파드에서 수신하는 포트
    - name: https                 # 두 번째 포트이름
      protocol: TCP               # 프로토콜 
      port: 443                   # 두 번째 서비스 포트 
      targetPort: 9377            # 실제 파드에서 수신하는 두 번째 포트
```

### 클러스터 IP 직접 지정

- 서비스 생성 시, `.spec.clusterIP` 필드로 원하는 클러스터 IP를 지정할 수 있다.
- 지정한 IP는 클러스터의 `service-cluster-ip-range ****`범위 내여야 하며, 잘못된 IP를 입력하면 오류가 발생한다.
  서

### 서비스 디스커버리 방법

쿠버네티스는 서비스를 찾는 두 가지 기본 방법을 지원한다.

1. **환경변수**
- 파드가 생성될 때, 해당 노드의 kubelet이 서비스별 환경 변수를 추가한다.
- `{SVCNAME}_SERVICE_HOST` 및 `{SVCNAME}_SERVICE_PORT` 환경 변수가 추가된다.
- 이 환경 변수는 파드가 생성될 때 이미 존재하는 서비스에 한해 추가된다.

1. **DNS**
- CoreDNS와 같은 DNS 서버가 클러스터 내 모든 서비스에 대해 DNS 레코드를 자동으로 생성한다.
- 파드는 서비스 이름으로 DNS 조회를 통해 클러스터 IP를 찾을 수 있다.
- 서비스 포트 이름이 지정된 경우, `_http._tcp.my-service.my-ns`와 같은 SRV 레코드로 포트 정보도 조회할 수 있다.
- ExternalName 서비스는 DNS를 통해서만 접근 가능하다.

<br>

## 4. 헤드리스 서비스

헤드리스 서비스(Headless Service)는 로드 밸런싱과 클러스터 IP를 제공하지 않는 특수한 서비스 유형이다.

- 명시적으로 클러스터 IP (`.spec.clusterIP`)에 "None"을 지정한다.
- 클러스터 IP가 할당되지 않고, kube-proxy가 이러한 서비스를 처리하지 않으며, 플랫폼에 의해 로드 밸런싱 또는 프록시를 하지 않는다.

### **동작 방식**

- **셀렉터가 있는 경우**
    - 컨트롤 플레인은 쿠버네티스 API 내에서 엔드포인트슬라이스 오브젝트를 생성
    - 서비스 하위(backing) 파드들을 직접 가리키는 A 또는 AAAA 레코드(IPv4 또는 IPv6 주소)를 반환 변경
- **셀렉터가 없는 경우**
    - 쿠버네티스 컨트롤 플레인은 엔드포인트슬라이스 오브젝트를 생성하지 않는다
    - DNS 시스템은 다음 중 하나를 탐색한 뒤 구성한다.
        - `type: ExternalName` 서비스에 대한 DNS CNAME 레코드
        - `ExternalName` 이외의 모든 서비스 타입에 대해, 서비스의 활성(ready) 엔드포인트의 모든 IP 주소에 대한 DNS A / AAAA 레코드

## 5. 서비스 퍼블리싱

클러스터 외부에서 접근 가능한 서비스를 만들고자 할 때, 쿠버네티스의 `ServiceType`을 통해 서비스의 노출 방식을 지정할 수 있다.
type 종류는 다음과 같으며,  중첩(nested) 기능으로 설계되어, 각 단계는 이전 단계에 더해지는 형태이다. 

- **ClusterIP (기본값)**:

                                                                  서비스를 클러스터 내부 IP에만 노출하며, 클러스터 내부에서만 접근 가능하다.

- **NodePort**:

  각 노드의 고정 포트(NodePort)를 통해 외부에서 접근할 수 있도록 서비스를 노출한다.

  내부적으로는 `ClusterIP` 서비스가 함께 생성된다.

- **LoadBalancer**:

  클라우드 공급자의 로드 밸런서를 이용해 서비스를 외부에 노출한다.

  일반적으로 NodePort 및 ClusterIP 서비스를 기반으로 구성된다.

- **ExternalName**:

  서비스를 외부 도메인 이름(CNAME 레코드)으로 매핑한다.

  프록시나 포워딩 없이 DNS를 통해 직접 연결된다.


**Ingress를 통한 서비스 노출**

`Ingress`는 서비스 유형은 아니지만, 클러스터의 외부 요청을 내부 서비스로 라우팅하는 진입점 역할을 한다.

하나의 IP 주소에 여러 서비스를 매핑할 수 있어, 라우팅 규칙을 하나의 리소스로 통합하는 데 유용하다.

### 1) NodePort 유형

NordPort는 클러스터 외부에서 노드의 IP와 특정 포트를 통해 서비스에 접근할 수 있도록 한다.

- 쿠버네티스 컨트롤 플레인은 `-service-node-port-range` ㅍ플래그로 정의된 범위에서 할당된다
- 서비스는 할당된 포트를 `.spec.ports[*].nodePort` 필드에 나타낸다
- 자유롭게 자체 로드 밸런싱 솔루션을 설정하거나, 쿠버네티스가 완벽하게 지원하지 않는 환경을 구성하거나, 하나 이상의 노드 IP를 직접 노출시킬 수 있다.
- 서비스의 프로토콜애 매치되도록 쿠버네티스는 포트를 추가로 할당한다. 클러스터 외부에서 클러스터의 아무 노드에 연결하여 `type: NodePort` 서비스로 접근할 수 있다.

**포트 직접 선택하기**

특정 포트 번호를 원한다면, `nodePort` 필드에 값을 명시할 수 있다. 

```yaml
	ports:
      # 기본적으로 그리고 편의상 `targetPort` 는 `port` 필드와 동일한 값으로 설정된다.
      - port: 80
        targetPort: 80
        # 선택적 필드
        # 기본적으로 그리고 편의상 쿠버네티스 컨트롤 플레인은 포트 범위에서 할당한다(기본값: 30000-32767)
        nodePort: 30007
```

**`type: NodePort` 서비스를 위한 커스텀 IP 주소 구성 [](https://kubernetes.io/ko/docs/concepts/services-networking/service/#service-nodeport-custom-listen-address)**

각 노드가 여러 네트워크에 연결되어 있는 경우 커스텀 IP주소를 설정할 수 있다.

- 특정 IP 대역에서만 NodePort 서비스를 노출하고 싶다면, kube-proxy 실행 시 `--nodeport-addresses` 플래그(또는 설정 파일의 `nodePortAddresses` 필드)를 사용해서 허용할 IP 블록을 지정할 수 있다.
- 예를 들어, `--nodeport-addresses=127.0.0.0/8`로 설정하면 NodePort 서비스는 오직 루프백(127.0.0.1 등)에서만 접근할 수 있다.

### 2) 로드벨런서 유형

`LoadBalancer` ****타입은 클라우드 공급자의 외부 로드 밸런서를 생성하여 서비스를 외부에 노출함하고 트래픽은 로드 밸런서를 거쳐 백엔드 파드로 전달된다,

```yaml
ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
status:
  loadBalancer:
    ingress:
      - ip: 192.0.2.127
```

- `loadBalancerIP` 필드로 고정 IP를 지정 가능하고,지정하지 않으면 임시 IP가 할당된다. 공급자가 지원하지 않으면 `loadBalancerIP`는 무시된다.
- `cloud-controller-manager` 컴포넌트를 통해 NodePort를 내부적으로 포함하며, 로드 밸런서가 해당 포트로 트래픽 전달한다.

**프로토콜 유형이 혼합된 로드밸런서**

- 로드밸런서 유형은 여러 포트를 정의할 경우, 모든 포트가 동일한 프로토콜을 사용해야 한다.
- `MixedProtocolLBService` 기능 게이트는 서로 다른 포트에 다른 프로토콜 사용을 허용한다.
- 클라우드 공급자가 혼합 프로토콜을 지원하지 않으면, 단일 프로토콜만 허용됨.

**로드밸런서 NodePort 할당 비활성화**

- `spec.allocateLoadBalancerNodePorts: false` 로 설정하여 노드 포트 할당을 비활성화 할 수 있다.
- 트래픽을 직접 파드로 라우팅하는 로드 밸런서 구현에서만 사용해야 한다.
- 기존 서비스에서 `false`로 변경해도, 기존 NodePort는 유지된다. 수동으로 `nodePorts` 필드를 제거해야 해제가 가능하다.

**로드벨런서 구현 클래스 지정**

`spec.loadBalancerClass` 필드를 설정하여 클라우드 제공자가 설정한 기본값 이외의 로드 밸런서 구현을 사용할 수 있다.

- 기본값은 `nil`이며, 이 경우 `-cloud-provider`로 설정된 기본 로드밸런서가 자동 적용된다.
- `spec.loadBalancerClass`가 지정되면 기본 로드밸런서는 무시되고, 해당 클래스를 감시하는 컨트롤러가 필요하다.
- 오직 `type: LoadBalancer` 서비스에서만 설정 가능하다.
- 한 번 설정하면 변경할 수 없다.
- 값은 `"example.com/internal-vip"` 형태의 레이블 스타일 문자열이며, 접두사 없는 값은 최종 사용자용으로 예약된다.

**내부 로드 벨런서**

- 혼합된 환경(내/외부 트래픽)에서는 내부 트래픽을 같은 네트워크 주소 블록 안에서 라우팅해야 할 수 있다.
- 수평 분할 DNS 구조에서는 내부용과 외부용으로 두 개의 LoadBalancer 서비스가 필요하다
- 내부 로드 밸런서를 설정하려면 클라우드 공급자별 어노테이션을 서비스에 추가해야 함

| 클라우드 | 어노테이션 키 | 값 예시 |
| --- | --- | --- |
| **GCP** | `cloud.google.com/load-balancer-type` | `Internal` |
| **AWS** | `service.beta.kubernetes.io/aws-load-balancer-internal` | `true` |
| **Azure** | `service.beta.kubernetes.io/azure-load-balancer-internal` | `true` |
| **IBM Cloud** | `service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type` | `private` |
| **OpenStack** | `loadbalancer.openstack.org/internal` | `true` |
| **Baidu Cloud** | `k8s.io/elb-internal` | `1` |
| **Tencent Cloud** | `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid` | `<subnet-id>` |
| **Alibaba Cloud** | `service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type` | `intranet` |
| **OCI** | `oci.oraclecloud.com/load-balancer-internal` | `true` |

**AWS에서 TLS/SSL 지원**

AWS 환경의 쿠버네티스 클러스터에서 TLS/SSL을 적용하려면 LoadBalancer 타입의 서비스에 어노테이션을 설정해야 한다.

**1. 인증서 지정 (필수)**

```yaml

metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012

```

- TLS에 사용할 인증서의 ARN을 지정함
- 인증서는 IAM에 업로드된 인증서 또는 AWS Certificate Manager(ACM) 인증서 가능

**2. 파드가 알려주는 프로토콜 지정**

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: (https|http|ssl|tcp)
```

- 파드가 사용하는 백엔드 프로토콜을 명시한다.
- 프로토콜에 따라 선택되는 프록시 계층이 달라진다
    - `http`, `https`: L7 (애플리케이션 계층) 프록시
        - 헤더 파싱 및 `X-Forwarded-For` 추가
    - `tcp`, `ssl`: L4 (전송 계층) 프록시
        - 헤더 수정 없이 그대로 전달


**3. SSL이 적용될 포트 지정 (혼합 환경에서 사용)**

```yaml
 metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
        service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443,8443"
```

- 지정된 포트(예: `443`, `8443`)만 SSL 인증서를 사용한다
- 나머지 포트(예: `80`)는 평문 HTTP 처리한다.

**4. SSL 네고시에이션 정책 지정 (선택)**

```yaml
 metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-0
```

- 쿠버네티스 v1.9 이상에서 사용 가능하다.
- AWS의 미리 정의된 SSL 정책을 적용한다.
- 사용 가능한 정책은 CLI로 확인:

```bash
aws elb describe-load-balancer-policies --query 'PolicyDescriptions[].PolicyName'
```

**AWS에서 지원하는 프록시 프로토콜 [](https://kubernetes.io/ko/docs/concepts/services-networking/service/#aws%EC%97%90%EC%84%9C-%EC%A7%80%EC%9B%90%ED%95%98%EB%8A%94-%ED%94%84%EB%A1%9D%EC%8B%9C-%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C)**

AWS에서 실행되는 클러스터에 대한 프록시 프로토콜 지원을 활성화하려면, 다음의 서비스 어노테이션을 사용할 수 있다.

```yaml
metadata:
name: my-service
annotations:
service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
```

**AWS ELB 접근 로그 관련 어노테이션**

AWS ELB 서비스의 접근 로그를 관리하기 위한 어노테이션이 있다.

- `service.beta.kubernetes.io/aws-load-balancer-access-log-enabled`

  : 접근 로그의 활성화 여부를 제어한다.

- `service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval`

  : 접근 로그를 게시하는 간격을 분 단위로 제어한다. 5분 또는 60분의 간격으로 지정할 수 있다

- `service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name`

  : 로드 밸런서 접근 로그가 저장되는 Amazon S3 버킷의 이름을 제어한다.

- `service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix`

  : Amazon S3 버킷을 생성한 논리적 계층을 지정한다.


**AWS의 연결 드레이닝(Draining)**

로드 밸런서에서 인스턴스를 제거할 때 기존의 연결을 종료하지 않고 일정 시간 동안 유지할 수 있도록 해주는 기능이다.

새로운 트래픽은 다른 인스턴스로 라우팅되고 기존 연결은 종료되지 않아 서비스 중단 없이 원활한 처리가 가능하다.

1. **`service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled`**
    - `true` 값으로 설정하여 Classic ELB의 연결 드레이닝을 활성화한다.
2. **`service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout`**
    - 연결 드레이닝의 최대 대기 시간을 초 단위로 설정한다.


**다른 AWS ELB 관련 어노테이션**

1. **`service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout`**
    - 유휴 상태의 연결에 대한 타임아웃을 설정한다.
2. **`service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled`**
    - 교차-영역 로드 밸런싱을 활성화한다.
3. **`service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags`**
    - ELB에 추가할 태그를 설정한다.
4. **`service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold`**
    - 백엔드가 정상으로 간주되기 위한 헬스 체크 성공 횟수를 설정한다.
5. **`service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold`**
    - 백엔드가 비정상으로 간주되기 위한 헬스 체크 실패 횟수를 설정한다.
6. **`service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval`**
    - 헬스 체크 간격을 설정한다.
7. **`service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout`**
    - 헬스 체크 타임아웃을 설정한다.

**AWS의 네트워크 로드 밸런서 지원**

- AWS 네트워크 로드 밸런서를 사용하기 위해서는 `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"` 어노테이션을 설정해야 힌다.
- NLB는 클라이언트의 IP 주소를 노드로 전달한다.

  이때, `.spec.externalTrafficPolicy`가 `Cluster`로 설정되면 클라이언트의 IP는 종단 파드에 전달되지 않으며, `Local`로 설정하면 클라이언트 IP가 파드로 전달된다.
  그러나 `Local` 설정에서는 트래픽 분배가 불균등할 수 있다. 파드가 없는 노드는 헬스 체크에서 실패하고 트래픽을 수신하지 못한다.

- 트래픽을 균등하게 분배하려면 `DaemonSet`을 사용하거나 파드 안티어피니티를 설정하여 파드가 동일한 노드에 배치되지 않도록 해야 한다.

**Tencent Kubernetes Engine (TKE) CLB 어노테이션**

TKE에서는 클라우드 로드 밸런서를 관리하기 위한 여러 어노테이션을 제공한다:

1. **`service.kubernetes.io/qcloud-loadbalancer-backends-label`**
    - 지정된 레이블을 가진 노드에 로드 밸런서를 바인딩한다.
2. **`service.kubernetes.io/tke-existed-lbid`**
    - 기존 로드 밸런서의 ID를 지정한다.
3. **`service.kubernetes.io/service.extensiveParameters`**
    - 로드 밸런서에 대한 사용자 지정 매개변수를 설정한다.
4. **`service.kubernetes.io/service.listenerParameters`**
    - LB 리스너에 대한 사용자 지정 매개변수를 설정한다.
5. **`service.kubernetes.io/loadbalance-type`**
    - 로드 밸런서의 유형을 지정한다
6. **`service.kubernetes.io/qcloud-loadbalancer-internet-charge-type`**
    - 퍼블릭 네트워크 대역폭 청구 방법을 설정한다.
7. **`service.kubernetes.io/qcloud-loadbalancer-internet-max-bandwidth-out`**
    - 대역폭 값을 지정한다.
8. **`service.kubernetes.io/local-svc-only-bind-node-with-pod`**
    - 로드 밸런서가 파드가 실행 중인 노드만 등록한다.

### **3) ExternalName 유형**

`ExternalName` 서비스는 클러스터 외부의 DNS 이름을 서비스로 매핑한다. 이 서비스를 사용하면 특정 DNS 이름을 통해 외부 리소스에 접근할 수 있다.

```yaml
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

- **`spec.externalName`**외부 DNS 이름을 지정한다.
- 서비스에 접근할 때, 클러스터 내 DNS 서비스는 `externalName`으로 지정된 값의 CNAME 레코드를 반환한다.
- 리다이렉션은 프록시 또는 포워딩을 통하지 않고 DNS 수준에서 발생한다
- HTTP나 HTTPS와 같은 프로토콜에서 사용 시, `Host` 헤더 불일치나 TLS 인증서 문제로 오류가 발생할 수 있다.

### 외부 IP

외부에서 클러스터 노드로 직접 접근할 수 있는 외부 IP 주소가 있다.

```yaml
spec:
  type: ClusterIP
  externalIPs:
    - 80.11.12.10
  ports:
    - port: 80
```

- 클라이언트는 `80.11.12.10:80`으로 접속하면 `my-service`를 통해 파드에 접근할 수 있습니다.
- 이 IP로 들어오는 트래픽은 서비스의 엔드포인트(파드 등)로 라우팅된다.
- 외부 IP는 쿠버네티스가 직접 관리하지 않으며, 클러스터 관리자가 설정 및 유지 책임을 진다.
- 모든 ServiceTypes에서 사용할 수 있다.

## 세션 스티킹

특정 클라이언트로부터의 연결이 매번 동일한 파드로 전달되도록 하고 싶다면, 클라이언트의 IP 주소 기반으로 세션 어피니티를 구성할 수 있다

## API 오브젝트

서비스는 쿠버네티스 REST API의 최상위 리소스이다.

## 가상 IP 주소 메커니즘

가상 IP 주소를 갖는 서비스를 노출하기 위해 쿠버네티스가 제공하는 메커니즘이다.

<br>
<hr>
<br>

# 인그레스

## 인그레스란?

> 인그레스는 클러스터 외부에서 클러스터 내부 서비스로 HTTP와 HTTPS 경로를 노출한다. 트래픽 라우팅은 인그레스 리소스에 정의된 규칙에 의해 컨트롤된다.
>

- 외부에서 클러스터 내부 서비스에 접근할 수 있도록 HTTP(S) 기반의 URL, 로드 밸런싱, SSL/TLS 종료, 가상 호스팅 등을 설정할 수 있는 리소스이다.
- 인그레스 동작을 실제로 수행하는 구성요소는 인그레스 컨트롤러이다

  → 일반적으로 로드 밸런서와 함께 동작하며, 에지 라우터나 프론트엔드 설정도 가능.

- HTTP/HTTPS가 아닌 포트나 프로토콜을 외부에 노출하려면`ervice.type: NodePort` 또는 `LoadBalancer`를 사용해야 한다.

## 전제 조건들

- 인그레스 리소스만 생성하면 동작하지 않는다

  → 반드시 인그레스 컨트롤러가 배포되어 있어야 함.

    - `ingress-nginx`와 같은 인그레스 컨트롤러를 직접 배포하거나, 클라우드에서 제공하는 컨트롤러를 사용.
- 여러 종류의 인그레스 컨트롤러 중에서 선택 가능하지만,

  이상적으로는 공통 사양을 따르도록 설계되어 있다.

<br>

## 인그레스 리소스

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress                # 인그레스 리소스의 이름
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /   # NGINX 인그레스 컨트롤러에서 경로 재작성 규칙 설정 (모든 요청 경로를 /로 재작성)
spec:
  ingressClassName: nginx-example      # 사용할 인그레스 컨트롤러 클래스 이름(여기서는 nginx-example)
  rules:
  - http:
      paths:
      - path: /testpath               # 외부에서 /testpath로 들어오는 요청을
        pathType: Prefix              # Prefix: /testpath로 시작하는 모든 경로에 적용
        backend:
          service:
            name: test                # test라는 서비스로 트래픽 전달
            port:
              number: 80              # test 서비스의 80번 포트로 전달

```

- **기본 구조 필드:**

  `apiVersion`, `kind`, `metadata`, `spec` 필드는 반드시 명시해야 한다.

- **이름 규칙:**

  `metadata.name`은 유효한 DNS 서브도메인이어야 한다.

- **spec 필드:**

  로드 밸런서나 프록시 서버 구성을 위한 HTTP(S) 규칙을 포함한다. 인그레스는 오직 HTTP/HTTPS 트래픽만 처리 가능.

- **ingressClassName:**

  생략하려면 기본 IngressClass가 정의되어 있어야 한다.

- **Ingress-NGINX 예외:**
    - `-watch-ingress-without-class` 플래그를 사용하면

  `ingressClassName` 없이도 동작 가능하지만,

  명시적으로 ingressClassName을 지정하는 것이 권장된다.


### 인그레스 규칙

- **선택적 호스트**

  호스트가 지정되지 않기에 지정된 IP 주소를 통해 모든 인바운드 HTTP 트래픽에 규칙이 적용된다.

- **경로 목록**

  각각 `service.name` 과 `service.port.name` 또는 `service.port.number` 가 정의되어 있는 관련 백엔드를 가지고 있다. 로드 밸런서가 트래픽을 참조된 서비스로 보내기 전에 호스트와 경로가 모두 수신 요청의 내용과 일치해야 한다.


### DefaultBackend

`.spec.rules`가 없으면, 모든 트래픽은 단일 기본 백엔드로 전송된다. 이 경우 `.spec.defaultBackend` 를 필수로 지정하게 된다.

- 요청을 처리할 기본 서비스(서비스명 + 포트)를 지정.
- 인그레스 리소스에서 명시할 수도 있고, 보통은 인그레스 컨트롤러의 구성 옵션으로 설정된다.
- 인그레스 리소스에 설정하지 않으면, 컨트롤러의 기본 동작에 따름.

### **리소스 백엔드**

인그레스 오브젝트와 동일한 네임스페이스 내에 있는 다른 쿠버네티스 리소스에 대한 ObjectRef이다. 

`Resource` 는 서비스와 상호 배타적인 설정이며, 둘 다 지정하면 유효성 검사에 실패한다. 

정적 자산이 있는 오브젝트 스토리지 백엔드로 데이터를 수신하려고 사용한다.

인그레스를 생성한 후 다음의 명령으로 확인할 수 있다.
`kubectl describe ingress ingress-resource-backend`

### 경로 유형

인그레스의 각 경로에는 해당 경로 유형이 있어야 한다.

- `ImplementationSpecific`:
    - 구현할 때 별도 `pathType` 으로 처리하거나, `Prefix` 또는 `Exact` 경로 유형과 같이 동일하게 처리할 수 있다.
- `Exact`
    - URL 경로의 대소문자를 엄격하게 일치시킨다.
- `Prefix`
    - URL 경로의 접두사를 `/` 를 기준으로 분리한 값과 일치시킨다.

### **다중 일치**

- 인그레스의 여러 경로가 요청과 일치하는 경우
    - 가장 긴 일치하는 경로가 우선된다.
- 두 개의 경로가 여전히 동일하게 일치하는 경우
    - 접두사 경로 유형보다 정확한 경로 유형을 가진 경로가 사용 된다.

<br>

## 호스트네임 와일드카드

도메인 이름에서 별표(*)를 사용해 여러 하위 도메인을 한 번에 지정하는 방식이다

`rules.host` 필드에서 도메인명을 정확히 지정하거나(`foo.bar.com`), 와일드카드(`*.foo.com`)로 설정할 수 있다.

- EX) 일치여부

| 설정된 호스트 값 | 요청의 Host 헤더 | 일치 여부 설명 |
| --- | --- | --- |
| `*.foo.com` | `bar.foo.com` | 일치 – 와일드카드는 `bar` 부분 |
| `*.foo.com` | `baz.bar.foo.com` | 불일치 – `baz.bar`는 단일 레이블 아님 |
| `*.foo.com` | `foo.com` | 불일치 – `*`는 `foo.com`과 직접 일치하지 않음 |

→ 와일드카드는 오직 한 단계의 DNS 레이블만 대체할 수 있다.

<br>

## 인그레스 클래스

인그레스는 서로 다른 컨트롤러에 의해 구현된다면, 각 인그레스에서 클래스를 구현해야하는 컨트롤러 이름을 포함하여 추가 구성이 포함된 IngressClass 리소스에 대한 참조 클래스를 지정해야 한다.

```yaml
spec:
  controller: example.com/ingress-controller
  parameters:
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: external-lb
```

- `.spec.parameters` 로 해당 인그레스클래스와 연관있는 다른 리소스를 참조할 수 있다.

### **인그레스클래스 범위**

인그레스클래스가 외부 컨트롤러와 통신할 때 사용할 파라미터을 지정할 때,  클러스터 범위인지, 네임스페이스 범위인지 설정할 수 있다.

- 클러스터 범위 (기본값)
    - `.spec.parameters.scope`를 지정하지 않거나 `Cluster`로 설정하면 클러스터 범위 리소스를 참조한다

    ```yaml
    parameters:
      scope: Cluster  # 이 파라미터 리소스가 클러스터 범위임을 명시 (기본값이므로 생략 가능)
      apiGroup: k8s.example.net  # 파라미터 리소스를 정의한 API 그룹
      kind: ClusterIngressParameter  # 참조할 리소스의 타입 
      name: external-config-1  # 참조할 리소스의 이름
    
    ```


- 네임스페이스 범위
    - `.spec.parameters.scope`를 `Namespace`로 지정하면 네임스페이스 범위 리소스를 참조합니다.
    - 네임스페이스를 `.spec.parameters.namespace`에 명시해야 한다.

```yaml
parameters:
  scope: Namespace  # 이 파라미터 리소스가 네임스페이스 범위임을 명시
  apiGroup: k8s.example.com  # 파라미터 리소스를 정의한 API 그룹
  kind: IngressParameter  # 참조할 리소스의 타입
  namespace: external-configuration  # 파라미터 리소스가 위치한 네임스페이스
  name: external-config  # 참조할 리소스의 이름

```

### 기본 인그레스클래스

기본 인그레스클래스를 설정하면, `ingressClassName` 필드가 지정되지 않은 새 Ingress 리소스에 자동으로 할당됩니다.

- `IngressClass` 리소스에 `ingressclass.kubernetes.io/is-default-class: "true"` 어노테이션을 추가하면, 해당 `IngressClass`가 클러스터의 기본 인그레스클래스로 설정된다.
- 클러스터에서는 하나의 기본 인그레스클래스만 설정해야 한다.
- 일부 인그레스 컨트롤러는 기본 인그레스클래스가 설정되지 않아도 작동할 수 있지만 명시하는 것이 권장됩니다.

```yaml
metadata:
  labels:
    app.kubernetes.io/component: controller  # IngressClass가 인그레스 컨트롤러와 관련 있음을 나타냄
  name: nginx-example  # IngressClass의 이름
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # 기본 IngressClass로 설정
spec:
  controller: k8s.io/ingress-nginx  # 이 IngressClass를 사용할 컨트롤러 지정
```
<br>

## 인그래스 유형들

### 단일 서비스로 지원되는 인그레스

- 단일 서비스를 노출하려면, 인그레스에서 규칙 없이 `defaultBackend`를 지정한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  defaultBackend:
    service:
      name: test  # 기본 백엔드로 사용될 서비스 이름
      port:
        number: 80  # 서비스의 포트 번호
```

- `kubectl apply -f` 명령어로 인그레스를 생성한 후, `kubectl get ingress test-ingress`를 사용하여 인그레스의 상태를 확인할 수 있다.

### 간단한 팬아웃

**팬아웃 구성**은 하나의 IP 주소(또는 호스트 이름)를 기반으로 **URI 경로에 따라 여러 서비스로 트래픽을 라우팅**하는 방식이다.


```yaml
	paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service1   # /foo 경로는 service1으로 라우팅
            port:
              number: 4200
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: service2   # /bar 경로는 service2로 라우팅
            port:
              number: 8080

```

- 인그레스 컨트롤러는 지정된 서비스(service1, service2)가 존재하면 자동으로 로드 밸런서를 구성하여 요청을 분기한다.
- 기본 백엔드(`default-http-backend`)를 요구하는 인그레스 컨트롤러의 경우 수동으로 생성한다.

### 이름 기반의 가상 호스팅

하나의 IP 주소에서 여러 호스트 이름(도메인) 을 기준으로 트래픽을 라우팅한다.

- Ingress 리소스의 `rules[].host` 필드에 따라 HTTP 요청의 Host 헤더를 검사하여 서비스로 전달합니다.

- 도메인 별로 서비스 분기

    ```yaml
    spec:
      rules:
      - host: foo.bar.com       # foo.bar.com 요청 → service1
        http:
    	    .....
      - host: bar.foo.com       # bar.foo.com 요청 → service2
        http:
          .....
    ```


- 기본 도메인 없이 fallback 처리 포함

    ```yaml
    spec:
      rules:
      - host: first.bar.com       # Host = first.bar.com → service1
        http:
          ....
      - host: second.bar.com      # Host = second.bar.com → service2
        http:
          ....
      - http:                     # Host 미지정 → 기본 fallback 처리
         ....
    ```

    - 만약 규칙에 정의된 호스트 없이 인그레스 리소스를 생성하는 경우, 이름 기반 가상 호스트가 없어도 인그레스 컨트롤러의 IP 주소에 대한 웹 트래픽을 일치시킬 수 있다
    - Host 헤더가 `first.bar.com`, `second.bar.com`에 해당되지 않을 경우, `service3`로 요청이 전달된다.

### TLS

인그레스 리소스는 **TLS 암호화를 통해 HTTPS(443 포트)** 접속을 지원한다.Ingress 리소스에서 `tls` 섹션을 사용하면 **HTTPS 트래픽을 수신하고 처리**할 수 있다.

- TLS Secret 생성

  TLS 설정 시 **인증서(`tls.crt`)와 개인 키(`tls.key`)** 를 포함한 **Secret** 을 먼저 생성해야 합니다.


```yaml
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: <base64 인코딩된 인증서>
  tls.key: <base64 인코딩된 키>
type: kubernetes.io/tls
```

- Ingress에서 TLS 설정 예시
    - 클라이언트 → Ingress 컨트롤러까지는 **HTTPS**, 그 이후는 **HTTP(일반 텍스트)** 로 처리됩니다.
    - `tls.hosts` 항목과 `rules[].host` 항목은 **정확히 일치**해야 TLS가 정상 작동합니다

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls  # 위에서 만든 Secret 참조
  rules:
  - host: https-example.foo.com
```

### 로드벨런싱

- 인그레스 컨트롤러는 기본적인 로드 밸런싱 정책을 기반으로 초기화된다.
- 고급 기능(세션 지속성, 동적 가중치 등)은 인그레스를 통해 직접 설정할 수 없으며, 대신 Service의 로드 밸런서를 통해 제공된다.
- 헬스 체크는 인그레스에서 직접 지원하지 않지만, 쿠버네티스의 Readiness Probe와 같은 기능으로 대체 가능하다.

<br>

## 인그레스 업데이트

기존 인그레스를 업데이트해서 새 호스트를 추가하려면, 리소스를 편집해서 호스트를 업데이트 할 수 있다.

- 기존 인그레스 리소스의 상태와 규칙을 확인한다.

  `bashkubectl describe ingress test`

- 새 호스트를 추가하려면 인그레스 리소스를 편집한다.

  `bashkubectl edit ingress test`

- 편집기에서 **`spec.rules`** 아래에 새로운 호스트 규칙을 추가한다.

    ```yaml
    textspec:
      rules:
      - host: foo.bar.com  # 기존 호스트
      - host: bar.baz.com  # 새로운 호스트 생성
        http:
          paths:
          - backend:
              service:
                name: service2
                port:
                  number: 80
            path: /foo
            pathType: Prefix
    ```


→ 편집 내용을 저장하면 `kubectl`이 자동으로 API 서버에 변경사항을 반영한다.

→ 인그레스 컨트롤러는 이 변경을 감지해 로드밸런서 구성을 새로 적용합니다.

- 인그레스 리소스의 YAML 파일을 직접 수정한 뒤, `kubectl replace -f <파일명>.yaml` 명령으로 적용해도 동일한 결과를 얻을 수 있다.

## 가용성 영역의 전체에서의 실패

장애 도메인에 트래픽을 분산시키는 기술은 클라우드 공급자마다 다르다.

## 대안

사용자는 인그레스 리소스를 직접적으로 포함하지 않는 여러가지 방법으로 서비스를 노출할 수 있다.

- `Service.Type=LoadBalancer`사용.
- `Service.Type=NodePort` 사용.