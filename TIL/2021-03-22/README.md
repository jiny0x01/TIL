# TIL

## 오늘 공부한 컨셉

+ gRPC 인터셉터
+ gRPC 데드라인
+ gRPC 취소처리
+ gRPC 에러처리

## 상세내용



### gRPC 고급기능

gRPC 애플리케이션 구축할 때 고려할 점

+ 수신과 발신
+ RPC에 대한 인터럽트 처리
+ 네트워크 지연 처리
+ 에러 처리
+ 서비스 소비자 간의 메타데이터 공유 등



## gRPC 인터셉터

gRPC 애플리케이션은 클라이언트/서버에 원격 함수 실행 전후 몇 가지 공통적인 로직을 실행할 필요가 있음

인터셉터라는 확장 메커니즘을 제공하여 로깅/인증/메트릭 등 특정 요구사항 충족을 위해 RPC 실행을 가로챌 수 있음

gRPC 인터셉터 유형

+ unary interceptor(단일 인터셉터) : 단순 RPC
+ streaming interceptor : 스트리밍 RPC



#### 서버 단일 인터셉터

아래 함수에서 gRPC로 들어오는 모든 단일 RPC 호출 전체를 제어할 수 있다.

```go
func orderUnaryServerInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    // 전처리 로직
    // 인자로 넘겨진 info를 통해 현재 RPC 호출에 대한 정보를 얻음
    log.Println("======[Server Interceptor] ", info.FullMethod) // RPC를 호출하기 전에 메시지를 가로챌 수 있음
    
    // 단일 RPC의 정상 실행을 완료하고자 핸들러를 호출함
    m, err := handler(ctx, req)
     
    log.Printf(" Post Proc Message : %s", m) // RPC 호출의 응답을 처리
    return m, err  // RPC 응답을 다시 돌려보냄
}

func main() {
   // 서버측에서 인터셉터를 등록함
  s := grpc.NewServer( grpc.UnaryInterceptor(orderUnaryServerInterceptor))
}
```

서버 측 단일 인터셉터 구현은 3가지로 구분할 수 있다.

1. 전처리(preprocessing phase)
   + 개발자는 RPC 호출에 대한 정보를 얻을 수 있음
   + RPC 호출을 수정할 수 있음
2. RPC 메서드 호출(invoker phase) 
   + gRPC UnaryHandler를 호출해야 RPC 메서드를 호출함
3. 후처리
   + 후처리가 필요하지 않은 경우, 핸들러 호출``(handler (ctx, req))`` 을 반환할 수 있다.



#### 서버 스트리밍 인터셉터

스트리밍 인터셉터는 전처리 단계와 stream operation interception phase를 포함한다.

```go
// 서버-스트림 인터셉터
// wrappedStream이 내부의 grpc.ServerStream을 wrapping하고
// RecvMsg와 SendMsg 메서드 호출을 가로챈다.

type wrappedStream struct {
	grpc.ServerStream
}

func (w *wrappedStream) RecvMsg(m interface{}) error {
  log.Printf("========[Server Stream Interceptor Wrapper] " + "Receive a message (Type: %d) at %s", m, time.Now().Format(time.RFC3339))
  return w.ServerStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
  log.Printf("======== [Server stream Interceptor Wrapper]" + "Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
  return w.ServerStream.SendMsg(m)
}

func newWrappedStream(s grpc.ServerStream) grpc.ServerStream {
  returnr &wrappedStream{s}
}

func orderServerStreamInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
  log.Println("======= [Sever Stream Interceptor] ", info.FullMethod)
  err := handler(srv, newWrappedStream(ss))
  if err != nil {
    log.Printf("RPC failed with error %v", err)
  }
  return err
}

//인터셉터 등록
s := grpc.NewServer(grpc.StreamInterceptor(orderServerStreamInterceptor))
```

+ 전처리 단계에서 스트리밍 RPC 호출이 서비스 구현으로 이동하기 전에 인터셉트할 수 있고
+ 전처리 단계 후에 원격 메서드의 RPC 호출을 실행하고자 StreamHandler를 호출한다
+ 전처리 단계 이후에도 grpc.ServerStream 인터페이스를 구현하는 wrapper stream 인터페이스를 사용해 스트리밍 RPC 메시지를 가로챌 수 있다.
+ hander(srv, newWrappedStreeam(ss))와 같이 grpc.StreamHandler를 호출할 때 이 래퍼 구조체를 전달할 수 있는데
  + grpc.ServerStream 래퍼는 gRPC 서비스에서 스트리밍 메시지를 가로챈다.



클라이언트도 서버와 대동소이하다.



### 데드라인

데드라인과 타임아웃은 분산 컴퓨팅에서 일반적으로 사용되는 패턴이다.

하나의 요청이 하나 이상의 서비스를 함께 묶는 여러 downstream RPC로 구성되는 예를 보자. 

이 경우 각 서비스 호출마다 개별 RPC를 기준으로 타임아웃을 적용할 수 있지만, 요청 전체 수명주기에는 직접 적용할 수 없다. 이런 경우에 데드라인을 사용한다.

gRPC API도 RPC 데드라인 사용을 지원하는데, 여러 가지 이유로 **항상 gRPC 애플리케이션에서 데드라인을 지정하는 것이 바람직하다**

```go
// 클라이언트 애플리케이션의 gRPC 데드라인
conn, err := grpc.Dial(address, grpc.WithInsecure())
if err != nil {
log.Fatalf("did not connect: %v", err)
}
defer conn.Close()
client := pb.NewOrderManagementClient(conn)

clientDeadline := time.Now().Add(time.Duration(2 * time.Second))
ctx, cancel := context.WithDeadline(context.Background(), clientDeadline)

defer cancel()

order1 := pb.Order{Id: "101", Items:[]string{"Iphone xs", "mac book"}, Destination: "San Jose, CA", Price:2300,00}
res, addErr := client.AddOrder(ctx, &order1)

if addErr != nil {
	got := status.Code(addErr)
	log.Printf("Error Occured -> addOrder : , %v: ", got)
} else {
log.Print("addOrder Response -> ", res.Value)
}
```

데드라인의 이상적인 값 선택 요소

+ 개별 서비스의 end-to-end 지연시간
+ RPC가 직렬화 되는지
+ 병렬로 호출될 수 있는지
+ 기본 네트워크의 지연시간과 다운스트림 서비스의 데드라인 값



#### 취소 처리

서버측에서 통신이 성공적으로 끝나지만 클라이언트는 실패할 수 있다. 클라이언트와 서버가 RPC 결과에 대해 다른 결론을 내릴 수 있는 여러 조건이 있을 수 있는데, RPC를 중단시키려고 할 때 RPC를 canceling하면 된다.

양방향 스트리밍 방식에 대해서 생각해보자. context.WithTimeout 호출에서 cancel 함수를 얻을 수 있고 cancel에 대한 참조가 있으면 RPC를 중단하려는 아무 위치에서나 이를 호출할 수 있다.

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
streamProcOrder, _ := client.ProcessOrders(ctx)
_ = streamProcOrder.Send(&wrapper.StringValue{Value:"102"})
_ = streamProcOrder.Send(&wrapper.StringValue{Value:"103"})
_ = streamProcOrder.Send(&wrapper.StringValue{Value:"104"})

channel := make(chan bool, 1)

go asncClientBidrectionalRPC(streamProcOrder, channel)
time.Sleep(time.Millisecond * 1000)

cancel()
log.Printf("RPC status : %s", ctx.Err())

_ = streamProcOrder.Send(&wrapper.StirngValue{Value:"101"})
_ = streamProcOrder.CloseSend()

<- channel

func asncClientBidirectionalRPC (
				streamProcOrder pb.OrderManagement_ProcessOrdersClient, c chan bool) {
		combinedShipment, errProcOrder := streamProcOrder.Recv()
		if errProcOrder != nil {
		log.Printf("Error Reciving message %v", errProcOrder)
		}
}
```

RPC를 취소하면 상대방이 context를 통해 취소를 확인할 수 있다.

위 예제에서 서버 애플리케이션은 ``stream.Context().Err() == context.Canceled`` 를 사용해 현재 context가 취소됐는지 여부를 확인한다.

gRPC 에러 코드의 전체 목록은 여기를 참조

> https://oreil.ly/E610Q0



에러 처리에 대한 use-case

```go
// 주문 관리 예제에서 AddOrder 원격 메서드 호출 시 잘못된 주문 ID로 요청이 들어올 때 에러 처리
if orderReq.Id == "-1" {
	log.Printf("Order ID is invalid! -> Recived Order Id %s", orderReq.Id)
}

errorStatus := status.New(codes.InvalidArgument, "Invalid information received")
ds, err := errorStatus.WithDetails(&epb.BadRequests_fieldViolation{
			Field:"ID",
			Description: fmt.Sprintf("order ID received is not valid %s : %s", orderReq.Id, orderReq.Description),
	},
)
if err != nil {
	return nil, errorStatus.Err()
}
return nil, ds.Err()

```

grpc.status 패키지에서 에러 상태를 간단하게 만들 수 있다.

```go
// 클라이언트에서 에러처리
order1 := pb.Order{Id: "-1",
		Items:[]string{"iPhone Xs", "mac book pro"}, 
		Destination:"San Jose, CA", Price:2300.00}
   
res, addOrderError := client.AddOrder(ctx, &order1)

if addOrderError != nil {
    errorCode := status.Code(addOrderError)
    if errorCode == codes.InvalidArgument {
    log.Printf("Invalid Argument Error : %s", errorCode)
    errorStatus := status.Convert(addOrderError)
    for _, d := range errorStatus.Details() {
    switch info := d.(type) {
    case *epb.BadRequest_FieldViolation:
    			log.Printf("Request Field Invalid: %s", info)
    default:
          log.Printf("Unexpected error type: %s", info)      
    	}
    }
} else {
   log.Printf("Unhandled error : %^s", errorCode)
} else {
log.Print("AddOrder response -> " , res.Value)
}
```





