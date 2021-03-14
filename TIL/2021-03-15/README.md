# TIL

## 오늘 공부한 컨셉
+ 고급스케줄링
  + horizontalPodAutoScaler
  + 수평적확장(scale out), 수직적확장(scale up)
  + taint, toleration
  + affinity, antiaffinity

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
