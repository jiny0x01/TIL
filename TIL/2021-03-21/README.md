# TIL

## 오늘 공부한 컨셉

## 상세내용



## gRPC 통신 패턴에서의 메시지를 흐름 이해

gRPC가 지원하는 4가지 통신 패턴. 각 패턴

+ 단순 RPC
  + gRPC 서버 / gRPC 클라이언트 통신에서 항상 단일 요청과 단일 응답만 있음
  
  + 클라이언트
  
    + half-close the connection을 하려면 요청 메시지 끝에 EOS 플래그를 추가
  
      + > half-close the connection : 클라이언트 측에서 연결을 닫아 더 이상 서버로 메시지를 보낼 수 없지만 서버에서 들어오는 메시지는 수신할 수 있는 상태
  
  + 서버
  
    + 전체 메시지를 받고 메시지를 만듬
    + 상태 정보와 트레일러 헤더를 보내서 통신 종료
  
+ 서버 스트리밍 RPC
  
  + 여러 메시지를 클라이언트에게 전송
  + 상태정보와 트레일러를 보내서 끝남
  
+ 클라이언트 스트리밍 RPC

  + 서버 스트리밍과 반대되는 개념
  + 헤더 프레임을 전송해 서버와 연결 설정
  + length-prefix 메시지를 데이터 프레임으로 서버에 보냄
  + EOS를 보내서 half-close the connection함

+ 양방향 스트림 RPC

  + 연결되면 클라이언트/서버는 상대방이 끝날 떄 까지 기다리지 않고 length-prefix 메시지를 보냄
  + 먼저 종료하면 메시지 못보내는 걸로 끝

### gRPC 구현 아키텍처

gRPC core layer를 바탕으로 상위 레이어의 모든 네트워크 작업을 추상화함

core layer는 핵심 기능의 확장을 제공함

+ 통신 보안용 인증 필터 (authentication filters)
+ deadline filter



### 



