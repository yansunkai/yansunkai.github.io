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
前面是监听端口和服务注册和普通rpc没什么区别，主要是s.Serve开始看。
```go
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

这里面主要是一个newHTTP2Transport构造一个http2的ServerTransport实现了ServerTransport接口。  
```go
func (s *Server) Serve(lis net.Listener) error {
    ...

	for {
		rawConn, err := lis.Accept()
		...
		go func() {
			s.handleRawConn(rawConn)
			s.serveWG.Done()
		}()
	}
}
```
```go
func (s *Server) handleRawConn(rawConn net.Conn) {
	rawConn.SetDeadline(time.Now().Add(s.opts.connectionTimeout))
	conn, authInfo, err := s.useTransportAuthenticator(rawConn)
	...

	// Finish handshaking (HTTP2)
	st := s.newHTTP2Transport(conn, authInfo)
	if st == nil {
		return
	}

	rawConn.SetDeadline(time.Time{})
	if !s.addConn(st) {
		return
	}
	go func() {
		s.serveStreams(st)
		s.removeConn(st)
	}()
}
```
```go
type ServerTransport interface {
	// HandleStreams receives incoming streams using the given handler.
	HandleStreams(func(*Stream), func(context.Context, string) context.Context)

	// WriteHeader sends the header metadata for the given stream.
	// WriteHeader may not be called on all streams.
	WriteHeader(s *Stream, md metadata.MD) error

	// Write sends the data for the given stream.
	// Write may not be called on all streams.
	Write(s *Stream, hdr []byte, data []byte, opts *Options) error

	// WriteStatus sends the status of a stream to the client.  WriteStatus is
	// the final call made on a stream and always occurs.
	WriteStatus(s *Stream, st *status.Status) error

	// Close tears down the transport. Once it is called, the transport
	// should not be accessed any more. All the pending streams and their
	// handlers will be terminated asynchronously.
	Close() error

	// RemoteAddr returns the remote network address.
	RemoteAddr() net.Addr

	// Drain notifies the client this ServerTransport stops accepting new RPCs.
	Drain()

	// IncrMsgSent increments the number of message sent through this transport.
	IncrMsgSent()

	// IncrMsgRecv increments the number of message received through this transport.
	IncrMsgRecv()
}
```
newHTTP2Server函数非常大，它主要就是建立http2的连接过程，发送settings帧。
http2和http1.*主要区别在一个tcp连接上可以双向发送多个流，每个流都是帧来分块传输，每个帧都有标记来区分属于哪个流。这样就可以双向全双工通信。
```go
func newHTTP2Server(conn net.Conn, config *ServerConfig) (_ ServerTransport, err error) {
	...
	framer := newFramer(conn, writeBufSize, readBufSize, maxHeaderListSize)
	...
	if err := framer.fr.WriteSettings(isettings...); err != nil {
		return nil, connectionErrorf(false, err, "transport: %v", err)
	}
	// Adjust the connection flow control window if needed.
	...
	t.framer.writer.Flush()
	...

	frame, err := t.framer.fr.ReadFrame()
	if err == io.EOF || err == io.ErrUnexpectedEOF {
		return nil, err
	}
	if err != nil {
		return nil, connectionErrorf(false, err, "transport: http2Server.HandleStreams failed to read initial settings frame: %v", err)
	}
	atomic.StoreUint32(&t.activity, 1)
	sf, ok := frame.(*http2.SettingsFrame)
	if !ok {
		return nil, connectionErrorf(false, nil, "transport: http2Server.HandleStreams saw invalid preface type %T from client", frame)
	}
	t.handleSettings(sf)

	go func() {
		t.loopy = newLoopyWriter(serverSide, t.framer, t.controlBuf, t.bdpEst)
		t.loopy.ssGoAwayHandler = t.outgoingGoAwayHandler
		if err := t.loopy.run(); err != nil {
			errorf("transport: loopyWriter.run returning. Err: %v", err)
		}
		t.conn.Close()
		close(t.writerDone)
	}()
	go t.keepalive()
	return t, nil
}
```

建立连接以后就通过HandleStreams来处理消息。从frame中读取帧并按照帧类型处理。
每个stream都是从收到HEADER帧开始创建和处理的，根据header里带的fields去处理decodeState，新建Stream。
```go
func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
	defer close(t.readerDone)
	for {
		frame, err := t.framer.fr.ReadFrame()
		atomic.StoreUint32(&t.activity, 1)
		if err != nil {
			if se, ok := err.(http2.StreamError); ok {
				warningf("transport: http2Server.HandleStreams encountered http2.StreamError: %v", se)
				t.mu.Lock()
				s := t.activeStreams[se.StreamID]
				t.mu.Unlock()
				if s != nil {
					t.closeStream(s, true, se.Code, false)
				} else {
					t.controlBuf.put(&cleanupStream{
						streamID: se.StreamID,
						rst:      true,
						rstCode:  se.Code,
						onWrite:  func() {},
					})
				}
				continue
			}
			if err == io.EOF || err == io.ErrUnexpectedEOF {
				t.Close()
				return
			}
			warningf("transport: http2Server.HandleStreams failed to read frame: %v", err)
			t.Close()
			return
		}
		switch frame := frame.(type) {
		case *http2.MetaHeadersFrame:
			if t.operateHeaders(frame, handle, traceCtx) {
				t.Close()
				break
			}
		case *http2.DataFrame:
			t.handleData(frame)
		case *http2.RSTStreamFrame:
			t.handleRSTStream(frame)
		case *http2.SettingsFrame:
			t.handleSettings(frame)
		case *http2.PingFrame:
			t.handlePing(frame)
		case *http2.WindowUpdateFrame:
			t.handleWindowUpdate(frame)
		case *http2.GoAwayFrame:
			// TODO: Handle GoAway from the client appropriately.
		default:
			errorf("transport: http2Server.HandleStreams found unhandled frame type %v.", frame)
		}
	}
}
```
operateHeaders函数首先通过StreamID来区分stream，然后把stream交给handle处理。
```go
func (t *http2Server) operateHeaders(frame *http2.MetaHeadersFrame, handle func(*Stream), traceCtx func(context.Context, string) context.Context) (fatal bool) {
	streamID := frame.Header().StreamID
	...
	s := &Stream{
		id:             streamID,
		st:             t,
		buf:            buf,
		fc:             &inFlow{limit: uint32(t.initialWindowSize)},
		recvCompress:   state.data.encoding,
		method:         state.data.method,
		contentSubtype: state.data.contentSubtype,
	}
	...
	handle(s)
	return false
}
```
handleStream就是从stream读出method和service名称，然后获取对应的method进行处理。  
这里分成了unary和stream类型的rpc。
```go
func (s *Server) serveStreams(st transport.ServerTransport) {
	defer st.Close()
	var wg sync.WaitGroup
	st.HandleStreams(func(stream *transport.Stream) {
		wg.Add(1)
		go func() {
			defer wg.Done()
			s.handleStream(st, stream, s.traceInfo(st, stream))
		}()
	}, func(ctx context.Context, method string) context.Context {
		if !EnableTracing {
			return ctx
		}
		tr := trace.New("grpc.Recv."+methodFamily(method), method)
		return trace.NewContext(ctx, tr)
	})
	wg.Wait()
}
```
```go
func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo) {
	sm := stream.Method()
    ...
	service := sm[:pos]
	method := sm[pos+1:]

	srv, knownService := s.m[service]
	if knownService {
		if md, ok := srv.md[method]; ok {
			s.processUnaryRPC(t, stream, srv, md, trInfo)
			return
		}
		if sd, ok := srv.sd[method]; ok {
			s.processStreamingRPC(t, stream, srv, sd, trInfo)
			return
		}
	}
```
processUnaryRPC就是冲stream中读取message length，然后读取data，交给md.Handler处理，最后sendResponse。 
```go
func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, md *MethodDesc, trInfo *traceInfo) (err error) {
	...
	d, err := recvAndDecompress(&parser{r: stream}, stream, dc, s.opts.maxReceiveMessageSize, payInfo, decomp)
	...
	df := func(v interface{}) error {
		if err := s.getCodec(stream.ContentSubtype()).Unmarshal(d, v); err != nil {
			return status.Errorf(codes.Internal, "grpc: error unmarshalling request: %v", err)
		}
		if sh != nil {
			sh.HandleRPC(stream.Context(), &stats.InPayload{
				RecvTime: time.Now(),
				Payload:  v,
				Data:     d,
				Length:   len(d),
			})
		}
		if binlog != nil {
			binlog.Log(&binarylog.ClientMessage{
				Message: d,
			})
		}
		if trInfo != nil {
			trInfo.tr.LazyLog(&payload{sent: false, msg: v}, true)
		}
		return nil
	}
	ctx := NewContextWithServerTransportStream(stream.Context(), stream)
	reply, appErr := md.Handler(srv.server, ctx, df, s.opts.unaryInt)
	
    ...
	if err := s.sendResponse(t, stream, reply, cp, opts, comp); err != nil {
	...
	return err
}
```
processStreamingRPC用来支持grpc的流式请求和响应的。  
stream模式一般是server注册一个StreamHandler来循环处理stream读进来的数据。  
stream的读写通过serverStream里面的SendMsg和RecvMsg函数实现。  
```go
func (s *Server) processStreamingRPC(t transport.ServerTransport, stream *transport.Stream, srv *service, sd *StreamDesc, trInfo *traceInfo) (err error) {
	...
	ss := &serverStream{
		ctx:                   ctx,
		t:                     t,
		s:                     stream,
		p:                     &parser{r: stream},
		codec:                 s.getCodec(stream.ContentSubtype()),
		maxReceiveMessageSize: s.opts.maxReceiveMessageSize,
		maxSendMessageSize:    s.opts.maxSendMessageSize,
		trInfo:                trInfo,
		statsHandler:          sh,
	}

	...
	if s.opts.streamInt == nil {
		appErr = sd.Handler(server, ss)
	...
	err = t.WriteStatus(ss.s, status.New(codes.OK, ""))
	if ss.binlog != nil {
		ss.binlog.Log(&binarylog.ServerTrailer{
			Trailer: ss.s.Trailer(),
			Err:     appErr,
		})
	}
	return err
}
```
```go
type StreamHandler func(srv interface{}, stream ServerStream) error

// StreamDesc represents a streaming RPC service's method specification.
type StreamDesc struct {
	StreamName string
	Handler    StreamHandler

	// At least one of these is true.
	ServerStreams bool
	ClientStreams bool
}
```

写的好累，后面有时间再看看proto的编解码吧。