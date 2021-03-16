# TIL

## 오늘 공부한 컨셉
1. 커스텀 리소스
2. 오퍼레이터
3. 워크플로우 
4. gRPC 장단점

## 상세내용

### 커스텀 리소스
쿠버네티스 API를 사용자가 원하는대로 확장한 리소스
쿠버네티스에서는 API를 쉽게 확장할 수 있도록 CustomResourceDefinition(CRD) 라는 리소스 지원함


```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
medata:
  name: mypods.crd.example.com
spec:
  group: crd.example.com
  version: v1
  scope: Nmaespaced
  names:
    plural: mypods      # 복수이름
    singular: mypod     # 단수이름
    kind: MyPod         # Kind 이름
    shortNames:         # 축약이름
    - mp
```

위에서 MyPod이라는 리소스를 정의햇으니 만들어보자
```
apiVersion: "crd.example.com/v1"
kind: MyPod
metadata:
  name: mypod-test
spec:
  uri: "any uri"
  customCommand: "custom command"
  image: nginx
```

#### 커스텀 컨트롤러
쿠버네티스 내부적으로 존재하지 않는 사용자 정의 리소스에 대해선(위에 만든 MyPod과 같은 경우) Custom Controller를 사용함.
Custom Controller를 만드는 작업은 언어에따라, 동작방식에 따라 다양함

#### Operator 패턴
Custome Resource와 Custom Controller의 조합을 이용하여 특정 애플리케이션이나 서비스의 생성과 삭제를 관리하는 패턴을 말함
쿠버네티스 코어 API에 포함되지 않은 애플리케이션을 쿠버네티스 native 리소스처럼 동작하게 만들 수 있음.

위 패턴을 잘활용하면 아래의 시나리오가 가능함
> 신규 프로젝트를 시작할 때 마다 CI/CD를 위한 Jenkins 애플리케이션을 매번 새로 생성하고 프로젝트가 완료된 이후에는 Jenkins를 삭제한다고 가정. Operator 패턴을 이용하여 Jenkins라는 사용자 리소스를 생성하여 새로운 Jenkins 애플리케이션이 필요할 때마다 Jenkins 리소스를 생성ㅎ마.

Operator 패턴을 이용하면, 동일한 애플리케이션을 여러 개 설치해야 하는 경우, 리소스 생성만으로 바로 애플리케이션을 사용할 수 있는 장점이 있다.

### Operator tools
+ kubebuilder : go언어
+ Operator Framework : go언어
+ metacontroller
+ KUDO 


### 유용한 Operators
helm chart 마냥 이미 만들어진 오퍼레터를 활용하는 경우가 더 많음

Helm Operator는 helm chart를 사용자 쿠버네티스 리소스처럼 관리할 수 있게 해주는 Operator
helm 사용자 정의 리소스를 작성하면 kelm chart가 설치되고 리소스가 삭제될 떄도 chart도 삭제된다.

### workflow 관리
쿠버네티스 core API에는 일회성 배치 작업을 수행하는 Job 리소스가 존재함.
간단한 retry 정책이나 parallel 실행이 가능하지만, 작업간 종속성 부여, 조건부 실행, 에러 핸들링 등 고급 워크플로우 관리 기능이 부재함.
이를 보안해서 Argo workflow가 있음.

Argo workflow 장점
1. 실행의 단위가 컨테이너 레벨에서 이뤄져서 고립성 높음. 개별 Job마다 실행 환경이 다른 경우 실행 환경이 서로 뒤엉키지 않음
2. 하나의 역할만 담당하는 Job을 단일하게 개발할 수 있어서 재사용성을 높일 수 있음. 데이터 입출력 인터페이스만 잘 맞춰 놓으면 단일한 역할을 담당하는 Job을 여러개 만들어서 레고 블록마냥 쌓아 올릴 수 있음


단점
1. Pod를 생성하고 삭제하는 비용이 적지 않음(이미지를 다운받고 가상 네트워크 디바이스를 연결하고 IP를 부여하고 컨테이너 실행...)
  + 작은 일을 처리하는 많은 Job을 생성하면 오히려 성능 저하 생김. 
  + 작업이 간단하고 리소스가 많이 필요하지 않으면 컨테이너 내부 프로세스나 스레드 레벨에서 처리하는게 좋음
2. 스탭마다 개별적인 컨테이너를 실행하기에 Job간의 데이터를 빠르게 공유하는 것이 비교적 쉽지 않음(volume 연결로 해결 가능)

장단점이 분명하지만 컨테이너끼리 작업 순서를 정의하고 싶을 때 argo workflow 사용

> TMI) workflow를 생성할때는 kubectl apply가 아니라 kubectl create 사용해야함. 매번 새로운 workflow를 생성해야해서.  apply로 생성하면 새로운 Workflow가 아닌 기존 workflow에 변경사항이 적용됨

argo workflow에서는 DAG도 지원함

#### 활용
컨테이너 기반의 워크플로우 엔진이 활용되는 곳 

1. 데이터 파이프라인
2. CI 파이프라인
  + 이미지 빌드, 테스트, 패키징 등 표준화 할 수 있어서 step으로 묶어서 파이프라인 처리해버리면 된다. CI 서버가 필요없어서 컴퓨팅 자원 최소화 가능





### gRPC

gRPC는 분산도니 이기종 애플리케이션을 연결/호출/운영/디버깅 할 수 있는 통신 기술

gRPC 개발할 때 가장 먼저 해야할 일

+ 인터페이스를 정의하는 것
  + 서비스를 사용하는 법
  + 원격으로 호출되는 메서드 종류
  + 해당 메서드를 호출하기 위한 파라미터
  + 메시지 형식 등
  + 이와 같은 서비스 정의에 사용되는 언어를 IDL(Interface Definition Language)

서비스 정의로 코드 만들기

+ 서버
  + 서버 스켈레톤
+ 클라이언트
  + 클라이언트 스텁



#### gRPC 서버

gRPC 서버를 실행하려면 서버에서 다음과 같은 처리가 필요하다

1. 상위 서비스 클래스를 오버라이드해서 생성된 서버 스켈레톤의 서비스 로직을 구현
2. gRPC 서버를 실행해 클라이언트 요청을 수신하고 응답함



클라이언트-서버 메시지 흐름

1. gRPC 클라이언트가 gRPC 서비스를 호출할 때 클라이언트의 gRPC 라이브러리는 프로토콜 버퍼를 사용해 RPC 프로토콜 버퍼형식으로 마샬링함
2. HTTP/2를 통해 전송
3. 서버는 요청을 언마샬링하고 각 프로시저 호출은 프 로토콜 버퍼에 의해 실행됨



#### SOAP(Simple Object Access Protocol)

서비스 지향 아키텍처에서 서비스간 XML 기반의 구조화된 데이터 교환용 표준 통신 기술

+ 메시지 포맷의 복잡성 때문에 분산 애플리케이션 구축의 민첩성을 저해함.



#### REST

텍스트 기반 전송 프로토콜로 구축되며 JSON처럼 사람이 읽을 수 있는 텍스트 포멧을 활용

- 따라서 서비스 간 통신에선 사람이 읽을 필요가 없어서 비효율적



### gRPC 장점

+ 프로세간 통신 효율성
+ 간단 명확한 서비스 인터페이스와 스키마
  + gRPC는 애플리케이션 개발용 contract-first 접근 방식을 권장함
    + 먼저 서비스 인터페이스를 정의한 후 나중에 구현 세부 사항을 작업함
+ 엄격한 타입 점검 형식
  + 정적 타이핑
+ 다양한 프로그래밍 언어지원
+ duplex streaming 
  + 대부분의 스트림은 읽기 또는 쓰기만 가능한데, 둘 다 가능한 스트림을 말함
+ 유용한 내장 기능
  + 인증
  + 암호화
  + 복원력(데드라인/타임아웃)
  + 메타 데이터 교환
  + 압축
  + 로드밸런싱
  + 서비스 검색
+ 최신 프레임워크에서 지원함

### gRPC 단점

+ 외부 서비스 부적함
  + 내부 프로세스 통신용으로는 적합
+ 서비스 정의가 바뀌면 개발 프로세스 복잡성 증가
  + 스키마 수정은 최신 서비스 간 통신 use-case에서 흔함
  + gRPC 서비스 정의가 변경되면 클라이언트-서버 코드 재생성해야함

