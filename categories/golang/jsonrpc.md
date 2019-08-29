# jsonRpcgolang默认的net/rpc是通过http传输gob编码的，不支持跨语言调用。  
jsonRpc是json编码的支持跨语言调用。

### jsonRpc Server
jsonRpc的server是直接拿到tcp的conn，然后给jsonrpc.ServerConn处理，而不是通过http的handle来处理。
```go
func main() {
	rpc.Register(new(Arith)) // 注册rpc服务

	lis, err := net.Listen("tcp", "127.0.0.1:8096")
	if err != nil {
		log.Fatalln("fatal error: ", err)
	}

	fmt.Fprintf(os.Stdout, "%s", "start connection")

	for {
		conn, err := lis.Accept() // 接收客户端连接请求
		if err != nil {
			continue
		}

		go func(conn net.Conn) { // 并发处理客户端请求
			fmt.Fprintf(os.Stdout, "%s", "new client in coming\n")
			jsonrpc.ServeConn(conn)
		}(conn)
	}
}
```

可以看到这里的http的rpc唯一区别就是序列化反序列化用到了serverCodec，而http的是用gobServerCodec。  
具体的编解码这里先不做介绍
```go
func ServeConn(conn io.ReadWriteCloser) {
	rpc.ServeCodec(NewServerCodec(conn))
}
```
```go
func NewServerCodec(conn io.ReadWriteCloser) rpc.ServerCodec {
	return &serverCodec{
		dec:     json.NewDecoder(conn),
		enc:     json.NewEncoder(conn),
		c:       conn,
		pending: make(map[uint64]*json.RawMessage),
	}
}
```
```go
func (server *Server) ServeCodec(codec ServerCodec) {
	sending := new(sync.Mutex)
	wg := new(sync.WaitGroup)
	for {
		service, mtype, req, argv, replyv, keepReading, err := server.readRequest(codec)
		if err != nil {
			if debugLog && err != io.EOF {
				log.Println("rpc:", err)
			}
			if !keepReading {
				break
			}
			// send a response if we actually managed to read a header.
			if req != nil {
				server.sendResponse(sending, req, invalidRequest, codec, err.Error())
				server.freeRequest(req)
			}
			continue
		}
		wg.Add(1)
		go service.call(server, sending, wg, mtype, req, argv, replyv, codec)
	}
	// We've seen that there are no more requests.
	// Wait for responses to be sent before closing codec.
	wg.Wait()
	codec.Close()
}
```

http的dial函数建立tcp连接后会发送一个http的connect请求，然后new一个gob的Codec。   
而jsonrpc建立tcp连接后直接通过conn创建json的clientCodec。
### jsonRpc Client
```go
func main() {
	conn, err := jsonrpc.Dial("tcp", "127.0.0.1:8096")
	if err != nil {
		log.Fatalln("dailing error: ", err)
	}

	req := ArithRequest{9, 2}
	var res ArithResponse

	err = conn.Call("Arith.Multiply", req, &res) // 乘法运算
	if err != nil {
		log.Fatalln("arith error: ", err)
	}
	fmt.Printf("%d * %d = %d\n", req.A, req.B, res.Pro)

	err = conn.Call("Arith.Divide", req, &res)
	if err != nil {
		log.Fatalln("arith error: ", err)
	}
	fmt.Printf("%d / %d, quo is %d, rem is %d\n", req.A, req.B, res.Quo, res.Rem)
}
```
```go
// Dial connects to a JSON-RPC server at the specified network address.
func Dial(network, address string) (*rpc.Client, error) {
	conn, err := net.Dial(network, address)
	if err != nil {
		return nil, err
	}
	return NewClient(conn), err
}
```
```go
func NewClient(conn io.ReadWriteCloser) *rpc.Client {
	return rpc.NewClientWithCodec(NewClientCodec(conn))
}
```

接下来的处理流程和http rpc是一样的，都是input goroutine处理rpcresponse，send来发送rpc request，seq来记录序号保证并发。
```go
func NewClientWithCodec(codec ClientCodec) *Client {
	client := &Client{
		codec:   codec,
		pending: make(map[uint64]*Call),
	}
	go client.input()
	return client
}
```
```go
func (client *Client) send(call *Call) {
	client.reqMutex.Lock()
	defer client.reqMutex.Unlock()

	// Register this call.
	client.mutex.Lock()
	if client.shutdown || client.closing {
		client.mutex.Unlock()
		call.Error = ErrShutdown
		call.done()
		return
	}
	seq := client.seq
	client.seq++
	client.pending[seq] = call
	client.mutex.Unlock()

	// Encode and send the request.
	client.request.Seq = seq
	client.request.ServiceMethod = call.ServiceMethod
	err := client.codec.WriteRequest(&client.request, call.Args)
	if err != nil {
		client.mutex.Lock()
		call = client.pending[seq]
		delete(client.pending, seq)
		client.mutex.Unlock()
		if call != nil {
			call.Error = err
			call.done()
		}
	}
}
```