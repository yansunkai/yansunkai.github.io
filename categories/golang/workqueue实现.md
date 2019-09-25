# workQueue实现
最近在看kubernetes源码，其中controller-manager中，用到了workqueue来对informer的api事件和控制器循环进行解耦，这在平时很多业务中可以借鉴。  
其中kubernetes的level trigger设计在workqueue实现中就有体现。   

level trigger（条件触发）： 是指每当状态变化时发生一个io事件，关心数据的最终状态，而不需要担心错过数据的变化过程。  
edge trigger（边缘触发）： 是只要满足条件就发生一个io事件，更关心事件本身的curd的动作。  

代码路径：D:\working_dir\kubernetes\staging\src\k8s.io\client-go\util\workqueue\queue.go
queue的数据结构：
```go
type Type struct {
	//存放队列中的数据
	queue []t

	//存放所有需要处理的数据，同时防止queue中数据重复
	dirty set

	//正在被处理的数据，同时防止同个时间段处理同个API对象
	processing set

	cond *sync.Cond

	shuttingDown bool

	metrics queueMetrics

	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.Clock
}
```

这里set是通过map的key不会是重复的特性来实现。value用空的struct不占空间。
```go
type empty struct{}
type t interface{}
type set map[t]empty

func (s set) has(item t) bool {
	_, exists := s[item]
	return exists
}

func (s set) insert(item t) {
	s[item] = empty{}
}

func (s set) delete(item t) {
	delete(s, item)
}
```

往队列添加数据的过程首先如果数据存在dirty中则退出（level trigger设计），如果不存在把item添加到dirty，然后判断当前item是否正在处理中，
如果正在处理中等处理完再添加到queue（防止同个时间段处理同个API对象），否则直接添加入queue。     
```go
// Add marks item as needing processing.
func (q *Type) Add(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	if q.shuttingDown {
		return
	}
	if q.dirty.has(item) {
		return
	}

	q.metrics.add(item)

	q.dirty.insert(item)
	if q.processing.has(item) {
		return
	}

	q.queue = append(q.queue, item)
	q.cond.Signal()
}
```

获取数据时，如果queue为空则等待，否则获取数据出队，然后往processing set中添加，表明当前item正在处理，并删除dirty。
```go
func (q *Type) Get() (item interface{}, shutdown bool) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	for len(q.queue) == 0 && !q.shuttingDown {
		q.cond.Wait()
	}
	if len(q.queue) == 0 {
		// We must be shutting down.
		return nil, true
	}

	item, q.queue = q.queue[0], q.queue[1:]

	q.metrics.get(item)

	q.processing.insert(item)
	q.dirty.delete(item)

	return item, false
}
```

Done表明item处理完成，则从processing set中删除，此时如果dirty中有item表明这个item是processing时被添加进来的，后续还需要处理所以入队。
```go
func (q *Type) Done(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.metrics.done(item)

	q.processing.delete(item)
	if q.dirty.has(item) {
		q.queue = append(q.queue, item)
		q.cond.Signal()
	}
}
```

## sync.cond
上面多个goroutine之间的同步用到了条件变量sync.cond。  
sync.cond主要有三个函数： 
Wait() 阻塞操作
Signal() 唤醒一个协程
Broadcast() 唤醒所有协程

使用wait函数一定要注意：使用前要加锁，返回后要解锁。   
主要是为了把外面条件判断和添加阻塞队列变为一个原子操作  
```go
func (c *Cond) Wait() {
    c.checker.check()
    t := runtime_notifyListAdd(&c.notify) // 添加阻塞列表
    c.L.Unlock() // 函数调用前已经加锁了，在添加阻塞队列后就可以撤销锁了
    runtime_notifyListWait(&c.notify, t) // 进行阻塞
    c.L.Lock() // 外层还有个解锁操作，加把锁
}
```

```go
func (c *Cond) Signal() {
    c.checker.check()
    runtime_notifyListNotifyOne(&c.notify) 
    // 唤醒一个
}

func (c *Cond) Broadcast() {
    c.checker.check()
    runtime_notifyListNotifyAll(&c.notify)
    // 唤醒多个
}
```