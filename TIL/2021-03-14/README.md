# TIL

## 오늘 공부한 컨셉

## 상세내용

### 컨트롤러
리소스를 지속적으로 관찰하며 리소스 생명주기에 따라 미리 정해진 작업을 수행함
current state와 desired state!

쿠버네티스에는 Job 리소스가 있음
+ 한 번 실행하고 완료되는 배치 작업을 수행
+ Job 컨트롤러는 새로운 Job 리소스가 생성되는지 control-loop를 돌면서 지속적으로 관찰함

### ReplicaSet
Pod 복제 담당

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: myreplicaset
spec:
    replicas: 2 
    selector:                  # 라벨링 시스템으로 복제할 Pod 선택
      matchLabels:
          run: nginx-rs
    template:                # 복제할 Pod 정의
        metadata:
            labels:
                run: nginx-rs
        spec:
            containers:
            - name: nginx
              image: nginx
```

#### 복제 개수 증가
> kubectl scale rs --replicas <number> <name> 

#### ReplicaSet 리소스 정리
> kubectl delete rs --all

### Deployment
+ 롤링 업데이트를 지원하고 롤링되는 Pod 비율 조절
+ 업데이트 히스토리 저장 및 롤백
+ Pod 개수 늘릴 수 있음(scale out)
+ 배포 상태 확인

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeploy
spec:
  replicas: 10
  selector:
    matchLabels:
      run: nginx
  strategy:
    type: RollingUpdate         # Recreate도 있음
    rollingUpdate:
      maxUnavailable: 25%       # 최대 중단 Pod 허용 개수. 업데이트 전 25% Pod가 내려갈 수 있음
      maxSurge: 25%             # 최대 초과 Pod 허용 개수. 10의 25%면 pod가 약 13개까지 생성될 수 있음
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
```

배포 상태 확인
> kubectl rollout status deployment mydeploy 

deployment는 deploy로 축약 가능

배포 히스토리 확인
> kubectl rollout history deployment mydeploy


롤백
> kubectl rollout undo deployment mydeploy

--to-reversion으로 특정 버전으로 롤백
> kubectl rollout undo deployment mydeploy --to-revision=1

revison은 배포 history 번호로 하면 된다.

Pod 개수 줄이기
> kubectl scale deployment mydeploy --replicas=5

올리는 건 replicas=10으로 맞춰주면 된다. 

edit으로 yaml 수정
> kubectl edit deploy mydeploy

YAML에 있는 ``replicas:`` 부분을 수정하면된다.

### StatefulSet
Deployment나 ReplicaSet과 다르게 복제된 Pod가 완벽히 동일하지않고 순서에 따라 고유의 역할을 가짐
동일한 이미지를 이용하여 Pod를 생성하지만 실행 시, 각기다른 역할을 가지며 서로의 역할을 교체하지 못할 때 사용
**프로세스간 서로 치환될 수 없는 클러스터를 구축할 떄 많이 사용**
+ 고유의 Pod 식별자가 필요한 경우
+  명시적으로 Pod마다 저장소가 지정되어야 하는 경우
  - 1번디스크는 pod1, 2번디스크는 pod2
+ Pod끼리 순서에 민감한 애플리케이션 
+ 애플리케이션이 순서대로 업데이트되야 할 떄
+ Pod의 내부적 상태를 저장하는 경우 StatefulSet사용 (Pod로 DB 구축하는 경우)

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysts
spec:
  serviceName: mysts
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  teplate:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: nginx-vol
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: nginx-vol
    spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysts
spec:
  clusterIP: None
  ports:
  - port: 8080
    protocol: TCP
    tartgetPort: 80
  selector:
    run: nginx
```

### DaemonSet
모든 노드에 동일한 Pod를 실행시키고자 할 때 사용함
리소스 모니터링, 로그 수집기 같은 것들이 유용

### Job
한번 실행하고 완료되는 일괄처리 프로세스용

### CronJob
Job을 주기적으로 실행할 수 있는 리소스

### helm 쿠버네티스 패키지 매니저
helm은 YAML 형식으로 구성되어 있으며 chart라고 한다.

helm chart 구조
+ values.yaml
  - 사용자가 원하는 값들을 설정하는 파일
+ templates/
  - 설치할 리소스 파일이 존재하는 디렉터리. 쿠버네티스 리소스가 YAML 파일 형태로 있음.
  - 각 설정값은 비워져 있고(placehold) values.yaml 설정값으로 채워짐

chart 생성
> helm create mychart

### Ingress 리소스
애플리케이션 계층에서 외부의 트래픽을 처리함
부하 분산, TLS 종료, 도메인 기반 라우팅 기능 제공

#### ingress controller
ingress 리소스 자체는 트래픽 처리에 대한 정보를 담고 있는 규칙에 가까움
ingress 규칙을 읽고 외부의 트래픽을 Service로 전달해주는게 ingress controller

#### 도메인 주소 테스트
Layer 7계층 통신이라 도메인 주소 가지고 있어야 테스트할 수 있음.
https://sslip.io를 사용하면 간단하게 도메인을 얻을 수 있음.

#### 라우팅
+ 도메인 기반 라우팅
  - IP 주소는 동일해도 도메인 주소를 기준으로 서로 다른 서비스로 HTTP 트래픽을 라우팅 할 수 있음
+ Path 기반 라우팅
 
#### Basic Auth
Ingress 리소스에 간단한 HTTP Authentication 기능을 추가할 수 있음
Basic Auth는 유저ID, 비밀번호를 HTTP 헤더로 전달하여 인증함.
헤더에 아래와 같이 user와 password를 콜론으로 묶어 base64로 인코딩하여 전달
> Authorization: Basic $base64(user:password)

접속
> curl -v -H "Authorization: Basic $(echo -n foo:bar | base64)" https://httpbin.org/basic-auth/foo/bar

ingress에 Basic Auth설정하기 위해 basic authenticatoin 파일이 필요함
htpasswd라는 툴로 생성가능
> sudo apt install -y apache2-utils
> htpasswd -cb auth foo bar # id: foo, pw: bar
> kubectl create secret generic basic-auth --from-file=auth  # 생성한  auth를 secret에 넣음
> kubectl get secret basic-auth -oyaml # Secret 생성된 것 확인

생성한 리소스를 Ingress에 설정해야함

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth  # secret 이름 지정
    nginx.ingress.kubernetes.io/auth-auth-realm: 'Authentication Required - foo' # 보안 메시지 및 인증 영역 설정
  name: apache-auth
spec:
  rules:
  - host: apache-auth.10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: apache
          serviccePort: 80
        path: /
```

### TLS 설정

cert-manager로 인증서 발급 자동화
https://cert-manager.io

인증서 발급 도와줄 Issuer 리소스 생성
https://letsencrypt.org 는 무료로 사용자들에게 정ㅇ식 인증서를 발급해주는 인증서 발급 기관

http01 solver를 이용한 도메인 인증을 성공하기 위해서는 반드시 공인 IP를 사용해야함.
Let's encrypt 서버에서 도메인 주소에 대한 소유권을 확인함
내부망에서 인증서를 발급하는 경우는 Let's encrypt에서 제공하는 DNS TXT record를 DNS 서버에 설정하며 됨


