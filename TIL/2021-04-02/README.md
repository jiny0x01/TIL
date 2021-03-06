# TIL

## 오늘 공부한 컨셉



+ 잘못 알고 있는 분산 시스템의 일반화 오류
+ 12 factor app
+ CAP(일관성, 가용성, 파티션) 이론
+ SLA(Service-Level-Agreement)
+ brownfield와 greenfield 시나리오. vm에서 클라우드로 진화하는 과정
+ 모놀리식에서 마이크로서비스로 마이그레이션 패턴
  + strangler 패턴
  + anti-corruption-layer 패턴



## 상세내용



## 분산 시스템의 오류

분산 시스템을 고려할 때 개발자/아키텍트가 알아야 하는 **몇 가지 잘못되거나 근거 없는 가정들**

1. 네트워크는 안정적이다.
   + 누가 안정적이라고 보장하는가? 잠재적인 네트워크 실패에 대응하도록 개발해야한다.
2. 네트워크 지연이 없다.
   + 네트워크 호출과 네트워크에 대한 대화를 자주하지 않는다.
   + 클라이언트 가까운 곳에 데이터를 두도록 클라우드 네이티브 앱을 설계한다.
     - 캐싱
     - CDN(content delivery network)
     - multiregion 배포 활용
3. 대역폭이 무한대다.
   + DDD와 CQRS(command query responsibility segregation) 같은 데이터 패턴은 대역폭이 커야하는 환경에서 유용함
4. 네트워크는 안전하다
   + 보안에 항상 높은 우선순위를 둬야 한다. 
   + defense-in-depth(심층 방어) 같은 방법을 고려
5. 토폴로지는 변하지 않는다(-> 변할 수 있다)
   + 토폴로지: 컴퓨터 네트워크를 물리적으로 연결해 놓은 것
   + 자원 소모량과 초당 요청에 따라 토폴로지를 추가하거나 제거해야함
6. 관리자는 한 명이다.
7. 전송 비용이 없다.
   + 클라우드 네이티브 관점에서 전송 비용을 바라보는 2가지 관점
     1. 전송은 네트워크에서 발생. 네트워크 비용은 클라우드 업체에서 제공함 -> 무료가 아님
     2. payload를 오브젝트로 변환하는 비용은 무료가 아님. 네트워크 호출 지연 외에도 직렬화/역직렬화를 고려해야하는 비싼 연산임
8. 네트워크는 동등하다.
   + 서로 다른 프로토콜의 존재를 고려하므로 동등하지 않음.



## CAP 이론

분산 시스템에서 자주 언급되는 이론. 모든 네트워크 공유 데이터 시스템의 아래 세 가지중 두 가지만 만족한다는 내용

+ Consistency(C, 일관성)는 데이터는 하나의 동일한 복제본을 갖고 있다

+ Availability(A, 고가용성)

+ Partition(P) 네트워크 파티션이 가능함. -> 네트워크 파티션이 **하위 시스템간에 통신 오류를 유발하더라도 데이터 처리 시스템이 데이터를 처리할 수 있는 능력**을 의미

  > 네트워크 파티션은 네트워크 장치의 장애로 인한 네트워크 분할뿐만 아니라 별도의 최적화를 위해 상대적으로 독립적 인 서브넷으로 네트워크 분해를 나타냅니다.
  >
  > 예를 들면) 네트워크에서 여러  노드 A/B는 같은 서브넷에 있고, 노드 C,D는 다른 서브넷에 있다고 가정. 네트워크가 두 서브넷 장치간 교환이 실패하면 partition이 발생함. 이 경우에선 A/B는 노드 C,D와 통신 할 수 없지만(묶어서), 모든 노드 A-D는 이 전과 동일하게 작동

현실에서는 항상 네트워크 파티션이 발생한다. 따라서 CAP 이론에 따르면 일관성과 고가용성만 최적화 할 수 있음.

NoSQL DB는 가용성을 최적화함. SQL 기반 시스템은 ACID 원리를 지키기 위해 일관성을 최적화함

> ACID(atomicity, consistency, isolation, durability)

## 12 factor app

클라우드 네이티브 앱을 개발하기 위한 방법론

1. 코드베이스
   + 애플리케이션당 하나의 코드베이스가 있어서 이 코드베이스로 개발/테스트/실서비스와 같은 환경에 배포할 수 있어야함.
2. 종속성
   + 온프레미스와 클라우드 환경의 차이 때문에 종속성을 읽어버리거나 버전 불일치로 많은 이슈가 발생
   + 컨테이너는 도커파일 안에 종속성을 선언하여 모든 종속성을 컨테이너 안에 넣고 패키징해서 문제를 많이 줄임
3. 설정
   + 코드와 엄격하게 분리해라
   + 쿠버네티스는 ConfigMap
4. 백엔드 서비스
   + 백엔드 서비스: 앱이 일반적인 동작을 위해 네트워크를 통해 사용하는 모든 서비스
   + 클라우드 네이티브 앱의 경우 벡엔드 서비스로 managed caching service 또는 database as a service(DBaaS) 등이 될 수 있다.
   + 결합도를 떨어뜨릴 수 있게 외부 설정 시스템에 저장된 설정 정보로 백엔드 서비스에 접근해라
5. 빌드/릴리즈/실행 분리
   + CI/CD 활용
6. 프로세스
   + 클라우드에선 연산 상태가 없어야함 -> 돈이랑 직결된 문제
   + 데이터는 프로세스 외부에 저장해야함 -> 탄력성을 갖게 됨
7. 데이터 격리
   + MSA의 핵심 요소
   + 각 서비스가 API를 통해서만 접근할 수 있도록 자신의 데이터를 관리. 다른 서비스에 데이터 직접 접근을 막아라
8. 동시성
   + 클라우드에선 확장과 자원 사용량 개선이 장점이다.
   + 각 서비스와 함수를 독립적이고 수평적으로 확장할 수 있음.
9. 폐기 가능
   + 함수나 컨테이너의 인스턴스 개수를 줄일 수 있는 것을 의미
10. 개발/실서비스 일치
    + 컨테이너는 서비스의 모든 종속성을 패키지할 수 있어야함
11. 로그
12. 관리 프로세스
    + 관리 작업은 수명이 짧은 프로세스로 실행하라는 의미(함수/컨테이너)

## Service-Level Agreement(SLA) : 서비스 수준 협약서

| 가용성(%) | 연간 다운타임 | 월간 다운타임 | 주간 다운타임 |
| --------- | ------------- | ------------- | ------------- |
| 99%       | 3.65일        | 7.20시        | 1.68시        |
| 99.9%     | 8.76시        | 43.2분        | 10.1분        |
| 99.99%    | 52.56분       | 4.32분        | 1.01분        |
| 99.999%   | 5.26분        | 25.9초        | 6.05초        |
| 99.9999%  | 31.5초        | 2.59초        | 0.605초       |

두 서비스의 서비스 수준 협약서 계산

서비스1이 99.95%, 서비스2가 99.90%: 0.9995 x 0.9990 = 0.9985005

=> 99.85%



## 서버리스 컴퓨팅

클라우드 공급자가 인프라 밑단과 스케일을 관리한다는 의미. 애플리케이션 자원에 자동으로 할당하거나 해제하면 개발자는 인프라 밑단을 어떻게 관리할지 신경쓰지 않아도 됨

개발자 관점에선 event-driven 프로그래밍 모델

경제적 관점에선 CPU 시간을 소모한 만큼만 비용 지불

FaaS(Function as a Service)를 서버리스라 생각하지만 그건 아님. 기술적인 측면에서만 맞는말

MS 에저 컨테이너 인스턴스(ACI), AWS Fargate, GCP 클라우드 함수의 서버리스 컨테이너가 서버리스의 좋은 예시

서버리스가 제공하는 다른 예시

+ API 관리
+ 머신러닝 서비스

**인프라 밑단 관리 신경쓰지 않고 기능을 쓸 수 있고! 사용자가 사용한 만큼 비용을 지불하는 모델이 서버리스**



## 함수

일반적으로는 서버리스 인프라인 AWS 람다, 에저 funtions와 같은 FaaS 오퍼링을 말함.

FaaS와 컨테이너 특징

| FaaS                      | 컨테이너화된 서비스          |
| ------------------------- | ---------------------------- |
| 한 가지만 수행            | 한 가지 이상 수행            |
| 의존성을 배포할 수 없음   | 의존성을 배포할 수 있음      |
| 한 종류의 이벤트에만 응답 | 한 종류 이상의 이벤트에 응답 |

FaaS 오퍼링의 경제성은 좋지만 적합하지 않은 두 가지 경우

1. vendor lock-in 피하고 싶을 때
   + 특정 FaaS 오퍼링에 맞는 함수를 개발하고 클라우드 공급자의 고수준 클라우드 서비스를 사용해야 함.
   + 따라서 전체 애플리케이션의 이식성이 떨어짐

2. 함수를 온프레미스나 사용자 클러스터에서 실행하고 싶을 때
   + 쿠버네티스 클러스터에서 실행할 수 있는 오픈소스 FaaS 런타임(Kubeless, OpenFasS 등)이 많이 있으니 설치형 FaaS 써라

FaaS나 FaaS 구현에 가장 중요한 부분은 **초기 구동 시간**이다.

일반적으로 실행 명령을 받은 함수는 매우 빠르게 실행을 원하기 때문. 컨테이너는 좋은 초기 구동 시간을 제공하지만 자원 격리를 충분하게 제공하지 않음



## VM에서 클라우드로 진화하게 된 과정

1.  brownfield 시나리오
   + 기존 애플리케이션을 **lift and shift**에서 현대화를 걸처 최적화로 가는 시나리오
2. greenfield 시나리오
   + 처음부터 클라우드로 개발하는 시나리오



### Lift And Shift

클라우드 이전하는 사용자가 처음엔 소프트웨어를 클라우드 장비 내에 직접설치함 

+ 운영 최소화 -> 비용 절감
+ 사용자가 전체 스택을 제어 -> 의존성 에러/ 런타임 충돌/ 자원 경합/ 격리 등 책임 따름



더 자세히 읽어볼 것: https://www.netapp.com/ko/knowledge-center/what-is-lift-and-shift/

### 탈모놀리식 해야하는 이유

+ 배포 시간이 더 빨라짐
+ 필요한 것만 업데이트 하면 됨
+ 특정 기능만 다른 스케일(업/아웃)이 필요할 수 있음
+ 특정 기능은 다른 언어로 개발해야할 수 있음
+ 코드가 너무 크고 복잡해짐

### 모놀리식 -> 마이크로서비스 전환하는 2가지 패턴

+ strangler pattern(스트랭글러 패턴)
  + 새로운 서비스나 기존 구성 요소를 마이크로서비스로 개발
  + 파사드나 게이트웨이에서 사용자 요청을 새로운 애플리케이션으로 보내게함
  + 점진적으로 마이그레이션
+ anti-corruption layer pattern(손상방지레이어패턴)
  + 스트랭글러와 비슷하지만 레거시 애플리케이션에 접근하기 위한 새로운 서비스가 필요할 때 사용



## 클라우드 네이티브 애플리케이션 기초

1. 운영 효율성
   + 모든 것을 자동화하기
     + IaC(Infrastructure as Code)
       + terraform
   + 모든 것을 모니터링하기
   + 모든 것을 문서화하기
   + 변화는 점진적으로
   + 장애 대비
2. 보안
   + depense in depth
     + 소스코드
     + 컨테이너 이미지
       + 항상 필요한 것만 추가 + 필요한 포트만 노출
     + 컨테이너 저장소
       + role-based access control(RBAC)를 이용해 저장소에 누가 접근 했는지 감사해야함.
       + Twistlock 과 같은 도구로 이미지의 취약점 스캔
     + 파드
       + 인증된 저장소에서만 컨테이너 이미지를 다운로드할 수 있는지를 확인해야 함
       + 쿠버네티스에서는 정책 컨트롤러를 사용
     + 클러스터와 오케스트레이터
       + 오케스트레이터를 호스팅 중인 클러스터에 인터넷으로 접근하는 게 필요한지
       + VPN이 적절한지 
       + 오케스트레이터의 control plain에는 보안 접근이 필요하고 감사 로그 켜야함
       + 쿠버네티스 RBAC 켜졌는지 확인
3. 신뢰성과 가용성
   + 신뢰성 : 장애나도 잘 돌아가는거
   + 가용성 : 일정 시간동안 이용할 수 있다는 거
4. 확장성과 비용