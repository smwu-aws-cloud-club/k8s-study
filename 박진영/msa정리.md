# MSA 정리

# MSA 개념 및 구성 요소 이해

## MSA vs Monolith 비교 (참고 [1](https://techblog.uplus.co.kr/msa-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C-api-%EB%AC%B8%EC%84%9C-%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0-d8381c015fed), [2](https://www.msap.ai/docs/msa-expert-from-concepts-to-practice/part-1-msa-fundamentals/chapter-1-introduction-to-msa/section-1-2-what-is-microservices-architecture/))

### MSA(Microservice Architecture)

- 독립적으로 배포가능한 서비스 단위로 비즈니스를 분리하는 아키텍처 모델
- 이론적으로 각각의 작은 서비스는 독립적인 비즈니스 로직과 데이터베이스를 가짐
- 장점
  - 서비스 간 의존성이 줄어들어 장애 발생시 격리와 대처가 쉽고 빠름
  - 팀간 독립적으로 배포할 수 있어 생산성 향상
- 단점
  - 각각의 도메인마다 API 문서가 생성됨→“문서의 파편화”
    (Open API Specification로 해결)
  - 서비스 간 통신 비용 발생
  - 분산 시스템의 복잡성
  - 테스트 복잡성 증가

### Monolith

- 전통적인 소프트웨어 아키텍처 모델
- 어플리케이션과 관련된 모든 요소가 하나의 통일된 단위로 개발
  → 비즈니스와 관련된 모든 요소가 결합&거대한 네트워크
- 장점:개발한 서비스는 배포가 간단하고 통합 테스트 시나리오 구성이 용이
- 단점:확장이 어려우며 서비스 간의 의존성이 강함

## spring MVC vs spring cloud msa

spring MVC

- Servlet API를 기반으로 하는 전통적인 웹(모놀리식) 애플리케이션 프레임워크
- DispatcherServlet이 HTTP 요청을 받아서, 컨트롤러에게 요청을 분배하고, 뷰를 렌더링
- 특징
  - 단일 어플리케이션 구조(모놀리식)
  - Model-View-Controller(모델-뷰-컨트롤러) 패턴 적용
  - RESTful 웹사이트 및 API 설계 지원
  - 유연한 데이터 바인딩, 다양한 뷰 기술 지원
  - 상대적으로 단순한 인프라와 배포 구조

[Spring Web MVC Reference1](https://docs.spring.io/spring-framework/reference/web/webmvc.html)

[Spring Web MVC 상세 API2](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/mvc.html)

spring cloud msa

- 마이크로서비스 아키텍처(분산 시스템) 구현을 위한 도구와 프레임워크 모음
- 특징
  - 비스 레지스트리, API Gateway, 구성 서버, 로드밸런싱 등 마이크로서비스 구현에 필요한 패턴과 솔루션 제공
  - 분산 구성/설정(Cloud Config), 서비스 디스커버리(Eureka 등), 부하분산, 장애감내(Circuit Breaker)
  - 컨테이너 및 오케스트레이션 환경(Kubernetes 등)과 연계하여 운영 및 자동화 지원
  - Cloud Native, DevOps, CI/CD 환경과의 통합
  [Spring Cloud Reference4](https://docs.spring.io/spring-cloud/docs/current/reference/html/)
  [Spring Cloud Train Documentation6](https://docs.spring.io/spring-cloud-release/reference/index.html)
  [Spring Cloud 프로젝트 소개7](https://spring.io/projects/spring-cloud)

| 항목              | Spring Web MVC                              | Spring Cloud MSA                                        |
| ----------------- | ------------------------------------------- | ------------------------------------------------------- |
| **적용 구조**     | 단일(모놀리식) 웹/REST 앱                   | 마이크로서비스 분산 시스템                              |
| **프레임워크**    | `spring-webmvc`                             | Spring Cloud, Netflix OSS, 다양한 Cloud 모듈            |
| **확장성**        | 전체 애플리케이션 단위                      | 서비스별(기능 단위)로 독립적 확장 가능                  |
| **배포 방식**     | 한 번에 배포(일괄)                          | 서비스 단위 개별 배포                                   |
| **장애 대응**     | 일부 장애가 전체에 영향                     | 장애 격리, 한 서비스 장애 시 다른 서비스 정상 작동      |
| **필수 컴포넌트** | DispatcherServlet, Controller, ViewResolver | Config Server, Discovery, Gateway, Circuit Breaker 등   |
| **API 설계**      | REST API/Ops 지원                           | REST + 메시징, 이벤트, 서비스 간 호출, 동적 라우팅 지원 |
| **운영/자동화**   | 필수적이지 않음                             | CI/CD, Monitoring, Logging, Cloud 환경 연계 자동화 중요 |
| **인프라 복잡도** | 상대적으로 단순                             | 복잡함(분산 트랜잭션, 서비스간 통신, 보안, 장애대응 등) |

# spring cloud msa 구성 요소 및 설계 방식 이해

| **폴더/파일**                | **설명**                                              |
| ---------------------------- | ----------------------------------------------------- |
| src/main/java                | 서비스별 자바 소스 코드 저장소                        |
| src/main/resources           | 설정 파일(application.yml/properties 등), 리소스 파일 |
| src/main/resources/static    | 정적 리소스(HTML, CSS, JS 등)                         |
| src/main/resources/templates | 템플릿 엔진 파일(Thymeleaf, JSP 등)                   |
| src/test/java                | 테스트 코드                                           |
| pom.xml 또는 build.gradle    | 빌드/의존성 관리, Spring Cloud 관련 starter 모듈 명시 |

- 프로젝트 루트에 서비스별 모듈(예: **`discovery-server`**, **`config-server`**, **`gateway-service`**, **`user-service`** 등)을 폴더별로 분리하는 **멀티 모듈 구조**가 흔합니다.
- 각 서비스는 독립 실행 가능(Spring Boot)하도록 설계합니다.
- 공통 라이브러리는 별도 **`common`** 모듈로 개발·공유할 수 있습니다.

| **기능**             | **설명**                                                                    | **대표 컴포넌트**               |
| -------------------- | --------------------------------------------------------------------------- | ------------------------------- |
| 분산/버전 설정 관리  | 중앙 서버에서 서비스별 구성·환경파일 관리, 동적 갱신 가능                   | Spring Cloud Config             |
| 서비스 등록/발견     | 각 서비스가 중앙 레지스트리에 자기 위치 등록, 서비스 간 위치·상태 동적 조회 | Eureka, Consul                  |
| API Gateway          | 모든 클라이언트/외부 요청의 단일 진입점, 인증·라우팅·정책 적용              | Spring Cloud Gateway            |
| 부하 분산            | 여러 인스턴스로 분산 호출, 각 서비스에 균형 있게 트래픽 배분                | Spring Cloud LoadBalancer       |
| 서비스 간 통신       | REST, 메시징 등으로 서비스 간 데이터 교환                                   | OpenFeign, RestTemplate, Stream |
| 장애 복원력/회로차단 | 특정 서비스 장애 시 전체 시스템 장애 확산 방지, 빠른 장애 감지/격리         | Resilience4j, Circuit Breaker   |
| 메시지 기반 통신     | 이벤트/비동기 메시징을 통한 확장성/비동기 처리                              | Spring Cloud Stream             |
| 분산 트랜잭션/추적   | 서비스 연동 과정 추적, 장애 진단 및 데이터 일관성 관리                      | Sleuth, Zipkin, Saga 등         |

- **Spring Cloud Config**: 외부/중앙 설정 관리
- **Spring Cloud Eureka**: 서비스 등록·발견
- **Spring Cloud Gateway**: 라우팅, 인증, API 게이트웨이
- **Spring Cloud LoadBalancer**: 트래픽 부하 분산
- **Spring Cloud Circuit Breaker**: 장애 복원력 제공
- **Spring Cloud Stream**: 이벤트·메시징 통신
- **Spring Cloud Sleuth/Zipkin**: 모니터링/분산 추적

### **설계 및 적용 원칙**

- **각 서비스 별도 모듈, 독립적 배포**가 기본 전제
- **서비스 등록 및 발견 패턴 적용**
  Eureka, Consul 등 레지스트리를 통한 동적 서비스 위치 관리 방식 채택. → k8s로 대체/병용 가능
- **API Gateway**를 통해 외부 트래픽 집중 관리
- **중앙 설정, 서비스 디스커버리, 동적 라우팅** 등 분산 아키텍처 최적화
- **회로 차단기, 장애 복원 패턴** 등 고가용성 확보
- **Kubernetes 등과 연계**하여 클라우드 네이티브 환경 지원

- **도메인 주도 설계(DDD)**: 각 서비스의 경계와 책임을 비즈니스 도메인 기준으로 구분.
- **자동화된 배포 및 인프라 관리**: DevOps, CI/CD, 컨테이너 등 자동화와 연계한 운영 체계 강조.
- **분산 환경 최적화**: 부하분산, 탄력성(Resilience), 장애격리 등을 위한 구조적 패턴 필수화.
- **외부설정·확장성 강화**: 환경 변화에 유연하게 대응할 수 있도록 외부화된 설정 관리와 유기적 확장 지원.

## springboot 환경 설정

https://spring.io/projects/spring-cloud#overview

https://jinaenya.tistory.com/48
