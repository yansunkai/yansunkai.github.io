# workpool实现

fasthttp包中使用协程池，通过复用goroutine，减轻处理大并发时runtime 调度 goroutine 的压力，从而提高处理能力。  

## workerPool的struct结构体，它通过workerChan来管理每个goroutine，当worker空闲时，workerChan.ch没有数据，对应的goroutine堵塞等待，
当有connect连接进来，通过获取workerChan，然后往chan net.Conn发送数据connect连接执行ServeHandler。  
```go
type workerPool struct {
	//每个协程要执行的任务，也就是http的serveHTTP处理函数。	
	WorkerFunc ServeHandler

	MaxWorkersCount int

	LogAllErrors bool

    //最大空闲时间
	MaxIdleWorkerDuration time.Duration

	Logger LoggerworkerPool

	lock         sync.Mutex
	workersCount int
	mustStop     bool

    //可用的worker列表
	ready []*workerChan

	stopCh chan struct{}

    //对象池
	workerChanPool sync.Pool

	connState func(net.Conn, ConnState)
}
```

 ```go
type workerChan struct {
	lastUseTime time.Time
	ch          chan net.Conn
}
```

## start
start主要每MaxIdleWorkerDuration时间执行clean操作，清除超过空闲事件的worker。
```go
func (wp *workerPool) Start() {
	if wp.stopCh != nil {
		panic("BUG: workerPool already started")
	}
	wp.stopCh = make(chan struct{})
	stopCh := wp.stopCh
	wp.workerChanPool.New = func() interface{} {
		return &workerChan{
			ch: make(chan net.Conn, workerChanCap),
		}
	}
	go func() {
		var scratch []*workerChan
		for {
			wp.clean(&scratch)
			select {
			case <-stopCh:
				return
			default:
				time.Sleep(wp.getMaxIdleWorkerDuration())
			}
		}
	}()
}
```

## clean
通过二分查找找出可清楚的最小lastUseTime，然后删除左边的队列。是=因为ready是FILO先进后出的，所以ready中越往后的空闲时间最短。  
```go
func (wp *workerPool) clean(scratch *[]*workerChan) {
	maxIdleWorkerDuration := wp.getMaxIdleWorkerDuration()

	// Clean least recently used workers if they didn't serve connections
	// for more than maxIdleWorkerDuration.
	criticalTime := time.Now().Add(-maxIdleWorkerDuration)

	wp.lock.Lock()
	ready := wp.ready
	n := len(ready)

	// Use binary-search algorithm to find out the index of the least recently worker which can be cleaned up.
	l, r, mid := 0, n-1, 0
	for l <= r {
		mid = (l + r) / 2
		if criticalTime.After(wp.ready[mid].lastUseTime) {
			l = mid + 1
		} else {
			r = mid - 1
		}
	}
	i := r
	if i == -1 {
		wp.lock.Unlock()
		return
	}

	*scratch = append((*scratch)[:0], ready[:i+1]...)
	m := copy(ready, ready[i+1:])
	for i = m; i < n; i++ {
		ready[i] = nil
	}
	wp.ready = ready[:m]
	wp.lock.Unlock()

	// Notify obsolete workers to stop.
	// This notification must be outside the wp.lock, since ch.ch
	// may be blocking and may consume a lot of time if many workers
	// are located on non-local CPUs.
	tmp := *scratch
	for i := range tmp {
		tmp[i].ch <- nil
		tmp[i] = nil
	}
}
```

## Serve
serve就是workerpool的处理函数，首先通过getCh获取空闲的workerChan然后把connect发送给ch来执行WorkerFunc  
```go
func (wp *workerPool) Serve(c net.Conn) bool {
	ch := wp.getCh()
	if ch == nil {
		return false
	}
	ch.ch <- c
	return true
}
```

## getCh
首先判断如果ready中没有可使用的workerChan表明没有空闲goroutine，需要创建goroutine。如果有空闲goroutine，从ready的最后面获取workerChan
如果是新创建的workerChan启动workerFunc，把对象put到workerChanPool。

```go
func (wp *workerPool) getCh() *workerChan {
	var ch *workerChan
	createWorker := false

	wp.lock.Lock()
	ready := wp.ready
	n := len(ready) - 1
	if n < 0 {
		if wp.workersCount < wp.MaxWorkersCount {
			createWorker = true
			wp.workersCount++
		}
	} else {
		ch = ready[n]
		ready[n] = nil
		wp.ready = ready[:n]
	}
	wp.lock.Unlock()

	if ch == nil {
		if !createWorker {
			return nil
		}
		vch := wp.workerChanPool.Get()
		ch = vch.(*workerChan)
		go func() {
			wp.workerFunc(ch)
			wp.workerChanPool.Put(vch)
		}()
	}
	return ch
}
```

## workerFunc
workerFunc就是从chan中获取connect，然后执行WorkerFunc。执行完成后执行release然后goroutine阻塞在获取ch.ch处待下次使用。  
前面如果获取到的为nil值表明clean这个goroutine所以退出循环。  

```go
func (wp *workerPool) workerFunc(ch *workerChan) {
	var c net.Conn

	var err error
	for c = range ch.ch {
		if c == nil {
			break
		}

		if err = wp.WorkerFunc(c); err != nil && err != errHijacked {
			errStr := err.Error()
			if wp.LogAllErrors || !(strings.Contains(errStr, "broken pipe") ||
				strings.Contains(errStr, "reset by peer") ||
				strings.Contains(errStr, "request headers: small read buffer") ||
				strings.Contains(errStr, "unexpected EOF") ||
				strings.Contains(errStr, "i/o timeout")) {
				wp.Logger.Printf("error when serving connection %q<->%q: %s", c.LocalAddr(), c.RemoteAddr(), err)
			}
		}
		if err == errHijacked {
			wp.connState(c, StateHijacked)
		} else {
			_ = c.Close()
			wp.connState(c, StateClosed)
		}
		c = nil

		if !wp.release(ch) {
			break
		}
	}

	wp.lock.Lock()
	wp.workersCount--
	wp.lock.Unlock()
}
```

## release
release主要是更新lastUseTime，然后把workerChan加入ready队列。
```go
func (wp *workerPool) release(ch *workerChan) bool {
	ch.lastUseTime = time.Now()
	wp.lock.Lock()
	if wp.mustStop {
		wp.lock.Unlock()
		return false
	}
	wp.ready = append(wp.ready, ch)
	wp.lock.Unlock()
	return true
}
```