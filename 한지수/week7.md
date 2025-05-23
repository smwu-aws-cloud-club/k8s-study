# 로깅 아키텍처

## ✅로깅 아키텍처란

> 클러스터에서 로그는 노드, 파드 또는 컨테이너와는 독립적으로 별도의 스토리지와 라이프사이클을 가져야한다. 이 개념을 클러스터-레벨 로깅이라고 한다.
>

클러스터-레벨 로깅은 로그를 저장, 분석, 쿼리하기 위해서는 별도의 백엔드가 필요하다. 쿠버네티스가 로그 데이터를 위한 네이티브 스토리지 솔루션을 제공하지는 않지만, 쿠버네티스에 통합될 수 있는 기존의 로깅 솔루션이 많이 있다.

## ✅파드와 컨테이너 로그

쿠버네티스는 실행중인 파드의 컨테이너에서 출력하는 로그를 감시한다.

- 초당 한 번씩 표준 출력에 텍스트를 기록하는 컨테이너를  포함하는 파드 매니페스트를 사용

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: counter
    spec:
      containers:
      - name: count
        image: busybox:1.28
        args: [/bin/sh, -c,
                'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
    
    ```


- 해당 파드를 실행하기 위한 명령어

    ```bash
    kubectl apply -f https://k8s.io/examples/debug/counter-pod.yaml
    ```


- 로그를 가져오기 위한 명령어

    ```bash
    kubectl logs counter
    ```


- 컨테이너 이전 인스턴스에 대한 로그를 검색하기 위한 명령어

    ```bash
    kubectl logs counter -c count
    ```

  파드에 여러 컨테이너가 있는 경우 명령어 `-c` 플래그와 컨테이너 로그를 지정해야 한다


### 노드가 컨테이너 로그를 처리하는 방법

- stdout(표준 출력) 및 stderr(표준 에러) 스트림에 의해 생성된 모든 출력은 컨테이너 런타임이 처리하고 리디렉션 시킨다

  → Kubelet : CRI 로깅 포맷으로 표준화

- 컨테이너가 재시작하는 경우 : kubelet은 종료된 컨테이너 하나를 로그와 함께 유지, 파드가 노드에서 축출되면 해당하는 모든 컨테이너와 로그가 함께 축출
- kubelet : 일반적으로 `kubelet logs`를 통해 접근 가능(최신 로그만 확인)

### 로그 로테이션

- 로테이션 구성 이후, kubelet은 컨테이너 로그를 로테이트하고 로깅 경로 구조를 관리
- kubelet은 이 정보를 컨테이너 런타임에 전송하고(CRI를 사용), 런타임은 지정된 위치에 컨테이너 로그를 기록
- kubelet 설정파일로`containerLogMaxSize` 및 `containerLogMaxFiles`를 설정 가능

  → 로그 파일의 최대 크기와 각 컨테이너에 허용되는 최대 파일 수를 각각 구성 가능


## ✅시스템 컴포넌트 로그

- 컨테이너에서 실행되는 것
    - 쿠버네티스의 스케줄러, 컨트롤러 매니저, API 서버는 파드로 실행된다 → 대부분 스태틱 파드로 실행
    - ectd는 컨트롤 플레인에서 실행된다 → 일반적으로 스태틱 파드, kube-proxy를 사용하는 경우는 데몬
- 실행 중인 컨테이너와 관련된  것
    - kubelet이 파드와 그룹회된 컨테이너를 실행시킨다

### 로그의 위치

노드의 운영체제에 따라 기록하는 방법이 다르다.

- 리눅스
    - systemd를 사용하는 시스템
        - `journald`에 작성
        - `journalctl` 로 확인
    - systemd를 사용하지 않는 시스템
        - `/var/log` 디렉터리의 `.log` 파일에 작성
        - kube-log-runner를 통해 간접적으로 실행하여 리디렉션
- 윈도우
    - 기본값 : C:\var\logs
    - 기타 배포 도구 : C:\var\log\kubelet
    - 다른 경로 : kube-log-runner를 통해 간접적으로 실행하여 리디렉션

## ✅클러스터-레벨 로깅 아키텍처

- 모든 노드에서 실행되는 노드-레벨 로깅 에이전트를 사용한다
- 애플리케이션 파드에 로깅을 위한 전용 사이드카 컨테이너를 포함한다
- 애플리케이션 내에서 로그를 백엔드로 직접 푸시한다.

### 노드 로깅 에이전트 사용

- 각 노드에 노드-레벨 로깅 에이전트를 포함하여 클러스터-레벨 로깅을 구현할 수 있다.

  → 로그를 노출하거나 백엔드로 전송하는 전용 도구이다. 일반적으로는 각 노드의 모든 애플리케이션 컨테이너 로그 디렉터리에 접근 가능한 컨테이너이다.

- 에이전트는 모든 노드에서 실행되어야 하므로 DaemonSet으로 배포하는 것이 일반적이다.
- 노드-레벨 로깅은 노드당 하나의 에이전트만을 생성하며, 애플리케이션 변경 없이 동작한다.
- 컨테이너는 로그를 stdout 및 stderr로 출력하며 형식은 정해져 있지 않다. 노드-레벨 에이전트는 이를 수집하여 취합을 위해 전달한다.

### 로깅 에이전트와 함께 사이드카 컨테이너 사용

- 사이드카 컨테이너는 애플리케이션 로그를 자체 `stdout`으로 스트리밍한다.
- 사이드카 컨테이너는 로깅 에이전트를 실행하며, 애플리케이션 컨테이너에서 로그를 가져오도록 구성된다.

**사이드카 컨테이너 스트리밍**

- 자체 `stdout` 및 `stderr` 스트림으로 로그를 출력한다.
- 로그 소스로는 파일, 소켓, journald 등을 사용할 수 있다.
- 로그 출력은 kubelet과 노드 로깅 에이전트가 자동으로 처리함.

예시: 파드는 단일 컨테이너를 실행하고, 컨테이너는 서로 다른 두 가지 형식을 사용하여 서로 다른 두 개의 로그 파일에 기록한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done      
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

- 두 컴포넌트를 컨테이너의 `stdout` 스트림으로 리디렉션한 경우
    - 동일한 로그 스트림에 서로 다른 형식의 로그 항목을 작성 (X)
    - 두 개의 사이드카 컨테이너를 생성 (O)

      → 각 사이드카 컨테이너는 공유 볼륨에서 특정 로그 파일을 테일(tail)한 다음 로그를 자체 `stdout` 스트림으로 리디렉션할 수 있다.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done      
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox:1.28
    args: [/bin/sh, -c, 'tail -n+1 -F /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox:1.28
    args: [/bin/sh, -c, 'tail -n+1 -F /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}

```

클러스터에 노드-레벨 로깅 에이전트가 설치되어 있다면, 사이드카 컨테이너의 `stdout/stderr` 로그 스트림을 자동으로 수집할 수 있고, 로그의 소스 컨테이너에 따라 로그 라인을 파싱하도록 에이전트를 구성할 수 있다.

다만, 로그를 파일에 기록한 후 다시 stdout으로 스트리밍하는 방식은 다음과 같은 단점이 있다:

- 파드의 CPU 및 메모리 사용량이 낮더라도, 디스크 사용량은 두 배로 늘어난다.
- 특히 단일 로그 파일만 사용하는 애플리케이션이라면, 스트리밍 사이드카를 사용하는 것보다 `/dev/stdout`으로 직접 출력하는 것이 효율적이다.

사이드카 컨테이너는 로그 파일을 자체적으로 로테이션할 수 없는 애플리케이션에 `logrotate` 등을 주기적으로 실행해주는 방식으로 활용될 수 있다.

**로깅 에이전트가 있는 사이드카 컨테이너**

노드-레벨 로깅 에이전트가 상황에 맞게 충분히 유연하지 않은 경우, 애플리케이션과 함께 실행하도록 특별히 구성된 별도의 로깅 에이전트를 사용하여 사이드카 컨테이너를 생성할 수 있다.

예시: 로깅 에이전트가 포함된 사이드카 컨테이너를 구현하는 데 사용할 수 있는 두 가지 매니페스트

1. luentd를 구성하는 컨피그맵(ConfigMap)이 포함

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluentd.conf: |
    <source>
      type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>

    <source>
      type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>

    <match **>
      type google_cloud
    </match>    

```

1. fluentd가 실행되는 사이드카 컨테이너가 있는 파드를 설명

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done      
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-agent
    image: registry.k8s.io/fluentd-gcp:1.30
    env:
    - name: FLUENTD_ARGS
      value: -c /etc/fluentd-config/fluentd.conf
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /etc/fluentd-config
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config

```

**애플리케이션에서 직접 로그 노출**

애플리케이션에서 직접 로그를 노출하거나 푸시하는 클러스터-로깅은 쿠버네티스의 범위를 벗어난다.

<br>
<hr>

# 리소스 모니터링 도구

## 리소스 모니터링 도구란?

> 쿠버네티스에서 애플리케이션 모니터링은 단일 모니터링 솔루션에 의존하지 않는다. 신규 클러스터에서는, 리소스 메트릭 또는 완전한 메트릭 파이프라인으로 모니터링 통계를 수집할 수 있다.
>

## ✅**리소스 메트릭 파이프라인**

리소스 메트릭 파이프라인은 Horizontal Pod Autoscaler(HPA) 컨트롤러와 kubectl top 유틸리티에서 사용하는 제한된 메트릭 집합을 제공한다. 이 메트릭은 경량의 단기 인메모리 저장소인 metrics-server에 의해 수집되며, metrics.k8s.io API를 통해 노출된다. metrics-server는 클러스터 내 모든 노드를 자동으로 발견하고 각 노드의 kubelet에 CPU와 메모리 사용량을 질의한다. kubelet은 쿠버네티스 마스터와 노드 간의 연결 고리 역할을 하며, 노드에서 실행 중인 파드와 컨테이너를 관리한다.

- kubelet은 각 파드를 해당 컨테이너와 매칭시키고, 컨테이너 런타임 인터페이스(CRI)를 통해 개별 컨테이너의 리소스 사용 통계를 가져온다.
- 컨테이너는 리눅스의 cgroup과 네임스페이스를 활용해 구현되며, 만약 컨테이너 런타임이 사용 통계치를 제공하지 않는 경우에는 kubelet이 cAdvisor 코드를 사용해 직접 통계를 조회한다.
- 이렇게 수집된 파드 리소스 사용량 통계는 metrics-server로 전달되고, metrics-server는 이를 metrics.k8s.io API를 통해 제공한다.

이 API는 인증이 필요한 kubelet의 읽기 전용 포트에서 `/metrics/resource/v1beta1` 경로를 통해 접근 가능하다. 따라서 클러스터 내 모든 노드의 리소스 사용 현황을 실시간으로 모니터링하고, 자동 확장이나 리소스 관리를 지원할 수 있다.

## ✅**완전한 메트릭 파이프라인**

완전한 메트릭 파이프라인은 더 풍부한 메트릭에 접근할 수 있도록 지원한다. 쿠버네티스는 Horizontal Pod Autoscaler(HPA)와 같은 메커니즘을 활용하여, 이런 메트릭을 기반으로 클러스터의 현재 상태에 맞게 자동으로 스케일링하거나 클러스터를 조정할 수 있다. 모니터링 파이프라인은 kubelet에서 메트릭을 수집하고, 이를 쿠버네티스 내의 `custom.metrics.k8s.io`와 `external.metrics.k8s.io` API를 구현한 어댑터를 통해 외부로 노출한다.

- CNCF 프로젝트인 프로메테우스는 기본적으로 쿠버네티스, 노드, 그리고 프로메테우스 자체를 모니터링할 수 있다.
- 그러나 CNCF 프로젝트가 아닌 완전한 메트릭 파이프라인 프로젝트는 쿠버네티스 공식 문서의 범위에 포함되지 않는다.