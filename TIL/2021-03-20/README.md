# TIL

## 오늘 공부한 컨셉

+ gRPC 동작 원리

## 상세내용



gRPC에서 클라이언트 스텁은 인코딩 메시지로 HTTP POST 요청을 생성한다. 

특정 필드의 필드 인덱스와 와이어 타입을 아렴ㄴ 아래 식을 사용해 필드의 태그 값을 결정할 수 있다. 필드 인덱스의 바이너리 표현을 3자리만큼 왼쪽으로 시프트연산하고 와이어 타입의 바이너리 값과 비트 OR 연산을 하면 된다.

> Tag value = (field_index << 3) | wire_type

> message ProductID {
>
> ​	string value = 1; 
>
>    string은 와이어 타입, 1은 필드 인덱스
>
> }

전송 동작 원리

1. 필드 인덱스와 와이어타입을 위의 식을 통해 계산하고

2. 문자열은 UTF-8로 인코딩, int32는 가변길이 정수 인코딩 기술을 사용하여 인코딩한다.

3. 메시지가 인코딩 되면 해당 태그와 값이 바이트 스트림으로 연결됨

4. 스트림의 끝은 0이라는 태그 값을 전송해 표시함

   

### 가변길이정수

하나 이상의 바이트를 사용해 정수를 직렬화 하는 방법

대부분의 숫자가 균등하게 분포돼 있지 않다는 아이디어를 기반으로 함

가변 길이 정수에서 마지막 바이트를 제외한 각 바이트에 앞으로 더 많은 바이트가 있음을 나타내고자 최상위 비트가 1이 된다. 각 바이트의 하위 7비트는 2의 보수 표현으로 저장되며 최하위 비트가 먼저 나오기 때문에 하위 그릅에 연속 비트를 추가해야한다.



300을 가변 길이 정수로 전송하는 예시

다음 2개 바이트를 전송한다고 가정

> 1010 1100 0000 0010

첫 번째 바이트의 최상위 비트가 1이기 때문에 다음 바이트까지 하나의 정수가 저장되고 하위 7비트만을 사용하기 때문에 다음과 같이 변경된다.

> 010 1100 000 0010 -> 000 0010 010 1100

이제 10진수로 변경하면 다음과 같이 계산된다.

> 256 + 32 + 8 + 4

최하위 그룹이 먼저 나오는 방식은 스트림 방식으로, 바이트를 전송할 때 계산이 쉽다는 장점이 있다. 위의 예를 다시 생각해보면 수신되는 바이트의 최상위 비트를 먼저 확인해 계속적으로 계산을 처리할 수 있다.

#### 부호 있는 정수

지그재그 인코딩이 부호 있는 정수를 부호 없는 정수로 변환하는데 사용함

변환한 다음에는 부호 없는 정수는 위에서 언급한 가변길이정수 인코딩 방법을 사용함

| 원래 값 | 매핑된 값 |
| ------- | --------- |
| 0       | 0         |
| -1      | 1         |
| 1       | 2         |
| -2      | 3         |
| 2       | 4         |

표를 보면 값 0은 0에 매핑되고, 다른 값은 지그재그 방식으로 양수에 매핑된다.

-1 -> 1

1 -> 2 ...

원래 음의 값은 홀수 양수로 매핑되고, 원래 양의 값은 짝수 양으로 매핑된다.

지그재그 인코딩 후에는 원래 값의 부호에 관계없이 양수로 매핑되는데, 얻어진 양수는 가변 길이 정수로 인코딩한다.

int32/int64와 같은 일반 타입을 사용하는 경우, 음수는 가변 길이 인코딩을 사용해 바이너리로 변환되기 때문에 음의 정수는 sint32/sint64와 같은 부호 있는 정수 타입을 사용하는 것이 좋다. 

음의 정수에 대한 가변길이정수 인코딩은 양의 정수보다 같은 바이너리 값을 나타내고자 더 많은 바이트가 필요하다. 따라서 음수값을 인코딩하는 효율적인 방법은 음수값을 양수로 바꾸고 양수값을 인코딩한다.

> 음의 정수는 정해진 바이트의 맨 앞 비트가 1이라서 크기에 상관없이 전체 바이트를 모두 사용함. 지그재그 인코딩으로 변환하면 적은 양의 바이트 사용하게됨

#### 비가변 길이 정수 숫자

실제 값에 상관없이 고정된 바이트 수를 할당

+ fixed64, sfixed64, double
+ fixed32, sfixed32, float

프로토콜 버퍼 인코딩에 대한 자세한 설명

> https://oreil.ly/hH_gL

### 인코딩 다음 단계. 

인코딩이 끝나면 메시지를 보낼 준비를 위해 메시지 프레임을 만든다.

message-framing은 정보를 쉽게 추출할 수 있도록 관련 정보와 커뮤니케이션을 구성하는 것 -> 상대방이 정보를 쉽게 추출하게 해야함

gRPC에서는 length-prefix 메시지 프레이밍 기술을 사용

#### length-framing

메시지 자체를 전송하기 전에 각 메시지의 크기를 기록하는 방식

+ 인코딩된 바이너리 메시지 앞에 메시지 크기를 지정하고자 4바이트가 할당됨

+ gRPC 통신은 최대 4GB 크기의 모든 메시지를 처리할 수 있다.

+ 메시지 길이는 4바이트 빅엔디언 정수로 표현된다.

+ 데이터의 압축 여부를 나타내는 1바이트 부호 없는 정수가 있다.
+ 압축 플래그가 1이면 HTTP 전송에서 선언된 헤더 중 하나인 메시지 인코딩 헤더에 선언된 메커니즘을 사용해 바이너리 데이터가 압축됐음을 나타냄
  + 값이 0이면 메시지 바이트 인코딩이 발생하지 않았음을 타나냄

수신측에서는

+ 메시지를 받으면 첫 번째 바이트를 읽어 메시지의 압축 여부를 확인
+ 수신자는 다음 4바이트를 읽어 인코딩된 바이너리 메시지의 크기를 얻음
  + 스트림에서 정확한 바이트 길이를 읽을 수 있음
+ 단순/단일 메시지의 경우 하나의 length-prefix 지정 메시지를 갖고
+ 스트리밍 메시지의 경우는 여러 개의 length-prefix 지정 메시지를 가짐



### gRPC가 네트워크를 통해 length-prefix 지정 메시지를 보내는 방법을 알아보자

gRPC 코어는 세 가지 전송 구현을 지원한다

+ HTTP/2
+ Cronet (https://oreil.ly/D0laq)
+ in-process (https://oreil.ly/lRgXF)



#### HTTP/2

용어정리

+ 스트림: 설정도니 영결에서 양방향 바이트 흐름. 스트림은 하나 이상의 메시지를 전달 가능
+ 프레임: HTTP/2에서 가장 작은 통신 단위, 프레임에는 프레임 헤더가 포함되 있으며 헤더를 통해 프레임이 속한 스트림을 식별
+ 메시지: 하나 이상의 프레임으로 구성된 논리적 HTTP 메시지에 매핑되는 온전한 프레임 시퀀스. 이는 클라이언트와 서버가 메시지를 독립 프레임으로 분류하고 인터리브한후 다른 쪽에서 다시 조립할 수 있는 메시지 멀티플렉스를 지원



**요청 메시지**

원격 호출을 시작하는 메시지다. gRPC에서 요청 메시지는 항상 클라이언트 애플리케이션에 의해 트리거 됨.

요청 메시지 -> 요청 헤더 -> length-prefix message -> 스트림 종료 플래그(EOS: End Of Stream)

```
HEADERS (flags = END_HEADERS)
:method = POST
:scheme = http         # TSL가 활성화되면 https로
:path = /ProductInfo/getProduct
:authority = abc.com
te = trailers					 # 호환되지 않는 프록시 탐지를 정의. gRPC는 trailers
grpc-timeout = 1S
content-type = application/grpc
grpc-encoding = gzip
authorization = Bearer xxxxxx
```

":"로 시작하는 헤더 이름은 reserved header다. HTTP/2에서는 reserved header가 다른 헤더보다 앞에 나온다.

전송할 데이터가 없으면서 요청 스트림을 종료해야 하는 경우, 구현은 END_STREAM 플래그를 사용해 아래 빈 데이터 프레임을 보낸다.

> DATA (flags = END_STREAM)
>
> <Length-Prefixed Message>



**응답 메시지**

응답 메시지의 세 가지 주요 요소

+ response header
+ length-prefix message
+ trailer

클라이언트에 응답으로 보낼 length-prefix 메시지가 없으면 응답헤더/트레일러만 구성해서 보낸다.

응답 헤더

```
HEADERS (flags = END_HEADERS)
:status = 200
grpc-encoding = gzip
content-type = application/grpc
```



