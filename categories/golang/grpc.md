# GRPC
在golang语言中，通过json这种通用编码来支持rpc跨语言调用，但是json性能不高，并且使用不方便。  
所以我们会使用protobuf来进行数据编解码，并且会自动生成服务端和客户端的代码使用非常方便。  

## 使用grpc
首先编写helloworld.proto文件， 主要是定义接口和数据类型，其实就是rpc的method，request，reply。  
```go
syntax = "proto3";

option java_multiple_files = true;
option java_package = "io.grpc.examples.helloworld";
option java_outer_classname = "HelloWorldProto";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```
接着使用grpc protobuf工具生成对应语言的函数库。  
protobuf支持生成各种语言的pb文件  
protoc安装参考： [link](http://doc.oschina.net/grpc?t=56831)  
```go
protoc --go_out=. helloworld.proto
```
helloworld.pb.go  
![grpc1](../../image/golang/grpc1.png)

然后就可以通过使用helloworld.pb.go编写服务端和客户端。

grpc server
```go
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.Name)
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```
grpc client
```go
func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	name := defaultName
	if len(os.Args) > 1 {
		name = os.Args[1]
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.Message)
}
```

## http2实现
gRPC主要基于HTTP2协议实现的。

