# TIL

## 오늘 공부한 컨셉
+ 고급스케줄링
  + horizontalPodAutoScaler
  + 수평적확장(scale out), 수직적확장(scale up)
  + taint, toleration
  + affinity, antiaffinity
+ 접근제어
+ 인증서 basic auth, X.509
+ RBAC(roll based access control
## 상세내용

### 고급 스케줄링
ReplicaSet, Deployment의 replica property로 서비스 가용성을 높일 수 있다.
일정 범위 안의 트래픽 안에서는 서비스 가용성이 유지되지만 순간적으로 트래픽 처리량이 늘어나면 replica의 최대 개수가 정해져 있어 트래픽에 대응하지 못한다.

#### Pod레벨에서 고가용성 확보 hpa
쿠버네티스에서는 Pod의 리소스 사용량에 따라 자동으로 확장하는 HorizontalPodAutoScaler(hpa) 리소스를 지원한다.

+ 수평적 확장 : 동일한 프로세스(또는 노드)의 개수를 늘리는 것 == scale out
+ 수직적 확장 : 프로세스(또는 노드)의 성능을 높이는 것 == scale up

hpa가 정상적으로 동작하려면 requests property가 정의되어야 한다.

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: heavy-cal
spec:
  maxReplicas: 50
  minReplicas: 1
  scaleTargetRef:                       # 모니터링할 타겟
    apiVersion: apps/v1
    kind: Deployment
    name: heavy-cal
  targetCPUUtilizationPercentage: 50    # 리소스 임계치
```
모니터링 대상의 리소스에 꼭 request property가 정의되어야 한다.
hpa가 해당 값을 기준으로 CPU 임계치를 계산한다.
heavy-cal hpa는 heavy-cal Deploymnet를 모니터링하다가 CPU 리소스가 50%를 넘겼을 떄 최대 50개까지 replica 개수를 증가시키는 역할을 수행한다.

#### Taint
Taint : 오염시키다, 오점을 남기다
Taint는 Node 리소스에 적용하는 설정값. 노드의 Taint(오점)을 남기면 Pod가(오염되어) 쉽게 스케줄링 못한다.
> kubectl taint nodes $NODE_NAME <KEY>=<VALUE>:<EFFECT>
+ key: 임의의 문자열
+ value: 임의의 문자열
+ effect: NoSchedule, PreferNoSchedule, NoExecute
   - PreferNoSchedule : taint된 노드에 새로운 Pod를 스케줄링하는 것을 지양함. 다른 노드에 먼저 스케줄링하고 마지막에 PreferNoSchedule에 스케줄링함(가장 약한 정책)
   - NoSchedule: taint된 노드에 새로운 Pod를 스케줄링하지 못하게 막음. 기존에 있던 Pod에는 영향 없음
   - NoExecute: taint된 노드에 새로운 Pod 스케줄링 못하게하고 기존 노드에 있던 Pod도 삭제
   
#### Toleration
Toleration: 견디다, 용인하다
Taint된 노드라도 Pod가 참을 수 있으면(tolerate) Taint된 노드에 스케줄링 할 수 있음

```
kind: Pod
metadata:
  name: tolerate
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "project"           #  taint된 key
    value: "A"
    operator: "Equal"        # Equal, Exists중 하나 선택. Equal == key,value 항상 같아야함, Exists : key만 동일하면 tolerate
    effect: "NoSchedule"     # 해당 effect에 대해 tolerate
```

#### Affinity와 AntiAffinity
Taint와 Tolerate가 특정 Pod 스케줄링을 피하거나 감수하고 실행하는 정책이라면
Affinity와 AntiAffinity는 특정 Node나 Pod와의 거리를 조절하는데 사용 (특정 Pod끼리 서로 가까이 스케줄링할 때)
Affinity : 친밀함
AnitiAffinity : 반-친밀함

#### NodeAffinity
Pod가 특정 Node에 할당되길 원할 때 사용. nodeSelector와 유사하지만 더 상세히 설정가능
```
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoreDuringExecution:        # 특정 노드에 스케줄링 되게 강제한다. 
        nodeSelectoTerms:                                   # 선택할 노드의 라벨정보
        - metchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

requiredDuringSchedulingIgnoreDuringExecution는 특정 노드에 스케줄링을 희망하는 것

#### PodAffinity
matchExpressions이 매칭되는 Pod가 동일한 노드에서 실행될 수 있도록 요청
Deployment 리소스를 이용하여 여러 Pod에 동일한 PodAffinity를 가지도록 생성해야함.

### PodAffinity와 PodAntiAffinity 활용법
높은 안정성을 갖는 서비스를 만들 수 있다.
어떤 웹서비스를 운영한다고 가정
+ 웹 서비스 뒤에는 cache 서버를 두고 필요한 데이터를 캐시 서버에서 꺼내서 사용
+ 서비스의 가용성을 위해 1개의 노드가 죽더라도 다른 노드에서 서비스를 지속할 수 있도록 web서버끼리, cache 서버끼리 최대한 서로 다른 노드에 스케줄링 되도록 설정
  + 각각의 web서버와 캐시서버는 네트워크 레이턴시를 최소화하기 위해 최대한 같은 노드에서 실행되도록 설정
  
캐시서버
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 2
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringScheduleingIgnoredDuringExecution:
          - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis
```


웹서버
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store 
  replicas: 2
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:                # web 서버끼리 멀리 스케줄링. app=web-store 라벨을 가진 Pod끼리 멀리 
          requiredDuringScheduleingIgnoredDuringExecution:
          - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:                      # web-cache 서버끼리 가까이 스케줄링. app=stroe라벨을 가진 Pod끼리 가까이 스케줄링
          requiredDuringScheduleingIgnoredDuringExecution:
          - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx 
```

### 클러스터 관리
클러스터 관리자가 일반 사용자에게 컴퓨터 자원(리소스) 사용량을 제한하기 위해 사용하는 것이 LimitRange와 ResourceQuota다.

#### LimitRange
기능은 2가지가 있다.
1. 일반 사용자가 리소스 사용량 정의를 생략해도 자동으로 Pod 리소스 사용량을 설정
2. 관리자가 설정한 최대 요청량을 일반 사용자가 넘지 못하게 제한

일반적으로 리소스 설정 없이 Pod를 생성하면 리소스 제약 없이 무제한으로 녿의 전체 리소스를 사용할 수 있다.
> kubectl run mynginx --image nginx
> kubectl get pod mynginx -oyaml | grep resources

이 경우, 일반 사용자가 생성한 Pod가 노드 전체의 리소스를 고갈시킬 위험이 있다.
이를 방지하기 위해 클러스터 관리자가 네임스페이스에 LimitRange를 설정한다.

```
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
spec:
  limits:
  - default:
      cpu: 400m
      memory: 512Mi
    defaultRequest:     # 생략시 사용되는 기본 request 설정값
      cpu: 300m
      memory: 256Mi
    max:
      cpu: 600m
      memory: 600Mi
    min:
      memory: 200Mi
    type: Container
```

위 yaml을 적용하고 리소스 설정없이 Pod를 생성하고 확인해보자
> kubectl run nginx-lr --image nginx
> kubectl get pod nginx-lr -oyaml | grep -A 6 resources

clean up
> kubectl delete limitrange <limit-range>

> 참고로! 리퀘스트는 컨테이너가 생성될 때 필요한 자원량이고 리미트는 생성된 컨테이너가 가질 수 있는 최대 자원량이다.
#### ResourceQuota
limitrange가 개별 Pod 생성에 관여했다면 ResourceQuota는 전체 네임스페이스에 제약 설정한다.
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: res-quota
spec:
  hard:
    limits.cpu: 700m
    limits.memory: 800Mi
    requests.cpu: 500m
    requests.memory: 700Mi
```
clean up
> kubectl delete resourcequota <resource-quota>

### 노드관리
온프레미스 환경에서 물리적인 디스크 손상, 내부 네트워크 장애가 있을 수 있다
클라우드 서비스에선 서버 타입 변경, 디스크 교체등으로 노드를 일시적으로 중단하고 관리해야 하는 경우가 있다.
쿠버네티스에서는 특정 노드를 유지보수 상태로 전환하여 새로운 Pod가 스케줄링 되지 않게 설정할 수 있다.
+ Cordon: 노드를 유지보수 모드로 전환
  - cordon : 저지선을 치다, 사람들의 출입을 통제하다
  - worker를 cordon
    * > kubectl cordon worker
+ Uncordon: 유지보수가 완료된 노드를 정상화
   - > kubectl uncordon worker
+ Drain: 노드를 유지보수 모드로 전환하며 기존의 Pod를 Evict한다.
  - Evict: Pod를 내쫓는다는 의미
  - > kubectl drain worker --ignore-daemonset
  - deamonSet은 무시한다. 
  - drain된 노드도 uncordon으로 되돌릴 수 있다.
+ PodDisruptionBudget(pdb)는 Drain시 모든 Pod가 한번에 죽기 때문에 서비스 부하가 한쪽에 몰려 응답 지연이 발생할 수 있는 것을 방지하기 위함
  - pdb는 운영중인 Pod의 개수를 항상 일정 수준으로 유지할 수 있게해줌

pdb yaml
```
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 9             # 최소 유지해야하는 Pod수
  selector:                   # 유지할 Pod
    matchLabels:
      app: nginx
```

  

### 접근제어
+ Authentication : 접속한 사람의 신분을 인증하는 단계(누가 접근하고 있는가?)
+ Authorization : 어떤 권한을 가지고 어떤 행동을 할 수 있는지 확인
+ Admission Control : 요청한 내용이 적절한지 확인함. LimitRange, ResourceQuota 기능이 Admission Control을 통해 Pod 요청이 적절한지 화깅ㄴ한 것

#### 사용자 인증
쿠버네티스에는 크게 5가지 사용자 인증 방식이 존재
1. HTTP Authentication : HTTP 프로토콜에서 제공하는 인증체계를 이용한 인증
2. X.509 Certificate: X.509 인증서를 이용한 상호 TLS(transport layer security) 인증
3. OpenID Connection
4. Webhook 인증 : Webhook 인증 서버를 통한 사용자 인증
5. Proxy 인증 : Proxy 서버를 통한 대리 인증

여기서는 많이 사용하는 2가지 HTTP Auth, X.509 cert에 대해서 다룸

#### HTTP Basic Authentication
KUBECONFIG파일은 쿠버네티스 마스터 API 서버와 통신하기 위해 필요한 정보를 담고 있는 파일임 $HOME/.kube/config에 위치

```
apiVersion: v1
clusters:
- cluster:
    # 쿠버네티스 서버 인증서가 base64로 인코딩되어 있음
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJkakNDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdGMyVnkKZG1WeUxXTmhRREUyTVRVMU56WTJORE13SGhjTk1qRXdNekV5TVRreE56SXpXaGNOTXpFd016RXdNVGt4TnpJegpXakFqTVNFd0h3WURWUVFEREJock0zTXRjMlZ5ZG1WeUxXTmhRREUyTVRVMU56WTJORE13V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFRL0pXQk0rTW4yTzk0bmRValJlZVY2M1VCSFRsRDhxYTRPZDN6clpCR1kKbERJY0Fzd2oyeDhvRUk0cjd6RFlGbnFCRVdNd1Z3bVRkVlRVbWZnb3pRS05vMEl3UURBT0JnTlZIUThCQWY4RQpCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBZEJnTlZIUTRFRmdRVXNaT3dVdWNFWklHS09Xa0NyZzJuCjFHVHFvcG93Q2dZSUtvWkl6ajBFQXdJRFJ3QXdSQUlnRzdURlRWZ1poT3pDWDRZQS9MNlZma1FpK1RERlo2QmYKbStFZFZheEdjK1FDSUVuZU5oa1B5blBCMlJETlNwN0xKTTIzR3dOVml6azdaWUJLMGsrQ001NGMKLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    password: 2ac6513b67e7a32e44975e5fa76bec7f
    username: admin
```

/var/lib/rancher/k3s/server/cred/passwd
위 파일에 myuser 사용자를 추가한다.
> mypassword,user,uid,group1[,group2, group3]
형식은 다음과 같다
> password,user,uid,group1[,group2,group3]

> system:masters group은 쿠 버네티스에서 따로 역할을 부여하지 않아도 모든 권한을 수행할 수 있는 마스터 그룹이다.

passwd 파일을 저장하고 KUBECONFIG 파일의 user 정보를 password: mypassword, username: myuser로 수정한다.
> sudo systemctl restart k3s

이런식으로 API 서버도 basic auth로 사용자를 인증하며 passwd파일을 통해 계정을 관리한다.

#### X.509 인증서

CSR(Certificate Signing Request) : 인증서 서명 요청. 쿠버네티스가 보유한 CA로부터 인증서 서명을 받기 위해 요청하는 문서

cloudflare에서 만든 cfssl 툴로 사용자 인증서 만들기
> wget -q --show-progress --https-only --timestamping https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssl https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssljson
> chmod +x cfssl cfssljson
> sudo mv cfssl cfssljson /usr/local/bin

인증서를 생성하는 방법
1. CSR 파일 생성
2. CA로부터 인증서 서명
3. 발급된 인증서를 KUBECONFIG 파일에 설정

CSR 파일을 생성하고 쿠버네티스 CA에 서명을 요청한다. k3s 쿠버네티스 CA는 다음 위치에 존재한다.
+ 인증서: /var/lib/rancher/k3s/server/tls/client-ca.crt
+ 개인키: /var/lib/rancher/k3s/server/tls/client-ca.key

사용자 인증서를 발급하기 위한 CA config 파일을 생성하고 인증서를 발급하다.
주의 깊게 봐야하는 문서는 다음과 같다.
+ 사용자 인증서: client-cert.pem
+ 사용자 개인키: client-cert-key.pem

위 파일은 CSR을 통해 쿠버네티스 CA로부터 사용자 인증서와 사용자 개인키를 발급받은 것이다.
해당 파일을 KKUBECONFIG 파일에 설정하면 X.509 인증 방식으로 변경된다.

#### 역할 기반 접근제어(RBAC: Role Based Access Control)
RBAC에는 크게 3가지 리소스가 있다.
+ Role(ClusterRole) : 어떤 권한을 소유하고 있는지 정의
+ Subjects: Role을 부여할 대상을 나타냄
+ RoleBinding(ClusterRoleBinding): Role과 Subject의 연결을 정의


##### Role
Role에는 2가지 리소스가 존재한다.
1. 네임스페이스 안에서 역할을 정의하는 Role 리소스
2. 네임스페이스 밖에서 클러스터 레벨로 역할을 정의하는 ClusterRole 리소스

```
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-viewer
rules:
- apiGroups: [""]
  resources:
  - pods
  verbs:      # 선택한 리소스에 대한 허용 동작 
  - get
  - watch
  - list 
```
ClusterRole도 위와 유사. metadata에 따로 namespace를 입력하지 않으며 클러스터 레벨에서 역할을 정의할 수 있음

##### subjects
Subject는 Role을 부여받을 객체로 3가지가 있다. 쿠버네티스에는 User와 Group은 리소스가 명시적으로 구현되어 있지않고 개념만 있다.
X.509 인증 방식에서는 인증서의 Common Name(CN) 부분이 쿠버네티스에서 User로 인식되고 Organization(O) 부분이 Group으로 인식된다.
1. User
2. Group
3. ServiceAccount
  + > kubectl get serviceaccount # or sa
  + 네임스페이스 레벨에서 동작함. 모든 namespace를 생성할 떄는 default ServiceAccount가 자동으로 생성됨
  + ServiceAccount는 Pod가 쿠버네티스와 통신할 때 사용하는 Identity다.
  + 생성은 > kubectl create sa mysa
 
#### RoleBinding, ClusterRoleBinding
Role과 Subject를 묶는 역할 -> 특정 사용자가 부여받은 특정 권한을 사용할 수 있게 함.
  
```
apiVersion: rbac.authorizaation.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default 
subjects:
- kind: ServiceAccount
  name: mysa
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

지금까지 한 작업은 다음과 같다.
1. Pod의 get,watch,list 권한을 가진 pod-viewer Role 생성
2. mysa라는 ServiceAccount 생성
3. Role과 ServiceAccount를 연결하는 read-pods RoleBinding 생성

#### 네트워크 접근 제어
모든 네트워크 제공자가 네트워크 접근 제어 기능을 제원하지 않고 Weave, Calico 등 일부 제품만 지원함
k3s는 flannel을 네트워크 제공자로 사용하고 있음. flannel에 Network Policy를 설정하기 위해 Canal 설치해야함
> kubectl apply -f https://docs.projectcalico.org/manifests/canal.yaml

쿠버네티스 네트워크 기본 정책
1. 기본적으로 클러스터에 네트워크 정책이 하나도 설정되어 있지 않다.
2. 하나도 설정된게 없으면 네임스페이스의 모든 트래픽이 열려 있다.(default-allow)
3. 만약 한개의 네트워크 정책이라도 설정되면, 정책에 영향을 받는 Pod에 대해서 해당 네트워크 정책 이외의 나머지 트래픽은 모두 막힌다.(default-deny)

#### NetworkPolicy 리소스
네트워크를 제어하기 위해서 NetworkPolicy 리소스를 사용해야한다.
네임스페이스 레벨에서 동작하며 라벨 셀렉터를 이용하여 특정 Pod에 네트워크 정책을 적용함

네트워크를 구성해보자
먼저 기본적으로 전체 인바운드 트래픽을 차단해서 외부 트래픽이 들어올 수 없게 **private zone**으로 만든다.
```
#deny-all.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}  
  ingress: []
```
ingress property에 빈 리스트가 선언되어 있음 -> 허용되는 인바운드 정책이 비어있다는 의미
전체 인바운드 트래픽을 허용할 때는 ingress 리스트에 빈 딕셔너리 {}로 선언했었음. -> 특정 조건을 가지지 않는 모든 트래픽을 의미

지금 전체 트래픽이 막혀있다.

웹서버를 만들고 외부 사용자들이 접근할 수 있도록 Pod의 80번 포트만 허용해주자.
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-open
  namespace: default
spec:
  podSelector:      # run=web인 Pod에 대해서 다음과 같은 정책을 허용
    matchLabels:
      run: web
    ingress:
    - from:
      - podSelector: {}
      ports:
      - protocol: TCP
        port: 80
```

이번에는 웹서버와 내부에서 통신할 앱서버를 만들어보자
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-from-web
  namespace: default
spec:
  podSelector:      # run=web인 Pod에 대해서 다음과 같은 정책을 허용
    matchLabels:
      run: web
  ingress:
  - from:
    - podSelector:
        matchLabels:
        run: web
      
```
ingress에서 podSelector로 run=web 라벨을 가진 Pod에 대해서 허용하고 있다. 포트는 전체 다 허용


#### Egress 아웃바운드 트래픽 제어
사용자에게 개발용 네임스페이스를 제공하고 그 속에서만 서비스를 개발하고, 운영중인 다른 네임스페이스에는 접근을 차단하고 싶을 때, 특정 네임스페이스의 아웃바운드를 모두 막을 수 있다.
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: dont-leave-dev
  namespace: dev
spec:
  podSelector: {}
  egress:
  - to:
    - podSelector: {}
```
 
#### 특정 아이피대역 요청 차단
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: block-metadata
  namespace: default
spec:
  podSelector: {}
  egress:
  - to:
    - ipBlock:
       cidr: 0.0.0.0/0
       except:
       - 169.254.169.254/32
```
특정 IP를 기준으로 아웃바운드를 막아야 하는 경우 유용하게 사용할 수 있음

#### AND와 OR
from property의 리스트 원소를 2개의 podSelector로 선언하면 AND로 작용한다.

ingress property 아래의 각각의 from을 선언하면 OR로 표현된다.

### 로깅과 모니터링
``kubectl logs`` 명령어를 사용하면 컨테이너 로그를 확인할 수 있지만 Pod가 죽게 되거나 1회성 배치 작업인 경우, Pod가 사라지게 되어 로그 기록도 같이 삭제된다.
클러스터 레벨의 로깅 시스템을 따로 구축하여 로그 기록을 저장하면 언제든지 과거 로그를 확인할 수 있다.

쿠버네티스는 kubelet과 같은 시스템 컴포넌트가 존재한다.
보통 systemd를 통해 쿠버네티스 컴포넌트를 실행하는데 이러한 시스템 컴포넌트도 로깅시스템으로 기록 확인할 수 있다.

위의 이유로 클러스터 레벨의 로깅 시스템을 따로 구축해서 사용한다.
여기서는 EFK스택( 엘리스틱서치, fluent-bit, kibana)로 로깅시스템 구축에 대해 알아보자

#### 엘라스틱서치
EFK스택에서 엘라스틱서치는 로그 저장소로 이용하며 아래 개념을 사용
+ index: document의 집합으로 idnex를 기준으로 데이터를 질의하고 저장한다. DB의 table 개념
+ shard: 성능 향상을 위해 index를 나눈 것. DB의 partition
+ document: 1개의 행
+ field: document안에 있는 열

장점
1. 비정형 데이터를 다룰 수 있는 유연함(text 기반 검색엔진이므로)
2. 확장성과 가용성
3. Full Text 검색이 빠름
4. 계층적인 데이터도 쿼리 가능
5. RESTful API 지원
6. 준 실시간 쿼리

단점
1. 업데이트 비용이 큼
2. 트랜젝션 기능 부재
3. JOIN 기능 부재


#### fluent-bit(fluentd의 경량버전)
로그수집기
많은 플러그인 지원

#### Kibana
웹에서 dashboard를 제공하는 데이터 시각화 플랫폼
KQL(kibana Query Language)를 지공하여 Kibana 플랫폼에서 엘리스틱서치로 쿼리 날릴 수 있음

### 리소스 모니터링
쿠버네티스 환경에서는 특정 서버라는 경계가 모호해지고 애플리케이션 단 위로 모니터링 대상이 세밀해짐
쿠버네티스에선 모니터링 agent를 설치하여 agent가 메트릭을 모니터링 시스템에 전달하는 push-based 방식보단 모니터링 시스템이 수집해야하는 대상을 찾아 직접 메트릭을 수집하는 pull-based를 쓴다.

#### prometheus

특징
+ key/value 형태의 time series 데이터 구조를 지님
+ PromQL이라는 유연한 질의 언어 제공
+ 분산 스토리지 사용하지 않음
+ HTTP를 이요한 pull 방식으로 메트릭 수집
+ 수집 대상은 service discovery로 찾음

정보수집 원리는 도커에서 
> docker stats <COTINAER_ID>
위에 명령어를 사용해서 리소스 정보를 확인한다.

### CI/CD
jenkins를 쿠버네티스에 셋팅하면 Pod가 생설될 때마다 젠킨스가 따라 붙는다. 기존에는 노드가 생길떄마다 젠킨스를 정적으로 붙여줘야 했다.

dynamic worker 장점
1. 쉽게 확장할 수 있음
2. 리소스 사용이 효율적
3. 클러스터 관리가 쉬워짐


단점
1. worker 노드 생성 시 지연시간 발생
2. worker에서 생성된 결과가 바로 사라짐 -> PCV 연결등으로 해결가능(외부에 저장해야함)
3. 빌드 디버깅이 어려움 -> Pod가 삭제되면 로그 기록이 사라져서

#### 도커 안에서 도커
젠킨스가 도커 컨테이너로 구동이 되고 젠킨스 안에서 도커 이미지를 생성한다.
2가지 방법이 있다.
1. 도커 컨테이너 안에 도커 데몬서버 자체가 들어있는 형태
2. 컨테이너 안에서 호스트 서버의 도커 서버를 참조하는 방법
후자의 방법을 사용하는게 좋다.

docker라는 이미지를 실행하면서 도커 소켓을 볼륨으로 연결한다.
> docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock docker

### Single Source Of Truth(SSOT)
단일 진실의 원천.
결과는 한가지 이유에서 비롯된다는 원칙
> e.g) 강아지가 울고 있는데(결과) 그 결과는 오직 배고파서(이유) 그런거라고 가정. 강아지가 배고프지 않으면 울지 않음. 

소프트웨어 배포 관점에서 크게 2가지 원인을 보자
1. GitOps에서 소프트웨어를 단일한 원천에서 배포하여 그 과정을 일원화함. (배포의 방법을 늘리지 않음) GitOps에서 배포 결과(진실)은 오직 단일한 원천에서 나오도록함
2. 소프트웨어 배포 과정이 단일 진실의 원천 원칙을 지켰다면 배포 결과를 완벽히 반영함 -> 문제가 생기면 원천이 어떻게 구성(YAML 정의서)되었는지 확인하면 됨


장점
1. 현재 배포 환경의 상태를 쉽게 파악
2. 빠르게 배포
3. 안정적으로 운영환경에 배포

GitOps의 원칙
1. YAML로 선언형으로 배포해야함
2. Git을 이용한 배포 버전 관리
3. 변경 사항 운영반영 자동화
4. 이상 탐지 및 자가 치유

GitOps의 구현체로 FluxCD, ArgoCD, Codefresh, Jenkins X 등 있음


