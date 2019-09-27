# rate limit实现
在kubernetes的controller中对workqueue实现了流量控制和延时重试机制。其中主要用到了ratelimit。  
包的路径是"golang.org/x/time/rate"，是通过令牌桶的算法实现。  
令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，
则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。  

## 代码解析
NewLimiter中有两个参数，第一个r就是qps，每秒的并发数, 第二个b为bucket size也是最多可获取到的token数量（当时间足够的时候同一时刻的最大并发数量）
```go
func NewLimiter(r Limit, b int) *Limiter {
	return &Limiter{
		limit: r,
		burst: b,
	}
}
```
```go
type Limiter struct {
	//每秒并发数
	limit Limit
	//桶的大小
	burst int

	mu     sync.Mutex
	//桶中已算出的令牌数
	tokens float64
	// 最近一次的更新时间
	last time.Time
	// 最近一次的事件执行时间
	lastEvent time.Time
}
```
Limiter主要开发3中函数：  
Allow，AllowN： 是否获取到令牌，也就是n个事件是否可以同时发生  
Reserve,ReserveN:  需要等多久才能等到n个事件发生  
Wait，WaitN： 阻塞当前直到lim允许n个事件的发生  

在实现上它们的套路是一样的，都是调用内部函数reserveN获取令牌，所以只要看下wait函数就可以了。  


### Wait
wait就是WaitN(ctx, 1)，也就是获取一个令牌。
```go
func (lim *Limiter) Wait(ctx context.Context) (err error) {
	return lim.WaitN(ctx, 1)
}
```

这里面关键是lim.reserveN获取Reserve，然后算出delay时间，new一个Timer进行等待。  
其中waitLimit就是最大等待时间，根据ctx.Deadline获取，如果未设置就是用InfDuration。  
```go
func (lim *Limiter) WaitN(ctx context.Context, n int) (err error) {
	if n > lim.burst && lim.limit != Inf {
		return fmt.Errorf("rate: Wait(n=%d) exceeds limiter's burst %d", n, lim.burst)
	}
	// Check if ctx is already cancelled
	select {
	case <-ctx.Done():
		return ctx.Err()
	default:
	}
	// Determine wait limit
	now := time.Now()
	waitLimit := InfDuration
	if deadline, ok := ctx.Deadline(); ok {
		waitLimit = deadline.Sub(now)
	}
	// Reserve
	r := lim.reserveN(now, n, waitLimit)
	if !r.ok {
		return fmt.Errorf("rate: Wait(n=%d) would exceed context deadline", n)
	}
	// Wait if necessary
	delay := r.DelayFrom(now)
	if delay == 0 {
		return nil
	}
	t := time.NewTimer(delay)
	defer t.Stop()
	select {
	case <-t.C:
		// We can proceed.
		return nil
	case <-ctx.Done():
		// Context was canceled before we could proceed.  Cancel the
		// reservation, which may permit other events to proceed sooner.
		r.Cancel()
		return ctx.Err()
	}
}
```

limit == Inf表示未限制。接着调用lim.advance(now)获取令牌，如果获取到的令牌数量小于请求数n表示需要等待。  
其中ok := n <= lim.burst && waitDuration <= maxFutureReserve表示，请求数量要小于桶的大小也就是最大并发数量同时等待时间不超过最大等待时间。    
最后构建Reservation返回，并更新state。  
其中tokens -= float64(n)表示未获取完的令牌，如果<0表示令牌不够需要等待，其中tokens需要存入lim.tokens供下次计算token使用。
```go
func (lim *Limiter) reserveN(now time.Time, n int, maxFutureReserve time.Duration) Reservation {
	lim.mu.Lock()

	if lim.limit == Inf {
		lim.mu.Unlock()
		return Reservation{
			ok:        true,
			lim:       lim,
			tokens:    n,
			timeToAct: now,
		}
	}

	now, last, tokens := lim.advance(now)

	// Calculate the remaining number of tokens resulting from the request.
	tokens -= float64(n)

	// Calculate the wait duration
	var waitDuration time.Duration
	if tokens < 0 {
		waitDuration = lim.limit.durationFromTokens(-tokens)
	}

	// Decide result
	ok := n <= lim.burst && waitDuration <= maxFutureReserve

	// Prepare reservation
	r := Reservation{
		ok:    ok,
		lim:   lim,
		limit: lim.limit,
	}
	if ok {
		r.tokens = n
		r.timeToAct = now.Add(waitDuration)
	}

	// Update state
	if ok {
		lim.last = now
		lim.tokens = tokens
		lim.lastEvent = r.timeToAct
	} else {
		lim.last = last
	}

	lim.mu.Unlock()
	return r
}
```

maxElapsed根据burst最大并发数-tokens已存在令牌数算出，表示现在这个时刻到桶满所需要的时间戳，
而elapsed表示最后一次获取token到这次所用的时间，然后这个时间如果>maxElapsed,就用maxElapsed，因为burst最大并发数的限制。    
然后根据elapsed算出这个时间段内增加的令牌数delta，再加上原先的tokens就是这次可获取的tokens。  
其中这个token用float64表示，这样可以精确算出获取到整个token所需要的时间。
```go
func (lim *Limiter) advance(now time.Time) (newNow time.Time, newLast time.Time, newTokens float64) {
	last := lim.last
	if now.Before(last) {
		last = now
	}

	// Avoid making delta overflow below when last is very old.
	maxElapsed := lim.limit.durationFromTokens(float64(lim.burst) - lim.tokens)
	elapsed := now.Sub(last)
	if elapsed > maxElapsed {
		elapsed = maxElapsed
	}

	// Calculate the new number of tokens, due to time that passed.
	delta := lim.limit.tokensFromDuration(elapsed)
	tokens := lim.tokens + delta
	if burst := float64(lim.burst); tokens > burst {
		tokens = burst
	}

	return now, last, tokens
}
```