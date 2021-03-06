# TIL

## 오늘 공부한 컨셉
+ kubectl 명령어 톱아보기
   - yaml 파일 즉석으로 쓰기, 수정 등
+ 자동완성
+ 리소스의 2가지 분류
  1. 네임스페이스 리소스(e.g Pod)
  2. 클러스터 리소스(e.g Node)
+ 클러스터 상태 확인
  - 3가지 명령어 유용할듯.
+ ConfigMap, Secret
+ 쿠버네티스 네트워크

## 상세내용

### kubectl describe
describe는 get과 비슷하게 pod 상태 정보를 보여주면서 Pod에 대한 EVNET기록까지 확인할 수 있다. 문제가 발생했을 때 get 명령과 디버깅 용도로 자주 사용됨.

> 도커에선 docker inspect 느낌


### 컨테이너 로깅
> kubectl logs <name>


### 컨테이너 명령 전달
> kubectl exec <NAME> -- {CMD}

### 컨테이너 정보 수정
> kubectl edit pod <NAME>


### yaml
yaml파일은 선언형 명령 질의서
yaml 파일에 작성한 내용을 토대로 kubectl apply -f <file.yaml>을 적하면 kubectl run처럼 Pod가 실행됨. 

멱등성(idempotent): 동일한 연산을 여러 번 적용해도 최종 결과가 달라지지 않는 성질. 대표적으로 set. 동일원소 추가해도 최종적으로 1개만 갖고있음.

yaml도 작성된 yaml기준으로 동작함. 이미 작동하고 있는 yaml의 내용을 수정하면 업데이트되어 적용됨

### namespace
쿠버네티스 클러스터를 논리적으로 나누는 역할
+ default
+ kube-system
  - 쿠버네티스 핵심 컴포넌트의 네임스페이스. 네트워크 설정, DNS 설정 등 중요한 역할을 담당하는 컨테이너가 존재
+ kube-public
  - 외부로 공개 가능한 리소스를 담고 있음
+ kube-node-lease
  - 노드가 살아있는지 마스터에 알리는 용도

특정 네임스페이스에 리소스를 생성하려면 --namespace (or -n)으로 설정가능. 아래 명령어는 kube-system 네임스페이스에 nginx 생성
> kubectl run mynginx-ns --image nginx --namespace kube-system

## 자동완성
https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion

bash
``echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc``

### 파이프라인으로 YAML 리소스 생성
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cat-nginx
spec:
  containers:
  - image: nginx
    name: cat-nginx
EOF
```

### -o jsonpath
특정 정보만 뽑아내기
> kubectl get node master -o jsonpath="{.status.addresses[0].address}"

-o jsonpath를 빼고 출력하면 master node에 대한 정보가 json 형태로 쭉 나온다. 그래서 jsonpath로 json 접근하여 특정정보만 뺴오면 된다.

더 많은 사용법
> https://kubernetes.io/docs/reference/kubectl/jsonpath

### 리소스 조회
> kubectl api-resources

리소스는 네임스페이스 리소스와 클러스터 리소스 2가지가 있는걸 어그저께 배웠다.
네임스페이스 리소스만 조회하려면
> kubectl api-resources --namedspaced=true

## 클러스터 상태 확인 중요!
쿠버네티스 클러스터가 정상적으로 동작하고 있는지 확인할 때
```
# 쿠버네티스 API 서버 작동 여부 확인
kubectl cluster-info

# 전체 노드 상태 확인
kubectl get node

# 쿠버네티스 핵심 컴포넌트의 Pod 상태 확인
kubectl get pod -n kube-system
```

많은 경우 위 3개의 명령으로 문제 해결 가능

## KUBECONFIG
kubectl은 내부적으로 KUBECONFIG($HOME/.kube/config) 설정 파일을 참조하여 마스터 주소, 인증 정보관리

> kubectl config view

KUBECONFIG는 크게 3가지 영역
+ clusters: kubectl이 보는 클러스터 정보
+ users: 쿠버네티스 클러스터에 접속하는 사용자
+ contexts: cluster와 user를 연결해주는 것

## 명령어 치트시트
https://kubernetes.io/docs/reference/kubectl/cheatsheet/

## Pod

### 가상환경 플랫폼 실행 단위
+ 가상머신 : instance
+ 도커 : container
+ 쿠버네티스 : Pod

### Pod 특징
+ 1개 이상의 컨테이너 실행. 보통 1Pod 1컨테이너. 상황에 따라 2~3개. 3개 이상 넘어가는 경우 거의 없음.
+ Pod내에 실행되는 컨테이너는 동일 노드에 할당. 동일한 생명주기 가짐
+ 고유의 Pod IP. NAT 통신 없이 Pod 고유의 IP를 이용하여 접근
+ pod끼리 ip 공유. 서로 포트를 이용해서 구분함
+ volume 공유. 파일시스템 기반으로 파일 주고 받을 수 있음.

> NAT(network access translation) : 여러 개의 내부 ip를 1개의 외부 ip로 연결하는 기술. 대표적으로 포트포워딩
> 포트포워딩 : 게이트웨이(외부망)의 반대쪽에 위치한 보호/내부망에 상주하는 호스트에 대한 서비스를 생성하기 위해 흔히 사용되며, 통신하는 목적지 IP 주소와 포트 번호를 내부 호스트에 다시 매핑함으로써 이루어진다


## YAML 파드 템플릿 생성
--dry-run과 -o로 조합해서 파드 생성하지 않고 yaml 생성
> kubectl run mynginx --image nginx --dry-run=client -o yaml > mynginx.yaml


### pod 생성 순서
1. kubectl로 Pod 정의를 쿠버네티스 마스터에 전달
2. 쿠버네티스 마스터는 YAML 유효성체크하고 특정 노드에 컨테이너 실행하도록 명령내림
3. 명령받은 노드(kubelet)는 컨테이너를 노드에 실행

## nodeSelector
라벨링으로 pod를 특정 node에 할당되도록 스케줄 가능
A라는 node는 SSD쓰고 B라는 노드는 HDD쓰면, Pod를 SSD에 할당하려면 A에 할당해야함. 기본적으로 자동으로 할당되는데 nodeSelector로 할당시켜줌

라벨 붙여놓은거 다 확인하기
> kubectl get node --show-lables

마스터 노드에 disktype=ssd 라벨 붙이고 워커노드에 disktype=hdd 붙이기

> kubectl label node master disktype=ssd
> kubectl label node worker disktype=hdd

disktype 라벨확인
> kubectl get node --show-labels | grep disktype

Pod의 YAML에 nodeSelector property 추가
```
apiVersion: v1
kind: Pod
metadata:
name: node-selector
spec:
   containers:
   - name: nginx
     image: nginx
   nodeSelector:
      disktype: ssd
```

> 만약 2개 이상의 노드에 동일한 라벨이 부여되는 경우, 쿠버네티스가 노드의 상태를 확인하여 최적의 노드 하나를 선택함.


### 실행 명령 파라미터 지정
```
apiVersion: v1
kind: Pod
metadata:
    name: cmd
spec:
    restartPolicy: OnFailure
    containers:
    - name: nginx
      image: nginx
      command: ["/bin/echo"]
      args: ["hello"]
```

command: 컨테이너의 시작 실행 명령. 도커의 entrypoint와 비슷
args: 실행 명령에 전달할 파라미터. 도커의 cmd와 비슷
restartPolicy : pod 재시작 정책 설정
  - Always : Pod 종료시 항상 재시작(디폴트)
  - Never: 재시작안함
  - OnFailure: 실패시에만 재시작

## 볼륨 연결
Pod의 데이터는 휘발성임. 지속적으로 저장하려면 host volume에 저장해야함
```
apiVersion: v1
kind: Pod
metadata:
    name: volume
spec:
    containers:
    - name: nginx
      image: nginx

      volumeMounts:  
      - mountPath: /container-volume
        name: my-volume

    volumes:
    - name: my-volume
      hostPath:
          path: /home
```

+ volumeMounts: 컨테이너 내부에 사용될 볼륨을 선언
   - mountPath: 컨테이너 내부에 볼륨이 연결될 위치 지정
   - name: volumeMounts와 volumes를 연결하는 식별자

### Pod 끼리 파일을 주고 받는 emptyDir
임시파일임
```
apiVersion: v1
kind: Pod
metadata:
    name: volume
spec:
    containers:
    - name: nginx
      image: nginx

      volumeMounts:  
      - mountPath: /container-volume
        name: my-volume

    volumes:
    - name: my-volume
      emptyDir: {} # 2개 이상의 컨테이너가 서로 디렉터리 공간을 공유할 수 있음. 파일 접근이므로 레이스컨디션 조심
```


## 리소스 관리 requests랑 limits
### request 컴퓨터 자원 최소 사용 보장
```
# 생략
spec:
    resources:
        requests:
            cpu: "250m"
            memory: "500MI"
```
CPU에서 1000m은 1core.
250m은 0.25core
메모리 Mi는 1Mib(2^20 bytes) 

### limit 컴퓨터 자원 최대 사용 보장
```
# 생략
spec:
    resources:
        limits:
            cpu: "500m"
            memory: "1GI"
```

## livenessProbe
컨테이너가 정상적으로 살아있는지 확인하기 위한 프로퍼티
```
# 생략
spec:
    containers:
        livenessProbe:
             httpGet:
               path: /live
               port: 80
```
+ httpGet: HTTP GET method를 이용하여 상태 확인 수행. HTTP 상태코드가 200번~300번대면 정상, 그 이외는 비정상 -> 컨테이너 종료 및 재시작

## readinessProbe
Pod가 생성직후 트래픽을 받을 준비가 완료되었는지 확인하는 프로퍼티
livenessProbe랑 똑같이 사용

http method 말고 exec의 반환값으로 구분하는 법도 있다.

```
# 생략
spec:
    containers:
        readinessProbe:
             exec:
               command:
                 - cat
                 - /tmp/ready
```
명령의 리턴값이 0이면 정상으로, 0 이외면 비정상으로 인식한다.
위 예제에서는 /tmp/ready가 있으면 0으로 인식. 파일이 있는걸로 간주되면 트래픽을 받을 준비가 된 것

## Pod 내에 2개의 컨테이너 실행
```
apiVersion: v1
kind: Pod
metadata:
    name: second
spec:
    containers:
    - name: nginx
      image: nginx
    - name: curl
      image: curlimages/curl
      command: ["/bin/sh"]
      args: ["-c", "while true; do sleep 5; curl localhost; done"]
```
컨테이너 안에 name으로 추가해주면 된다.

### 2개 이상의 컨테이너 로그확인
Pod내에 2개 이상의 컨테이너가 존재해서 어떤 컨테이너의 로그를 볼지 지정해야한다.
> kubectl logs second -c nginx
or
> kubectl logs second -c curl

두 번쨰 컨테이너에서 curl을 실행하기전에 **5초간 대기하는 이유는 쿠버네티스 Pod 내부 컨테이너끼리의 실행 순서를 보장하지 않는다. 따라서 nginx가 제대로 실행된 후 curl을 실행하기 위해서다.**

sidecar 패턴
Pod안에 2개이상의 컨테이너를 실행하는 이유는 메인 컨테이너가 본연의 임무를 수행하고, 서브 컨테이너가 메인 컨테이너를 보조하기 위함.

e.g
+ 메인컨테이너 : 웹 서빙
+ 서브컨테이너 : 웹 로그 전송

오토바이 사이드카 닮아서 지어진 이름

### initContainers property 
컨테이너 실행 순서는 보장되지 않음. 명시적으로 메인 컨테이너가 실행되기 전에 미리 초기화를 수행하기 위함
```
#생략
spec:
   containsers:
      #생략
   initContainers:
   - name: git
      # 나머지는 containers랑 같음
```
메인 컨테이너 실행 전에 초기화를 위해 먼저 실행되는 컨테이너

## ConfigMap
메타데이터를 저장하는 리소스. 
ConfigMap에 모든 설정값을 저장해놓고  Pod에서는 필요한 정보를 불러올 수 있음

1. 생성하는 법
> kubectl create configmap <key> <data-source>

.properties 파일 작성하고 --from-file 옵션으로 ConfigMap만들기
> kubectl create configmap <config_name> --form-file=<.properties>

--from-literal로 값 바로 설정
> kubectl create configmap special-config --from-literal=sepecial.power=10 --from-literal=special.strength=20

2. YAML에다 직접 작성해도 된다.

3. ConfigMap을 volume으로 마운트하여 파일처럼 사용 가능.
여태까지 배운거에서는 hostPath(호스트에 저장), emptyDir(Pod간 디렉터리공유)이 끝이였는데 하나 추가된다.
```
# 생략
spec:
   # skip
   volumes:
   - name: game-volume
     configMap:
       name: game-config
```
 
 4. 환경변수로 사용
```
 #skip
 spec:
     env:
     - name: special_env
       valueFrom:
         configMapKeyRef:
            name: special-config
            key: special.power
```
 + name: 환경변수의 key 지정
 + valueFrom: 기존의 value property 대신 valueFrom을 사용하여 다른 리소스의 정보를 참조하는 것을 선언
    - configMapKeyRef: ConfigMap의 키를 참조
      * name : ConfigMap의 이름 설정
      * key : ConfigMap내에 포함된 설정값 중 특정 설정값을 명시적으로 선택

5. 4에서는 1개의 설정값만 사용했다.(special.power) envFrom으로 모든 설정값을 사용해보자.
```
 #skip
 spec:
     containers:
       envFrom:
       - configMapRef:
         name: monster-config
```

configMap을 2가지로 작성하고 3가지로 적용하는 방법에 대해 알아봤다.
생성
1. kubectl create configmap
2. YAML에 직접 작성. kind를 ConfigMap으로 설정

적용
1. volume 마운트 -> 생성된 ConfigMap
2. env 환경변수로 사용 - 1개
3. envFrom 환경변수로 사용 - 여러개


## Secret
tmpfs라는 메모리 기반 파일시스템에 사용해서 보안에 강함
base64로 인코딩(암호화는 아님! base64로 디코딩하면 바로 보임ㅋㅋ)

Secret 리소스 만들어보기
```
apiVersion: v1
kind: Secret
metadata:
   name: user-info
type: Opaque
data:
   username: YWRtaW4=            # admin
   password: cGFzc3dvcmQxMjM=    # password123
```
+ type: 기본 Opaque. 다른 타입도 있음
+ data: 저장할 민감데이터

### stringData property. 인코딩을 쿠버네티스가 처리해주길 원할 때
```
apiVersion: v1
kind: Secret
metadata:
   name: user-info
type: Opaque
stringData:
   username: admin
   password: password123
```

> kubectl apply -f user-info-stringdata.yaml

> kubectl get secret user-info-stringdata -o yaml

명령형 커맨드로 생성도 위의 configMap과 같다.
```
cat user-info.properties
# username=admin
# password=password123
```

--from env-file 옵션으로 properties 파일로부터 Secret을 생성
> kubectl create secret generic user-info-from-file --from-env-file=user-info.properties

> kubectl get secret user-info-from-file -o yaml
--from-file, --from-literal 지원함.

### 활용
configMap과 마찬가지
```
# skip
spec:
   # skip
   volumes:
   -name: secret
   secret:
      secretNmae: user-info
```

환경변수는 
```
valueFrom:
   secretKeyRef:
       name: user-info
       key: password
```

envFrom은
```
containers:
    envFrom:
    - secretRef:
       name: user-info 
```

## Downward API
Pod의 메타데이터를 컨테이너에게 전달할 수 있는 메커니즘
실행되는 Pod의 정보를 컨테이너에 노출하고 싶을 때 사용
```
#SKIP
volumes:
  downwardAPI:          # DownwardAPI 볼륨 사용 선언
   item:                # 메타데이터로 사용할 아이템 리스트 지정
   - path: "lables"     # 볼륨과 연결될 컨테이너 내부 path
     fieldRef:          # 참조할 필드
      fieldPath: matadata.labels   #Pod의 메타데이터 필드
 ```
 
 env도 설정가능. 찾아보자
 
 ## Service가 왜 있을까?
 Pod는 불안정한 리소스로 본다. 무슨 이유로든 종료될 수 있어서 사용자가 불편함을 겪을 수 있다.
 그래서 안전한 Service로 end-point를 제공한다.
 
리버스 프록시: 클라이언트 서버 구조에서 서버로 전송되는 요청을 대신 받아 원래의 서버로 전달해주는 대리 서버를 의미. 서버 요청되는 부하를 분산시키고 보안을 높이는 용도

### 라벨 셀렉터로 Pod를 선택하는 이유
쿠버네티스에선 각 리소스의 관계를 느슨한 연결 관계로 표현하고자함. 
느슨한 관계 : 특정 리소스를 간접 참조
Service에서 Pod의 이름이나 IP로 직접 참조하면 Pod의 생명주기에 따라 사용자가 매번 새로운 Pod 정보를 Service에 등록 및 삭제해야함.

라벨링 시스템을 통해 느슨한 관계를 유지할 경우  Service에서 바라보는 특정 라벨을 가지고 있는 어떠한 Pod에도 트래픽을 전달할 수 있다.
Pod입장에서도 Service를 직접 참조할 필요 없이 라벨로 간접참조하면 된다.

```
# myservice.yaml
apiVersion: v1
kind: Service
metadata:
    labels:
       hello: world
    name: myservice
spec:
    ports:
    - port: 8080
    protocol: TCP
    targetPort: 80
    selector:
        run: mynginx       # run=mynginx 라벨을 가진 Pod에 Service 트래픽 전달
```

Pod 하나를 생성하고 ip 주소를 확인
> kubectl run mynginx --image nginx
> kubectl get pod -o wide

서비스를 생성하고 조회
> kubectl apply -f myservice.yaml
> kubectl get service #축약시 svc
서비스IP주소확인

curl 요청할 client pod 생성
> kubectl run client --image nginx

Pod IP로 접근
> kubectl exec client -- curl 10.42.0.226

서비스 IP로 접근
> kubectl exec client -- curl 10.43.152.73:8080

서비스 DNS로 접근
> kubectl exec client -- curl myservice:8080

이것으로 서비스를 end-point로 하여 pod에 접근하는 방법을 알아보았다.

### Service 도메인 주소 법칙
``<서비스 이름>.<네임스페이스>.svc.cluster.local``

도메인 주소 생략
+ 서비스이름.네임스페이스 == 서비스이름.네임스페이스.svc.cluster.local
+ 서비스이름 == 서비스이름.네임스페이스.svc.cluster.local

### 클러스터 DNS 서버
Pod의 DNS 설정 확인
> kubectl exec client -- cat /etc/resolv.conf

네임서버 IP가 나오는데 쿠버네티스의 모든 Pod가 이 IP로 DNS 조회함.
kube-system 네임스페이스의 Service 리소스를 조회하면 kube-dns라는 Service가 주인임
> kubectl get svc -n kube-system
> kubectl get svc kube-dns -n kube-system --show-labels

kube-dns의 Label을 보면 k8s-app=kube-dns임. 해당 라벨로 Pod를 확인해보면
> kubectl get pod -n kube-system -l k8s-app=kube-dns

coredns-xxx 라는 Pod가 조회됨. 
coredns는 쿠버네티스에서 제공하는 클러스터 DNS 서버. 이 친구 덕분에 쿠버네티스 클러스터 안에서 자체적인 도메인 네임 시스템을 갖게 됨.

### Service 종류
4가지
+ ClusterIP
  - 클러스터 내부에서만 접근가능
  - 외부에서 접근하지 못하는데 존재하는이유
    * 네트워크 보안/관리를 위해. 1~2개의 외부 서비스 end-point 외에는 직접 트래픽을 받는 경우가 드뭄
    * 더 확장된 쿠버네티스 네트워킹을 위한 기본 빌딩블럭으로 사용
+ NodePort
  - 외부트래픽을 Serivce까지 전달
  - 쿠버네티스에서 제공하는 NodePort range는 30000~32767까지
  - NodePort는 Pod가 위치한 노드 뿐만 아니라 모든 노드에서 동일하게 서비스 end-point를 제공한다.
    * 예를 들어, 생성한 node-port Pod가 마스터 노드에 위치한다고 가정, 마스터 노드, 워커 노드 모두 동일한 NodePort로 서비스 접근 가능
  - 쿠버네티스는 클러스터 시스템이라 특정 노드에서만 서비스가 동작하지 않고 모든 노드에서 동일하게 적용. kube-proxy가 해줌
    * kube-proxy는 리눅스 커널의 netfilter를 이용하여 커널 레벨에서 특정 트래픽을 중간에서 가로채 다른곳으로 라우팅 해주는 역할 수행
+ LoadBalancer
  - NodePort만으로 외부 트래픽을 받을 수 있는데 LoadBalancer가 필요한 이유
    1. 보안적 측면 : 호스트 서버의 노드포트 대역을 외부에 공개할 필요가 없음. 로드밸랜서만 외부 네트워크에 위치시키켜 호스트 정보를 숨김
    2. 클러스터 앞단에 로드밸런서를 배치하면 각각의 서버 IP(master, worker)를 직접 알 필요가 없음. 
+ ExternalName
  - 외부 DNS주소에 클러스터 내부에서 사용할 새로운 별칭을 만듬(내부에서 외부로 갈 떄)
  - 쿠버네티스 클러스터에 편입되지 않는 외부 서비스에 쿠버네티스 네트워킹 기능을 연결하고 싶은 경우 사용

### 네트워크 모델 특징
+ 각 Node간 NAT(network address translation)없이 통신 가능해야함
+ 각 Pod간 NAT 없이 통신 가능해야함
+ Node와 Pod간 NAT 없이 통신이 가능해야 함
+ 각 Pod는 고유의 IP 부여받음
+ 각 Pod IP는 네트워크 provider를 통해 할당받음
+ Pod IP는 클러스터 내부에서 어디서든 접근 가능해야함.


장점
+ 모든 리소스(Node, Pod)가 각각 모든 리소스(Node, Pod, Service)를 고유의 IP로 접근가능
+ NAT 통신으로 인한 부작용에 대해 신경 쓸 필요 없다.
+ 기존 TCP TDP IP 프로토콜 그대로 이용 가능
+ Pod끼리 네트워킹이 어느 노드에서든 동일하게 동작함. (호스트 서버와의 종속성이 없어져서 이식성이 높아짐)

Clean up
> kubectl delete pod --all
> kubectl delete svc --all
