# channel实现

代码路径：./src/runtime/chan.go

```go

type hchan struct {
	qcount   uint           // total data in the queue   队列中的当前数据的个数
	dataqsiz uint           // size of the circular queue channel的大小
	buf      unsafe.Pointer // points to an array of dataqsiz elements  数据缓冲区，存放数据的环形数组
	elemsize uint16         // channel 中数据类型的大小 （单个元素的大小）
	closed   uint32         // 表示 channel 是否关闭的标识位
	elemtype *_type // element type  队列中的元素类型
    // send 和 recieve 的索引，用于实现环形数组队列
	sendx    uint   // send index    当前发送元素的索引
	recvx    uint   // receive index    当前接收元素的索引
	recvq    waitq  // list of recv waiters    接收等待队列；由 recv 行为（也就是 <-ch）阻塞在 channel 上的 goroutine 队列
	sendq    waitq  // list of send waiters    发送等待队列；由 send 行为 (也就是 ch<-) 阻塞在 channel 上的 goroutine 队列
 
	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
    // lock保护hchan中的所有字段，以及此通道上阻塞的sudoG中的几个字段。  
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
    // 保持此锁定时不要更改另一个G的状态（特别是，没有准备好G），因为这可能会因堆栈收缩而死锁。
	lock mutex
}
 
 
/**
发送及接收队列的结构体
等待队列的链表实现
*/
type waitq struct {
	first *sudog
	last  *sudog
}
```

有上述的结构体我们大致可以看出channel其实就是由一个环形数组实现的队列，用于存储消息元素；  
两个链表实现的 goroutine 等待队列，用于存储阻塞在 recv 和 send 操作上的 goroutine；  
一个互斥锁，用于各个属性变动的同步，只不过这个锁是一个轻量级锁。  
其中 recvq 是读操作阻塞在 channel 的 goroutine 列表，  
sendq 是写操作阻塞在 channel 的 goroutine 列表。  
列表的实现是 sudog，其实就是一个对 g 的结构的封装。  
和select类似，hchan其实只是channel的头部。  
头部后面的一段内存连续的数组将作为channel的缓冲区，即用于存放channel数据的环形队列。   
qcount 和 dataqsiz 分别描述了缓冲区当前使用量【len】和容量【cap】。若channel是无缓冲的，则size是0，就没有这个环形队列了。   


## send 有以下四种情况：【都是对不为nil的chan的情况】  

向已经close的chan写数据，抛panic。  
有 goroutine 阻塞在 channel recv 队列上，此时缓存队列( hchan.buf)为空(即缓冲区内无元素)，直接将消息发送给 reciever goroutine,只产生一次复制  
当 channel 缓存队列( hchan.buf )有剩余空间时，将数据放到队列里，等待接收，接收后总共产生两次复制  
当 channel 缓存队列( hchan.buf )已满时，将当前 goroutine 加入 send 队列并阻塞。  

## receive 有以下四种情况：【都是对不为nil的chan的情况】

从已经 close 且为空的 channel recv 数据，返回空值  
当 send 队列不为空，分两种情况：【一】缓存队列为空，直接从 send 队列的sender中接收数据 元素；【二】缓存队列不为空，此时只有可能是缓存队列已满，从队列头取出元素，并唤醒 sender 将元素写入缓存队列尾部。由于为环形队列，因此，队列满时只需要将队列头复制给 reciever，同时将 sender 元素复制到该位置，并移动队列头尾索引，不需要移动队列元素。【这就是为什么使用环形队列的原因】  
缓存队列不为空，直接从队列取队头元素，移动头索引。  
缓存队列为空，将 goroutine 加入 recv 队列，并阻塞。  

## close channel 的工作
 将 c.closed 设置为 1。  
 唤醒 recvq 队列里面的阻塞 goroutine  
 唤醒 sendq 队列里面的阻塞 goroutine  