# TIL

## 오늘 공부한 컨셉

## 상세내용



protobuf 설치

> go get -u github.com/golang/protobuf

protoc 플러그인 설치

> go get -u github.com/golang/protobuf/protoc-gen-go

proto 컴파일

> protoc -I <proto 디렉터리> <proto 디렉터리>/<.proto 파일> --go_out=plugins=grpc:<저장경로>





### 서버 스트리밍 RPC

단순 RPC에서는 gRPC 서버와 gRPC 클라이언트 간의 통신에서 항상 단일 요청과 단일 응답을 가짐

서버 스트리밍 RPC 에서는 서버가 클라이언트의 요청 메시지를 받은 후 일련의 응답을 다시 보냄. 이러한 일련의 응답을 스트림이라함.

서버는 응답을 보내고 서버의 상태 정보를 후행 메타데이터로 클라이언트에게 전달해 스트림의 끝을 알림

