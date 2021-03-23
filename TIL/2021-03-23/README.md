# TIL

## 오늘 공부한 컨셉

+ 멀티플렉싱
+ 메타데이터
+ 로드밸런싱
+ TLS, mTLS
+ basic auth, Oauth2.0 붙이는법. JWT도 지원함

## 상세내용

## 멀티플렉싱

지금까지는 하나의 gRPC 서비스가 등록되고 gRPC가 클라이언트/서버에 1:1로 매칭되는 경우만 보았다.

gRPC를 사용하면 동일한 gRPC 서버에서 여러 gRPC 서비스를 실행할 수 있고 여러 gRPC 클라이언트 스텁에 동일한 gRPC 클라이언트 연결을 사용할 수 있다. 이를 **multiplexing** 이라 한다.

e.g) OrderManagement 서비스 예에서는 하나의 gRPC 서버에서 주문 관리 목적에 필요한 다른 서비스를 실행해 클라이언트 애플리케이션이 동일한 연결을 재사용해서 두 서비스를 모두 호출할 수 있다고 가정.  이 경우 각각의 서버 등록 기능을 통해 두 서비스를 동일한gRPC 서버에 등록할 수 있다.

```go
func main() {
    initSampleData()
    lis, err := net.Listen("tcp", port)
    if err != nil {
		    log.Fatalf("failed to listen: %v", err)
    }
    grpcServer := grpc.NewServer()
    
    //grpc orderMgtServer에 주문 관리 서비스 등록
    ordermgt_pb.RegisterOrderManagementServer(grpcServer, &orderMgtServer{})
    
    // grpc orderMgtServer에 Greeter 서비스 등록
    hello_pb.RegisterGreeterServer(grpcServer, &helloServer{})
}
```



MSA에서 gRPC 멀티플렉싱의 강력한 용도중 하나는. 한 서버 프로세스에서 동일한 서비스의 여러 주요 버전을 호스팅 하는 것.

이를 통해 API 변경 후 서비스가 레거시 클라이언트를 수용할 수 있다.

서비스 계약에 대한 이전 버전이 더 이상 사용되지 않으면 서버에서 제거하면 된다.

예를 들면

1. 서버 프로세스에서 v1.0 API를 gRPC로 지원중.
2. API 버전이 v2.0로 업데이트 되어 서버 프로세스에서 v1.0과 v2.0을 동시에 멀티플랙싱
3. v1.0을 지원 종료하면서 서버 프로세스는 v1.0을 제거하면됨. v2.0을 제공중이니 서비스엔 문제없음



## 메타데이터

대부분의 서비스의 BM로직 및 소비자와 직접 관련된 정보는 원격 메서드 호출 인자의 일부분이다.

특정 조건에서 RPC의 비즈니스 컨텍스트와 관련이 없는 RPC 호출 정보를 공유할 수 있는데, RPC 인자의 일부가 돼서는 안된다.

이런 경우는 gRPC 메타데이터를 사용할 수 있다. gRPC 헤더를 사용해 클라이언트나 서버에서 생성한 메타데이터를 클라이언트와 서버 애플리케이션 간에 교환할 수 있다. 메타데이터는 key/value다.

메타데이터의 일반적인 사용

+ gRPC 애플리케이션간 보안 헤더를 교환하는 것
+ gRPC 애플리케이션간 임의의 정보를 교환하는 것
+ gRPC 메타데이터 API는 인터셉터 내부에서 많이 사용됨

### 메타데이터 생성과 조회

``metadata.New(map[string]string{"key1": "val1", "key2": "val2"})`` 형식으로 만들어짐

``metadata.Pairs`` 를 사용하면 메타데이터를 쌍으로 만들 수 있음. 동일한 키를 가진 메타데이터가 목록으로 합쳐짐

```go
// 메타데이터 생성법1
md := metadata.New(map[string]string{"key1" : "val1", "key2" : "val2"})

// 메타데이터 생성법2
md := metadata.Pairs(
	"key1", "val1",
	"key1", "val1-2",
	"key2", "val2"
)
```



바이너리 데이터를 메타데이터로 설정가능. 데이터 전송전에 base64로 인코딩하고 전송후 디코딩함

메타데이터 조회는 ``metadata.FromIncomingContext(ctx)``와 함께 RPC 호출의 수신 콘텍스트를 사용해 수행할 수 있음

```go
func (s *server) AddOrder(ctx context.Context, orderReq *pb.Order) (*wrappers.StringValue, error) {
	md, metadataAvailable := metadata.FromIncomingContext(ctx)
	// 'md' 메타데이터 맵에서 필요한 메타데이터를 읽음
}
```



메타데이터를 생성하고 RPC 호출 콘텍스트에 메타데이터를 넣어서 클라이언트에서 서버로 메타데이터를 보낼 수 있음

Go에서는 2가지 방법이 있음

1. NewOutgoingContext
   1. 새 context를 생성하면서 새롭게 메타데이터를 생성
2. AppendToOutgoingContext
   1. 기존 context에 메타데이터를 추가할 수 있음. 
   2. 이 경우 사용하던 context의 기존 메타데이터가 replace됨

```go
// grpc 클라이언트에서 메타데이터 전송
md := metadata.Pairs(
  "timestamp", time.Now().Format(time.StampNano),
  "kn", "vn",
)
mdCtx := metadata.NewOutgoingContext(context.Background(), md)

ctxA := metadata.AppendToOutgoingContext(mdCtx, "k1", "v1", "k1", "v2", "k2", "v3")

// simple RPC
response, err := client.SomeRPC(ctxA, someRequest)

// streaming RPC
stream, err := client.SomeStreamingRPC(ctxA)
```



```go
// grpc 클라이언트에서 메타데이터 읽기
var header, trailer metadata.MD

// simple RPC
r, err := client.SomeRPC(
	ctx,
	someRequest,
	grpc.Header(&header),
	grpc.Trailer(&trailer),
)

// 헤더 트레일러 맵 처리

// streaming RPC
stream, err := client.SomeStreamingRPC(ctx)

// 헤더 조회
header, err := stream.Header()

// trailer 조회
trailer := stream.Trailer()
```



```go
// 서버 메타데이터 읽기
// simple RPC
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    md, ok := meatadata.FromIncomingContext(ctx)
    
    // md에 메타데이터 있으니 활용
}

// streaming RPC
func (s *server) SomeStreamingRPC(stream pb.Service_someStreamingRPCServer) error {
    md, ok := metadata.FromIncomingContext(stream.Context())

   // 메타데이터 읽기
}
```



```go
// 서버 메타데이터 전송
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    //헤더 생성 및 전송
    header := metadata.Pairs("ehader-key", "val")
    grpc.SendHeader(ctx, header)
    
 		// 트레일러 생성과 지정
 		trailer := metadata.Paris("trailer-key", "val")
 		grpc.SetTrailer(ctx, trailer)
}

func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
		// 헤더 생성과 전송
		header := metadata.Pairs("header-key", "val")
		stream.SendHeader(header)
		
		// 트레일러 생성과 지정
		trailer := metadata.Paris("trailer-key", "val")
		stream.SetTrailer(trailer)
}
```



### Name resolving

서비스 이름에 대한 백엔드 IP의 목록을 반환한다.

```go
const (
		exampleSchema					= "example"
		exampleServiceName 		= "lb.example.grpc.io"
)
var addrs = []string{"localhost:50051", "localhost:50052"}

type exampleResolverBuilder struct{}

func (*exampleResolverBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOption) (resolver.Resolver, error) {
  r := &exampleResolver{
    target:			target,
    cc:					cc,
    addrsStore:		map[string][]string{
      exampleServiceName: addrs,
    },
  }
  r.start()
  return r, nil
}

func (*exampleResolverBuilder) Scheme() string { return exampleScheme }

type exampleResolver struct {
  target				resolver.Target
  cc						resolver.ClientConn
  addrsStore		map[string][]string
}

func (r *exampleResolver) struct() {
  addrStrs := r.addrsStore[r.target.Endpoint]
  addrs := make([]resolver.Address, len(addrStrs))
  for i, s:= range addrStrs {
    addrs[i] = resolver.Address{Addr: s}
  }
  r.cc.UpdateState(resolver.State{Addresses: addrs}))
}

func (*exampleResolver) ResolveNow(o resolver.ResolveNowOption) {}

func (*exampleResolver) Close() {}

func init() {
  resolver.Register(&exampleResolverBuild{})
}
```



gRPC 애플리케이션에서 로드밸런싱을 할 때는 name resolver를 사용해야 한다.



## 로드밸런싱

고가용성과 확장성을 만족시키기 위해서 실제 서비스 환경에서는 둘 이상의 gRPC 서버를 실행해야 한다.

서비스들 사이에 RPC 호출을 분산시키려면 일부 entity에서 처리해야 하는데, 이것이 로드밸런싱의 역할이다.

gRPC에는 일반적으로 로드밸런서 프록시와 클라이언트 측 로드밸런싱 2가지 메커니즘이 적용된다.



**로드밸런서 프록시**

1. 클라이언트가 LB(load balancer) 프록시에 RPC를 요청한다.
2. LB 프록시는 백엔드 gRPC 서버 중 하나에게 RPC 호출을 분배한다.
3. LB 프록시는 각 백엔드 서버의 load를 추적하여 알맞게 분배한다.

백엔드 서비스의 topology는 gRPC 클라이언트에 공개되지 않으며 클라이언트는 로드밸런서의 엔드포인트만 알고 있다.

> topology : (컴퓨터 네트워크의 요소를 물리적으로 연걸해 놓은 것 또는 그러한 방식)

Nginx, Emvoy proxy 등과 같은 로드밸런싱 솔루션을 gRPC 애플리케이션의 LB 프록시로 사용할 수 있다.



**클라이언트 측 로드밸런싱**

gRPC 클라이언트 레벨에서 로드밸런싱 로직을 구현할 수 있다. 이 방법에서는 여러 백엔드 gRPC 서버를 인식하고 각 RPC에 사용할 하나의 서버를 선택한다.

로드밸런싱 로직은 클라이언트 애플리케이션의 일부로 개발되거나 lookaside 로드밸런서라고 하는 전용 서버에서 구현될 수 있다.

```go
pickfirstConn, err := grpc.Dial(
    fmt.Sprintf("%s:///%s",
    		// exampleScheme = "example"
    		// exampleServiceName = "lb.example.grpc.io"
    		exampleScheme, exampleServiceName),
    // pick_first가 기본값
   값
   grpc.WithBalanceName("pick_first"),
   grpc.WithInsecure(),
)
if err != nil {
		log.Fatalf("did not connect: %v", err)
}
defer pickfirstConn.Close()

log.Println("==== Calling helloworld.Greeter/SayHello " + "with pick_first ====")
makeRPCs(pickfirstCOnn, 10)

// 라운드로빈 정책으로 다른 클라이언트 커넥션을 만든다.
roundrobinConn, err := grpc.Dial()

defer roundrobinConn.Close()
makeRPCs(roundrobinConn, 10)
```

gRPC에서는 pick_first/round-robin 2가지 로드밸런싱 정책이 지원된다.

pick_first : 첫 번째 주소에 연결시도 -> 연결되면 모든 RPC에 이를 사용. 실패하면 다음 주소 시도

round_robin : 모든 주소에 연결. 한 번에 하나씩 RPC를 각 백엔드에 보냄



### 압축

네트워크 대역폭을 효율적으로 사용하기 위해 RPC가 실행될 때 압축할 수 있다.

클라이언트에서 RPC를 수행할때 compressor를 설정해 구현한다.

``client.AddOrder(ctx, &order1, grpc.UseCompressor(gzip.Name))``  과 같이 사용한다.

위 코드에선 gzip 패키지를 ``"google.golang.org/grpc/encoding/gzip"``이 제공한다.



## 보안 적용

#### TLS를 사용한 gRPC 채널 인증

두 애플리케이션간 정보보호와 데이터 무결성 제공을 목표로함.

클라이언트 서버 간의 연결이 안전한 경우는 다음 속성 중 하나 이상을 만족시켜야 함.

+ 연결은 private하게
+ 연결은 reliable하기(신뢰적)



서버에서 단방향 보안 연결 사용

+ 단방향 보안은 가장 간단한 방법
+ 서버는 공개키/개인키 쌍으로 초기화해야함

```go
package main
import (
		"crypto/tls"
		"errors"
		pb "productinfo/server/ecommerce"
		"google.golang.org/grpc"
		"gogole.golang.org/grpc/credentials"
		"log"
		"net"
)

var (
		port = ":50051"
		crtFile = "server.crt"
		keyFile = "sever.key"
)

func main() {
	cert, err := tls.LoadX509KeyPair(crtFile, keyFile)
	if err != nil {
			log.Fatalf("failed to load key pair : %s", err)
	}
	opts := []grpc.ServerOption{
			grpc.Creds(credentials.NewServerTLSFromCert(&cert))
	}
	
	s := grpc.NewServer(otps...)
	
	pb.RegisterProductInfoSever(s, &server{})
	
	lis, err := net.Listen("tcp", port)
	if err != nil {
			log.Fatalf("failed to listen: %v", err)
	}
	
	if err := s.Serve(lis); err != nil {
			log.Fatalf("failed to serve: %v", err)
	}
}
```

단방향 TLS에서는 서버의 자격만 인증한다. 이 다음에는 클라이언트/서버 인증을 알아보자



#### gRPC 서버에서 mTLS 활성화

코드가 기니까 여기서 참고

> https://github.com/grpc-up-and-running/samples/tree/master/ch06/mutual-tls-channel/go

서버측 흐름은

1. 서버 인증서와 키를 통해 X.509 키 쌍을 생성
2. Ca에서 인증서 풀을 생성
3. CA 인증서를 인증서 풀에 추가
4. TLS 자격증명을 생성해 들어오는 모든 연결에 대해 TLS를 활성화함
5. TLS 서버 자격증명을 사용해 새 gRPC 서버 인스턴스를 만듦
6. 생성된 API를 호출해 gRPC 서비스를 새로 작성된 gRPC 서버에 등록
7. 지정된 포트로 TCP 리스너 생성
8. gRPC 서버를 리스너에 바인딩하고  해당 포트에서 수신 메시지를 리스닝

클라이언트측 흐름은

1. 서버 인증서와 키를 통해 X.509 키 쌍을 생성
2. Ca에서 인증서 풀을 생성
3. CA 인증서를 인증서 풀에 추가
4. 전송 자격증명을 연결 옵션으로 추가. ServerName은 인증서의 Common Name과 같아야함
5. 다이얼 옵션을 지정해 서버와 안전한 연결 설정
6. 연결을 지정해 스텁 생성. 스텁에는 서버를 호출하기 위한 원격 메서드가 포함되어있음
7. 모든 처리가 끝나면 연결을 닫음



#### gRPC authentication

사용자 인증 정보를 통신에 지정하는 법을 알아보자. gRPC는 built-in으로 basic authentication을 지원하지 않아서 클라이언트 context에 커스텀 자격증명으로 추가해야 한다.

```go
type basicAuth struct {
		username string
		password string
}

func (b basicAuth) GetRequestMetadata(ctx context.Context, in ...string) (map[string]string, error) {
		auth := b.username + ":" + b.password // username:base64(password)가 basic auth
		enc := base64.StdEncoding.EncodingTostring([]byte(auth))
		return map[string]string {
				"authorization": "Basic " + enc, }, nil
		}
}

// 해당 인증 정보를 전달하고자 채널 보안이 필요하닞 여부를 지정한다. 채널 보안을 사용하는 것이 좋다.
func (b basicAuth) RequireTransportSecurity() bool {
		return true
}
```



```go
// basic auth를 사용하는 gRPC 보안 클라이언트 애플리케이션

package main

import (
	"context"
	"google.golang.org/grpc/credentials"
	"log"
	"path/filepath"
	"time"

	wrapper "github.com/golang/protobuf/ptypes/wrappers"
	pb "github.com/grpc-up-and-running/samples/ch02/productinfo/go/proto"
	"google.golang.org/grpc"
)

const (
	address = "localhost:50051"
	hostname = "localhost"
	crtFile = filepath.Join("ch06", "secure-channel", "certs", "server.crt")
)

func main() {
	creds, err := credentials.NewClientTLSFromFile(crtFile, hostname)
	if err != nil {
		log.Fatalf("failed to load credentials: %v", err)
	}
	opts := []grpc.DialOption{
		// transport credentials.
		grpc.WithTransportCredentials(creds),
	}

	// Set up a connection to the server.
	conn, err := grpc.Dial(address, opts...)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewProductInfoClient(conn)

	// Contact the server and print out its response.
	name := "Sumsung S10"
	description := "Samsung Galaxy S10 is the latest smart phone, launched in February 2019"
	price := float32(700.0)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.AddProduct(ctx, &pb.Product{Name: name, Description: description, Price: price})
	if err != nil {
		log.Fatalf("Could not add product: %v", err)
	}
	log.Printf("Product ID: %s added successfully", r.Value)

	product, err := c.GetProduct(ctx, &wrapper.StringValue{Value: r.Value})
	if err != nil {
		log.Fatalf("Could not get product: %v", err)
	}
	log.Printf("Product: ", product.String())
}
```



```go
// basic auth 확인 가능한 서버

package main

import (
	"context"
	"crypto/tls"
	"errors"
	wrapper "github.com/golang/protobuf/ptypes/wrappers"
	"github.com/google/uuid"
	pb "github.com/grpc-up-and-running/samples/ch02/productinfo/go/proto"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"log"
	"net"
	"path/filepath"
)

const (
	port = ":50051"
	crtFile = filepath.Join("ch06", "secure-channel", "certs", "server.crt")
	keyFile = filepath.Join("ch06", "secure-channel", "certs", "server.key")
)

// server is used to implement ecommerce/product_info.
type server struct {
	productMap map[string]*pb.Product
}

// AddProduct implements ecommerce.AddProduct
func (s *server) AddProduct(ctx context.Context, in *pb.Product) (*wrapper.StringValue, error) {
	out, err := uuid.NewUUID()
	if err != nil {
		log.Fatal(err)
	}
	in.Id = out.String()
	if s.productMap == nil {
		s.productMap = make(map[string]*pb.Product)
	}
	s.productMap[in.Id] = in
	return &wrapper.StringValue{Value: in.Id}, nil
}

// GetProduct implements ecommerce.GetProduct
func (s *server) GetProduct(ctx context.Context, in *wrapper.StringValue) (*pb.Product, error) {
	value, exists := s.productMap[in.Value]
	if exists {
		return value, nil
	}
	return nil, errors.New("Product does not exist for the ID" + in.Value)
}

func main() {
	cert, err := tls.LoadX509KeyPair(crtFile, keyFile)
	if err != nil {
		log.Fatalf("failed to load key pair: %s", err)
	}
	opts := []grpc.ServerOption{
		// Enable TLS for all incoming connections.
		grpc.Creds(credentials.NewServerTLSFromCert(&cert)),
	}

	s := grpc.NewServer(opts...)
	pb.RegisterProductInfoServer(s, &server{})
	// Register reflection service on gRPC server.
	//reflection.Register(s)

	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```



### Oauth2

통신에 토큰을 지정하는 방법을 알아보자. 이 예제에선 인증 서버가 없어서 토큰 값을 임의의 문자열로 하드코딩한다.

```go
클라이언트
package main

import (
	"context"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/credentials/oauth"
	"log"
	"path/filepath"
	"time"

	wrapper "github.com/golang/protobuf/ptypes/wrappers"
	pb "github.com/grpc-up-and-running/samples/ch02/productinfo/go/product_info"
	"golang.org/x/oauth2"
	"google.golang.org/grpc"
)

const (
	address = "localhost:50051"
	hostname = "localhost"
	crtFile = filepath.Join("ch06", "secure-channel", "certs", "server.crt")
)

func main() {
	// Set up the credentials for the connection.
	perRPC := oauth.NewOauthAccess(fetchToken())

	creds, err := credentials.NewClientTLSFromFile(crtFile, hostname)
	if err != nil {
		log.Fatalf("failed to load credentials: %v", err)
	}
  
  // 동일한 연결의 모든 RPC 호출에 단일 OAuth 토큰을 적용하도록 gRPC 다이얼 옵션을 구성한다.
  // 통신마다 OAuth 토큰을 적용하려면 CallOption을 사용해 gRPC를 구성하면 된다.
	opts := []grpc.DialOption{
		grpc.WithPerRPCCredentials(perRPC),
		// transport credentials.
		grpc.WithTransportCredentials(creds),
	}

	// Set up a connection to the server.
	conn, err := grpc.Dial(address, opts...)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewProductInfoClient(conn)

	// Contact the server and print out its response.
	name := "Sumsung S10"
	description := "Samsung Galaxy S10 is the latest smart phone, launched in February 2019"
	price := float32(700.0)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.AddProduct(ctx, &pb.Product{Name: name, Description: description, Price: price})
	if err != nil {
		log.Fatalf("Could not add product: %v", err)
	}
	log.Printf("Product ID: %s added successfully", r.Value)

	product, err := c.GetProduct(ctx, &wrapper.StringValue{Value: r.Value})
	if err != nil {
		log.Fatalf("Could not get product: %v", err)
	}
	log.Printf("Product: ", product.String())
}

func fetchToken() *oauth2.Token {
	return &oauth2.Token{
		AccessToken: "some-secret-token",
	}
}
```



서버

```go

package main

import (
	"context"
	"crypto/tls"
	"errors"
	wrapper "github.com/golang/protobuf/ptypes/wrappers"
	"github.com/google/uuid"
	pb "github.com/grpc-up-and-running/samples/ch02/productinfo/go/product_info"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
	"log"
	"net"
	"path/filepath"
	"strings"
)

// server is used to implement ecommerce/product_info.
type server struct {
	productMap map[string]*pb.Product
}

var (
	port = ":50051"
	crtFile = filepath.Join("ch06", "secure-channel", "certs", "server.crt")
	keyFile = filepath.Join("ch06", "secure-channel", "certs", "server.key")
	errMissingMetadata = status.Errorf(codes.InvalidArgument, "missing metadata")
	errInvalidToken    = status.Errorf(codes.Unauthenticated, "invalid token")
)

// AddProduct implements ecommerce.AddProduct
func (s *server) AddProduct(ctx context.Context, in *pb.Product) (*wrapper.StringValue, error) {
	out, err := uuid.NewUUID()
	if err != nil {
		log.Fatal(err)
	}
	in.Id = out.String()
	if s.productMap == nil {
		s.productMap = make(map[string]*pb.Product)
	}
	s.productMap[in.Id] = in
	return &wrapper.StringValue{Value: in.Id}, nil
}

// GetProduct implements ecommerce.GetProduct
func (s *server) GetProduct(ctx context.Context, in *wrapper.StringValue) (*pb.Product, error) {
	value, exists := s.productMap[in.Value]
	if exists {
		return value, nil
	}
	return nil, errors.New("Product does not exist for the ID" + in.Value)
}

func main() {
	cert, err := tls.LoadX509KeyPair(crtFile, keyFile)
	if err != nil {
		log.Fatalf("failed to load key pair: %s", err)
	}
  
  // TLS 서버 인증서와 함께 grpc.ServerOption을 추가한다. grpc.UnaryInterceptor 함수를 사용해 클라이언트의 모든 요청을 가로채는 인터셉터를 추가한다.
	opts := []grpc.ServerOption{
		// Enable TLS for all incoming connections.
		grpc.Creds(credentials.NewServerTLSFromCert(&cert)),

		grpc.UnaryInterceptor(ensureValidToken),
	}

	s := grpc.NewServer(opts...)
	pb.RegisterProductInfoServer(s, &server{})
	// Register reflection service on gRPC server.
	//reflection.Register(s)

	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

// valid validates the authorization.
func valid(authorization []string) bool {
	if len(authorization) < 1 {
		return false
	}
	token := strings.TrimPrefix(authorization[0], "Bearer ")
	// Perform the token validation here. For the sake of this example, the code
	// here forgoes any of the usual OAuth2 token validation and instead checks
	// for a token matching an arbitrary string.
	return token == "some-secret-token"
}

// ensureValidToken ensures a valid token exists within a request's metadata. If
// the token is missing or invalid, the interceptor blocks execution of the
// handler and returns an error. Otherwise, the interceptor invokes the unary
// handler.
func ensureValidToken(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	md, ok := metadata.FromIncomingContext(ctx)
  // 토큰의 유효성을 체크한다.
	if !ok {
		return nil, errMissingMetadata
	}
	// The keys within metadata.MD are normalized to lowercase.
	// See: https://godoc.org/google.golang.org/grpc/metadata#New
	if !valid(md["authorization"]) {
		return nil, errInvalidToken
	}
	// Continue execution of handler after ensuring a valid token.
	return handler(ctx, req)
}
```





gRPC에는 두 가지 유형의 자격증명이 지원된다

1. 채널 자격증명 : TLS 등 채널에 지정되고
2. 호출 자격증명 : Oauth2.0, basic auth와 같은 것은 호출에 지정된다.

