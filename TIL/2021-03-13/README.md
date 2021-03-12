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

## 상세내용

### kubectl describe
describe는 get과 비슷하게 pod 상태 정보를 보여주면서 Pod에 대한 EVNET기록까지 확인할 수 있다. 문제가 발생했을 때 get 명령과 디버깅 용도로 자주 사용됨.

> 도커에선 docker inspect 느낌


### 컨테이너 로깅
> kubectl logs <name>


### 컨테이너 명령 전달
> kubectl exec <NAME> -- <CMD>


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


