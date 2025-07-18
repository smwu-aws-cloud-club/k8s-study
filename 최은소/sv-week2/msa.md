# MSA vs Monolith 아키텍처 비교

## 모놀리식 아키텍처 (Monolithic Architecture)

모놀리식 아키텍처는 애플리케이션의 모든 구성 요소가 하나의 단일 단위로 통합되어 있는 전통적인 소프트웨어 아키텍처이다.

**핵심 특징:**
- 단일 배포 단위 (Single Deployable Unit)
- 통합된 코드베이스 (Unified Codebase)
- 공유 데이터베이스 (Shared Database)
- 프로세스 내 통신 (In-Process Communication)

**장점:**
- **단순성**: 하나의 코드베이스로 관리 용이
- **빠른 개발**: 초기 개발 속도가 빠름
- **쉬운 테스트**: 통합 테스트가 간단
- **단순한 배포**: 하나의 배포 단위
- **강한 일관성**: ACID 트랜잭션 지원
- **디버깅 용이**: 단일 프로세스에서 디버깅
- **성능 최적화**: 내부 호출로 인한 높은 성능

**단점:**
- **확장성 제한**: 부분적 확장 불가
- **기술 종속**: 단일 기술 스택에 의존
- **단일 장애점**: 전체 시스템 장애 위험
- **코드 복잡성**: 애플리케이션 증가 시 복잡도 급증
- **배포 리스크**: 작은 변경도 전체 배포 필요
- **팀 간 의존성**: 개발팀 간 조율 필요

## 마이크로서비스 아키텍처 (Microservices Architecture)

마이크로서비스 아키텍처는 애플리케이션을 작고 독립적인 서비스들의 집합으로 구성하는 분산 아키텍처 접근 방식이다.

**핵심 특징:**
- 독립적인 서비스 (Independent Services)
- 서비스별 데이터베이스 (Database per Service)
- 네트워크 기반 통신 (Network-based Communication)
- 분산 배포 (Distributed Deployment)

**장점:**
- **독립적 확장**: 서비스별 개별 확장 가능
- **기술 다양성**: 서비스별 최적 기술 스택 선택
- **장애 격리**: 부분 장애가 전체에 미치는 영향 최소화
- **팀 자율성**: 서비스별 독립적 개발 및 배포
- **지속적 배포**: 빈번한 배포 가능
- **비즈니스 정렬**: 서비스가 비즈니스 도메인과 일치
- **재사용성**: 서비스 재사용 가능

**단점:**
- **분산 복잡성**: 분산 시스템의 복잡성 증가
- **네트워크 지연**: 서비스 간 통신 오버헤드
- **데이터 일관성**: 분산 트랜잭션 관리의 어려움
- **운영 복잡성**: 모니터링, 로깅, 보안 복잡성 증가
- **개발 오버헤드**: 초기 설정 및 인프라 구축 비용
- **테스트 복잡성**: 통합 테스트의 어려움

# Spring MVC vs Spring Cloud MSA 비교

## Spring MVC

Spring MVC는 Spring Framework의 웹 모듈로서, Model-View-Controller 아키텍처 패턴을 따르는 전통적인 웹 애플리케이션 개발 프레임워크이다.

**핵심 특징:**
- **MVC 패턴**: Model, View, Controller의 명확한 분리
- **DispatcherServlet**: 모든 HTTP 요청을 처리하는 중앙 컨트롤러
- **단일 애플리케이션**: 하나의 배포 가능한 WAR/JAR 파일
- **전통적인 웹 개발**: 서버 사이드 렌더링 및 RESTful API 제공

**핵심 컴포넌트:**
- **DispatcherServlet**: 중앙 요청 처리기
- **Controller**: 비즈니스 로직 처리
- **Service Layer**: 비즈니스 서비스 구현
- **Repository**: 데이터 접근 계층
- **View Resolver**: 뷰 렌더링 처리

**장점:**
- **개발 단순성**: 단일 프로젝트로 모든 기능 구현
- **빠른 개발**: 프로토타입 및 중소규모 애플리케이션에 적합
- **통합 테스트**: 전체 애플리케이션 테스트가 간단
- **트랜잭션 일관성**: ACID 트랜잭션으로 강한 일관성 보장
- **성능 최적화**: 메모리 내 호출로 높은 성능
- **단순한 배포**: 하나의 배포 파일로 간단한 배포

**단점:**
- **확장성 제한**: 특정 기능만 확장하기 어려움
- **기술 종속성**: 단일 기술 스택에 의존
- **팀 확장성**: 대규모 팀에서 코드 충돌 가능성
- **단일 장애점**: 한 부분의 오류가 전체 시스템에 영향
- **배포 위험**: 작은 변경도 전체 시스템 재배포 필요

## Spring Cloud MSA

Spring Cloud는 분산 시스템 및 마이크로서비스 아키텍처 구현을 위한 도구 모음으로, 클라우드 네이티브 애플리케이션 개발을 지원한다.

**핵심 특징:**
- **분산 시스템**: 독립적인 서비스들의 집합
- **서비스 디스커버리**: 서비스 간 자동 발견 및 연결
- **API Gateway**: 단일 진입점을 통한 라우팅
- **분산 구성 관리**: 중앙화된 설정 관리

**핵심 컴포넌트:**
- **Service Discovery**: Eureka, Consul
- **API Gateway**: Spring Cloud Gateway
- **Config Management**: Spring Cloud Config
- **Circuit Breaker**: Resilience4J
- **Load Balancing**: Spring Cloud LoadBalancer

**장점:**
- **독립적 확장성**: 서비스별 개별 확장 가능
- **기술 다양성**: 서비스별 최적 기술 스택 선택
- **장애 격리**: 서비스 간 장애 격리 및 복구
- **팀 자율성**: 서비스별 독립적 개발 및 배포
- **지속적 배포**: 개별 서비스의 빈번한 업데이트
- **클라우드 친화적**: 컨테이너 및 클라우드 환경에 최적화

**단점:**
- **복잡성 증가**: 분산 시스템의 복잡성
- **네트워크 오버헤드**: 서비스 간 통신 비용
- **데이터 일관성**: 분산 트랜잭션의 복잡성
- **운영 복잡성**: 여러 서비스의 모니터링 및 관리
- **초기 비용**: 인프라 구축 및 학습 비용

# Spring Cloud MSA 구성 요소 및 설계 방식

## Spring Cloud 핵심 구성 요소

### 1. Service Discovery & Registry (서비스 발견 및 등록)

**목적**: 마이크로서비스들이 서로를 동적으로 찾고 통신할 수 있도록 지원

**주요 구성요소:**
- **Spring Cloud Netflix Eureka**: 서비스 레지스트리 서버
- **Spring Cloud Consul**: HashiCorp Consul 기반 서비스 발견
- **Spring Cloud Zookeeper**: Apache Zookeeper 기반 서비스 발견

### 2. API Gateway (API 게이트웨이)

**목적**: 모든 클라이언트 요청의 단일 진입점 제공 및 라우팅 관리

**주요 구성요소:**
- **Spring Cloud Gateway**: 리액티브 방식의 API 게이트웨이
- **Spring Cloud Netflix Zuul**: 전통적인 서블릿 기반 게이트웨이 (유지보수 모드)

### 3. Configuration Management (설정 관리)

**목적**: 분산 환경에서 중앙화된 설정 관리

**주요 구성요소:**
- **Spring Cloud Config**: Git 기반 설정 서버
- **Spring Cloud Consul Config**: Consul KV 기반 설정
- **Spring Cloud Vault**: HashiCorp Vault와 통합

### 4. Circuit Breaker (회로 차단기)

**목적**: 장애 전파 방지 및 시스템 복원력 향상

**주요 구성요소:**
- **Spring Cloud Circuit Breaker**: 추상화 레이어
- **Resilience4J**: 함수형 프로그래밍 기반 회로 차단기
- **Netflix Hystrix**: 전통적인 회로 차단기 (유지보수 모드)

### 5. Load Balancing (로드 밸런싱)

**목적**: 서비스 인스턴스 간 요청 분산

**주요 구성요소:**
- **Spring Cloud LoadBalancer**: 클라이언트 사이드 로드밸런서
- **Netflix Ribbon**: 전통적인 로드밸런서 (유지보수 모드)

### 6. Distributed Tracing (분산 추적)

**목적**: 마이크로서비스 간 요청 흐름 추적 및 모니터링

**주요 구성요소:**
- **Spring Cloud Sleuth**: 분산 추적 라이브러리
- **Zipkin**: 분산 추적 시스템
- **Jaeger**: 분산 추적 플랫폼