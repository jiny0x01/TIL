# TIL

## 오늘 공부한 컨셉

- 도커랑 쿠버네티스 차이
- 도커 명령어 ENTRYPOINT, VOLUME(바운드마운트), USER 새로 앎
- desired state
- 마스터와 워커
- 컨트롤러 라벨 셀렉트 네임스페이스

## 상세내용



## 도커 기초

### 장점

+ 표준화 -> 프로세스의 실행을 표준화

+ 이식성 -> 도커 플랫폼 위에서 동일한 실행 환경으로 프로세스 작동

+ 가벼움 -> 실행되는 애플리케이션별로 커널 공유. 다른 vm보다 가벼움

+ 보안 -> 컨테이너는 고립된 환경. 보안 유리함

  

### 컨테이너 가상머신 차이 

공통

+ 리소스 가상화 및 고립시키는 점

동작방식

+ 가상머신

  - 서버에 하이퍼바이저 설치하고 그 위에 OS를 올리고 애플리케이션을 올림

    > 하이퍼바이저 : 하드웨어를 가상화하는 것을 제어하고 각각의 가상머신을 관리하는 역할. 하드웨어의 물리적 리소스를 VM에게 제공하고 I/O 명령처리 해줌

+ 컨테이너

  + 도커 데몬 위에서 컨테이너를 올려서 가상화 지원
  + 소스코드랑 라이브러리를 패키징해서 별도의 실행환경 만드는 것 
  + 동일한 호스트에서 커널을 공유하며 실행함. 하지만 개별적인 User space를 가짐

운영체제 위에 운영체제를 올리는 것보다 가벼운 것이 특징



### 컨테이너 실행

프로세스 실행하듯이 컨테이너를 실행하면 됨

``docker run <IMAGE>:<TAG> [<argv>]``

### 도커 이미지 주소

``<레지스트리 이름>/<이미지 이름>:<TAG>``

> e.g) dockerio/docker/whalesay:lastest

위 예시는 레지스트리 이름이 dockerio/docker/wahilesay, TAG는 lastest

### 실행중이 컨테이너에 명령 전달

> docker exec <CONTAINER ID> <CMD>

### 컨테이너 호스트 파일 복사

> docker cp <HOST_PATH> <CONTAINER_ID>:<CONTAINER_PATH>

 ### Dockerfile 매개변수 전달

ARG로 Dockerfile 안에서 매개변수를 설정

```dockerfile
FROM ubuntu:18.04

...

ARG my_ver=1.0

...
```

ARG 지시자에 my_ver=1.0이 있다. 도커 빌드할 때 해당 변수의 값을 조작할 수 있다.

> docker build . -t hello:2 --build-arg my_ver=2.0

컨테이너가 돌아가는 상태에서도 환경변수를 변경할 수 있다.

> docker run -e my_ver=2.5 hello:2



### ENTRYPOINT와 CMD 차이

CMD는 override할 수 있음. ENTRYPOINT는 이미지를 실행 가능한 **바이너리**로 만듦. 이미지 실행 시 무조건 호출되고 파라미터 전달시 파라미터가 ENTRYPOINT의 파라미터로 전달됨.



### 네트워크

외부 트래픽을 컨테이너 내부로 전달하기 위해서 로컬 호스트 서버와 컨테이너의 포트를 매핑시켜 트래픽을 포워딩한다.

아래 예제는 호스트 5000번 포트를 컨테이너 80번 포트로 매핑

> docker run -p 5000:80 -d nginx



### volume 컨테이너 데이터 지속보관을 위함

컨테이너는 휘발성 프로세스라서 프로세스 종료시 데이터가 유실됨.

볼륨 마운트로 호스트의 파일시스템을 컨테이너와 연결하여 필요한 데이터를 로컬 호스트에 저장가능

> docker run -p 6000:80 -v $(pwd):/usr/share/nginx/html/ -d nginx

로컬호스트의 6000포트와 컨테이너 80포트 연결하고 $(pwd)와 /usr/share/nginx/html을 볼륨마운트했다.



### User

컨테이너는 디폴트로 root다.  root가 아닌 일반 유저를 사용하게 할 수 있다.

```docker
FROM ubuntu:18.04

# ubuntu 유저 생성
RUN adduser --disabled-password --gecos "" ubuntu

#컨테이너 실행시 ubuntu로 실행
USER ubuntu
```

컨테이너 실행 시 --user 옵션으로 명시적으로 유저를 입력할 수 잇다.

> docker run --user root -it my-user bash



## 쿠버네티스

### desired state

에어컨 - 특정 온도가 될때까지 시스템이 알아서 온도를 조절함

쿠버네티스 - 사용자의 요청에 의해 현재 상태가 desired state가 될 때 까지 사전에 미리 정의된 특정 작업을 수행

### controller

control-loop를 돌면서 리소스를 지속적으로 모니터링 해서 사용자가 생성한 리소스의 이벤트에 따라 정의된 작업을 실행

### 네임스페이스

클러스터 위에서 논리적으로 서로 다른 네임스페이스로 클러스터 환경을 나눌 수 있음

+ 네임스페이스마다 서로 다른 권한 설정 가능

+ 네트워크 정책 설정 가능

쿠버네티스의 모든 리소스 2가지 구분

+ 네임스페이스 레벨 리소스
  + Pod
  + Deployment
  + Service
+ 클러스터 레벨 리소스
  + Node
  + PersistentVolume
  + StorageClass

### Label Selector

특정 리소스에 명령을 전달하거나 정보를 확인하고 싶을 때 라벨 사용

리소스에 key-value형태의 라벨을 붙이고 셀렉터로 특정 key-value를 가진 리소스 추출



### 마스터

+ kube-apiserver : 마스터로 전달되는 모든 요청을 받는 REST API 서버
  + kubectl을 이용하여 api 서버에 명령을 보내고 응답을 받음. 
  + 마스터와 통신한다는 뜻이 이 api 서버와 통신한다는 의미
+ etcd : 클러스터내 모든 메타 정보를 저장하는 저장소
  + 쿠버네티스 클러스터에 필요한 데이터 저장하는 DB역할
  + 분산형 key-value 저장소
+ kube-scheduler : 사용자의 요청에 따라 적절하게 컨테이너를 워커 노드에 배치하는 스케줄러
+ kube-controller-manger : 현재 상태와 desired state를 지속적으로 확인하고 특정 이벤트에 따라 특정 동작을 수행하는 컨트롤러
+ colud-controller-manager : 클라우드 플랫폼에 특화된 리소스를 제어하는 클라우드 컨트롤러
+ 마스터는 단일 서버로 구성할 수 있으며 고가용성을 위해 여러 서버를 묶어 클러스터 마스터로 구축가능

### 노드

+ kubelet : 마스터의 명령에 따라 컨테이너의 라이프 사이클을 관리하는 노드 관리자
+ kube-proxy : 컨테이너의 네트워킹을 책임지는 프록시
+ container runtime : 실제 컨테이너를 실행하는 컨테이너 실행환경







